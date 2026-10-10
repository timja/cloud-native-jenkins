# Target 3: Shared build records and history (R5)

| | |
|---|---|
| **Roadmap tier** | R5 (core of it) |
| **Status** | Planned |
| **Depends on** | Target 1 (item store, change events), Target 2 (cluster-wide build numbers, item ownership) |
| **Evidence** | [E2](../findings/E2-disposable-controller.md) (history lost with a controller; identity collisions), [E3](../findings/E3-shared-home.md) (each controller lists only its own builds), [E7](../findings/E7-ecosystem-scan.md) (212 plugins use build-dir `File` APIs), [E9](../findings/E9-runtime-job-state.md) (branch pruning deletes history), ET1/ET2 observations |

## Goal
Every controller lists, and can fully open, **every completed build** of every job, wherever it ran:
- the build page
- the console log
- the Pipeline graph and step logs
- test results and other plugin data stored in the build directory

History survives the controller that ran the build being destroyed. Deleting, renaming and editing builds (e.g. the description) on any controller is reflected everywhere.

**Out of scope:**
- **Running builds** stay visible only on the controller running them, and appear elsewhere once they complete. Live cross-controller view and adoption are R3/R4.
- **Per-item UUID identity** (see "Identity" below).
- **Moving artifacts** off `archive/`; use an external `ArtifactManager`.
- **Freestyle live log streaming** to the store.

## Why a snapshot, not routing
Targets 1 and 2 worked by routing `XmlFile`/`TextFile` keys transparently. That does not work for builds. The research (code refs in the appendix) found that most of a completed build is read and written through raw `File` APIs:

| File | I/O | Routable? |
|---|---|---|
| `build.xml` | `XmlFile` read; Pipeline's non-atomic durability writes raw streams | partly |
| `log`, `log-index` (Pipeline `FileLogStorage`, final) | `FileOutputStream`, `RandomAccessFile`, `AnnotatedLargeText(File)` | no |
| `workflow-completed/flowNodeStore.xml` | `XStream.fromXML(File)`, raw stream writes | no |
| `changelog*.xml` | `ChangeLogParser.parse(File)`; Pipeline stores an *absolute* path in build.xml | no |
| plugin data (`junitResult.xml`, coverage, warnings, …) | 212 plugins use `Run#getRootDir` etc. (E7) | no |

Also:
- Core indexes builds by `dir.list()` (`AbstractLazyLoadRunMap.loadNumberOnDisk`).
- `RunMap.retrieve` checks `new File(d, "build.xml").exists()`.

So: **when a build completes, its owner publishes a snapshot of the build directory to the store. Other controllers materialise the snapshot into their local `builds/N/` before anything reads it.** Every existing reader (core, Pipeline, plugins) then works unchanged, and history is shared. The store only ever holds *completed*, immutable-ish records, so running-build write amplification and core's "reload forces COMPLETED" behaviour (`Run.reload`) never come into play.

## Design (core, on top of the T2 stack, `@Restricted(Beta)` where public)

### Store layout
These are not "owned" keys: `XmlFile` does not route them. Core writes them explicitly through the SPI.
- `jobs/<item path>/builds/<N>/files.tgz` holds the build directory, excluding:
  - `archive/` (use an external `ArtifactManager`)
  - running-only state: `program.dat`, `workflow/`, `stashes/`
  - caches such as `pipeline-graph-view-*.json`
  - files over a size cap (default 64 MB, configurable; skipped files are listed in the record)
- `jobs/<item path>/builds/<N>/build.xml` is the record. It is written **after** `files.tgz`, so its existence is the commit marker, and it is small, so later edits (description, keep-forever) republish only this.

Both live under the item prefix, so T1's `PersistenceStore.move` carries history along on rename and move.

### Core pieces
| Piece | Where | What |
|---|---|---|
| `BuildRecords.publish(run)` | `RunListener.onFinalized` (owner) | Tar + gzip the build dir (exclusions above), write `files.tgz`, then `build.xml` |
| republish on change | `SaveableListener.onChange` for a *completed* `Run`, unless `PersistenceStores.isApplyingRemoteChange()` | Re-write `build.xml` only |
| `BuildRecords.delete(run)` | `RunListener.onDeleted` (origin only) | Delete the build's keys |
| index merge | `AbstractLazyLoadRunMap.loadNumberOnDisk` | Union of `dir.list()` and stored build numbers (`store.children(<buildDir key>)` with a `build.xml`), like T1's `storedItemNames` |
| lazy materialise | `RunMap.retrieve(File)` | No local `build.xml` but stored → extract `files.tgz` + `build.xml` into `builds/N/`, then load as today. A fresh controller only fetches builds that are actually opened |
| remote events | `ItemChangeApplier` | `builds/N/build.xml` WRITTEN → materialise, then `recognizeNumber(N)` (new build) or `Run.reload()` (edited record); refresh peephole permalinks. DELETED → remove from the index (`LazyBuildMixIn.removeRun`) and delete the local dir **without** firing `RunListener.onDeleted`, so external cleanup runs once, on the origin |
| numbering guard | `AbstractLazyLoadRunMap.proposeNewNumber` (JENKINS-27530) | Relax when numbers come from `StoredBuildNumberAssigner`: a later remote build may already be indexed when a locally assigned earlier number is put |
| permalinks | `PeepholePermalink` | Recompute for the job when a remote record arrives or is deleted (the default cache is per controller, and its listeners fire only on the owner) |

### Identity (the subset in this target)
Cluster-wide numbering (T2) already makes `(job full name, number)` unique while the job exists. Two holes:
1. **Delete + recreate a job with the same name.** Today numbering continues only because the `nextBuildNumber` key happens to survive item deletion (an accident: nothing tests it). This target makes it explicit and tested: item deletion keeps the job's `nextBuildNumber`, so a recreated job never reuses numbers. It also fixes `move` rejecting a destination whose only leftover key is that `nextBuildNumber`.
2. **Remote item deletion fires `ItemListener.onDeleted` on every controller.** So plugins that clean up external data, e.g. junit-sql-storage's `DELETE … WHERE job=?`, run N times and race with a recreated job. Fix: plugins skip cleanup while `PersistenceStores.isApplyingRemoteChange()`. This is a junit-sql-storage PR.

**Deferred to a later target:**
- a per-item UUID that survives renames and is used by plugins' external keys
- junit-plugin `load(Run)` API, junit-sql `job_id` column, azure-artifact-manager keying by id
- data keyed under an old name after a rename

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | core | `cnj/t3-m1-build-snapshots` | `cnj/t2-m2-trigger-coordination` | `BuildRecords` publish / republish / delete; index merge; lazy materialise in `RunMap.retrieve`; JENKINS-27530 relaxation; tests on `InMemoryPersistenceStore` (fresh home lists and opens builds made "elsewhere") |
| M2 | core | `cnj/t3-m2-remote-build-events` | M1 | `ItemChangeApplier` handles build records (materialise + recognize / reload / silent removal); permalink refresh; tests |
| M3 | core | `cnj/t3-m3-explicit-numbering` | M2 | Keep `nextBuildNumber` on item delete (explicit + tested); `move` ignores a leftover `nextBuildNumber` at the destination |
| M4 | junit-sql-storage | `cnj/t3-m4-remote-delete` | `origin/master` | Skip cleanup listeners while applying a remote change |
| M5 | cloud-native-jenkins | `cnj/t3-m5-et3` | `cnj/t2-m3-postgres-coordination` | Store plugin: nothing expected beyond large `bytea` values; `ET3-build-records.sh` |

## E-T3 acceptance experiment (definition of done)
ET setup (two tmpfs controllers, S0, Postgres, token auth). Pass criteria:
1. **Freestyle:** a build on A with a JUnit report (file storage, `junitResult.xml` in the build dir). B lists it with the same result, duration and console text, and the test report shows the same counts.
2. **Pipeline:** a multi-stage build with step output on A. B lists it, the `consoleText` is byte-identical, and the flow graph has the same nodes, with step logs readable on B.
3. **Interleaved history:** 10 builds on A and 10 on B of one job. Both list builds 1..20. `lastBuild`, `lastSuccessfulBuild` and `lastFailedBuild` agree on both.
4. **Edit and delete:** set a description on B for a build that ran on A → visible on A. Delete a build on A → gone on B, including its local dir.
5. **Disposable controller:** destroy A, which ran half the builds, and start a fresh A. Fresh A and B both list all builds and can open their console and test results.
6. **Running builds:** a 60 s build on A is not listed on B while running, and appears on B within seconds of completing.
7. **Rename:** rename the job on B. History (and console of old builds) is intact on both under the new name, and numbering continues.
8. **Delete + recreate:** delete the job and recreate it with the same name. Its first build is numbered after the old ones, and no old build appears in its history.

## Risks
- **Snapshot size.** Logs of very large builds go into Postgres `bytea` (max 1 GB per value). There is a size cap with a listed-but-skipped marker. An object-store backend for `files.tgz` (S3/Azure blob via the same SPI) is the follow-up.
- **Non-owner edits to a completed build** race with the owner's republish. Both write `build.xml` with last-writer-wins on rare manual edits; CAS conflicts are counted.
- **Plugins that rewrite build-dir files after `onFinalized`.** Those files are captured only if the plugin also triggers a `build.xml` save. A full republish trigger can be added if E-T3 finds such a plugin.
- **Absolute `changelogFile` paths in Pipeline build.xml (JENKINS-27704).** These resolve only if `JENKINS_HOME` is the same path on every controller, which is true in the harness. Upstream fix: relativise.
- **Discarders run on every controller.** Deletes are idempotent. Coordination with the T2 owner is a follow-up.

## Appendix: research
The three research reports (core build records, Pipeline storage, plugin identity) are summarised above. Key code refs:
- `AbstractLazyLoadRunMap.java:175-231`
- `RunMap.java:181-260`
- `Run.java:360, 1569-1630, 2007-2013`
- `LazyBuildMixIn.java:110-195`
- `PeepholePermalink.java:97-343`
- workflow-api `FileLogStorage` (raw I/O), `LogStorage.of` (`:161-175`)
- workflow-support `BulkFlowNodeStorage` (`:79-179`)
- workflow-cps `CpsFlowExecution.createStorage`/`optimizeStorage` (`:549-605`)
- junit-sql-storage `TestResultCleanupListener`

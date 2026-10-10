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

**Running builds** appear on other controllers as a **placeholder** ("#N running on controller A"), replaced by the full record when they complete.

**Large data stays out of Postgres** (decision 2026-10-10):
- Pipeline logs go through **pluggable log storage** (`LogStorageFactory`): CloudWatch, Log Analytics, Elasticsearch or object storage.
- Build snapshots go to **object storage**.
- Postgres holds only small records (`build.xml`, placeholders).

**Out of scope:**
- Live cross-controller interaction with running builds (console follow, stop, input) and adoption: R3/R4.
- **Per-item UUID identity** (see "Identity" below).
- **Moving artifacts** off `archive/`; use an external `ArtifactManager`.
- **Freestyle live logs.** Core has no log storage SPI; freestyle logs travel in the completed-build snapshot.

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

### Where things live
| Data | Store | Key |
|---|---|---|
| build record (small) | `PersistenceStore` (Postgres) | `jobs/<item path>/builds/<N>/build.xml`. Written **last**, as the commit marker; later edits (description, keep-forever) republish only this |
| running placeholder (small) | `PersistenceStore` | `jobs/<item path>/builds/<N>/running.xml`: origin, controller name, start time, display name. Written on start, deleted on publish |
| build directory snapshot (large) | **new `BlobStore` SPI** (object storage; Azurite in the harness) | `jobs/<item path>/builds/<N>/files.tgz` |
| Pipeline log + index (large, streamed) | **`LogStorageFactory`** (workflow-api's pluggable log storage) | backend-specific; the harness implementation uses one append blob for `log` and one for `log-index` per build |

The snapshot excludes:
- `archive/` (use an external `ArtifactManager`)
- running-only state: `program.dat`, `workflow/`, `stashes/`
- caches such as `pipeline-graph-view-*.json`
- for Pipeline builds, `log` and `log-index` (they are in log storage)

All keys live under the item prefix. T1's `PersistenceStore.move` therefore carries records along on rename and move; `BlobStore` gets the same `move(prefix)` contract.

### New SPI: `jenkins.persistence.BlobStore` (Beta)
- `put(key, InputStream, length)`, `get(key)` returning an `InputStream` (or null), `delete(prefix)`, `move(fromPrefix, toPrefix)`
- resolved through `BlobStoreProvider`, the same way as `PersistenceStoreProvider`

`files.tgz` goes here. With no `BlobStore`, build snapshots are not shared: records are, but materialisation has no files and history shows build metadata only.

### Pipeline logs: a `LogStorageFactory`
The research found `FileLogStorage` is `final` and does raw file I/O, but `LogStorage.of` lets any `LogStorageFactory` take over every build.
- **Harness implementation** (in the store plugin): a factory whose storage **delegates to `FileLogStorage` locally while running, and tees each flushed chunk of `log` and `log-index` to append blobs**.
- **Reads:** a controller without the local files downloads both blobs into a local cache file and serves them through `FileLogStorage.forFile(cache)`. This reuses its parsing of step spans and `log-index` unchanged.
- **Running builds elsewhere:** the same download gives a read-only console for a running placeholder, refreshed on each poll by range-reading from the last offset.

CloudWatch, Log Analytics or Elasticsearch plug in as alternative `LogStorageFactory` implementations: anything keyed by (job, build) that every controller can read.

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
| running placeholders | `RunListener.onStarted` / publish | Write and delete `running.xml`. `ItemChangeApplier` keeps a per-job map of placeholders from events, loaded from the store at startup |
| placeholder display | `RunningElsewhereAction` (job-level `TransientActionFactory`, `@Exported`) | Lists "#N running on controller A since …" on the job page and in the REST API; links to the read-only console (Pipeline, via log storage) |
| stale placeholders | SPI: optional `PersistenceStore.liveOrigins()` | If an origin is no longer live (T2 membership), its placeholders are removed by whichever controller notices (claimed like a trigger tick), so a crashed owner does not leave "running forever" builds |

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
| M1 | core | `cnj/t3-m1-build-snapshots` | `cnj/t2-m2-trigger-coordination` | `BlobStore` SPI + in-memory implementation; `BuildRecords` publish / republish / delete; index merge; lazy materialise in `RunMap.retrieve`; JENKINS-27530 relaxation; tests (fresh home lists and opens builds made "elsewhere") |
| M2 | core | `cnj/t3-m2-remote-build-events` | M1 | `ItemChangeApplier` handles build records (materialise + recognize / reload / silent removal); permalink refresh; tests |
| M3 | core | `cnj/t3-m3-running-placeholders` | M2 | `running.xml` records, `RunningElsewhereAction`, `liveOrigins()` stale cleanup; tests |
| M4 | core | `cnj/t3-m4-explicit-numbering` | M3 | Keep `nextBuildNumber` on item delete (explicit + tested); `move` ignores a leftover `nextBuildNumber` at the destination |
| M5 | junit-sql-storage | `cnj/t3-m5-remote-delete` | `origin/master` | Skip cleanup listeners while applying a remote change |
| M6 | cloud-native-jenkins | `cnj/t3-m6-blob-and-log-storage` | `cnj/t2-m3-postgres-coordination` | Store plugin: Azure Blob `BlobStore` (Azurite in the harness); append-blob `LogStorageFactory` (tee while running, download-to-cache for reads); `liveOrigins()` from T2 membership; Azurite added to `compose.et1.yaml`; `ET3-build-records.sh` |

## E-T3 acceptance experiment (definition of done)
ET setup (two tmpfs controllers, S0, Postgres, token auth). Pass criteria:
1. **Freestyle:** a build on A with a JUnit report (file storage, `junitResult.xml` in the build dir). B lists it with the same result, duration and console text, and the test report shows the same counts.
2. **Pipeline:** a multi-stage build with step output on A. B lists it, the `consoleText` is byte-identical, and the flow graph has the same nodes, with step logs readable on B.
3. **Interleaved history:** 10 builds on A and 10 on B of one job. Both list builds 1..20. `lastBuild`, `lastSuccessfulBuild` and `lastFailedBuild` agree on both.
4. **Edit and delete:** set a description on B for a build that ran on A → visible on A. Delete a build on A → gone on B, including its local dir.
5. **Disposable controller:** destroy A, which ran half the builds, and start a fresh A. Fresh A and B both list all builds and can open their console and test results.
6. **Running builds:**
   - A 60 s Pipeline build on A appears on B as a placeholder ("running on A") within seconds of starting, and B's read-only console shows its output so far.
   - On completion, the placeholder is replaced by the full build on B.
   - SIGKILL A during a second running build: B removes the stale placeholder within about a minute.
7. **Rename:** rename the job on B. History (and console of old builds) is intact on both under the new name, and numbering continues.
8. **Delete + recreate:** delete the job and recreate it with the same name. Its first build is numbered after the old ones, and no old build appears in its history.
9. **Large data is not in Postgres:** no `files.tgz` and no log content in `jenkins_store`, and the largest `jenkins_store` value is under 1 MB. Snapshots and logs are in the blob container.

## Risks
- **Log storage semantics.** A `LogStorage` must preserve the `log-index` step spans that the Pipeline UI depends on. Delegating to `FileLogStorage` and shipping both files keeps that exact. A native CloudWatch or Elasticsearch backend has to reproduce it, which is what pipeline-cloudwatch-logs does.
- **Append-blob limits.** Azure caps an append blob at 50 000 blocks. The tee batches flushes (e.g. every 1 s or 1 MB), and very long-running builds may need rollover to a second blob.
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

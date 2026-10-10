# ET3: Target 3 acceptance (shared build records and history)

**Result: PASS**, 48/48 checks, 0 failures. Raw evidence: `raw/ET3/`
- `results.md`
- `jenkins_store.txt`: every key in Postgres with its size
- `blobs.txt`: every blob in the container
- `pl-graph-B.txt`: flow graph and step logs of a Pipeline build, as read on B
- controller logs, `BUILD`

The ET1 checks were re-run on the same build, now with Azurite in the compose file: 27/27 pass (`raw/ET1/`).

- **Script:** `validation/experiments/ET3-build-records.sh`
- **Setup:** two tmpfs controllers, S0, Postgres, Azurite (Azure Blob Storage), API-token auth
- **Build:** core `cnj/t3-m4-explicit-numbering`, cloudbees-folder `cnj/t1-m2-folder-children`, plugin on `cnj/t3-m6-blob-and-log-storage`

| Scenario | Result |
|---|---|
| 1 freestyle build with a JUnit report (file storage) on A | ✅ B lists it with the same result, duration and console text; the test report has the same counts (1 pass, 1 fail) |
| 2 multi-stage Pipeline on A | ✅ `consoleText` byte-identical on B; flow graph (19 nodes) and the 7 step logs identical. The log is in append blobs (`log`, `log-index`) next to the snapshot (`files.tgz`) |
| 3 10 builds on A + 10 on B of one job, one failing | ✅ both list 1..20; `lastBuild`, `lastSuccessfulBuild` and `lastFailedBuild` agree and point at the right builds |
| 4 description set on B for a build that ran on A; a build deleted on A | ✅ description visible on A; the deleted build is gone on B, including its local directory |
| 5 A destroyed and started fresh | ✅ fresh A starts with no build directories, lists every build, and opens consoles (its own, B's, freestyle), test results and the flow graph |
| 6 60 s Pipeline on A | ✅ placeholder "running on jenkins-a" on B after 2-3 s; B's read-only console shows output so far; replaced by the full build on completion |
| 6b SIGKILL of A during a running build | ✅ B removed the stale placeholder 21 s after the kill (20 s membership timeout + cleanup period) |
| 7 job renamed on B | ✅ history and old consoles intact on both under the new name; nothing left under the old name in blob storage; numbering continues (#21) |
| 8 job deleted on A, recreated on B | ✅ first build of the recreated job is #2; no old build in its history on either controller |
| 9 what Postgres holds | ✅ no snapshots or logs in `jenkins_store`; largest value 2768 bytes; 75 blobs in the container |

## Observations
- **Lazy materialisation works as intended.** A remote completion costs the other controllers one Postgres event and an empty directory. The snapshot and log are downloaded the first time a build is opened. Permalinks are updated by a self-validating hint, so they don't force a download either.
- **Bug found by ET3 (first run, scenario 7: 2 failures):**
  - **Symptom:** a controller that had not yet opened any build of a job lost most of its history when another controller renamed the job.
  - **Cause:** during the rename, `Fingerprint.ProjectRenameListener` walks every build in `onLocationChanged` before `LazyBuildMixIn`'s listener updates the run map's base directory. So builds were loaded from the old location, failed, and were marked unloadable.
  - **Stock Jenkins** has a latent form of this: unloaded builds of a renamed job are dropped until the next reload.
  - **Fixed in core M1:** `LazyBuildMixIn.ItemListenerImpl` runs first, and `RunMap.retrieve` materialises only into the job's current build directory. The regression test `BuildRecordsTest#renameBeforeBuildsAreMaterialized` fails without the fix (`expected [2, 1] but was []`).
  - **Upstreaming:** worth proposing upstream as a stand-alone fix.
- **Rename ordering.**
  - Blobs now move before the store entries, because moving the entries notifies the other controllers. The destination is checked for conflicts before any blob moves.
  - Moves copy blobs one by one, so renaming a job with a long history is slow (O(builds) copies). Azure's server-side copy would avoid the download and upload.
- **Placeholders** appear as fast as the store's LISTEN/NOTIFY, about 2-3 s here including the 2 s poll in the script.
- **Live console from another controller** lags by the log shipper period (1 s) plus `FileLogStorage`'s own buffering, which flushes with growing delay up to about 10 s.

## Follow-ups
- Server-side blob copy for renames; append-blob rollover for very long builds (50 000 blocks ≈ 14 h at 1 s batches).
- A resumed Pipeline build (after a controller restart) keeps its old origin in the placeholder, so another controller may remove it while it still runs. Rewrite placeholders when builds resume.
- Freestyle has no live cross-controller console: core has no log storage SPI, so freestyle logs only travel in the completed snapshot.
- Only `RunListener`/`SaveableListener` changes after completion are republished. A plugin that rewrites build-directory files without saving `build.xml` would not be picked up.

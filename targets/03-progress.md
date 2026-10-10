# Target 3: progress and branches

Design: [03-shared-build-records.md](03-shared-build-records.md).

**Done means:** `validation/experiments/ET3-build-records.sh` passes (the 9 E-T3 criteria).

**Policy:** same as Targets 1 and 2:
- local only, nothing pushed
- stacked branches
- integration branches only if needed

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t3-m1-build-snapshots` | `cnj/t2-m2-trigger-coordination` | `BlobStore` SPI + in-memory impl; `BuildRecords` publish / republish / delete; index merge; lazy materialise; JENKINS-27530 relax | ✅ 594d337001; `BuildRecordsTest` 6/6 + RunMap, lazy-load, Run, Job, T1/T2 suites green |
| M2 | core | `cnj/t3-m2-remote-build-events` | T3 M1 | applier for build records; permalink refresh | 🚧 next |
| M3 | core | `cnj/t3-m3-running-placeholders` | T3 M2 | `running.xml`, `RunningElsewhereAction`, `liveOrigins()` cleanup | ⏳ |
| M4 | core | `cnj/t3-m4-explicit-numbering` | T3 M3 | keep `nextBuildNumber` on delete; `move` ignores leftover `nextBuildNumber` | ⏳ |
| M5 | junit-sql-storage | `cnj/t3-m5-remote-delete` | `origin/master` | skip cleanup while applying a remote change | ⏳ |
| M6 | cloud-native-jenkins | `cnj/t3-m6-blob-and-log-storage` | `cnj/t2-m3-postgres-coordination` | Azure Blob `BlobStore`, append-blob `LogStorageFactory`, Azurite in harness, `ET3-build-records.sh` | ⏳ |

## Log
- 2026-10-10: Target 3 started: M1 (fetch a remote build's files only when it is first opened).
- 2026-10-10: **T3 M1 committed** (594d337001). Decisions made while implementing:
  - A store opts in with `PersistenceStore.storesBuildRecords(jobKey)` (default false). Record keys are written directly and never routed, so running builds keep saving `build.xml` locally.
  - `children(jobs/<item>/builds)` must mean "committed builds", so M3 placeholders go under a separate prefix, `jobs/<item>/running/<N>.xml`, not `builds/<N>/running.xml`.
  - Saves made while a build is loading are never republished (avoids churn from data migrations).
  - Materialisation runs under the run map's lock, like a disk load; a slow object store delays other reads of that job's builds. Acceptable for now.
  - Known gap until M4: deleting a job leaves its build records in the store.

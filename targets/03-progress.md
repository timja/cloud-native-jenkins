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
| M1 | core | `cnj/t3-m1-build-snapshots` | `cnj/t2-m2-trigger-coordination` | `BlobStore` SPI + in-memory impl; `BuildRecords` publish / republish / delete; index merge; lazy materialise; JENKINS-27530 relax | 🚧 in progress |
| M2 | core | `cnj/t3-m2-remote-build-events` | T3 M1 | applier for build records; permalink refresh | ⏳ |
| M3 | core | `cnj/t3-m3-running-placeholders` | T3 M2 | `running.xml`, `RunningElsewhereAction`, `liveOrigins()` cleanup | ⏳ |
| M4 | core | `cnj/t3-m4-explicit-numbering` | T3 M3 | keep `nextBuildNumber` on delete; `move` ignores leftover `nextBuildNumber` | ⏳ |
| M5 | junit-sql-storage | `cnj/t3-m5-remote-delete` | `origin/master` | skip cleanup while applying a remote change | ⏳ |
| M6 | cloud-native-jenkins | `cnj/t3-m6-blob-and-log-storage` | `cnj/t2-m3-postgres-coordination` | Azure Blob `BlobStore`, append-blob `LogStorageFactory`, Azurite in harness, `ET3-build-records.sh` | ⏳ |

## Log
- 2026-10-10: Target 3 started: M1 (fetch a remote build's files only when it is first opened).

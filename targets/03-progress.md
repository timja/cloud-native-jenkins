# Target 3: progress and branches

Design: [03-shared-build-records.md](03-shared-build-records.md).

**Done means:** `validation/experiments/ET3-build-records.sh` passes (the 9 E-T3 criteria).

> **Status: DONE (2026-10-10).** ET3 passes 48/48 checks. Evidence: `findings/ET3-build-records.md` on `cnj/t3-m6-blob-and-log-storage`.

**Policy:** same as Targets 1 and 2:
- local only, nothing pushed
- stacked branches
- integration branches only if needed

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t3-m1-build-snapshots` | `cnj/t2-m2-trigger-coordination` | `BlobStore` SPI + in-memory impl; `BuildRecords` publish / republish / delete; index merge; lazy materialise; JENKINS-27530 relax | ✅ 2623792def (rebased); `BuildRecordsTest` 7/7 + RunMap, lazy-load, Run, Job, T1/T2 suites green |
| M2 | core | `cnj/t3-m2-remote-build-events` | T3 M1 | applier for build records; permalink refresh | ✅ bcfaeb03e6; `RemoteBuildEventsTest` 4/4 + PeepholePermalinkTest, RunMap, Job, Run green |
| M3 | core | `cnj/t3-m3-running-placeholders` | T3 M2 | `running/<N>.xml`, `RunningElsewhereAction`, `liveOrigins()` cleanup | ✅ bdaa76bfcd; `RunningBuildsTest` 3/3 + FreeStyleProject, AbstractProject green |
| M4 | core | `cnj/t3-m4-explicit-numbering` | T3 M3 | delete stored history with the job, keep `nextBuildNumber`; `move` replaces leftover `nextBuildNumber` (keeps the higher) | ✅ 65d22559ad; `DeletedJobNumberingTest` 4/4 + Items, AbstractItem, all `jenkins.persistence` suites green |
| M5 | junit-sql-storage | `cnj/t3-m5-remote-delete` (worktree `../junit-sql-storage-plugin-cnj`) | `origin/master` | skip cleanup while applying a remote change | ✅ 65b2435 (local-core pom bump, revert before upstreaming) + d77f566; new test + `testResultCleanup` green |
| M6 | cloud-native-jenkins | `cnj/t3-m6-blob-and-log-storage` | `cnj/t2-m3-postgres-coordination` | Azure Blob `BlobStore`, append-blob `LogStorageFactory`, Azurite in harness, `ET3-build-records.sh` (worktree `../cloud-native-jenkins-t3`) | ✅ 747ffce; Azurite + Postgres plugin tests green; **ET3 PASS** (48/48), ET1 re-run 27/27 |

## Log
- 2026-10-10: Target 3 started: M1 (fetch a remote build's files only when it is first opened).
- 2026-10-10: **T3 M1 committed** (594d337001). Decisions made while implementing:
  - A store opts in with `PersistenceStore.storesBuildRecords(jobKey)` (default false). Record keys are written directly and never routed, so running builds keep saving `build.xml` locally.
  - `children(jobs/<item>/builds)` must mean "committed builds", so M3 placeholders go under a separate prefix, `jobs/<item>/running/<N>.xml`, not `builds/<N>/running.xml`.
  - Saves made while a build is loading are never republished (avoids churn from data migrations).
  - Materialisation runs under the run map's lock, like a disk load; a slow object store delays other reads of that job's builds. Acceptable for now.
  - Known gap until M4: deleting a job leaves its build records in the store.
- 2026-10-10: **T3 M2-M5 committed.** Notes:
  - M2 keeps loading lazy: a remote build only gets an empty local dir plus `recognizeNumber`. Permalinks get a hint (`Cache.Some(N)`), which is checked when the permalink is next resolved, so no build is downloaded just to update `lastSuccessfulBuild`.
  - M2: a remote edit replaces the local `build.xml` and drops the loaded instance (`unload`), so the next access reloads it with `onLoad` (`Run.reload()` skips `onLoad`).
  - M3: stale placeholders are removed without a claim; deletion is idempotent. Follow-up: a Pipeline build resumed after a restart keeps the old origin in its placeholder, so others may remove it while it still runs.
  - M4: rename onto a deleted job's name replaces its leftover `nextBuildNumber` and keeps the higher number.
- 2026-10-10: First ET3 run: 46/48. Scenario 7 failed on A: a rename applied before any build was materialised lost history. The cause was listener ordering (`Fingerprint.ProjectRenameListener` loads builds before `LazyBuildMixIn` updates the base dir). Fixed in M1 with a regression test; the stack was rebased.
- 2026-10-10: Also fixed in M1: blobs move before store entries on rename, after a destination conflict check.
- 2026-10-10: **ET3 PASS** (48/48).

## Review and merge order
1. core `cnj/t3-m1-build-snapshots` → `m2` → `m3` → `m4`, stacked on the T2 core branches. The `LazyBuildMixIn` listener-ordering fix in M1 could go upstream on its own first.
2. junit-sql-storage `cnj/t3-m5-remote-delete` (drop the pom bump commit once core is released).
3. cloud-native-jenkins `cnj/t3-m6-blob-and-log-storage`, on `cnj/t2-m3-postgres-coordination`.

## Follow-ups
See `findings/ET3-build-records.md`:
- server-side blob copy on rename
- append-blob rollover
- rewrite placeholders when a Pipeline build resumes
- a freestyle live console (needs a core log SPI)
- republishing for plugins that write files without saving `build.xml`

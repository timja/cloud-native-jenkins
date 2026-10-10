# Target 5: progress and branches

Design: [05-shared-queue.md](05-shared-queue.md).

**Done means:** `validation/experiments/ET5-shared-queue.sh` passes (the E-T5 criteria).

> **Status: DONE (2026-10-10).** ET5 passes 35/35, in five runs on the final build; ET4 (39/39) and ET3 (48/48) still pass on it. Evidence: `findings/ET5-shared-queue.md` on `cnj/t5-m5-shared-queue`.

**Policy:** same as Targets 1-4:
- local only, nothing pushed
- stacked branches
- integration branches only if needed

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t5-m1-cluster-queue` | `cnj/t4-m2-build-leases` | `ClusterQueue` SPI; publish/fold/mirror; `QueueStartGate`; cluster ids; REST fallback; cancel anywhere | ✅ caf1b3d25b (rebased); `SharedQueueTest` 8/8 + QueueTest, queue/*, crash suites green |
| M2 | core | `cnj/t5-m2-cluster-blocking` | T5 M1 | stop requests; cluster-wide non-concurrent (exclusive job lease) and upstream/downstream blocking | ✅ 9c99962d41 (plus the dependency graph fix); `ClusterCoordinationTest` 6/6 + all `jenkins.persistence`, AbstractProject, BuildTrigger, ReverseBuildTrigger green |
| M3 | core | `cnj/t5-m3-cluster-periodic-work` | T5 M2 | `@ClusterSingleton` + `PeriodicWorkCoordinator` | ✅ c97f5e6b35; new test + PeriodicWork, AsyncPeriodicWork, FingerprintCleanupThread, Telemetry green |
| M4 | lockable-resources | `cnj/t5-m4-cluster-locks` | `origin/master` | store-backed locks (worktree: `../lockable-resources-plugin`, fresh clone) | ✅ aeb5405 (local-core pom bump, revert before upstreaming) + c7c8d9b (with the Pipeline waiter wake-up fix); `ClusterLocksTest` 2/2 + LockStep (33), FreeStyle, restart, manager suites green |
| M5 | cloud-native-jenkins | `cnj/t5-m5-shared-queue` | `cnj/t4-m3-pipeline-failover` | Postgres queue, reclaim, milestone storage, periodic-work claims, harness, ET5 (worktree `../cloud-native-jenkins-t5`) | ✅ 455f84e; plugin tests green (23); **ET5 PASS** (35/35) |

## Log
- 2026-10-10: Target 5 started.
- 2026-10-10: T5 M1-M3 (core) committed. Notes:
  - Mirroring bug found by the M2 tests: a controller's own WAITING event can arrive after its item has left the queue, and the mirror recreated it, so it built twice. Fixed in M1: never mirror your own items, and re-read the record before mirroring.
  - Non-concurrent jobs take an exclusive lease on the job in the start gate, so races between controllers cannot start two builds. The lease is taken over on adoption.
- 2026-10-10: T5 M4 (lockable-resources) committed. The plugin funnels every lock through `LockableResourcesManager.lock` and `freeResources`. Cluster mode adds a create-only store write per resource, and shows resources held elsewhere through the existing `remoteLockedBy`. Also fixed: `LockStepExecution` completed the step without running its body when a lock failed after resources looked available. With several controllers that is a normal race, so the step now waits.
- 2026-10-10: Agent placement needs no extra logic for jobs with an assigned label. A controller without the agent connected has no executor that can take the item, so only the controller with the agent claims it. Pipeline `node` labels are not known at queue time; that stays a follow-up.
- 2026-10-10: ET5 run history and fixes are in `findings/ET5-shared-queue.md`. The bugs it and the regression runs found:
  - the dependency graph was not rebuilt for jobs changed elsewhere
  - a cause folded in elsewhere was lost on the item's own controller
  - Pipeline lock waiters woke 13 s late
  - controllers starting at once raced on schema creation
  - a woken zombie appended to a taken-over log
  - Pipelines placed away from their agent; the requesting controller now goes first, with a 10 s grace
- 2026-10-10: **ET5 PASS** (35/35 x5); ET4 39/39 and ET3 48/48 on the same build.

## Review and merge order
1. core `cnj/t5-m1-cluster-queue` → `m2` → `m3`, on the T4 core stack.
2. lockable-resources `cnj/t5-m4-cluster-locks` (drop the pom bump commit once core is released).
3. cloud-native-jenkins `cnj/t5-m5-shared-queue`, on `cnj/t4-m3-pipeline-failover`.

## Follow-ups
- Pipeline placement aware of `node` labels, or moving a running build to its agent's controller.
- Remote completion of `build` steps (today pinned).
- Event-driven reclaim.
- lockable-resources UI paths (reserve, steal) across controllers.

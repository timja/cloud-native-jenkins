# Target 5: progress and branches

Design: [05-shared-queue.md](05-shared-queue.md).

**Done means:** `validation/experiments/ET5-shared-queue.sh` passes (the E-T5 criteria).

**Policy:** same as Targets 1-4:
- local only, nothing pushed
- stacked branches
- integration branches only if needed

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t5-m1-cluster-queue` | `cnj/t4-m2-build-leases` | `ClusterQueue` SPI; publish/fold/mirror; `QueueStartGate`; cluster ids; REST fallback; cancel anywhere | ✅ bc541b9cfd; `SharedQueueTest` 6/6 + QueueTest, queue/*, crash suites green |
| M2 | core | `cnj/t5-m2-cluster-blocking` | T5 M1 | stop requests; cluster-wide non-concurrent (exclusive job lease) and upstream/downstream blocking | ✅ c8c07bfac6; `ClusterCoordinationTest` 5/5 + all `jenkins.persistence`, AbstractProject, BuildTrigger, ReverseBuildTrigger green |
| M3 | core | `cnj/t5-m3-cluster-periodic-work` | T5 M2 | `@ClusterSingleton` + `PeriodicWorkCoordinator` | ✅ 7946fa35ff; new test + PeriodicWork, AsyncPeriodicWork, FingerprintCleanupThread, Telemetry green |
| M4 | lockable-resources | `cnj/t5-m4-cluster-locks` | `origin/master` | store-backed locks (worktree: `../lockable-resources-plugin`, fresh clone) | ✅ aeb5405 (local-core pom bump, revert before upstreaming) + 769096a; `ClusterLocksTest` 2/2 + LockStep (33), FreeStyle, restart, manager suites green |
| M5 | cloud-native-jenkins | `cnj/t5-m5-shared-queue` | `cnj/t4-m3-pipeline-failover` | Postgres queue, reclaim, milestone storage, periodic-work claims, harness, ET5 (worktree `../cloud-native-jenkins-t5`) | 🚧 in progress |

## Log
- 2026-10-10: Target 5 started.
- 2026-10-10: T5 M1-M3 (core) committed. Notes:
  - Mirroring bug found by the M2 tests: a controller's own WAITING event can arrive after its item has left the queue, and the mirror recreated it, so it built twice. Fixed in M1: never mirror your own items, and re-read the record before mirroring.
  - Non-concurrent jobs take an exclusive lease on the job in the start gate, so races between controllers cannot start two builds. The lease is taken over on adoption.
- 2026-10-10: T5 M4 (lockable-resources) committed. The plugin funnels every lock through `LockableResourcesManager.lock` and `freeResources`. Cluster mode adds a create-only store write per resource, and shows resources held elsewhere through the existing `remoteLockedBy`. Also fixed: `LockStepExecution` completed the step without running its body when a lock failed after resources looked available. With several controllers that is a normal race, so the step now waits.
- 2026-10-10: Agent placement needs no extra logic for jobs with an assigned label. A controller without the agent connected has no executor that can take the item, so only the controller with the agent claims it. Pipeline `node` labels are not known at queue time; that stays a follow-up.

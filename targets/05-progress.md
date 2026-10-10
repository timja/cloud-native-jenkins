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
| M4 | lockable-resources | `cnj/t5-m4-cluster-locks` | `origin/master` | store-backed locks | ⏳ |
| M5 | cloud-native-jenkins | `cnj/t5-m5-shared-queue` | `cnj/t4-m3-pipeline-failover` | Postgres queue, placement, milestone storage, build-step pinning, ET5 | ⏳ |

## Log
- 2026-10-10: Target 5 started.
- 2026-10-10: T5 M1-M3 (core) committed. Notes:
  - Mirroring bug found by the M2 tests: a controller's own WAITING event can arrive after its item has left the queue, and the mirror recreated it, so it built twice. Fixed in M1: never mirror your own items, and re-read the record before mirroring.
  - Non-concurrent jobs take an exclusive lease on the job in the start gate, so races between controllers cannot start two builds. The lease is taken over on adoption.

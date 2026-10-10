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
| M1 | core | `cnj/t5-m1-cluster-queue` | `cnj/t4-m2-build-leases` | `ClusterQueue` SPI; publish/fold/mirror; `QueueStartGate`; cluster ids; REST fallback; cancel anywhere | 🚧 in progress |
| M2 | core | `cnj/t5-m2-cluster-blocking` | T5 M1 | stop requests; cluster-wide non-concurrent and upstream/downstream blocking | ⏳ |
| M3 | core | `cnj/t5-m3-cluster-periodic-work` | T5 M2 | `PeriodicWorkCoordinator`, cluster-scoped periodic work | ⏳ |
| M4 | lockable-resources | `cnj/t5-m4-cluster-locks` | `origin/master` | store-backed locks | ⏳ |
| M5 | cloud-native-jenkins | `cnj/t5-m5-shared-queue` | `cnj/t4-m3-pipeline-failover` | Postgres queue, placement, milestone storage, build-step pinning, ET5 | ⏳ |

## Log
- 2026-10-10: Target 5 started.

# Target 2: progress and branches

Design: [02-cluster-scheduling.md](02-cluster-scheduling.md).

**Done means:** `validation/experiments/ET2-scheduling.sh` passes.

**Policy:** same as Target 1:
- local only, nothing pushed
- stacked branches
- integration branches only if needed

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t2-m1-stored-build-numbers` | `cnj/t1-m4-item-change-applier` | `TextFile` routing; `StoredBuildNumberAssigner` (strict CAS increment) | in progress |
| M2 | core | `cnj/t2-m2-trigger-coordination` | T2 M1 | `TriggerCoordinator` extension point consulted in `Trigger.checkTriggers` | not started |
| M3 | cloud-native-jenkins | `cnj/t2-m3-postgres-coordination` | `cnj/t1-m6-postgres-store` | plugin owns `nextBuildNumber`; membership heartbeat; rendezvous affinity + tick claims; `ET2-scheduling.sh` | not started |

## Log
- 2026-10-10: Target 2 scoped (R2 subset: build numbers + exactly-once cron-style triggers). One hook covers all cron-style triggers: `Trigger.checkTriggers`.

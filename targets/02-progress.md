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
| M1 | core | `cnj/t2-m1-stored-build-numbers` | `cnj/t1-m4-item-change-applier` | `TextFile` routing; `StoredBuildNumberAssigner` (strict CAS increment, retry) | ✅ 4 new tests (incl. 220 concurrent assignments across two origins) + JobTest, TextFileTest, T1 suites green |
| M2 | core | `cnj/t2-m2-trigger-coordination` | T2 M1 | `TriggerCoordinator` extension point consulted in `Trigger.checkTriggers` (both paths) | ✅ new test + SCMTriggerTest green |
| M3 | cloud-native-jenkins | `cnj/t2-m3-postgres-coordination` (worktree `../cloud-native-jenkins-t2`) | `cnj/t1-m6-postgres-store` | plugin owns `nextBuildNumber` + `scm-revision-hash.xml`/`scm-last-seen-revision-hash.xml`; membership heartbeat; rendezvous affinity + tick claims; `ET2-scheduling.sh` | ✅ 12 store tests green; ET2 running |

## Log
- 2026-10-10: Target 2 scoped (R2 subset: build numbers + exactly-once cron-style triggers). One hook covers all cron-style triggers: `Trigger.checkTriggers`.
- 2026-10-10: Found that branch-api reads `scm-revision-hash.xml` through a fresh `XmlFile` per call, so owning those keys in the store shares indexing state with no branch-api change. This addresses ET1's "indexing on B rebuilt branches".
- 2026-10-10: T2 M1, M2 (core) and M3 (plugin) committed; ET2 running.

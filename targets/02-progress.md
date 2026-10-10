# Target 2: progress and branches

Design: [02-cluster-scheduling.md](02-cluster-scheduling.md).

**Done means:** `validation/experiments/ET2-scheduling.sh` passes.

> **Status: DONE (2026-10-10).** ET2 passes with 0 failures on the API-token harness, and ET1 still passes on the same build (27/27). Evidence: `findings/ET2-scheduling.md` on `cnj/t2-m3-postgres-coordination`.

**Policy:** same as Target 1:
- local only, nothing pushed
- stacked branches
- integration branches only if needed

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t2-m1-stored-build-numbers` | `cnj/t1-m4-item-change-applier` | `TextFile` routing; `StoredBuildNumberAssigner` (strict CAS increment, retry) | ✅ 4 new tests (incl. 220 concurrent assignments across two origins) + JobTest, TextFileTest, T1 suites green |
| M2 | core | `cnj/t2-m2-trigger-coordination` | T2 M1 | `TriggerCoordinator` extension point consulted in `Trigger.checkTriggers` (both paths) | ✅ new test + SCMTriggerTest green |
| M3 | cloud-native-jenkins | `cnj/t2-m3-postgres-coordination` (worktree `../cloud-native-jenkins-t2`) | `cnj/t1-m6-postgres-store` | plugin owns `nextBuildNumber` + `scm-revision-hash.xml`/`scm-last-seen-revision-hash.xml`; membership heartbeat; rendezvous affinity + tick claims; `ET2-scheduling.sh` | ✅ 12 store tests green; **ET2 PASS** |

## Log
- 2026-10-10: Target 2 scoped (R2 subset: build numbers + exactly-once cron-style triggers). One hook covers all cron-style triggers: `Trigger.checkTriggers`.
- 2026-10-10: Found that branch-api reads `scm-revision-hash.xml` through a fresh `XmlFile` per call, so owning those keys in the store shares indexing state with no branch-api change. This addresses ET1's "indexing on B rebuilt branches".
- 2026-10-10: T2 M1, M2 (core) and M3 (plugin) committed; ET2 running.
- 2026-10-10: Harness switched to API-token auth (fixed token provisioned by `init.groovy.d`) on `main`; M6 and T2 M3 rebased onto it. The cookie-jar race that broke concurrent POSTs is gone.
- 2026-10-10: Fixed the ET2 cron check: the invariant is one build per claimed tick (`Trigger.Cron` catch-up can fire two ticks in one wall-clock minute).
- 2026-10-10: **ET2 PASS.**

## Review and merge order
1. core `cnj/t2-m1-stored-build-numbers` → `cnj/t2-m2-trigger-coordination`, stacked on the T1 core branches.
2. cloud-native-jenkins `main` (harness token auth), then `cnj/t1-m6-postgres-store` → `cnj/t2-m3-postgres-coordination`.

## Follow-ups (remaining R2)
- Webhook fan-in, and routing "Build Now" / "Scan now" to the owner. Today they run where they are received; numbering stays correct.
- A shared or partitioned queue; quiet-period, `build` and `lock` coordination.
- Cluster-singleton `PeriodicWork` beyond triggers, e.g. `BackgroundGlobalBuildDiscarder`.

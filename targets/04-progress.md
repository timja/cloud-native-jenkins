# Target 4: progress and branches

Design: [04-running-build-failover.md](04-running-build-failover.md).

**Done means:** `validation/experiments/ET4-pipeline-failover.sh` passes (the E-T4 criteria).

> **Status: DONE (2026-10-10).** ET4 passes 39/39 on the first run, and ET3 still passes (48/48) on the same build. Evidence: `findings/ET4-pipeline-failover.md` on `cnj/t4-m3-pipeline-failover`.

**Decisions (2026-10-10):**
- agents attach through a failover URL; build placement is a follow-up
- save points ship asynchronously (≤1 s)
- workflow-cps gets a `ProgramSavedListener` hook
- cloud provisioning is left for later

**Policy:** same as Targets 1-3:
- local only, nothing pushed
- stacked branches
- integration branches only if needed

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | workflow-cps | `cnj/t4-m1-program-saved-listener` | release `4391.v652b_b_b_1b_7c00` | `ProgramSavedListener` (worktree `../workflow-cps-plugin-cnj`) | ✅ ebbf16c8; new test + FlowDurabilityTest, PersistenceProblemsTest green |
| M2 | core | `cnj/t4-m2-build-leases` | `cnj/t3-m4-explicit-numbering` | `BuildLeases` SPI, fencing, `BuildAdopter` | ✅ d9d1f814de; `BuildLeasesTest` 5/5 + all `jenkins.persistence` suites, Run, Job, Executor green |
| M3 | cloud-native-jenkins | `cnj/t4-m3-pipeline-failover` | `cnj/t3-m6-blob-and-log-storage` | Postgres leases, save point shipper, `PipelineAdopter`, harness, ET4 (worktree `../cloud-native-jenkins-t4`) | ✅ 3ea33aa; plugin tests green (21, incl. leases); **ET4 PASS** (39/39), ET3 re-run 48/48 |

## Log
- 2026-10-10: Target 4 planned and started.
- 2026-10-10: M1 and M2 committed; M3 plugin and harness written. The harness adds an `agents` compose profile: `lb` (nginx primary/backup) and `agent1` (inbound WebSocket). JCasC `t4-agents.yaml` sets `MAX_SURVIVABILITY`.
- 2026-10-10: **ET4 PASS** (39/39). Adoption took 26-49 s after a kill or pause (20 s membership timeout + 30 s cleanup period). No step re-ran; the partitioned owner fenced itself, and only B published.

## Review and merge order
1. workflow-cps `cnj/t4-m1-program-saved-listener`: independent, and the first candidate for upstream.
2. core `cnj/t4-m2-build-leases`, on the T3 core stack.
3. cloud-native-jenkins `cnj/t4-m3-pipeline-failover`, on `cnj/t3-m6-blob-and-log-storage`.

## Follow-ups
See `findings/ET4-pipeline-failover.md`:
- event-driven adoption, for faster failover
- moving a running build to the controller holding its agent
- an optional synchronous commit for non-idempotent steps
- `lock`, `milestone` and `build` across controllers (Target 5)

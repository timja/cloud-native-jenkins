# Target 2: Cluster-wide build numbers and exactly-once triggers

| | |
|---|---|
| **Roadmap tier** | R2 (subset) |
| **Status** | In progress; see [02-progress.md](02-progress.md) |
| **Depends on** | Target 1 (shared item config, persistence SPI) |
| **Evidence** | [E3](../findings/E3-shared-home.md) (cron fired 8× in 3 min on 2 controllers; `nextBuildNumber` collision), [E9](../findings/E9-runtime-job-state.md) (triggers defined by builds), ET1 observations (`findings/ET1-shared-items.md` on `cnj/t1-m6-postgres-store`) (build numbers diverged; indexing on B rebuilt branches) |

## Goal
With N controllers sharing item configuration (Target 1):
1. **Build numbers are cluster-wide.** Every build of a job gets a unique number from one sequence, whichever controller runs it. The sequence survives disposable controllers and follows renames and moves.
2. **Cron-style triggers fire exactly once per tick across the cluster.** This covers `TimerTrigger`, `SCMTrigger` polling, Pipeline `pipelineTriggers`, and folder periodic indexing (`PeriodicFolderTrigger`).
3. **Each item's triggers fire on a stable owner.** Polling baselines and indexing state are still per controller (R5), so a stable owner is what prevents duplicate builds of the same revision. When the owner dies, a survivor takes over within about a minute.

**Out of scope** (later R2/R3 targets):
- webhook fan-in
- a shared queue
- forwarding user-initiated builds and "Scan now" to the owner
- quiet-period, `build` and `lock` coordination
- other `PeriodicWork`

## Design

### Build numbers
- **Core: `TextFile` routing.** Like `XmlFile` in Target 1, `TextFile` serves keys owned by the store from the store. `Job.getNextBuildNumberFile()` thereby becomes `jobs/…/nextBuildNumber` in the store and moves with the item (`PersistenceStore.move`).
- **Core: `StoredBuildNumberAssigner`.** An `@Extension` of the Beta `Job.BuildNumberAssigner`, ranked above the default. It is active when the job's `nextBuildNumber` is routed:
  1. Read the entry, or seed it from the in-memory number if absent.
  2. Write `n+1` with **strict** CAS against the version read, retrying on `ConflictException` (never last-writer-wins).
  3. `fastUpdateNextBuildNumber(n+1)` and return `n`.
- **Plugin:** owns `jobs/**/nextBuildNumber` in addition to item `config.xml`.

### Triggers
- **Core: `jenkins.triggers.TriggerCoordinator`** (Beta `ExtensionPoint`). `boolean shouldRun(TriggeredItem item, Trigger<?> trigger, Calendar tick)` is consulted in `Trigger.checkTriggers` before `t.run()`, on both the per-item and the synchronous-polling paths. With no implementation, every trigger runs, as today.
- **Plugin: `PostgresTriggerCoordinator`:**
  - **Membership.** `jenkins_members(origin, heartbeat_at)`. Each controller heartbeats every 5 s, and members with a heartbeat younger than 20 s are alive.
  - **Affinity.** owner(item) = rendezvous hash over alive members of the *top-level* item name, so a folder's or multibranch project's whole subtree has one owner. A controller only considers firing triggers it owns in its own view.
  - **Claim.** `insert into jenkins_trigger_claims(item, trigger, tick, origin) … on conflict do nothing`. Only the inserter runs the trigger, which keeps it at-most-once even while two controllers briefly disagree on membership. Old claims are pruned.
  - **Failure mode.** While views diverge, a tick can be missed (nobody believes it owns the item). Missed ticks are preferred to doubled ones: the next tick runs, and cron jobs miss at most a tick during failover.

## Branches
| # | Repo | Branch | Based on |
|---|---|---|---|
| M1 | core | `cnj/t2-m1-stored-build-numbers` | `cnj/t1-m4-item-change-applier` |
| M2 | core | `cnj/t2-m2-trigger-coordination` | T2 M1 |
| M3 | cloud-native-jenkins | `cnj/t2-m3-postgres-coordination` | `cnj/t1-m6-postgres-store` |

## E-T2 acceptance experiment (definition of done)
`validation/experiments/ET2-scheduling.sh` runs on the ET1 setup (two tmpfs controllers, S0, Postgres). Pass criteria:
1. **Cron exactly once.** A `* * * * *` pipeline runs for 4 minutes with A and B up. The total builds across A and B equal the number of ticks, and no tick is doubled.
2. **Unique, contiguous numbers.** 20 manual builds on A and 20 on B of one job give 40 builds numbered exactly 1..40, with no duplicates across controllers.
3. **SCM polling exactly once.** A job polls a shared git repo every minute. One commit gives exactly one build cluster-wide over the following 3 minutes. A second commit gives exactly one more.
4. **Periodic indexing exactly once.** A multibranch project has a 1-minute periodic indexing trigger. Adding a branch gives exactly one build of it cluster-wide, and existing branches are not rebuilt.
5. **Failover.** Stop the controller that owns the cron job. The survivor fires it within 2 minutes, and there is still never more than one build per tick.
6. **Rejoin.** Start a fresh controller (empty home). Ownership rebalances, and cron stays at exactly one build per tick across the transition.
7. **Continuity.** A fresh controller and a renamed job both continue the number sequence instead of restarting at 1.

# Target 5: A cluster-wide build queue (R2 remainder)

| | |
|---|---|
| **Roadmap tier** | R2: queue, placement, quiet period, cross-build coordination, cluster-singleton periodic work |
| **Status** | **Done** (2026-10-10): ET5 passes 35/35, see [05-progress](05-progress.md) |
| **Depends on** | T1 (store, events), T2 (membership, claims, cluster build numbers), T3 (build records, running placeholders), T4 (leases, adoption) |
| **Evidence** | [E3](../findings/E3-shared-home.md) (each controller has its own queue), [E5](../findings/E5-stateless-http.md) (agents connect to one controller), [E7](../findings/E7-ecosystem-scan.md) (131 plugins define periodic work), ET2 (manual builds run where they are requested) |

## Goal
A build requested on any controller (Build Now, the REST API, a trigger, a webhook, an upstream build) joins **one** cluster-wide queue:
- it is visible and cancellable on every controller;
- it is deduplicated and folded once, with the quiet period applied cluster-wide;
- it is blocked by builds running anywhere;
- it starts **exactly once**, on a controller that can run it;
- it survives the controller that accepted it dying.

The coordination that today lives in one JVM works across controllers:
- non-concurrent jobs and upstream/downstream blocking
- `milestone`
- `lock` (lockable-resources)
- the `upstream` trigger
- `build` with `wait: true`
- abort from any controller
- background jobs that must run once per cluster

**Out of scope:**
- Cloud provisioning (`NodeProvisioner` demand across controllers): later, with the cloud work.
- Moving a *running* build to the controller that holds an agent. T4 adoption makes this possible later. For now, a `node` block waits for its agent as today.
- Webhook endpoints themselves: an event received on any controller schedules into the shared queue. That gives "handled once" for builds; plugin-specific event fan-out is a follow-up.

## Why a mirror plus claim, not a new Queue
Research (core `Queue.java`, 3286 lines) found that the queue is deeply in-JVM:
- Items hold live `Task` references and are deduplicated by `Task.equals` over in-memory lists.
- Futures (`QueueTaskFuture`, `WorkUnitContext`) are JVM-local.
- Blocking checks local executors and run maps.
- Listeners fire under a local `ReentrantLock`.
- Replacing `Queue` is not possible from a plugin and would break much of the ecosystem.

It does have the right seams:
- `QueueDecisionHandler`: veto at schedule time.
- `QueueTaskDispatcher.canRun` / `canTake`: keep an item queued with a cause of blockage.
- `QueueListener`: lifecycle events.
- `QueueIdStrategy` and `Queue.State` (both Beta).
- The `LoadBalancer`.

So, as for builds in T3: **the store holds the cluster queue, and every controller mirrors it into its local `Queue`.** The local queue keeps doing what it is good at (quiet period, blocking, matching executors and labels). Before any controller *starts* an item, it must **claim** it in the store, which is a compare-and-set, so exactly one wins. The losers drop their mirror.

## Design

### 1. Cluster queue records (store plugin; Postgres)
- **Table** `jenkins_queue`:
  - `id` (from a cluster sequence, also used by `QueueIdStrategy`, so ids are unique cluster-wide and match `Run.getQueueId`)
  - `task` (item full name)
  - `dedup_key`: task plus a hash of the `QueueAction` identity, e.g. parameter values, so that `ParametersAction.shouldSchedule` semantics hold
  - `actions` (XStream XML of the persistable actions)
  - `due_at` (the quiet period, in DB time)
  - `state`: `waiting` | `claimed` | `started` | `cancelled`
  - `claimed_by` / `claim_epoch`
  - `build_number` once started
  - `origin`, `created_at`
- **Events:** the existing LISTEN/NOTIFY event log carries the changes (`queue/<id>` keys), so every controller mirrors new, folded, cancelled and claimed items.

### 2. Scheduling and folding (core hooks)
Core Beta SPI is `ClusterQueue`, obtained through the store like `leases()`.

**Scheduling:**
- `Queue.scheduleInternal` keeps its local dedup.
- When it creates a `WaitingItem` locally for a job, and not while mirroring a remote change, it publishes it with `insert … on conflict (dedup_key) where state = 'waiting'`.
- If the dedup key already exists, the item folds into the existing record:
  - `CauseAction` causes are merged;
  - `due_at` moves to the earlier of the two, as core does locally;
  - the local item is replaced by the mirror of the existing record.
- So "Build Now" on A and B within a quiet period gives one item and one build, with both causes.

**Mirroring:**
- A remote record enters the local queue as a `WaitingItem` with the record's id, due time and actions, under `applyingRemoteChange`, so it is not re-published.
- Cancelling or starting a record elsewhere removes the local mirror without firing user-visible cancellation (no `onLeft` with a cancelled future for the start case).

**Not shared (stay local):**
- Pipeline `node` blocks (`ExecutorStepExecution.PlaceholderTask`): they belong to a running build on its owner.
- Other subtasks and `TransientTask`s.
- Items carrying `BuildTriggerAction` (`build` with `wait: true`) are shared, but pinned to the upstream's controller (see 6).

### 3. Claiming and placement
**New core hook (Beta):** a `QueueStartGate` is consulted in `Queue.maintain` right before an item is handed to an executor. That covers both `makePending` (heavyweight) and `makeFlyWeightTaskBuildable` (flyweight Pipeline parents, which bypass the `LoadBalancer`).
- Returning false leaves the item buildable, with a cause such as "Claimed by jenkins-b".
- The store plugin's gate claims the record: `update … set state='claimed', claimed_by=me where id=? and state='waiting'`. If it fails, the local mirror is dropped.
- On `Executor` start, the record becomes `started` with the build number (`WorkUnit.setExecutable`). From then on the build is a T3/T4 running build: it has a placeholder and a lease.

**Placement:**
- **Default:** first come, first served. Any controller with a free matching executor claims, which spreads load.
- **Affinity for agent labels:** if an item's label matches only agents that are connected to another live controller, this controller does not claim it for a grace period, so the controller with the agent gets it first. Connected agents are published per controller in the membership table.
- **Job owner affinity:** an optional per-job setting, defaulting to off, uses the T2 rendezvous owner instead.

**Crashes:**
- A claimed-but-not-started item whose claimer is gone (T2 membership) goes back to `waiting` and is claimed again.
- A started item is a running build, so T4 handles it.
- Waiting items need no owner, so they survive any controller.

### 4. Ids, REST and cancellation
- `QueueIdStrategy` comes from the cluster sequence.
- **Queue items from any controller:**
  - `GET /queue/item/<id>/api/json` on any controller resolves to the local mirror.
  - If there is none, it resolves from the store record, as a read-only `RemoteQueueItem` with `why`, `cancelled` and `executable {number, url}`.
  - Clients that poll the `Location` returned by "Build Now" therefore work behind a non-sticky load balancer.
- **Cancel anywhere:** `doCancelItem` on any controller marks the record `cancelled`, and every mirror leaves. A `build` step waiting on it fails on its own controller (6).
- **Stop anywhere:**
  - Stopping a build running on another controller (from its placeholder, or `job/x/N/stop` routed by the placeholder) writes a stop request `jobs/<item>/running/<N>.stop`.
  - The owner, which holds the lease, interrupts the build.
  - This is the primitive that `milestone` and queue cancellation reuse.

### 5. Cluster-wide blocking
A core `QueueTaskDispatcher.canRun` consults the store:
- **Non-concurrent job:** blocked while a build of the job is running anywhere, using placeholders and leases from T3/T4. The cause is "Build #N is already in progress on jenkins-b".
- **`blockBuildWhenUpstreamBuilding` / `Downstream`:** the same check over the `DependencyGraph` (identical everywhere, from shared config) against running placeholders and the cluster queue's claimed/started records.

### 6. Cross-build coordination
- **`upstream` trigger and freestyle "Build other projects":**
  - They fire on the controller that ran the upstream build. `RunListener.onCompleted` fires only there, and the dependency graph is identical everywhere.
  - That controller schedules into the shared queue, so each downstream is queued once.
  - ET5 verifies they never fire twice: remote completion fires no listeners.
- **`build` step (pipeline-build-step):**
  - Its `StepContext` lives in the upstream's JVM, and completion is delivered by `RunListener.onFinalized` on the controller that ran the downstream. So items with a `BuildTriggerAction` are **pinned to the upstream's controller**: the gate lets only that controller claim them.
  - `wait: false` items are not pinned.
  - The general fix, delivering downstream completion to a remote upstream through the store, is a follow-up.
- **`milestone`:**
  - pipeline-milestone-step already has a `MilestoneStorage` extension point. The store plugin implements it with per-job milestone state in the store, updated with CAS.
  - `cancel(job, number, …)` writes a stop request (4), so a newer build passing a milestone on A cancels an older one running on B.
- **`lock` (lockable-resources):**
  - All lock state and the waiting queue live in one `GlobalConfiguration` (`LockableResourcesManager`, guarded by `syncResources`), so it needs a cluster mode.
  - Taking a resource is a CAS on `lockable-resources/<name>` in the store, recording the holder build and controller.
  - Each controller keeps its local waiters and retries on release events, ordered by enqueue time stored with the waiter, so fairness is approximate.
  - Locks held by a build follow its lease. If the build is adopted (T4), the lock stays held; if it is abandoned, the lock is released.
  - This is a branch of lockable-resources.

### 7. Cluster-singleton periodic work
- **New core hook (Beta):** `PeriodicWork` consults a `PeriodicWorkCoordinator` before each run, the same pattern as T2's `TriggerCoordinator`. It is applied only to work declared cluster-scoped through an opt-in marker, because most periodic work (E7: 131 plugins) is legitimately per JVM.
- **Core classification** (from research):
  - Cluster singletons: `BackgroundGlobalBuildDiscarder`, `FingerprintCleanupThread` (with shared fingerprints), `Telemetry`.
  - Per controller: `LoadStatisticsUpdater`, `MemoryUsageMonitor`, `HudsonHomeDiskUsageChecker`, `DailyCheck`.
  - Agent-bound: `WorkspaceCleanupThread`, `ComputerRetentionWork`, `ConnectionActivityMonitor`, which run on the controller holding the agent.
- **Store plugin:** claims one run per period through `jenkins_trigger_claims`, reusing the T2 table and rendezvous owner.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | core | `cnj/t5-m1-cluster-queue` | `cnj/t4-m2-build-leases` | `ClusterQueue` SPI and an in-memory implementation; publish, fold and mirror; `QueueStartGate` in `maintain` (heavy and flyweight); cluster `QueueIdStrategy`; `RemoteQueueItem` REST fallback; cancel anywhere; tests |
| M2 | core | `cnj/t5-m2-cluster-blocking` | M1 | stop requests (`running/<N>.stop`) honoured by the lease owner; cluster-wide non-concurrent and upstream/downstream blocking via `QueueTaskDispatcher`; tests |
| M3 | core | `cnj/t5-m3-cluster-periodic-work` | M2 | `PeriodicWorkCoordinator` plus a cluster-scoped marker; applied to `BackgroundGlobalBuildDiscarder`, `FingerprintCleanupThread`, `Telemetry`; tests |
| M4 | lockable-resources | `cnj/t5-m4-cluster-locks` | `origin/master` | store-backed lock state and cross-controller waiters |
| M5 | cloud-native-jenkins | `cnj/t5-m5-shared-queue` | `cnj/t4-m3-pipeline-failover` | Postgres `jenkins_queue`, claims and reclaim; agent-aware placement; `MilestoneStorage` in the store; `build`-step pinning; periodic-work claims; harness (lockable-resources and milestone plugins); `ET5-shared-queue.sh` |

## E-T5 acceptance experiment (definition of done)
ET setup (two tmpfs controllers, S0, Postgres, Azurite, token auth, `agent1` through the failover URL).
1. **Fold once:**
   - A job has a 15 s quiet period. "Build Now" on A, then on B 5 s later.
   - Exactly one build, with both causes. The queue item has the same id on A and B.
2. **Exactly once, spread:**
   - 40 parameterized builds are submitted alternately to A and B.
   - Each runs exactly once; both controllers run some.
   - Queue ids are unique cluster-wide. `/queue/item/<id>/api/json` resolves to the right executable on both controllers, including after the item left.
3. **Non-concurrent:**
   - A non-concurrent job is building on A (30 s). "Build Now" on B.
   - B's item shows "already in progress on jenkins-a", then runs after it. The two builds never overlap.
4. **Cancel anywhere:** an item in its quiet period is created on A and cancelled on B. It is gone on both and never runs.
5. **Stop anywhere:** a build running on A is stopped from B. It is aborted on A.
6. **Queue survives its controller:**
   - Three items are waiting in their quiet period, accepted by A. A is SIGKILLed.
   - They all run on B. An item A had claimed but not started is reclaimed.
7. **Agent affinity:** with `agent1` connected to B, an item for label `agent1` scheduled on A runs on B.
8. **`milestone`:** build #1 on A sleeps before `milestone 1`; build #2 on B passes `milestone 1` first. #1 is cancelled on A.
9. **`lock`:** two builds on different controllers `lock('res')` for 20 s each. They never overlap, and both complete.
10. **Upstream / `build`:**
    - Pipeline `upstream` trigger: the downstream runs exactly once per upstream completion, whichever controller ran the upstream.
    - `build(job: 'down', wait: true)` from A completes, with `down` run on A.
11. **Singleton periodic work:** over 5 minutes, `BackgroundGlobalBuildDiscarder` runs once per period cluster-wide (claims table), not once per controller.

## Risks
- **Mirror divergence.** Local queue semantics (folding, `QueueAction.shouldSchedule` by plugins, `FoldableAction`) must be reproduced from serialized actions. Plugins with non-persistable actions lose them across controllers. ET5 records which actions in the harness survive.
- **Claim latency** adds a store round-trip per start. A lost claim leaves a short window where a build could look buildable on two controllers; it never starts twice.
- **Approximate fairness** for queue order and `lock` waiters across controllers.
- **`build` step pinning** reduces load spreading for build-step-heavy pipelines until remote step completion exists.
- **lockable-resources is a large plugin** with many entry points (UI reserve/steal, labels, scripts). M4 covers `lock` acquire/release and waiting first; the UI paths come after.

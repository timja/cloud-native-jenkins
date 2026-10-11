# JEP-CNJ-4: Cluster scheduling: build numbers, triggers, a shared queue

| | |
|---|---|
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-CNJ-1 |

## Abstract
Several controllers schedule the same jobs. Five pieces make that work:
- build numbers are unique across the cluster
- every trigger fires once per tick, cluster-wide (cron, SCM polling, branch indexing, webhooks)
- a shared queue gives every queue item one id on all controllers, and lets one controller start it
- quiet periods, folding, blocking, `lock` and `milestone` hold across controllers
- periodic work that must run once is declared a cluster singleton

## Specification

### Build numbers
`Job.BuildNumberAssigner` (an existing Beta hook) is implemented by `StoredBuildNumberAssigner`, a compare-and-set increment of the job's stored `nextBuildNumber`. Numbers continue across controllers, fresh controllers, renames and deletions (JEP-CNJ-2).

### Triggers
- **`TriggerCoordinator`.** A controller claims `(item, trigger, tick)` in the store before running a trigger, and only the first claim runs. Cron triggers, SCM polling and periodic folder indexing all go through it. A tick missed while the store is unreachable is skipped, never run twice.
- **Affinity.** Each item's triggers prefer one live controller, by hash over the live members. A controller leaving moves its items to the others.
- **Webhooks.** `notifyCommit` and SCM events may reach any controller, or several. They schedule through the shared queue, so they result in one build.

### Shared queue (`ClusterQueue` store facet)
- **Ids.** Queue item ids come from a cluster sequence (`QueueIdStrategy`), so an item has the same id everywhere. While the store is unreachable, a local id from a disjoint range is used.
- **Mirroring.** An item entering a controller's queue is published as WAITING, and mirrored into every controller's queue, so `queue/item/<id>` and cancel work anywhere.
- **Folding.** Duplicate requests fold across controllers, as on one controller: the same task with the same parameters, within the quiet period.
- **Claiming.** A buildable item is started by the controller that claims it:
  - The requesting controller goes first, for a grace period, since it may hold the agents the build needs.
  - A label-bound item goes where an agent with the label is; an agent being provisioned is reserved for its item.
  - After that, any controller that can run it may claim it.
  - `build(wait: true)` items are pinned to their caller.
- **Blocking.** `SharedBlocking` blocks a non-concurrent job while it builds anywhere in the cluster.
- **Reclaim.** Items claimed by a controller that is gone go back to WAITING:
  - on the next maintenance pass (every 15 s)
  - at once when a controller is seen leaving
  - never right after a database outage (JEP-CNJ-3's stability rule)
- **Recording starts.** An item's start is recorded with retries, since an item left claimed would run again if its controller later went away.

### Cross-build coordination
- **`lock`** (lockable-resources) keeps its resources and queue in the store; one controller grants each resource at a time.
- **`milestone`** ordering is kept in the store (`ClusterMilestoneStorage`).
- **`@ClusterSingleton` periodic work.** A `PeriodicWorkCoordinator` runs work marked `@ClusterSingleton` on one controller only. In core these are the global build discarder, fingerprint cleanup and telemetry. Unmarked periodic work keeps running on every controller, since much of it is legitimately per JVM.

## Motivation
- **E3:** on two controllers, a cron trigger fired 8 times in 3 minutes, and only the JENKINS-23152 directory check avoided a build number collision.
- **E5:** an agent connects to one controller only, so where a build may start matters.
- **E7:** 131 plugins (37 in the BOM) define periodic work, so blanket leader election would be wrong.

## Reasoning
- **Claims instead of leader election** for triggers. There is no failover gap when a controller dies, since the next tick's claim simply goes to another controller.
- **A mirrored queue,** not a single global scheduler. Every controller keeps Jenkins' own queue semantics (causes of blockage, the UI, `QueueListener`s), and the store arbitrates only who starts what.
- **The requesting controller goes first,** which keeps label-bound and agent-affine work local without a global view of executors.

## Backwards compatibility
- Without a store, nothing changes.
- `@ClusterSingleton` is opt-in, and plugins with periodic work that must run once need to adopt it.
- **Known limitation:** an `upstream` trigger that fires from a build completing on another controller is not covered (ET2 did not test it).

## Security
Queue items keep their authentication (`QueueItemAuthenticator`), which is serialised with their actions and checked by the controller that starts them.

## Testing
- **Core:** `StoredBuildNumberAssignerTest`, `ClusterCoordinationTest`, `SharedQueueTest`.
- **Store plugin:** `PostgresPersistenceStoreTest` (claims, queue, counters).
- **Acceptance:**
  - ET2: unique numbers for 40 concurrent builds; cron, polling and indexing exactly once, also across failover and rejoin
  - ET5: shared queue, folding, cancel, blocking, `lock`, `milestone`
  - ET7: webhook fan-in

## Prototype implementation
- **Core:** `cnj/t2-*`, `cnj/t5-*`.
- **lockable-resources:** `cnj/t5-m4-cluster-locks`.
- **Store plugin:** `PostgresTriggerCoordinator`, `SharedQueueMaintenance`, `ClusterMilestoneStorage`.
- **Findings:** `findings/ET2-scheduling.md`, `ET5-shared-queue.md`.

## References
- E3 (shared home) in this repository
- `Job.BuildNumberAssigner` and `QueueIdStrategy` (existing Beta hooks)

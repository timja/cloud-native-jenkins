# JEP-252: Ownership of running builds: leases, fencing, adoption

| | |
|---|---|
| **JEP** | 252 |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-250, JEP-251 |

## Abstract
Every running build has exactly one owner: the controller holding its lease. A Pipeline ships save points (its program state and flow nodes) to blob storage as it runs. When its owner dies, another controller takes the lease, with a higher epoch, and resumes the Pipeline from the latest save point, with no shared disk. An owner that lost its lease (for example after a pause or a partition) is fenced: it stops the build and never publishes it. Builds nothing can resume are closed as aborted rather than vanishing.

## Specification

### Placeholders and leases
- **Placeholders.** `RunningBuilds` writes a placeholder `jobs/<job>/running/<N>.xml` when a build starts. It records the owner's origin and name, the start time, the lease epoch, and properties such as the latest save point. It is removed after the build's record is published (JEP-251).
- **`BuildLeases`** (a store facet):
  - `acquire(key)` returns an epoch, and succeeds only if the lease is free or its holder is no longer a live member
  - `holds(key, epoch)`
  - `release(key, epoch)`
- **Fencing.** Every placeholder update is a compare-and-set on the placeholder as the owner last wrote it, and fails if another controller adopted the build. A controller checks its leases:
  - periodically
  - whenever its membership may have lapsed (heartbeats missing for longer than the member timeout)

  On evidence from the store that a lease is lost, it stops the build locally (`lost`) and never publishes it.
- **No fencing on missing evidence.** If the store cannot be reached, a lease's state is unknown. The build carries on, while its log and save points wait until the heartbeat is back.

### Save points (Pipelines)
- workflow-cps calls a `ProgramSavedListener` whenever it saves `program.dat` (a durability-hint hook).
- The store plugin then copies `program.dat`, the changed flow nodes and `build.xml` to blob storage under `…/builds/<N>/running/`. Artifacts and stashes belong in an artifact manager (JEP-202) that keeps them in shared storage as they are made; the reference setup uses azure-artifact-manager, in the same blob container. Without one, save points also carry the build directory's other files changed since the last one (up to 64 MiB each), so that an adopted build still finds them.
- It makes the log durable up to that point, then commits the save point by recording its id in the placeholder. The reference log storage (JEP-251) ships its pending output. Another log storage needs to be told too: **proposed**, an optional flush on JEP-210's `LogStorage` (workflow-api), called before a save point commits. A synchronous implementation does nothing; a buffering one (CloudWatch batches its events) sends its buffer. Without it, an adopted build's log can miss the lines written just before the save point.
- Commits are coalesced: about two placeholder writes per build for a 200-step test Pipeline.
- `PERFORMANCE_OPTIMIZED` Pipelines have no save points and are not adopted.

### Adoption
- **A sweep looks for placeholders whose owner is not a live member.** It runs every 30 s, and at once when a controller is seen leaving the cluster (JEP-255): when its database session lock is freed (its process died), when its heartbeat deadline passes, or when a cleanly stopped controller announces it is leaving.
  - It acquires the lease (only one controller wins).
  - It asks the `BuildAdopter` extensions to prepare an adoption.
  - The Pipeline adopter stages the save point and the log into the home, and resumes the build as owner with the new epoch.
- **What else happens on adoption:**
  - A stop request made while the owner was gone is honoured.
  - `input` steps, `lock`, `milestone` and the build's place in the shared queue carry over (JEP-253).
- **Workspaces.** A workspace is on the agent, not the controller, so an adopted build finds what its earlier stages left there as long as the agent survives: a static or inbound agent reconnects to the adopting controller (agents follow the build's lease), and a later stage reads the files an earlier stage wrote. A Kubernetes pod agent belongs to the controller that started it and goes with it; the adopted build's `node` block is retried on a new pod with an empty workspace. Files a later stage needs from such a pod must be carried with `stash`/`unstash` or archived, through an artifact manager that keeps them in shared storage (JEP-202).
- **Builds nothing can resume** (freestyle, `PERFORMANCE_OPTIMIZED`, or a save point that cannot be found) are closed as ABORTED: a minimal record, with a console saying why, after what the build had printed (shipped while it ran, JEP-251). This also covers builds that were running when a backup was taken, after a restore.
- **No takeovers right after an outage.** A controller only takes others for gone after its own heartbeats have reached the database without a gap for longer than the member timeout. Right after a database outage, every heartbeat is stale, the live controllers' included.

### Restarts and handover
- **A controller restarted with its home** (for example by a safe restart to apply plugin changes) takes over the leases of the builds it was running *before* they are loaded and resumed. It uses `RunListener.allowLoad`, as an adopter would. If another controller already adopted a build, the local copy is not loaded. That happens often: a dead process is seen gone within seconds, sooner than a restart completes, so a build is checked against the store, not against what this controller had cached.
- **Handover by label.** A Pipeline waiting at `node('label')` for a label only another live controller has is handed over:
  1. its save point is flushed
  2. a handover request names the target
  3. the lease is released
  4. the target takes the build over and resumes it, where the `node` step gets an executor

  A build is handed over only while it holds no other executor, so nested or parallel `node` blocks for labels on different controllers wait instead of moving back and forth; a build is handed over at most four times. Running a `node` block on another controller's agent without moving the build would mean relaying Remoting between JVMs, which this JEP does not attempt.
- **Safe restarts** pause Pipelines at a save point (they resume afterwards, here or elsewhere) and wait for builds that cannot survive a restart, as on one controller.

## Motivation
- An experiment killed a controller running Pipelines and started another on the same home. The Pipelines survived, but only through the shared disk (`program.dat`, flow nodes, `FlowExecutionList`), plus a manual move of the agent's DNS name.
- Nothing stops a partitioned or paused controller from carrying on. Two controllers can then run, and publish, the same build.

## Reasoning
- **Epoch-fenced leases** make the store the arbiter, so no consensus protocol is needed between controllers.
- **Save points in blob storage at the moments Pipelines already persist** cost little: 849 ms against 253 ms with `PERFORMANCE_OPTIMIZED` for 200 `echo` steps, and most of that is `MAX_SURVIVABILITY`'s own local writes.
- **Failover time** is the time to notice a dead owner plus the time to stage the save point. A dead process frees its database session lock at once, so its builds were adopted 5.0–6.3 s after a `SIGKILL` and 6.5 s after a clean stop.
  - Before session locks, liveness was the heartbeat timeout alone (20 s, heartbeats every 5 s), and failover took 16–23 s; before the event-driven sweep, the 30 s sweep made it up to about 50 s.
  - A paused or partitioned owner keeps its lock, so it is still waited for until the heartbeat timeout (15 s): a controller paused for 12 s lost nothing. A lower timeout would be faster, but then a GC pause or a slow store makes others take over a healthy controller's builds (fenced safely, but the work restarts from its last save point).
- **"Unknown is not lost"** keeps a database outage from aborting every running build in the cluster. Before this rule, an outage of more than 20 s did exactly that.

## Backwards compatibility
- Without a store, nothing changes. `BuildAdopter` is a new extension point.
- Pipeline steps that are not idempotent (for example a `sh` step whose process died with its controller) run again from the last save point after an adoption, as after a restart on one controller.
- **Known limitation:** running only a `node` body on another controller's agent would need remote executors. Handover moves the whole build instead.

## Security
- Adoption runs the build as it was configured, with the same authorization (`QueueItemAuthenticator`) as on its original controller.
- Save points have the same content as the build's `program.dat` and flow nodes.

## Testing
- **Core:** `RunningBuildsTest`, `BuildLeasesTest`.
- **Store plugin:** `PostgresPersistenceStoreTest` (leases, no takeover before the controller has been steady).
- **Acceptance** (scripts in `validation/experiments/` of the cloud-native-jenkins repository; each runs two controllers against one store):
  - `ET4-pipeline-failover.sh`: a Pipeline survives its controller being killed during an agent `sh` step and at `input`; a later stage reads what an earlier stage left in the workspace, with the controller killed between them; a paused controller is fenced; a healthy controller's build is never adopted; a `PERFORMANCE_OPTIMIZED` build is closed; the cost of save points
  - `ET12-remainder.sh`: a Pipeline waiting for a label only another controller has is handed over, once, with parallel `node` blocks on different controllers; after every controller is killed, fresh controllers adopt or close the builds that were running
  - `ET13-cluster-plugins.sh`: a Pipeline survives rolling restarts of every controller
  - `ET14-store-outage.sh`: database and blob-store outages cause no aborts or takeovers
  - `ET15-logs-api-failover.sh`: the time from a controller being killed, stopped or paused to its builds being adopted; a freestyle build closed as aborted keeps what it printed
  - `ET9-kubernetes.sh`: a build whose controller's pod was deleted is retried on a new agent pod and gets an earlier stage's files from its stash

## Prototype implementation
- **Core:** prototype branches of Jenkins core, `cnj/t4-*`, `cnj/t12-m1-remainder`, `cnj/t14-m1-resilience`, `cnj/t15-remaining`.
- **workflow-cps:** prototype branch `cnj/t4-m1-program-saved-listener`.
- **Store plugin** (`impl/persistence-store-postgresql` in the cloud-native-jenkins repository): `SavePoints`, `PipelineAdopter`, `Handover`.
- **Findings:** documents in the cloud-native-jenkins repository, on its prototype branches: `findings/ET4-pipeline-failover.md`, `ET12-remainder.md`, `ET13-cluster-plugins.md`, `ET14-store-outage.md` and `ET15-logs-api-failover.md`.

## References
- `findings/E4-pipeline-failover.md` in the cloud-native-jenkins repository: cold failover of running Pipelines through a shared home
- [Pipeline durability settings](https://www.jenkins.io/doc/book/pipeline/scaling-pipeline/)

# Target 4: Running Pipeline builds survive their controller (R4 + R3)

| | |
|---|---|
| **Roadmap tier** | R4 (Pipeline data) + R3 (running builds: leases, fencing, adoption) |
| **Status** | **Done** (2026-10-10): ET4 passes 39/39, see [04-progress](04-progress.md) |
| **Depends on** | Target 1 (store, events), Target 2 (membership heartbeat), Target 3 (running placeholders, log shipping, build records) |
| **Evidence** | [E4](../findings/E4-pipeline-failover.md) (cold failover works, but only on a shared disk, and nothing fences a partitioned owner), [E1](../findings/E1-home-inventory.md), [E9](../findings/E9-runtime-job-state.md), [ET3](../findings/ET3-build-records.md) |

## Goal
When the controller running a Pipeline build dies, another controller **adopts** the build within about a minute and **resumes** it from its last save point. That covers steps on an agent and waiting at `input`. The build completes with the right result and a complete log, under the same number, listed on every controller. No shared filesystem: every controller still starts with an empty home.

A controller that is **cut off but still alive** (partitioned) must stop working on a build it no longer owns. Exactly one controller ever completes a build.

**Out of scope:**
- Cloud provisioning (Kubernetes, EC2, Docker), cloud agents, and orphan cleanup: later.
- Freestyle builds. They cannot resume after a restart, even on one controller.
- Cross-controller live interaction while the owner is alive, such as following the console from another controller or `input` and abort routed to the owner. That is the R3 "routing" item.
- `PERFORMANCE_OPTIMIZED` builds. They only save at shutdown, so a crash leaves nothing to resume. HA mode forces `MAX_SURVIVABILITY` (see Durability).

## What E4 needed from the shared disk, and where it comes from now
| File (in the build dir unless noted) | Written | Source after this target |
|---|---|---|
| `program.dat`: the CPS continuation, plus pickles such as `ExecutorPickle`, `FilePathPickle` and `SecretPickle` | atomic rename at every step boundary in durable modes (`CpsThreadGroup.saveProgram`, `:607-643`) | blob, per save point |
| `build.xml` while running: CPS heads, start nodes, script, iota | `WorkflowRun.save` / `CpsFlowExecution.saveOwner` at head changes | blob, per save point |
| `workflow/<id>.xml` flow nodes (`SimpleXStreamFlowNodeStorage`, one file per node, atomic in MAX mode) | per node | blob per node, uploaded once and again only when changed |
| `log`, `log-index` | continuously | already in append blobs (T3) |
| `FlowExecutionList.xml` (home) | at register/unregister | not needed: the adopter registers the build itself |
| `secrets/` (`SecretPickle`, agent HMAC `secrets/jenkins.slaves.JnlpSlaveAgentProtocol.secret`) | once | S0 seed (identical on every controller). The seed must include the JNLP secret file |
| agent-side: `durable-task` control dir, workspace | on the agent | unchanged: the agent survives and reconnects |

Loading a build directory that holds all of these runs the existing resume path unchanged:
1. `WorkflowRun.onLoad` (`WorkflowRun.java:574-628`) runs `CpsFlowExecution.onLoad`, which calls `loadProgramAsync(File)` (`CpsFlowExecution.java:923`). The run prints "Resuming build at … after Jenkins restart".
2. `ExecutorPickle` re-queues the `node` block for the **same agent name**; `ExecutorStepExecution.java:801-804` pins the label to the computer name. It waits while the agent is offline.
3. `DurableTaskStep` re-finds the `sh` process through its control dir once the agent's channel is back.
4. A pending `input` is restored from `program.dat`, and its `InputAction` ids from `build.xml`.

## Design

### 1. Consistent save points (workflow-cps, one small hook)
Polling the files would pair a `program.dat` from one save with a `build.xml` or flow nodes from another, which can break resume. So workflow-cps gets an extension point fired on the CPS VM thread right after `saveProgram` has atomically renamed `program.dat`. Flow nodes are flushed and the owner saved just before (`saveProgramIfPossible`, `CpsThreadGroup.java:582`):

```java
// workflow-cps, @Restricted(Beta.class)
public interface ProgramSavedListener extends ExtensionPoint {
    /** program.dat, flow nodes and build.xml are consistent on disk; called on the CPS VM thread. */
    void onProgramSaved(CpsFlowExecution execution, File programDat);
}
```

This is the only Pipeline plugin change. It is generic: it is useful for any backup or replication of running builds, so it can go upstream.

### 2. Shipping save points (store plugin)
The listener does not upload on the CPS VM thread; that would slow every step. It copies the save point into a staging directory and a shipper uploads the latest one asynchronously every `≤ 1 s`, coalescing intermediate ones. A crash therefore loses at most about 1 s of progress.

Resuming from a slightly older save point re-runs steps after it, which is at-least-once. That is the same guarantee Pipeline gives today after a crash. `sh` is safe: it re-attaches to the still-running process through the control dir pickled in `program.dat`.

Each save point is uploaded under a sequence number, then committed by one CAS write of the manifest, which is the commit marker. A torn upload is never visible.

| Data | Where | Key |
|---|---|---|
| `program.dat` | blob | `jobs/<item>/builds/<N>/running/program.dat.<seq>` |
| running `build.xml` | blob | `.../running/build.xml.<seq>` |
| flow nodes | blob | `.../running/workflow/<id>.xml`, re-uploaded only when changed. Nodes are immutable after their head moves on, so the cost is O(nodes) per build, not per save point |
| manifest: owner origin, **lease epoch**, `seq`, node list | `PersistenceStore` | the T3 placeholder `jobs/<item>/running/<N>.xml` gains these fields |

Older `<seq>` blobs are deleted after the next commit. Everything under `builds/<N>/running/` is deleted when the completed record is published. `BuildRecords` already excludes `program.dat` and `workflow/**` from the snapshot.

### 3. Ownership leases and fencing (core SPI + Postgres)
- **SPI (optional, Beta):** `PersistenceStore.leases()` returns `BuildLeases` or null:
  - `acquire(key, origin)` → epoch or null
  - `renew(origin)`
  - `holds(key, epoch)`
  - `release(key, epoch)`
- **Postgres:** a table `jenkins_build_leases(key pk, owner_origin, epoch, renewed_at)`.
  - The epoch comes from the existing version sequence, so it only grows.
  - `acquire` is `insert … on conflict do update … where renewed_at < now() - timeout` (DB time, as membership), returning the new epoch.
  - The membership heartbeat renews all of a controller's leases in the same statement.
- **Owner:**
  - It takes the lease in `RunningBuilds.started`, before any work.
  - Every manifest commit is a CAS on the placeholder, conditioned on `epoch == mine`. This is the fencing token.
  - The final record publish and placeholder delete are conditioned the same way.
  - Append-blob log writes stop as soon as a commit is fenced.
- **Partitioned owner:** a fenced commit, or a failed lease renewal past the timeout, means another controller may own the build. The old owner then:
  - interrupts its local execution, with cause "ownership lost";
  - skips publishing and placeholder deletion, because its epoch no longer matches;
  - deletes its local copy of the build.

  Steps running on agents in that window are the same exposure as today's crash-then-resume. The agent can only be connected to one controller, so in practice only the controller holding the agent's channel makes progress.

### 4. Adoption (core hook + store plugin)
- **Core (Beta):** `RunningBuilds.removeStale()` first offers each stale placeholder to `BuildAdopter` extensions. It only deletes the placeholder, as T3 does today, if no adopter takes it, for example for a freestyle build or a `PERFORMANCE_OPTIMIZED` one.
- **Store plugin, `PipelineAdopter`:**
  1. Acquire the lease (one winner cluster-wide).
  2. Download the committed save point into `builds/<N>/`: `build.xml`, `program.dat` and `workflow/*.xml`. Also download `log` and `log-index` from their append blobs, so the shipper resumes appending at the right offset.
  3. Add N to the run map (`recognizeNumber`). Load the run; `WorkflowRun.onLoad` then resumes it.
  4. Rewrite the placeholder with this controller's origin and the new epoch, so other controllers show "running on B".
- **Restarted controller:** the same mechanism covers a controller that restarts with a new origin. Its old builds are adopted by whichever controller notices, possibly itself. This fixes the T3 follow-up "resumed build keeps its old origin".

### 5. Agents
- Permanent inbound agents are defined identically on every controller (S0, JCasC). Their secrets match because the JNLP HMAC key is in the seed.
- An agent connects to **one** controller at a time; core rejects a second connection only per controller.
- **This target:** each agent reaches the controllers through a URL that **fails over**, not one that load-balances. In the harness that is nginx with `jenkins-a` as primary and `jenkins-b` as `backup`, for the agent port only. The remoting WebSocket client reconnects to the same URL every 10 s (`Engine.runWebSocket`, `:552-701`), so it lands on the survivor.
- **Limitation, documented:** while both controllers are up, a build on B that needs an agent attached to A waits. Placing a build where its agent is connected, or tunnelling agent channels, is the R3 routing follow-up.

### 6. Durability
- HA mode needs `MAX_SURVIVABILITY` or `SURVIVABLE_NONATOMIC`, which save at every step boundary. The harness sets the global default through JCasC.
- A `PERFORMANCE_OPTIMIZED` build is not adopted. Its placeholder is removed, as in T3, and the build is marked failed with "not resumable after controller loss".
- ET4 measures the per-step cost of shipping.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | workflow-cps | `cnj/t4-m1-program-saved-listener` | the release the harness runs (`4391.v652b_b_b_1b_7c00`) | `ProgramSavedListener` fired after `saveProgram`'s atomic rename; JenkinsRule test |
| M2 | core | `cnj/t4-m2-build-leases` | `cnj/t3-m4-explicit-numbering` | `BuildLeases` SPI and an in-memory implementation; lease taken in `RunningBuilds.started`; epoch-fenced placeholder, record and delete; `BuildAdopter` hook in `removeStale`; tests |
| M3 | cloud-native-jenkins | `cnj/t4-m3-pipeline-failover` | `cnj/t3-m6-blob-and-log-storage` | Postgres leases (renewed by the heartbeat); save point shipper and manifest; `PipelineAdopter`; fencing of a partitioned owner; log offset alignment; harness changes (below); `ET4-pipeline-failover.sh` |

**Harness additions:**
- `jenkins/inbound-agent` (WebSocket) as `agent1`, defined by JCasC on both controllers.
- nginx with primary and backup for agents.
- JCasC sets the `MAX_SURVIVABILITY` default.
- The seed includes the JNLP secret.

## E-T4 acceptance experiment (definition of done)
ET setup (two tmpfs controllers, S0, Postgres, Azurite, token auth) plus `agent1` connected through the failover URL.

The Pipeline under test:
- `stage('a') { echo }`
- `node('agent1') { sh 'for i in $(seq 1 60); do echo tick $i; sleep 1; done' }`
- `input 'continue?'`
- `node('agent1') { sh 'echo after input' }`
- `echo done`

Pass criteria:
1. **Kill during `sh`:**
   - SIGKILL A at about tick 20. B adopts within 60 s (placeholder "running on jenkins-b"), and the agent reconnects to B.
   - The `sh` continues to tick 60 without restarting: tick 1 appears once and the output is contiguous after the resume point. Then `input` is approved on B.
   - The build ends `SUCCESS` with the same number. The log contains the pre-kill output, "Resuming build", and the rest.
   - After A restarts fresh, both controllers list the build with the same result and console.
2. **Kill while waiting at `input`:** B adopts, the input is approved on B, and the build ends `SUCCESS`.
3. **Controller-only Pipeline** (`sleep 60`, no agent): B adopts and the build completes on time.
4. **Partition:**
   - `docker pause` A during a build, until its lease expires and B adopts.
   - When A is unpaused, it reports "ownership lost" and does not publish or append.
   - The build completes once, on B. The store has exactly one record, B's, and no appends from A after the fence.
5. **No false adoption:** a long build on a healthy A is never adopted (with B up throughout).
6. **Not resumable:** a `PERFORMANCE_OPTIMIZED` build on a killed A is not adopted. Its placeholder is removed, and B does not show it as running forever.
7. **Cost:** report shipping per save point (bytes, latency) for a 200-step Pipeline, and confirm the build is not slower by more than X% (X to be measured).

## Risks
- **At-least-once steps** after resume from a save point up to 1 s old. Non-idempotent controller-side steps (e.g. `httpRequest` POST) may repeat. This is the same as Pipeline after a crash today, now with a bound.
- **Agent pinned to one controller** while both are up (see Agents).
- **`program.dat` size** for deep `parallel` builds. Each save point uploads it in full. If ET4 shows large ones, add delta or compression.
- **Plugins holding running state outside `program.dat`** (`lock`, `milestone`, the `build` step, as in R4). Not covered; record what ET4 observes.
- **Old workflow-durable-task-step checkout locally (2021).** Work on M1 needs the release source of workflow-cps, not the local branch.

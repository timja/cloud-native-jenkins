# E4: Cold failover of running Pipelines to another controller

- **Script:** `validation/experiments/E4-pipeline-failover.sh`
- **Raw evidence:** `raw/E4/`

## Method
1. A runs on the shared home with an inbound WebSocket agent. The agent addresses the controller via a DNS alias `jenkins`, standing in for a k8s Service.
2. Two builds are in flight:
   - `sleep #1` is running `sh` on the agent (tick 20 of 120)
   - `input #1` is paused on `input`
3. A is **SIGKILLed** (not a graceful shutdown).
4. B starts on the same home and the `jenkins` alias moves to B. The agent process keeps running and reconnects.

## Result: both builds completed on B
- B served requests 8s after A was killed.
- The agent reconnected with the same secret. It is an HMAC derived from the shared `secrets/master.key`.
- `sleep #1`: "Resuming build … after Jenkins restart" → "Waiting for reconnection of agent1" → "Ready to run". The `durable-task` process on the agent had kept running, and B followed it to `tick 120`, `Finished: SUCCESS`.
- `input #1` was approved on B, ran its post-input `node` block and finished `SUCCESS`.

The first attempt failed with `missing workspace`, but that was the harness's fault: the agent *container* was recreated, which destroyed the workspace and the running process. With a long-lived agent, resume works.

## Implications
- **This baseline has to survive externalisation.** It worked because B could read the dead controller's state:
  - `program.dat`
  - `workflow/*.xml` flow nodes
  - `build.xml`, `log`, `log-index`
  - `FlowExecutionList.xml`
  - `secrets/` (`SecretPickle`, the agent HMAC)

  All of it comes from the shared disk, so R4 has to provide every one of these from the store.
- Cold failover needs no code changes **when the filesystem is shared**. Without one, nothing in this list exists on B.
- What is still missing for *safe* failover:
  - An ownership lease. Nothing stopped A and B from both running these builds had A only been partitioned rather than killed.
  - Fencing on `program.dat` writes.
  - Agent routing that doesn't depend on moving a DNS alias.

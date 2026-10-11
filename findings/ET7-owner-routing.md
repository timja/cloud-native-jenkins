# ET7: Target 7 acceptance (requests reach the owner of a build or an agent)

**Result: PASS**, 35/35 checks, in two consecutive runs on the final build. Raw evidence from the last run: `raw/ET7/` (`results.md`, controller and agent logs, `BUILD`).

- **Script:** `validation/experiments/ET7-owner-routing.sh`
- **Setup:**
  - two tmpfs controllers, S0, Postgres, Azurite, API-token auth
  - `agent1` through the failover URL, connected to A
  - the round-robin load balancer `rr`
  - builds start on A, and every request about them goes to B (or through `rr`)
- **Build:**
  - core `cnj/t6-m6-stateless-sessions` (`ab593f3b4e`, including the queue fix below)
  - plugin on `cnj/t7-m1-owner-routing`
  - other plugins as ET6

| Criterion | Result |
|---|---|
| 1 Live console | ✅ B's `progressiveText`, polled from the last offset every second, never fell more than 2 lines behind A's console (10/10 polls), and had every line in order |
| 2 Build page | ✅ `/job/tick/1/` and its REST API answer on B (`200`, building), forwarded to A (`X-CNJ-Forwarded-From: jenkins-a`), and 4/4 through `rr`. The Pipeline graph view is not installed in the harness, so it is not checked |
| 3 `input` | ✅ approved through the browser form (form-encoded, with a cluster crumb, through `rr`): the build continued on A with the submitted value. Another `input` was aborted from B over REST |
| 4 Stop | ✅ `/job/long/1/stop` on B aborted the build on A |
| 5 Replay | ✅ replaying from B a build running on A started a new build with the edited script |
| 6 Freestyle console | ✅ a freestyle build's live console on A is followed from B, and grows |
| 7 Agent | ✅ on B, `/computer/agent1/` says online (answered by A); its log and script console work from B, and the channel table names A. After A was SIGKILLed, agent1 reconnected to B (38 and 43 s in the two runs) and the table names B |
| 8 `input` after failover | ✅ an `input` pending on A when A was killed: B adopted the build, the `input` was approved on B, and the build completed with the step after it |
| 9 Webhook fan-in | ✅ `git/notifyCommit` sent to A, to B, and to both at once each started exactly one build for the commit. A multibranch push event sent to B started exactly one branch build |
| 10 No loops, no leaks | ✅ requests for a completed build were not forwarded (the counter was unchanged). A request already forwarded was not forwarded again (B answered it itself). A dead owner's build was answered locally, never `502`. No routing warnings in the logs |

## How it works
- **`OwnerRoutingFilter` (store plugin):** a URL for a build this controller does not have, but which a running placeholder places on another controller, is forwarded to that controller's internal URL (from the membership table). So is a URL for an agent whose channel another live controller holds (the new `jenkins_channels` table, kept by a `ComputerListener`).
- **What is forwarded:** method, headers, cookies and body. A form body already parsed by the crumb filter is rebuilt from the parameters. The answer is streamed back, flushed as it arrives.
- **When it is not forwarded:** when the owner is not alive, or the request was forwarded once already, it is answered locally.
- **Shared with T6:** `Forwarding` is now shared with the bound-object forwarding.

## Bugs found by ET7, now fixed
- **A webhook delivered to both controllers built twice** (run 1, scenario 9; a Target 5 race in core).
  - **Cause:** when the same job was scheduled on two controllers at once, the second to publish folded its local item into the first's and dropped it on another thread. That thread could run before the scheduling thread had added the item, find nothing to drop, and leave an unshared item that built a second time.
  - **Fix (core, on `cnj/t6-m6-stateless-sessions`):** the queue gate refuses to start an item folded elsewhere, and the drop waits for the item to appear. `SharedQueueTest` has a new test racing the two. ET5 still passes (35/35).
- **The routing filter was sometimes not installed** (runs 2-4, scenario 2: a `404` on B).
  - **Cause:** it was installed after `PLUGINS_STARTED` only if the store existed, but the store exists from `EXTENSIONS_AUGMENTED`, and initialisers after `PLUGINS_STARTED` may run before that. Run 1 passed by ordering luck.
  - **Fix:** always installed; each request checks for the store. The import and the peer-version check of Target 10 had the same latent problem.
- **Harness:** an `input` with parameters logs "Input requested", not its message; the agent's log page loads its text progressively (the check now reads `logText`).

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | 32/35 | the double build from a webhook to both (fixed); 2 harness checks |
| 2-4 | stopped at 2 | the routing filter not installed (initialisation order; fixed) |
| 5-6 | 35/35 ×2 | final build |

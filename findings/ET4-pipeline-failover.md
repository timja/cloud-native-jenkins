# ET4: Target 4 acceptance (running Pipeline builds survive their controller)

**Result: PASS**, 39/39 checks, 0 failures, on the first run. Raw evidence: `raw/ET4/`
- `results.md`
- controller and agent logs
- `BUILD`

ET3 was re-run on the same build: 48/48 pass (`raw/ET3/`).

- **Script:** `validation/experiments/ET4-pipeline-failover.sh`
- **Setup:**
  - two tmpfs controllers, S0, Postgres, Azurite, API-token auth
  - `agent1`, an inbound WebSocket agent, reaching the controllers through an nginx URL that fails over (A primary, B backup)
  - `MAX_SURVIVABILITY` by JCasC
- **Build:**
  - core `cnj/t4-m2-build-leases`
  - workflow-cps `cnj/t4-m1-program-saved-listener`
  - cloudbees-folder `cnj/t1-m2-folder-children`
  - plugin on `cnj/t4-m3-pipeline-failover`

| Scenario | Result |
|---|---|
| 3 controller-only Pipeline (30 × `echo`/`sleep 2`), SIGKILL of A | ✅ B adopted it **26 s** after the kill and logged "Resuming build". Completed `SUCCESS` with every step logged once and none repeated. A fresh A shows the same result and console |
| 1 `sh` on agent1 (60 ticks), SIGKILL of A at tick 20, then `input` | ✅ B adopted it after **33 s**, and agent1 reconnected to B through the failover URL. The `sh` step re-attached to its still-running process and continued to tick 60: every tick present, tick 1 once, so it was not restarted. `input` was approved on B, the post-input agent step ran, `SUCCESS`. The log has the pre-kill output, the resumption and the rest, in order. A fresh A shows the same |
| 2 waiting at `input` on B, SIGKILL of B | ✅ A adopted it after **49 s** and had the pending input. Approved on A, the agent step after `input` ran, `SUCCESS` |
| 4 partition: `docker pause` of A during a build, then unpause | ✅ B adopted it **31 s** after the pause. Once unpaused, A logged "Ownership of pt #1 was lost" and stopped. The build completed once, on B. Only B published the record. A lists B's build, with its own copy discarded, and its console is B's: no appends from A after the fence |
| 5 a 30 s build on a healthy A | ✅ never adopted, and never shown on B as anything but "running on jenkins-a" |
| 6 `PERFORMANCE_OPTIMIZED` build, SIGKILL of A | ✅ not adopted (no save point). B removed its placeholder and does not run it |
| 7 cost | ℹ️ 200 `echo` steps: 849 ms with save points, 253 ms with `PERFORMANCE_OPTIMIZED`. Most of the difference is `MAX_SURVIVABILITY`'s own local writes. Shipping coalesced the save points into **2** placeholder commits for the whole build. No save points are left in blob storage after completion |

## Observations
- **Adoption latency is 26-49 s** after a crash or pause. It is the sum of three things: the 20 s membership timeout, then the stale cleanup period (30 s, so up to 30 s more), then resume. Both settings can be tuned (`…memberTimeoutMillis`, `jenkins.persistence.RunningBuilds.cleanupPeriod`). Sub-10 s failover would need a shorter timeout and the cleanup driven by membership events rather than polling.
- **No step repeated in these runs**, even though save points ship asynchronously (up to 1 s behind). `sleep`, `sh` and `input` are durable steps that resume rather than re-run. A step that completes between the last shipped save point and a crash would run again. That was not observed here, but it is possible by design.
- **Agents follow the controller through the failover URL** (remoting's 10 s reconnect loop). The S0 seed now includes the JNLP HMAC key, so the agent's secret is valid on every controller.
- **Fencing works at two levels.** The partitioned owner learns it lost the build from the database, and conditional appends keep its log out of the shared blob. Its local copy is discarded, and it picks up the record the new owner publishes.
- **Harness note.** After scenario 3 the agent stayed connected to B, the survivor, which is correct. The script restarts the agent container so that scenario 1 starts with the agent on A.

## Follow-ups
- Event-driven adoption, where a member-gone event triggers it immediately, for faster failover.
- Moving a running build to the controller holding its agent, which is now possible with the adoption machinery. This is the R3 placement item; Target 5 covers queue-time placement.
- `lock`, `milestone` and the `build` step across controllers: Target 5.
- A step that completes inside the ≤1 s shipping window re-runs on adoption. If that matters, add an option to commit synchronously after non-idempotent steps.

## Target 15
| Scenario | Result |
|---|---|
| 2b (item 23) A later stage reads what an earlier stage left in the workspace (inbound agent), the owner killed between them | ✅ the other controller adopts the build and the agent reconnects to it; the read stage finds the file and prints it after the resumption |
| 6 (item 12) A `PERFORMANCE_OPTIMIZED` build closed as aborted | ✅ B does not run it again, and its console keeps what it printed on A (what A had shipped before it died), then why it ended |

**Harness fixes:** the order check of 2b searched for the file's content from the start of the console, where the write stage's `echo` already printed it; the check of 6 expected a line printed on A just before the kill, which A had not necessarily shipped yet.

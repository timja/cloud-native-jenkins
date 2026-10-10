# ET5: Target 5 acceptance (a cluster-wide build queue and cross-build coordination)

**Result: PASS**, 35/35 checks, in two consecutive runs (runs 6 and 7) on the final build. Raw evidence from run 7: `raw/ET5/`
- `results.md`
- `queue.txt`: every shared queue record and its state
- `res-events.txt`: the store's record of who held the lock
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET5-shared-queue.sh`
- **Setup:**
  - two tmpfs controllers, S0, Postgres, Azurite, API-token auth
  - `agent1` through the failover URL
  - lockable-resources with cluster locks and the resource `res` declared on both controllers
- **Build:**
  - core `cnj/t5-m3-cluster-periodic-work`
  - lockable-resources `cnj/t5-m4-cluster-locks`
  - workflow-cps `cnj/t4-m1-program-saved-listener`
  - cloudbees-folder `cnj/t1-m2-folder-children`
  - plugin on `cnj/t5-m5-shared-queue`

| Scenario | Result |
|---|---|
| 1 "Build Now" on A, then on B 5 s later, with a 15 s quiet period | ✅ B's request folded into A's queue item: the same id (1) on both, in both queues. Exactly one build, with both requests as causes |
| 2 40 parameterized builds submitted alternately to A and B | ✅ numbered 1..40, each request ran exactly once, spread over both controllers (20/20 and 19/21 in two runs). Queue ids are unique cluster-wide; `queue/item/<id>` resolves to the same build on both |
| 3 a non-concurrent job building on A, "Build Now" on B | ✅ B's request waited (exclusive job lease), and the two builds did not overlap |
| 4 an item in its quiet period created on A, cancelled on B | ✅ `204` on B; gone from A's queue and reported cancelled there |
| 5 a build running on A stopped from B | ✅ aborted on A: "Stopped from jenkins-b by admin" |
| 6 three items waiting on A, SIGKILL of A | ✅ all three ran on B |
| 7 a job tied to `agent1`, scheduled on B while the agent is connected to A | ✅ ran on A, the only controller with an executor that can take it |
| 8 `milestone`: #1 on A waits before milestone 1, #2 on B passes it first | ✅ #1 cancelled on A (stop request) and never passed the milestone |
| 9 `lock('res')` from A and B at once | ✅ both completed. The store's events show the holds strictly alternating. The waiter on the other controller got `res` in the same instant it was released (0.0 s, against 13 s before the fix below) |
| 10 freestyle "Build other projects" and `build(wait: true)` | ✅ an upstream build on either controller triggered exactly one downstream build; `build(wait: true)` from A completed, with the callee pinned to A |
| 11 cluster-singleton periodic work | ✅ at most one controller is allowed to run `BackgroundGlobalBuildDiscarder` per period, with exactly one claim recorded; per-controller work (`LoadStatisticsUpdater`) is not coordinated |

## Bugs found by ET5 and the core tests, now fixed
- **Duplicate build from a stale mirror** (M2 tests).
  - **Cause:** a controller's own WAITING event could arrive after its item had already started, and the mirror recreated it.
  - **Fix (M1):** a controller never recreates its own items, and the record is re-read before mirroring.
- **Downstream builds never triggered** (ET5 run 1, scenario 10).
  - **Cause:** a controller that loaded a job created elsewhere did not rebuild its dependency graph. This bug dates from Target 1.
  - **Fix (M2):** rebuild the graph after applying any item change, with a test.
- **Folded cause lost** (ET5 run 2, scenario 1). This failed only when the controller that created the item also won the claim.
  - **Cause:** it ignored changes to its own items arriving from the store.
  - **Fix (M1):** apply them while the item is still queued locally, with a test.
- **Pipeline lock waiters woke late** (ET5 run 3, seen as a 13 s gap in the store's lock events).
  - **Cause:** on a remote release, `ClusterLocks` only woke Jenkins' build queue, which serves freestyle builds. Pipeline `lock` steps wait in the plugin's own queue, so they waited for the plugin's periodic safety net.
  - **Fix (M4):** `refreshQueue()`.
- **Harness:**
  - counted the shell's `-x` trace lines
  - called a static interface method from the script console (`VerifyError`)
  - scenario 11 assumed no automatic run had claimed the period yet
  - scenario 9 measured overlap from the wrong pair of timestamps; it now uses the store's lock events
  - the restarted A lost the lock resource declared through the script console; it is now declared again after the restart

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | 28/34 | the dependency graph was not rebuilt for jobs changed elsewhere (fixed); 5 harness issues |
| 2 | 33/34 | a request folded in elsewhere lost its cause when the item's creator also ran it (fixed) |
| 3 | 34/34 | clean |
| 4 | 33/34 | scenario 11's check was wrong (the period had already been claimed) |
| 5 | 34/35 | scenario 9 measurement bug, which uncovered the lock wake-up delay (fixed) |
| 6, 7 | **35/35, 35/35** | final build |

## Regression: earlier acceptance experiments on the Target 5 build
The shared queue changes how every build starts, so ET3 and ET4 were re-run on this build.
- **ET3: 48/48.**
- **ET4: 39/39** on the final build, after fixing what its earlier runs found:
  - **The controller a build was requested on now goes first** (found by ET4 scenario 1).
    - **Problem:** with first come, first served, a Pipeline triggered on A could start on B and then wait for an agent connected to A. A Pipeline's `node` labels are only known when it runs.
    - **Fix (core M1):** other controllers only claim an item 10 s after it is due, or at once if its controller is gone (`jenkins.persistence.SharedQueue.originGraceMillis`).
  - **Concurrent schema creation** (a controller failed to start).
    - **Problem:** A and B ran `create table if not exists` at the same moment, which Postgres does not make safe. This race dates from Target 1; the queue's extra tables made it likelier.
    - **Fix (plugin):** a transaction-scoped advisory lock serializes it, with a test that opens six stores at once.
  - **A woken zombie could append to the log** (ET4 scenario 4).
    - **Problem:** after `docker pause`, A's log shipper ran before its heartbeat noticed the gap. Its stray bytes made B's next conditional append fail, so B stopped shipping.
    - **Fix (plugin):** no log or save point uploads while this controller's heartbeat may have lapsed. As a backstop, an owner that still meets a conflict rewrites the log from its local copy.
  - **Harness:** the log-order check required the exact kill tick to be shipped before the resume; it now requires earlier output.

## Observations
- **Placement goes to the requesting controller first, then whoever can run it.** "Build Now" on A runs on A if A can take it, and load spreads because requests arrive at both controllers (scenario 2). A job tied to an agent only starts where the agent is connected, because no other controller has an executor for it.
- **Placement of Pipeline jobs.** A Pipeline is placed before its `node` labels are known. The requesting controller going first makes the common case right, but a Pipeline requested on a controller without its agent still waits there (E5, R3).
- **`build(wait: true)` is pinned** to the upstream's controller, because its step context lives there. Remote step completion would remove the pin.
- **Lock fairness across controllers is approximate.** Each controller has its own waiters and retries when a release event arrives.

## Follow-ups
- Pipeline placement aware of `node` labels, or moving a running build to its agent's controller (T4 adoption makes this possible).
- Remote completion of `build` steps.
- Event-driven reclaim of claimed items, instead of the 15 s sweep.
- The lockable-resources UI paths (reserve, steal, unlock from the page) across controllers.

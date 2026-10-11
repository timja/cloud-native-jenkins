# Target 7: Requests reach the owner (R3 routing, agent channels, webhooks)

| | |
|---|---|
| **Roadmap tier** | R3 (routing to the owner, agent connections), R4 (`input` approvals), R2 (webhook fan-in) |
| **Status** | **Done** (2026-10-11): ET7 passes 35/35, see [07-progress](07-progress.md) |
| **Depends on** | T3 (running placeholders), T4 (leases, adoption), T5 (shared queue, stop requests), T6 (membership with internal URLs, request forwarding, stateless sessions) |
| **Evidence** | [E4](../findings/E4-pipeline-failover.md) (agents need a moved DNS alias), [E5](../findings/E5-stateless-http.md) (an agent connects to one controller), ET3–ET6 |

## Goal
A user or tool can use a build or agent from whichever controller the load balancer picks:
- follow the console of a build running elsewhere, live
- approve or abort its `input` step, stop it, replay it
- open an agent's page, log and script console wherever the agent is connected

A webhook delivered to any controller starts exactly one build, and an `input` still waiting when its owner dies can be approved after adoption.

## Today
- On B, a build running on A appears only as a placeholder (`RunningElsewhereAction` on the job). `/job/p/<N>/` and everything below it give `404` on B until the build completes.
- Stop works through T5 stop requests, but only from that placeholder action, not from the build's own URLs.
- An agent connected to A is an offline computer on B. Its page on B shows it as never connected.

## Design
### 1. Owner routing (store plugin)
- A servlet filter, generalising T6's `BoundRequestForwarder`, which forwards a request when its URL is for a build running on another controller.
- **Recognising such a URL:**
  - It has the form `/job/a/job/b/<N>/…`, or the same with `/view/x/` prefixes.
  - The job exists locally but has no build `N`.
  - `RunningBuilds.elsewhere(job)` has a placeholder for `N`. The placeholder names the owner, and the membership table gives the owner's internal URL.
- **Forwarding:**
  - Method, headers, body and cookies are forwarded, so the owner authenticates the user from the T6 login cookie, crumb or API token.
  - Responses are streamed back. That covers `progressiveText`/`progressiveHtml` console follow, `input` forms, `stop`, `replay` and `pipeline-graph`.
- A request already forwarded once is never forwarded again (the `X-CNJ-Forwarded-From` header).
- **If the owner is unreachable,** the request is not forwarded, so the user gets B's own page. After adoption that is the real build.

### 2. Agent channels (store plugin)
- A `ComputerListener` records which controller holds each agent's channel in a `jenkins_channels` table: agent, controller, connected-since. The row is deleted on disconnect, and expires with its controller's membership.
- Requests for `/computer/<name>/…` on a controller without the channel are forwarded to the holder:
  - the agent page
  - its log
  - the script console
  - `systemInfo`
  - disconnect and launch actions
- The node configuration is static state (T6), so `/computer/<name>/configure` is not forwarded.
- **Reconnection after failover** is the agent's own retry through the failover URL (T4). T7 measures the gap and checks the channel row moves.

### 3. `input` after adoption
- The pending `InputAction` is in the build's actions. Save points and the snapshot carry it (T3/T4), so the adopter has it.
- T7 verifies an `input` pending on A at the time A dies can be approved on B afterwards. Any fix needed goes in the adopter.

### 4. Webhook fan-in
- `git/notifyCommit` and SCM events arrive at one controller through the load balancer.
- Polling triggered on that controller schedules builds into the shared queue (T5), which folds duplicates. Multibranch events go to the indexing coordinator (T2).
- T7 verifies the paths end to end: a hook to either controller, or to both (redelivery), starts one build.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | cloud-native-jenkins | `cnj/t7-m1-owner-routing` | `cnj/t6-m7-static-state` | owner routing for builds running elsewhere; agent channel table and `/computer` routing; tests |
| M2 | cloud-native-jenkins | `cnj/t7-m2-et7` | M1 | harness, `ET7-owner-routing.sh`, and any adopter fixes found |
| (core) | core | `cnj/t7-*` | `cnj/t6-m6-stateless-sessions` | only if needed (e.g. a hook for the routing check, adopter fixes) |

## E-T7 acceptance experiment (definition of done)
ET setup as ET6 (round-robin LB, stateless sessions), plus `agent1` connected to A through the failover URL. Every request goes to B or through the LB.
1. **Live console:** for a Pipeline running on A, following the console from B (`progressiveText` with `start` offsets) sees new lines within 2 s of A writing them.
2. **Build page:** `/job/p/<N>/` of a build running on A answers `200` on B and shows it running. So do its REST API and the Pipeline graph.
3. **`input`:**
   - A build on A waiting at `input` is approved from B (browser form, through the LB) and continues on A.
   - Another is aborted from B (REST).
4. **Stop:** `/job/p/<N>/stop` on B stops a build running on A.
5. **Replay:** replaying, from B, a build running on A starts a new build with the edited script.
6. **Freestyle console:** the live console of a freestyle build running on A, followed from B.
7. **Agent:** on B, `/computer/agent1/` shows the agent online (answered by A), and its log and script console work. After A dies, the agent reconnects to B within the failover window and the channel row moves to B.
8. **`input` after failover:** an `input` pending on A when A is SIGKILLed is approved on B after adoption, and the build completes.
9. **Webhook fan-in:**
   - `git/notifyCommit` sent to A, to B, and to both at once, for a job polling a git repo: exactly one build per commit each time.
   - A multibranch push event to either controller: exactly one branch build.
10. **No loops, no leaks:**
    - Forwarded requests are never forwarded twice.
    - Requests for local builds are not forwarded.
    - A request for a build of a dead owner is answered locally, not with `502`.

## Decisions taken without asking (the roadmap is to be completed without stopping)
- **Forward instead of reading remotely.** Serving console or `input` from the store on B would need every plugin's UI to be store-aware. Forwarding reuses the owner's own UI.
- **Agent channels stay where they connect.** Moving a channel between controllers, or multiplexing it, is out of scope. Requests and builds go to the holder (T5 placement already does this for builds).

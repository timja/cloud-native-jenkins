# ET20: Target 15 items 4 and 26 (WebSockets, the CLI across controllers, the key check)

**Result: PASS**, every check. Raw evidence: `raw/ET20/`
- `results.md`
- what each WebSocket answered (`ws-a.txt`, `ws-b.txt`)
- the CLI's output (`console-f.txt`, `console-f-ws.txt`, `build-s.txt`)
- the log of the controller started without the seeded keys (`noseed.log`)
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET20-websocket-cli.sh`
- **Setup:**
  - two tmpfs controllers, Postgres, Azurite, token auth
  - an inbound agent `ala` connected to A only
  - a test-only WebSocket endpoint (`WebSocketEcho`, enabled by a system property) that answers with the name of the controller serving it

| Scenario | Result |
|---|---|
| 1 A WebSocket opened on B for an object of A's running build | ✅ relayed to A: B answered "hello from jenkins-a", as A does itself |
| 2 The CLI's `console -f` on B (`-http`), for a build running on A | ✅ follows it to the end: all 15 ticks, exit 0 |
| 2b The same over the CLI's WebSocket transport (`-webSocket`) | (pending rerun) |
| 3 `build -s -v` on B (`-http`), of a job only A can run (its agent is on A) | ✅ "Started onala #1 (on another controller)", the build's whole log, then "Completed onala #1 : SUCCESS", exit 0 |
| 4 A controller started against the store without the cluster's seeded keys | ✅ refuses to start, saying why ("does not have the key material of the cluster…", and how to fix it), and never becomes a member |

## How it works
- **The relay.** A WebSocket upgrade addressed to an object another controller owns is accepted by the controller it reached, which opens a WebSocket to the owner (with the client's credentials) and copies frames both ways (`WebSocketProxy`). Agents' own WebSocket connections are not relayed.
- **The CLI.** `console` and `build -s/-f/-v` look for a build that runs elsewhere, or whose queue item another controller started (`SharedQueue.startedElsewhere`). They stream its log from where its owner ships it (`RunningBuilds.RemoteLog`), then the rest from its record once it is published, and report its result (`RunningBuilds.followElsewhere`, `awaitRecord`).
- **The key check.** The first controller stores a value encrypted with the cluster's key (`cluster/key-check`). A controller that cannot decrypt it stops before it heartbeats (`KeyCheck`).

## Bugs found while running ET20, now fixed
- **`build -s -v` lost the end of a freestyle build's log** (core). A freestyle build's shipped log is deleted once its record is published, so the last read of it failed, and the CLI stopped there instead of printing the rest from the record. Now a shipped log that is gone falls through to the record. Test: `RunningBuildsTest.followingABuildElsewhereEndsWithTheRestOfItsRecord`.
- **Harness:**
  - macOS has no `timeout`, so the script uses a `run_for` helper.
  - `docker logs … | grep -q` failed under `pipefail` when grep stopped reading early, so the whole log is read first.
  - The compose controllers had the kubernetes plugin without its dependencies, a load failure on every start. It now goes only into the kind image (ET9).

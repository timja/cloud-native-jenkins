# ET14: Target 14 M1 acceptance (surviving outages of the persistence store and the blob store)

**Result: PASS**, every check, in two consecutive runs on the final build (and again in the final regression sweep). Raw evidence: `raw/ET14/`
- `results.md`
- the pages answered during the outage
- the consoles
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET14-store-outage.sh`
- **Setup:** two tmpfs controllers, Postgres, Azurite, token auth.

| Scenario | Result |
|---|---|
| 1 Postgres paused for 10 s while a Pipeline (on A) and a freestyle build (on B) run | ✅ both completed SUCCESS. Both consoles are whole (every line once, in order, and the end). Each record was published by one controller, and there is no second build |
| 2 Postgres paused for 60 s | ✅ the same. During the outage every page (`/`, `/api/json`, a job page, `/login`) answered 200 on both controllers, the slowest in under 6 s. A configuration change made after the outage reached the other controller |
| 3 Postgres restarted | ✅ the same |
| 4 A build requested while Postgres is paused | ✅ it ran once the store was back, exactly once |
| 5 Azurite stopped for 60 s | ✅ the same as 1: logs and save points were shipped once it was back |
| Throughout | ✅ no build was aborted, adopted or closed, and no queue item was put back for a controller wrongly taken for gone |
| 6 A restarted while Postgres is down | ✅ A does not serve, and logs that it waits for the store. It comes up about 20 s after the store is back. Exactly two live members (no ghost from the attempts), and the whole history is there |

## What was wrong before (found by research, then shown here)
- **An outage longer than the member timeout (20 s) aborted every running build in the cluster.**
  - **Cause:** a controller whose heartbeats were lapsing treated all its leases as lost without asking the database, stopped its builds and discarded them.
  - **And on recovery:** every heartbeat was stale at once, so whichever controller looked first adopted the others' Pipelines, closed their freestyle builds as aborted, and put their claimed queue items back (duplicates).
- **One synchronized JDBC connection with no timeouts.** A database that silently drops packets would hang the controller for about 15 minutes, holding the Queue and Jenkins locks. A slow one starved the heartbeat.
- **A completed build whose publication failed vanished from the cluster**, and the tail of a Pipeline log not shipped at the end was lost.
- **Change events could be skipped** (sequence numbers are taken at insert, not at commit).
- **Startup with the store unreachable failed**, and a blob-store failure left a "ghost" member heartbeating.

## How it works now
- **Connections.** Two HikariCP pools:
  - work
  - heartbeats, leases and membership

  Every connection has socket, connect, statement and lock timeouts and TCP keepalive. A call is retried on another connection only if it broke before the commit was sent.
- **Fail fast.** While the database is unreachable, ordinary calls fail at once (for 5 s at a time), and the heartbeat closes the breaker. Pages render without the store's data. A key never used before, needed meanwhile, is temporary and never written.
- **Leases are fenced only on evidence from the database.** Unknown is not lost: the build carries on, while its log and save points wait.
- **No takeovers after an outage.** A controller judges others gone (adopting, reclaiming queue items, taking leases) only after its own heartbeats have been steady for longer than the member timeout.
- **Recovery:**
  - Publication of completed builds, registration of started builds, failed applies and queue-start records are all retried.
  - A dead controller's placeholder is kept until its aborted record is written.
  - The listener re-reads gaps in event numbers for 60 s, delivering late-committed events.
  - A log's tail shipped late republishes the build, so other controllers fetch the whole log.
- **Startup.** Both stores are waited for, with backoff, for up to 10 minutes. Postgres is closed again if the blob store cannot be created. Every key the cluster shares is copied locally at startup.
- **Azure client:** bounded tries, and response timeouts.

## Bugs found while running ET14, now fixed
- Every page hung while the database was paused: a status icon per job waited 10 s for a pool connection, then retried. Fixed by failing fast.
- Pages answered 500 during the outage: an HMAC key never used on that controller was read from the store. Fixed by local copies at startup, plus a temporary key when the store is unreachable.
- **Harness:**
  - `docker compose up` restarted Postgres with the controller (`--no-deps`)
  - `grep -q` after `docker logs` fails under `pipefail`
  - a `curl` timeout under `set -e`
  - a check that did not wait for the retried publication

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | stopped at 4 | pages hanging during the outage (fail fast); `set -e` |
| 2 | 2 failures | 500s from the key read; boot check harness |
| 3 | 2 failures | the key never used anywhere (temporary key); `grep -q` under `pipefail` |
| 4-5 | all ✅ ×2 | final build |

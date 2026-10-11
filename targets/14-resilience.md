# Target 14: Resilience, limits and scale

| | |
|---|---|
| **Roadmap tier** | X (cross-cutting), R3, R4, R5 |
| **Status** | **Done** (2026-10-11): ET14, ET15 and ET16 pass, and ET1–ET13 pass on the same build, see [14-progress](14-progress.md) |
| **Depends on** | Targets 1–13 |

## Goal
Close the known edge cases left after Targets 1–13, in priority order (agreed with the user; upstreaming is excluded):

| # | Item | Decision |
|---|---|---|
| M1 | **Store outages:** Postgres or the blob store down, paused or restarted | Degrade and keep builds running: nothing is aborted, adopted or run twice. Pooled connections (HikariCP) |
| M2 | **Long logs:** Azure append blobs take at most 50,000 blocks | Logs in segments; coalesced shipping |
| M3 | **Controllers occasionally not coming up** in regression runs | Find the cause; add diagnostics |
| M4 | **REST API for builds running on another controller** | Full parity: `builds`, `allBuilds` and `lastBuild` |
| M5 | **Faster failover** (it was up to ~50 s) | About 15 s |
| M6 | **Scale:** thousands of jobs and builds on a laptop | Measure, and fix what does not hold |

## Design
- **M1, outages:**
  - two connection pools (work, and control: heartbeats, leases and membership), each with timeouts
  - fail fast while the store is unreachable
  - leases fenced only on evidence from the database
  - no takeovers until a controller's own heartbeats have been steady for longer than the member timeout
  - publication, registration, applies and queue starts retried
  - late-committed events re-read
  - startup waits for both stores
  - shared keys copied locally at startup
- **M2, logs:** `log`, `log.1`, … of at most 49,000 blocks each. Shipping is coalesced (3 s or 32 KiB), and moves keep append blobs appendable.
- **M3, boot:**
  - reloading a user changed elsewhere happens under its lock, and not while a save is pending
  - harness diagnostics: thread dumps and Postgres activity and locks on a timeout
- **M4, REST:** builds elsewhere are exported as stand-ins with a Run's properties. `/lastBuild` goes to the newer build elsewhere, and `isBuilding()` counts builds elsewhere (renames are refused).
- **M5, failover:** membership is watched on the heartbeat thread. When a controller leaves, its leases are reclaimed and its builds adopted at once.
- **M6, scale:** prefix listings use an index (`text_pattern_ops` with an escaped `LIKE`), and events are pruned by an index.

## Acceptance experiments
- **ET14** (store outages): Postgres paused for 10 s and for 60 s, restarted, and paused while a build is requested; Azurite stopped for 60 s; a controller started while Postgres is down. Every build completes once, with its whole log, and pages answer throughout.
- **ET15** (logs, REST, failover):
  - a log rolled over into several segments, also across a failover
  - B's API shows A's running build
  - renames refused from B
  - adoption within 30 s, measured three times
- **ET16** (scale):
  - 2,000 jobs and 2,000 builds
  - warm pages under 5 s
  - a restart within 180 s
  - 50 running Pipelines adopted within 90 s

# Target 14: progress and branches

Design: [14-resilience.md](14-resilience.md). **Done means:** `ET14-store-outage.sh`, `ET15-logs-api-failover.sh` and `ET16-scale.sh` pass, and ET1–ET13 still pass.

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1, M3–M5 | core | `cnj/t14-m1-resilience` | `cnj/t12-m1-remainder` | Retried publication, registration, applies and queue starts, with no ghost store (27140111d2). A user reloaded under its lock (91f302e55e). Builds elsewhere in the REST API (8953b3e09b). `sweepSoon` (f5d4f9a5f7). Shared keys kept locally, and a temporary key while the store is unreachable (fe86b50059, 8ca603a041). `isBuilding` counts builds elsewhere (82d1bbce2b) | ✅ `RunningBuildsTest`, `BuildRecordsTest`, `ClusterCoordinationTest` green |
| M1–M6 | cloud-native-jenkins | `cnj/t14-m1-resilience` | `cnj/t13-m1-cluster-plugins` | HikariCP pools with timeouts; fail fast; lease fencing by the database; liveness judged only after steady heartbeats; late events; waiting for the stores at boot; Azure timeouts. Log segments with coalesced shipping. The `lastBuild` redirect. Membership watched on the heartbeat thread. Indexed prefix listings. Boot diagnostics in the harness. ET14, ET15, ET16 | ✅ ET14, ET15, ET16 pass. `PostgresPersistenceStoreTest` 18/18, `AzureBlobStoreTest` 6/6 |

## Results
- **ET14** ([findings](../findings/ET14-store-outage.md)):
  - Postgres paused for 10 s and for 60 s, and restarted; Azurite stopped for 60 s. Every build completed once, with its whole console. Pages answered 200 throughout, the slowest in under 6 s.
  - A controller started with the store down comes up about 20 s after the store is back.
- **ET15** ([findings](../findings/ET15-logs-api-failover.md)):
  - logs in 5–7 segments, whole and in order, also across a failover
  - REST parity for builds elsewhere
  - renames refused from the other controller
  - failover in 16–23 s (it was about 50 s)
- **ET16** ([findings](../findings/ET16-scale.md)):
  - 2,000 jobs seen on B 10 s after A started creating them
  - warm `/` in 0.4–0.7 s
  - a restart in 14 s
  - 50 running Pipelines adopted within 24 s
- **M3** ([note](../findings/T14-boot-hang.md)): not reproduced. One race was fixed, and diagnostics were added.
- **Final regression sweep**, on core 82d1bbce2b and plugin fd83053: ET1–ET16 all pass, ET15 twice (ET9, ET10 and ET13 on kind).

## Log
- 2026-10-11: Target 14 planned from the known edge cases, in the user's priority order (upstreaming excluded). The decisions were: degrade and keep builds, HikariCP, about 15 s failover, full REST parity.
  - ET14 found pages hanging during an outage (fixed by failing fast) and 500s from reading a key from the store (fixed by local copies and a temporary key).
  - ET15 found that a rename from the controller not running the build was accepted (fixed in core).
- 2026-10-11 (kind): the hand-testing release was reinstalled and reseeded after the sweep. On fresh pods, `installPlugins` answered 200 and did nothing: the chart had copied `UpdateCenter.never` and `DownloadService.never` from the offline harness, from before runtime installs existed, so no controller ever fetched the update site. Fixed in the chart (`updateSites.fetch`, default true): each controller fetches at startup and daily, as on one controller (`DailyCheck` is per controller, not a cluster singleton). Verified by a rolling upgrade of the release.

## Follow-ups
- Failover is 16–23 s, against the target of about 15 s. The floor is the member timeout (20 s); lowering it trades false deaths (GC pauses, slow disks) for speed.
- Scale beyond two controllers and 20k+ builds; storms of branch indexing.
- Update-site metadata could be shared through the store (one fetch per cluster), which matters only for rate-limited or proxied sites.
- CLI `install-plugin` from a file stays local (from Target 13).

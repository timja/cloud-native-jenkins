## ET14 results (2026-10-12T15:36Z)

| | Check |
|---|---|
| ✅ | 1: the Pipeline completed SUCCESS |
| ✅ | 1: the freestyle build completed SUCCESS |
| ✅ | 1: both consoles are whole (every tick once, and the end) |
| ✅ | 1: each build was published, by one controller only |
| ✅ | 1: and there is no second build |
| ℹ️ | 2 pages during the outage (port, page, status, seconds): 8081 / 200 0.594723;8082 / 200 5.321082;8081 /api/json 200 5.017015;8082 /api/json 200 0.027260;8081 /job/p2/ 200 0.157677;8082 /job/p2/ 200 0.105830;8081 /login 200 0.017468;8082 /login 200 0.013426; |
| ✅ | 2 every page answered within 10 s during the outage (degraded, not hung) |
| ✅ | 2: the Pipeline completed SUCCESS |
| ✅ | 2: the freestyle build completed SUCCESS |
| ✅ | 2: both consoles are whole (every tick once, and the end) |
| ✅ | 2: each build was published, by one controller only |
| ✅ | 2: and there is no second build |
| ✅ | 2 a configuration change works after the outage, and reaches the other controller |
| ✅ | 3: the Pipeline completed SUCCESS |
| ✅ | 3: the freestyle build completed SUCCESS |
| ✅ | 3: both consoles are whole (every tick once, and the end) |
| ✅ | 3: each build was published, by one controller only |
| ✅ | 3: and there is no second build |
| ℹ️ | 4 the build request during the outage: curl exit 28 |
| ✅ | 4 it ran once the store was back |
| ✅ | 4 and only once |
| ✅ | 5: the Pipeline completed SUCCESS |
| ✅ | 5: the freestyle build completed SUCCESS |
| ✅ | 5: both consoles are whole (every tick once, and the end) |
| ✅ | 5: each build was published, by one controller only |
| ✅ | 5: and there is no second build |
| ✅ | no build was aborted or taken over |
| ✅ | no queue item was put back for a controller wrongly taken for gone |
| ℹ️ | 6 A while the store is down: /login 000 |
| ✅ | 6 A is not serving while it waits for the store |
| ✅ | 6 A logs that it waits |
| ✅ | 6 A comes up once the store is back |
| ℹ️ | 6 A was up 18 s after the store came back |
| ✅ | 6 exactly two live members (no ghost from the attempts) |
| ✅ | 6 the history is all there on the restarted A |

Failures: 0

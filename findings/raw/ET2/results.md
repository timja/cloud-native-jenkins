## ET2 results (2026-10-11T16:45Z)

| | Check |
|---|---|
| ✅ | both controllers are live members |
| ✅ | 2 forty builds across A and B are numbered exactly 1..40 (got: 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 ) |
| ✅ | 2 no build number was used on both controllers |
| ℹ️ | 2 builds per controller: A=20 B=20 |
| ℹ️ | 1 cron1 builds: #1@B #2@B #3@B #4@B #5@B  |
| ✅ | 1 cron fired (≥3 builds in ~4.5 min: 5) |
| ✅ | 1 exactly one cron build per claimed tick across controllers |
| ℹ️ | 1: 5 builds, 5 claimed ticks; wall-clock minutes with 2 builds (cron catch-up): 1 |
| ✅ | 1 cron build numbers are unique across controllers |
| ✅ | 1 all cron builds ran on one controller (affinity) |
| ✅ | 3 first poll (no previous build) builds once |
| ✅ | 3 one commit → exactly one more build |
| ✅ | 3 no further builds without commits (still 2) |
| ✅ | 3 second commit → exactly one more build |
| ✅ | 3 polling never built the same change twice (3 builds total: 3) |
| ✅ | 3 polling stayed on one controller (affinity) |
| ✅ | 4 periodic indexing did not rebuild existing branches (main: 1, feature: 1) |
| ✅ | 4 a new branch is built by periodic indexing |
| ✅ | 4 the new branch was built exactly once cluster-wide (1) |
| ✅ | 4 existing branches still built once |
| ℹ️ | 4 periodic indexing owner: B |
| ✅ | 5 survivor took over cron within 2.5 min (2 builds after the kill) |
| ✅ | 5 still exactly one cron build per claimed tick |
| ℹ️ | 5: 18 builds, 18 claimed ticks; wall-clock minutes with 2 builds (cron catch-up): 1 |
| ✅ | 5 cron build numbers still unique |
| ✅ | 6 cron kept firing across the rejoin (3 builds in 3 min) |
| ✅ | 6 exactly one cron build per claimed tick across failover and rejoin |
| ℹ️ | 6: 21 builds, 21 claimed ticks; wall-clock minutes with 2 builds (cron catch-up): 1 |
| ✅ | 6 cron build numbers unique across all incarnations |
| ✅ | 7 fresh controller continues the sequence (#41, expected 41) |
| ✅ | 7 renamed job continues the sequence (#42, expected 42) |

Failures: 0

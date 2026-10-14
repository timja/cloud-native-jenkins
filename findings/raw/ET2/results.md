## ET2 results (2026-10-14T09:59Z)

| | Check |
|---|---|
| ✅ | both controllers are live members |
| ✅ | 2 forty builds across A and B are numbered exactly 1..40 (got: 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 ) |
| ✅ | 2 no build number was used on both controllers |
| ℹ️ | 2 builds per controller: A=20 B=20 |
| ℹ️ | 1 cron1 builds: #1@B #2@B #3@B #4@B #5@B #6@B  |
| ✅ | 1 cron fired (≥3 builds in ~4.5 min: 6) |
| ✅ | 1 exactly one cron build per claimed tick across controllers |
| ℹ️ | 1: 6 builds, 6 claimed ticks; wall-clock minutes with 2 builds (cron catch-up): 1 |
| ✅ | 1 cron build numbers are unique across controllers |
| ✅ | 1 all cron builds ran on one controller (affinity) |
| ✅ | 3 first poll (no previous build) builds once |
| ✅ | 3 one commit → exactly one more build |
| ✅ | 3 no further builds without commits (still 2) |
| ✅ | 3 second commit → exactly one more build |
| ✅ | 3 the polling log reads the same on both controllers (served by the one that polled) |
| ✅ | 3 polling never built the same change twice (3 builds total: 3) |
| ✅ | 3 polling stayed on one controller (affinity) |

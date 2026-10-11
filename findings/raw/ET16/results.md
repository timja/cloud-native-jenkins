## ET16 results (2026-10-11T17:47Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 8 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 11 s after A started creating |
| ℹ️ | 2 the builds took 97 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 8081 / 8.999764 1.673091;8082 / 6.260604 1.561495;8081 /api/json?tree=jobs[name,color] 0.154075 0.020090;8082 /api/json?tree=jobs[name,color] 0.237992 0.080130;8081 /job/job-3/ 1.219422 0.069177;8082 /job/job-3/ 0.331408 0.023025;8081 /job/job-3/api/json?depth=1 0.049657 0.009744;8082 /job/job-3/api/json?depth=1 0.050754 0.008182;8081 /manage/cluster/ 0.129192 0.022764;8082 /manage/cluster/ 0.083492 0.023513; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 178857K, B used 351311K |
| ℹ️ | 3 store: 4226 keys, 4363 kB, 20531 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.198 ms |
| ✅ | 4 B is up within 180 s |
| ℹ️ | 4 B was up after 20 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 B's / after the restart: cold 10.047254 s, warm 0.744081 s |
| ℹ️ | 5 B's / with 50 builds running on A: 1.372004 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 50 builds, the last 29 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

## ET16 results (2026-10-12T18:41Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 6 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 6 s after A started creating |
| ℹ️ | 2 the builds took 41 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 8081 / 4.311765 0.435642;8082 / 2.098513 0.392969;8081 /api/json?tree=jobs[name,color] 0.089231 0.016719;8082 /api/json?tree=jobs[name,color] 0.028907 0.012985;8081 /job/job-3/ 0.248554 0.013599;8082 /job/job-3/ 0.126561 0.014390;8081 /job/job-3/api/json?depth=1 0.021396 0.006189;8082 /job/job-3/api/json?depth=1 0.021882 0.005818;8081 /manage/cluster/ 0.076881 0.028739;8082 /manage/cluster/ 0.117483 0.020552; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 153711K, B used 268519K |
| ℹ️ | 3 store: 5730 keys, 4580 kB, 30638 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.101 ms |
| ℹ️ | 4 fresh-1: serving the API 11.3 s after the container started (container and JVM start to initialization 2.9 s, initialization 6.8 s); first / then 1.435921 s |
| ✅ | 4 fresh B #1 is up within 180 s |
| ℹ️ | 4 fresh-2: serving the API 13.0 s after the container started (container and JVM start to initialization 2.8 s, initialization 7.2 s); first / then 1.532045 s |
| ✅ | 4 fresh B #2 is up within 180 s |
| ℹ️ | 4 fresh-3: serving the API 10.3 s after the container started (container and JVM start to initialization 2.3 s, initialization 6.2 s); first / then 1.573340 s |
| ✅ | 4 fresh B #3 is up within 180 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 median time to serving: 11.3 s; warm / afterwards 0.589932 s |
| ℹ️ | 4 runtime-plugin: serving the API 10.8 s after the container started (container and JVM start to initialization 2.8 s, initialization 6.5 s); first / then 1.385285 s |
| ✅ | 4 fresh B with next-build-number installed at runtime is up within 180 s |
| ✅ | 4 ... and has next-build-number |
| ℹ️ | 5 B's / with 50 builds running on A: 0.748196 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 52 builds, the last 9 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

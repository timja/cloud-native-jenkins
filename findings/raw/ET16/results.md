## ET16 results (2026-10-13T18:34Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 5 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 5 s after A started creating |
| ℹ️ | 2 the builds took 43 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 8081 / 5.387513 0.641776;8082 / 2.123295 0.419704;8081 /api/json?tree=jobs[name,color] 0.073753 0.011976;8082 /api/json?tree=jobs[name,color] 0.035175 0.011197;8081 /job/job-3/ 0.243562 0.012162;8082 /job/job-3/ 0.195384 0.012377;8081 /job/job-3/api/json?depth=1 0.025595 0.009070;8082 /job/job-3/api/json?depth=1 0.032347 0.009454;8081 /manage/cluster/ 0.068459 0.026565;8082 /manage/cluster/ 0.057286 0.019481; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 157551K, B used 155978K |
| ℹ️ | 3 store: 5730 keys, 4581 kB, 32640 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.093 ms |
| ℹ️ | 4 fresh-1: serving the API 10.9 s after the container started (container and JVM start to initialization 2.6 s, initialization 6.3 s); first / then 1.610303 s |
| ✅ | 4 fresh B #1 is up within 180 s |
| ℹ️ | 4 fresh-2: serving the API 9.9 s after the container started (container and JVM start to initialization 2.2 s, initialization 6.1 s); first / then 1.355405 s |
| ✅ | 4 fresh B #2 is up within 180 s |
| ℹ️ | 4 fresh-3: serving the API 10.1 s after the container started (container and JVM start to initialization 2.2 s, initialization 6.5 s); first / then 1.431384 s |
| ✅ | 4 fresh B #3 is up within 180 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 median time to serving: 10.1 s; warm / afterwards 0.356263 s |
| ℹ️ | 4 runtime-plugin: serving the API 10.0 s after the container started (container and JVM start to initialization 2.2 s, initialization 6.4 s); first / then 1.444157 s |
| ✅ | 4 fresh B with next-build-number installed at runtime is up within 180 s |
| ✅ | 4 ... and has next-build-number |
| ℹ️ | 5 B's / with 50 builds running on A: 0.678503 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 50 builds, the last 9 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

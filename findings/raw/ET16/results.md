## ET16 results (2026-10-12T15:56Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 6 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 6 s after A started creating |
| ℹ️ | 2 the builds took 40 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 8081 / 3.853765 0.391048;8082 / 2.081510 0.547409;8081 /api/json?tree=jobs[name,color] 0.051027 0.015912;8082 /api/json?tree=jobs[name,color] 0.030095 0.013616;8081 /job/job-3/ 0.219141 0.011311;8082 /job/job-3/ 0.116615 0.010215;8081 /job/job-3/api/json?depth=1 0.019724 0.006356;8082 /job/job-3/api/json?depth=1 0.017123 0.005494;8081 /manage/cluster/ 0.059532 0.020330;8082 /manage/cluster/ 0.052935 0.020726; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 334155K, B used 262049K |
| ℹ️ | 3 store: 5728 keys, 4578 kB, 30638 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.074 ms |
| ℹ️ | 4 fresh-1: serving the API 11.6 s after the container started (container and JVM start to initialization 3.1 s, initialization 7.1 s); first / then 1.470066 s |
| ✅ | 4 fresh B #1 is up within 180 s |
| ℹ️ | 4 fresh-2: serving the API 10.5 s after the container started (container and JVM start to initialization 2.4 s, initialization 6.9 s); first / then 1.606641 s |
| ✅ | 4 fresh B #2 is up within 180 s |
| ℹ️ | 4 fresh-3: serving the API 10.6 s after the container started (container and JVM start to initialization 2.4 s, initialization 7.0 s); first / then 1.336832 s |
| ✅ | 4 fresh B #3 is up within 180 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 median time to serving: 10.7 s; warm / afterwards 0.540406 s |
| ℹ️ | 4 runtime-plugin: serving the API 10.8 s after the container started (container and JVM start to initialization 2.6 s, initialization 6.9 s); first / then 1.473878 s |
| ✅ | 4 fresh B with next-build-number installed at runtime is up within 180 s |
| ✅ | 4 ... and has next-build-number |
| ℹ️ | 5 B's / with 50 builds running on A: 0.904516 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 50 builds, the last 9 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

## ET16 results (2026-10-12T09:59Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 6 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 6 s after A started creating |
| ℹ️ | 2 the builds took 42 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 8081 / 4.318194 0.358583;8082 / 1.986341 0.411462;8081 /api/json?tree=jobs[name,color] 0.129580 0.020282;8082 /api/json?tree=jobs[name,color] 0.025279 0.013863;8081 /job/job-3/ 0.182404 0.011689;8082 /job/job-3/ 0.105911 0.013718;8081 /job/job-3/api/json?depth=1 0.022425 0.006948;8082 /job/job-3/api/json?depth=1 0.018166 0.005956;8081 /manage/cluster/ 0.065313 0.020256;8082 /manage/cluster/ 0.050626 0.018534; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 224637K, B used 260841K |
| ℹ️ | 3 store: 5728 keys, 4578 kB, 30637 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.091 ms |
| ℹ️ | 4 fresh-1: serving the API 11.0 s after the container started (container and JVM start to initialization 2.8 s, initialization 7.1 s); first / then 1.411127 s |
| ✅ | 4 fresh B #1 is up within 180 s |
| ℹ️ | 4 fresh-2: serving the API 14.3 s after the container started (container and JVM start to initialization 3.1 s, initialization 8.6 s); first / then 1.470564 s |
| ✅ | 4 fresh B #2 is up within 180 s |
| ℹ️ | 4 fresh-3: serving the API 11.3 s after the container started (container and JVM start to initialization 3.0 s, initialization 7.0 s); first / then 1.290729 s |
| ✅ | 4 fresh B #3 is up within 180 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 median time to serving: 11.3 s; warm / afterwards 0.481902 s |
| ℹ️ | 4 runtime-plugin: serving the API 10.8 s after the container started (container and JVM start to initialization 2.7 s, initialization 6.7 s); first / then 1.542564 s |
| ✅ | 4 fresh B with next-build-number installed at runtime is up within 180 s |
| ✅ | 4 ... and has next-build-number |
| ℹ️ | 5 B's / with 50 builds running on A: 0.786461 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 50 builds, the last 10 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

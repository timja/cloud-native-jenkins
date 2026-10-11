## ET16 results (2026-10-11T21:43Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 64 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 65 s after A started creating |
| ℹ️ | 2 the builds took 92 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 8081 / 4.103652 0.548718;8082 / 3.455048 0.782524;8081 /api/json?tree=jobs[name,color] 0.159715 0.021431;8082 /api/json?tree=jobs[name,color] 0.058407 0.019990;8081 /job/job-3/ 0.286632 0.019005;8082 /job/job-3/ 0.461200 0.030060;8081 /job/job-3/api/json?depth=1 0.055082 0.020672;8082 /job/job-3/api/json?depth=1 0.288813 0.015755;8081 /manage/cluster/ 0.244079 0.045360;8082 /manage/cluster/ 0.167196 0.041061; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 189903K, B used 241149K |
| ℹ️ | 3 store: 4227 keys, 4481 kB, 20533 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.101 ms |
| ℹ️ | 4 fresh-1: serving the API 20.4 s after the container started (container and JVM start to initialization 4.9 s, initialization 9.7 s); first / then 19.003424 s |
| ✅ | 4 fresh B #1 is up within 180 s |
| ℹ️ | 4 fresh-2: serving the API 16.3 s after the container started (container and JVM start to initialization 3.9 s, initialization 10.8 s); first / then 5.352857 s |
| ✅ | 4 fresh B #2 is up within 180 s |
| ℹ️ | 4 fresh-3: serving the API 12.7 s after the container started (container and JVM start to initialization 2.6 s, initialization 7.0 s); first / then 6.756909 s |
| ✅ | 4 fresh B #3 is up within 180 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 median time to serving: 16.4 s; warm / afterwards 0.606916 s |
| ℹ️ | 4 runtime-plugin: serving the API 12.8 s after the container started (container and JVM start to initialization 3.0 s, initialization 7.1 s); first / then 6.851768 s |
| ✅ | 4 fresh B with next-build-number installed at runtime is up within 180 s |
| ✅ | 4 ... and has next-build-number |
| ℹ️ | 5 B's / with 50 builds running on A: 0.788712 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 50 builds, the last 11 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

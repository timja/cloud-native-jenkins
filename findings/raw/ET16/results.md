## ET16 results (2026-10-14T11:34Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 107 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 108 s after A started creating |
| ℹ️ | 2 the builds took 122 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 20481 / 33.535672 0.346617;20482 / 1.210799 0.298823;20481 /api/json?tree=jobs[name,color] 0.022343 0.008724;20482 /api/json?tree=jobs[name,color] 0.024642 0.009948;20481 /job/job-3/ 0.113076 0.009763;20482 /job/job-3/ 0.099278 0.009700;20481 /job/job-3/api/json?depth=1 0.013291 0.004193;20482 /job/job-3/api/json?depth=1 0.013726 0.004449;20481 /manage/cluster/ 0.045562 0.015019;20482 /manage/cluster/ 0.040592 0.016099; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 118914K, B used 303771K |
| ℹ️ | 3 store: 5729 keys, 4581 kB, 32638 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.055 ms |
| ℹ️ | 4 fresh-1: serving the API 9.5 s after the container started (container and JVM start to initialization 2.1 s, initialization 5.2 s); first / then 1.090387 s |
| ✅ | 4 fresh B #1 is up within 180 s |
| ℹ️ | 4 fresh-2: serving the API 9.8 s after the container started (container and JVM start to initialization 2.2 s, initialization 5.3 s); first / then 0.996929 s |
| ✅ | 4 fresh B #2 is up within 180 s |
| ℹ️ | 4 fresh-3: serving the API 9.6 s after the container started (container and JVM start to initialization 2.2 s, initialization 5.2 s); first / then 0.980987 s |
| ✅ | 4 fresh B #3 is up within 180 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 median time to serving: 9.7 s; warm / afterwards 0.301269 s |
| ℹ️ | 4 runtime-plugin: serving the API 9.7 s after the container started (container and JVM start to initialization 2.2 s, initialization 5.2 s); first / then 1.165775 s |
| ✅ | 4 fresh B with next-build-number installed at runtime is up within 180 s |
| ✅ | 4 ... and has next-build-number |
| ℹ️ | 5 B's / with 50 builds running on A: 1.539324 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 50 builds, the last 8 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

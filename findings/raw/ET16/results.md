## ET16 results (2026-10-11T15:49Z): 2000 jobs, 2000 builds, 50 running at once

| | Check |
|---|---|
| ℹ️ | 1 creating them took 8 s on A |
| ✅ | 1 B lists all 2000 jobs within 120 s of their creation |
| ℹ️ | 1 B had them all 10 s after A started creating |
| ℹ️ | 2 the builds took 46 s (published: 2000) |
| ℹ️ | 3 pages (port, page, cold s, warm s): 8081 / 3.122988 0.700294;8082 / 3.131911 0.420490;8081 /api/json?tree=jobs[name,color] 0.116816 0.017304;8082 /api/json?tree=jobs[name,color] 0.043824 0.017147;8081 /job/job-3/ 0.273184 0.030446;8082 /job/job-3/ 0.225551 0.018068;8081 /job/job-3/api/json?depth=1 0.021722 0.007435;8082 /job/job-3/api/json?depth=1 0.031073 0.007264;8081 /manage/cluster/ 0.090940 0.031138;8082 /manage/cluster/ 0.256939 0.018629; |
| ✅ | 3 every page answers warm within 5 s on both controllers |
| ℹ️ | 3 heap: A used 129461K, B used 124058K |
| ℹ️ | 3 store: 4226 keys, 4362 kB, 20531 events |
| ✅ | 3 listing a job's builds uses the key index |
| ℹ️ | 3 a job's builds listed in Execution Time: 0.122 ms |
| ✅ | 4 B is up within 180 s |
| ℹ️ | 4 B was up after 14 s |
| ✅ | 4 and lists every job |
| ℹ️ | 4 B's / after the restart: cold 6.325239 s, warm 0.399356 s |
| ℹ️ | 5 B's / with 50 builds running on A: 0.712338 s |
| ✅ | 5 B adopts all 50 within 90 s |
| ℹ️ | 5 adopted 53 builds, the last 24 s after the kill |
| ✅ | 5 all of them complete |

Failures: 0

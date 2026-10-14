## ET17 results (2026-10-14T11:33Z): 3 controllers, 10 jobs x 20000 builds, 30 multibranch projects x 10 branches

| | Check |
|---|---|
| ℹ️ | 1 build records inserted in 4 s: 200010 records |
| ℹ️ | 2 all three up 24 s after starting together |
| ✅ | 2 started within 300 s |
| ℹ️ | 2 controller on 20281: job page cold 0.639137 s, warm 0.028935 s; builds{0,50} 0.252181 s; lastSuccessfulBuild 0.006084 s; newest console 0.003956 s; heap used 260657K |
| ✅ | 2 20281: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 20282: job page cold 0.502415 s, warm 0.026817 s; builds{0,50} 0.276292 s; lastSuccessfulBuild 0.004705 s; newest console 0.003835 s; heap used 291055K |
| ✅ | 2 20282: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 20283: job page cold 0.492328 s, warm 0.018576 s; builds{0,50} 0.202671 s; lastSuccessfulBuild 0.005118 s; newest console 0.004867 s; heap used 227779K |
| ✅ | 2 20283: the warm job page with 20000 builds under 2 s |
| ✅ | 2 the job lists its newest build |
| ℹ️ | 2 deleting a job with 20000 builds took 0 s |
| ✅ | 2 ... and the others drop it within 60 s |
| ✅ | 2 ... with its records (its build numbers are kept, so a recreated job never reuses one) |
| ℹ️ | 3 C killed 21 s into the storm; VM swap used 96 of 63423 MiB |
| ✅ | 3 every branch job (300) on A and B |
| ℹ️ | 3 storm settled 21 s after the scans |
| ✅ | 3 every branch built (a published record each: none lost) |
| ℹ️ | 3 successful: 300; closed as aborted (were running on C when it was killed): 0 |
| ✅ | 3 ... each aborted one closed because C was gone (5 consoles) |
| ℹ️ | 3 records by result: SUCCESS=300  |
| ℹ️ | 3 branches with more than one build: 0 |
| ✅ | 3 ... exactly once: no branch built successfully twice (concurrent scans of a project) |
| ℹ️ | 3 conflicts and merges (A;B;C) before: 0 0;0 0;0 0; after (A;B): 0 0;7 7; |
| ℹ️ | 3 heap: A used 346561K, B used 447786K; VM swap used 96 of 63423 MiB |
| ℹ️ | 3 top statement: 4745 ms, 7181 calls: with recursive s(seg) as ((select split_part(substr(key, $1), $7, $8) from jenkins_store where key ~>=~ $2 and key ~<~ $ |
| ✅ | 3 C, restarted, has every branch job |

Failures: 0

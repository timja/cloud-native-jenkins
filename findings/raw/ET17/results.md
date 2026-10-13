## ET17 results (2026-10-13T18:38Z): 3 controllers, 10 jobs x 20000 builds, 30 multibranch projects x 10 branches

| | Check |
|---|---|
| ℹ️ | 1 build records inserted in 4 s: 200010 records |
| ℹ️ | 2 all three up 17 s after starting together |
| ✅ | 2 started within 300 s |
| ℹ️ | 2 controller on 8081: job page cold 0.496552 s, warm 0.023042 s; builds{0,50} 0.204604 s; lastSuccessfulBuild 0.008604 s; newest console 0.006338 s; heap used 148760K |
| ✅ | 2 8081: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8082: job page cold 0.550757 s, warm 0.019189 s; builds{0,50} 0.267209 s; lastSuccessfulBuild 0.007181 s; newest console 0.005570 s; heap used 136355K |
| ✅ | 2 8082: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8083: job page cold 0.636583 s, warm 0.017356 s; builds{0,50} 0.220870 s; lastSuccessfulBuild 0.007717 s; newest console 0.006231 s; heap used 154847K |
| ✅ | 2 8083: the warm job page with 20000 builds under 2 s |
| ✅ | 2 the job lists its newest build |
| ℹ️ | 2 deleting a job with 20000 builds took 1 s |
| ✅ | 2 ... and the others drop it within 60 s |
| ✅ | 2 ... with its records (its build numbers are kept, so a recreated job never reuses one) |
| ℹ️ | 3 C killed 22 s into the storm; VM swap used 1023 of 1023 MiB |
| ✅ | 3 every branch job (300) on A and B |
| ℹ️ | 3 storm settled 28 s after the scans |
| ✅ | 3 every branch built (a published record each: none lost) |
| ℹ️ | 3 successful: 239; closed as aborted (were running on C when it was killed): 61 |
| ✅ | 3 ... each aborted one closed because C was gone (5 consoles) |
| ℹ️ | 3 records by result: ABORTED=61 SUCCESS=239  |
| ℹ️ | 3 branches with more than one build: 0 |
| ✅ | 3 ... exactly once: no branch built successfully twice (concurrent scans of a project) |
| ℹ️ | 3 conflicts and merges (A;B;C) before: 2 2;1 1;2 2; after (A;B): 3 3;1 1; |
| ℹ️ | 3 heap: A used 312205K, B used 312592K; VM swap used 1023 of 1023 MiB |
| ℹ️ | 3 top statement: 3241 ms, 5558 calls: with recursive s(seg) as ((select split_part(substr(key, $1), $7, $8) from jenkins_store where key ~>=~ $2 and key ~<~ $ |
| ✅ | 3 C, restarted, has every branch job |

Failures: 0

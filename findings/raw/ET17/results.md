## ET17 results (2026-10-12T12:31Z): 3 controllers, 10 jobs x 20000 builds, 30 multibranch projects x 10 branches

| | Check |
|---|---|
| ℹ️ | 1 build records inserted in 4 s: 200010 records |
| ℹ️ | 2 all three up 18 s after starting together |
| ✅ | 2 started within 300 s |
| ℹ️ | 2 controller on 8081: job page cold 0.644459 s, warm 0.022436 s; builds{0,50} 0.275848 s; lastSuccessfulBuild 0.007923 s; newest console 0.007951 s; heap used 271376K |
| ✅ | 2 8081: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8082: job page cold 0.691454 s, warm 0.023503 s; builds{0,50} 0.522861 s; lastSuccessfulBuild 0.017543 s; newest console 0.006566 s; heap used 135829K |
| ✅ | 2 8082: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8083: job page cold 0.780165 s, warm 0.031429 s; builds{0,50} 0.235357 s; lastSuccessfulBuild 0.007431 s; newest console 0.004702 s; heap used 128065K |
| ✅ | 2 8083: the warm job page with 20000 builds under 2 s |
| ✅ | 2 the job lists its newest build |
| ℹ️ | 2 deleting a job with 20000 builds took 1 s |
| ✅ | 2 ... and the others drop it within 60 s |
| ✅ | 2 ... with its records (its build numbers are kept, so a recreated job never reuses one) |
| ℹ️ | 3 C killed 22 s into the storm |
| ✅ | 3 every branch job (300) on A and B |
| ℹ️ | 3 storm settled 35 s after the scans |
| ✅ | 3 every branch built (a published record each: none lost) |
| ℹ️ | 3 successful: 186; closed as aborted (were running on C when it was killed): 114 |
| ✅ | 3 ... each aborted one closed because C was gone (5 consoles) |
| ℹ️ | 3 branches with more than one build: 0 |
| ✅ | 3 ... exactly once: no branch built successfully twice (concurrent scans of a project) |
| ℹ️ | 3 conflicts and merges (A;B;C) before: 1 1;2 2;1 1; after (A;B): 3 3;2 2; |
| ℹ️ | 3 heap: A used 338308K, B used 320307K |
| ℹ️ | 3 top statement: 3452 ms, 5345 calls: with recursive s(seg) as ((select split_part(substr(key, $1), $7, $8) from jenkins_store where key ~>=~ $2 and key ~<~ $ |
| ✅ | 3 C, restarted, has every branch job |

Failures: 0

## ET17 results (2026-10-12T18:46Z): 3 controllers, 10 jobs x 20000 builds, 30 multibranch projects x 10 branches

| | Check |
|---|---|
| ℹ️ | 1 build records inserted in 5 s: 200010 records |
| ℹ️ | 2 all three up 18 s after starting together |
| ✅ | 2 started within 300 s |
| ℹ️ | 2 controller on 8081: job page cold 0.581977 s, warm 0.023242 s; builds{0,50} 0.230982 s; lastSuccessfulBuild 0.010121 s; newest console 0.005880 s; heap used 133346K |
| ✅ | 2 8081: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8082: job page cold 0.792041 s, warm 0.070917 s; builds{0,50} 0.239455 s; lastSuccessfulBuild 0.007104 s; newest console 0.005403 s; heap used 198633K |
| ✅ | 2 8082: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8083: job page cold 0.939341 s, warm 0.030100 s; builds{0,50} 0.240144 s; lastSuccessfulBuild 0.007147 s; newest console 0.008969 s; heap used 155338K |
| ✅ | 2 8083: the warm job page with 20000 builds under 2 s |
| ✅ | 2 the job lists its newest build |
| ℹ️ | 2 deleting a job with 20000 builds took 1 s |
| ✅ | 2 ... and the others drop it within 60 s |
| ✅ | 2 ... with its records (its build numbers are kept, so a recreated job never reuses one) |
| ℹ️ | 3 C killed 22 s into the storm |
| ✅ | 3 every branch job (300) on A and B |
| ℹ️ | 3 storm settled 30 s after the scans |
| ❌ | 3 every branch built (a published record each: none lost) |
| ℹ️ | 3 successful: 213; closed as aborted (were running on C when it was killed): 84 |
| ✅ | 3 ... each aborted one closed because C was gone (5 consoles) |
| ℹ️ | 3 branches with more than one build: 0 |
| ✅ | 3 ... exactly once: no branch built successfully twice (concurrent scans of a project) |
| ℹ️ | 3 conflicts and merges (A;B;C) before: 2 2;2 2;1 1; after (A;B): 6 6;2 2; |
| ℹ️ | 3 heap: A used 218177K, B used 487105K |
| ℹ️ | 3 top statement: 3373 ms, 5539 calls: with recursive s(seg) as ((select split_part(substr(key, $1), $7, $8) from jenkins_store where key ~>=~ $2 and key ~<~ $ |
| ✅ | 3 C, restarted, has every branch job |

Failures: 1

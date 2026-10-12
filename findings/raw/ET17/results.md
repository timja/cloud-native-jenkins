## ET17 results (2026-10-12T16:00Z): 3 controllers, 10 jobs x 20000 builds, 30 multibranch projects x 10 branches

| | Check |
|---|---|
| ℹ️ | 1 build records inserted in 5 s: 200010 records |
| ℹ️ | 2 all three up 20 s after starting together |
| ✅ | 2 started within 300 s |
| ℹ️ | 2 controller on 8081: job page cold 0.799149 s, warm 0.019508 s; builds{0,50} 0.278796 s; lastSuccessfulBuild 0.009981 s; newest console 0.007122 s; heap used 149872K |
| ✅ | 2 8081: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8082: job page cold 1.028789 s, warm 0.026776 s; builds{0,50} 0.230180 s; lastSuccessfulBuild 0.012497 s; newest console 0.011987 s; heap used 167549K |
| ✅ | 2 8082: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8083: job page cold 0.862441 s, warm 0.033907 s; builds{0,50} 0.332623 s; lastSuccessfulBuild 0.008606 s; newest console 0.005902 s; heap used 142818K |
| ✅ | 2 8083: the warm job page with 20000 builds under 2 s |
| ✅ | 2 the job lists its newest build |
| ℹ️ | 2 deleting a job with 20000 builds took 0 s |
| ✅ | 2 ... and the others drop it within 60 s |
| ✅ | 2 ... with its records (its build numbers are kept, so a recreated job never reuses one) |
| ℹ️ | 3 C killed 23 s into the storm |
| ✅ | 3 every branch job (300) on A and B |
| ℹ️ | 3 storm settled 45 s after the scans |
| ❌ | 3 every branch built (a published record each: none lost) |
| ℹ️ | 3 successful: 199; closed as aborted (were running on C when it was killed): 91 |
| ✅ | 3 ... each aborted one closed because C was gone (5 consoles) |
| ℹ️ | 3 branches with more than one build: 0 |
| ✅ | 3 ... exactly once: no branch built successfully twice (concurrent scans of a project) |
| ℹ️ | 3 conflicts and merges (A;B;C) before: 1 1;2 2;1 1; after (A;B): 1 1;12 12; |
| ℹ️ | 3 heap: A used 417374K, B used 250483K |
| ℹ️ | 3 top statement: 7031 ms, 5311 calls: with recursive s(seg) as ((select split_part(substr(key, $1), $7, $8) from jenkins_store where key ~>=~ $2 and key ~<~ $ |

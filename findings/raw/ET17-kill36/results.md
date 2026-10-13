## ET17 results (2026-10-13T18:48Z): 3 controllers, 10 jobs x 20000 builds, 30 multibranch projects x 10 branches

| | Check |
|---|---|
| ℹ️ | 1 build records inserted in 10 s: 200010 records |
| ℹ️ | 2 all three up 25 s after starting together |
| ✅ | 2 started within 300 s |
| ℹ️ | 2 controller on 8081: job page cold 0.653615 s, warm 0.043496 s; builds{0,50} 0.276576 s; lastSuccessfulBuild 0.012063 s; newest console 0.006747 s; heap used 136842K |
| ✅ | 2 8081: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8082: job page cold 1.196717 s, warm 0.045816 s; builds{0,50} 0.519796 s; lastSuccessfulBuild 0.021946 s; newest console 0.018657 s; heap used 168562K |
| ✅ | 2 8082: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 8083: job page cold 1.214629 s, warm 0.028763 s; builds{0,50} 0.376676 s; lastSuccessfulBuild 0.012059 s; newest console 0.007981 s; heap used 167193K |
| ✅ | 2 8083: the warm job page with 20000 builds under 2 s |
| ✅ | 2 the job lists its newest build |
| ℹ️ | 2 deleting a job with 20000 builds took 1 s |
| ✅ | 2 ... and the others drop it within 60 s |
| ✅ | 2 ... with its records (its build numbers are kept, so a recreated job never reuses one) |
| ℹ️ | 3 C killed 38 s into the storm; VM swap used 1023 of 1023 MiB |
| ✅ | 3 every branch job (300) on A and B |
| ℹ️ | 3 storm settled 51 s after the scans |
| ✅ | 3 every branch built (a published record each: none lost) |
| ℹ️ | 3 successful: 245; closed as aborted (were running on C when it was killed): 55 |
| ✅ | 3 ... each aborted one closed because C was gone (5 consoles) |
| ℹ️ | 3 records by result: ABORTED=55 SUCCESS=245  |
| ℹ️ | 3 branches with more than one build: 0 |
| ✅ | 3 ... exactly once: no branch built successfully twice (concurrent scans of a project) |
| ℹ️ | 3 conflicts and merges (A;B;C) before: 2 2;1 1;2 2; after (A;B): 2 2;2 2; |
| ℹ️ | 3 heap: A used 463195K, B used 390168K; VM swap used 1023 of 1023 MiB |
| ℹ️ | 3 top statement: 8563 ms, 5462 calls: with recursive s(seg) as ((select split_part(substr(key, $1), $7, $8) from jenkins_store where key ~>=~ $2 and key ~<~ $ |
| ✅ | 3 C, restarted, has every branch job |

Failures: 0

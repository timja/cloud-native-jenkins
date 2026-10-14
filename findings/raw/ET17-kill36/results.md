## ET17 results (2026-10-14T11:33Z): 3 controllers, 10 jobs x 20000 builds, 30 multibranch projects x 10 branches

| | Check |
|---|---|
| ℹ️ | 1 build records inserted in 4 s: 200010 records |
| ℹ️ | 2 all three up 13 s after starting together |
| ✅ | 2 started within 300 s |
| ℹ️ | 2 controller on 20181: job page cold 0.473567 s, warm 0.022711 s; builds{0,50} 0.227576 s; lastSuccessfulBuild 0.006645 s; newest console 0.005139 s; heap used 143274K |
| ✅ | 2 20181: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 20182: job page cold 0.561797 s, warm 0.021722 s; builds{0,50} 0.221695 s; lastSuccessfulBuild 0.005254 s; newest console 0.003845 s; heap used 133122K |
| ✅ | 2 20182: the warm job page with 20000 builds under 2 s |
| ℹ️ | 2 controller on 20183: job page cold 0.447021 s, warm 0.021195 s; builds{0,50} 0.196685 s; lastSuccessfulBuild 0.005089 s; newest console 0.003969 s; heap used 171017K |
| ✅ | 2 20183: the warm job page with 20000 builds under 2 s |
| ✅ | 2 the job lists its newest build |
| ℹ️ | 2 deleting a job with 20000 builds took 1 s |
| ✅ | 2 ... and the others drop it within 60 s |
| ✅ | 2 ... with its records (its build numbers are kept, so a recreated job never reuses one) |
| ℹ️ | 3 C killed 38 s into the storm; VM swap used 96 of 63423 MiB |
| ✅ | 3 every branch job (300) on A and B |
| ℹ️ | 3 storm settled 38 s after the scans |
| ✅ | 3 every branch built (a published record each: none lost) |
| ℹ️ | 3 successful: 300; closed as aborted (were running on C when it was killed): 0 |
| ✅ | 3 ... each aborted one closed because C was gone (5 consoles) |
| ℹ️ | 3 records by result: SUCCESS=300  |
| ℹ️ | 3 branches with more than one build: 0 |
| ✅ | 3 ... exactly once: no branch built successfully twice (concurrent scans of a project) |
| ℹ️ | 3 conflicts and merges (A;B;C) before: 2 2;0 0;2 2; after (A;B): 7 7;0 0; |
| ℹ️ | 3 heap: A used 380128K, B used 414463K; VM swap used 96 of 63423 MiB |
| ℹ️ | 3 top statement: 5483 ms, 1789 calls: select seq, kind, key, old_key, version, origin from jenkins_store_events where seq > $1 order by seq |
| ✅ | 3 C, restarted, has every branch job |

Failures: 0

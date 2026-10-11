## ET5 results (2026-10-11T17:19Z)

| | Check |
|---|---|
| ✅ | agent1 is connected to A |
| ✅ | both controllers share the queue |
| ℹ️ | 1 queue item ids: A=1 B=1 |
| ✅ | 1 the second request folded into the same queue item |
| ✅ | 1 the item is in both queues |
| ✅ | 1 exactly one build |
| ✅ | 1 and B agrees |
| ✅ | 1 with both requests as causes (got 2) |
| ✅ | 2 forty builds, numbered 1..40 |
| ✅ | 2 every request ran exactly once |
| ℹ️ | 2 builds per controller:   20 host=jenkins-a   20 host=jenkins-b  |
| ✅ | 2 both controllers ran some |
| ✅ | 2 queue ids are unique cluster-wide |
| ✅ | 2 queue/item/<id> resolves to the build on both controllers |
| ✅ | 3 B's request waits while #1 builds on A |
| ✅ | 3 the two builds did not overlap (#1 ended 1791739228, #2 started 1791739228) |
| ✅ | 4 cancelled on B |
| ✅ | 4 gone from A's queue |
| ✅ | 4 reported cancelled on A |
| ✅ | 5 aborted on A |
| ✅ | 5 the log says who stopped it |
| ✅ | 6 all three run on B |
| ✅ | 6 they ran on jenkins-b |
| ✅ | 7 it runs, on the controller holding the agent |
| ✅ | 7 it ran on A (whose executor on agent1 took it) |
| ✅ | 8 #2 passes milestone 1 |
| ✅ | 8 #1 on A is cancelled |
| ✅ | 8 #1 never passed the milestone |
| ✅ | 9 both complete |
| ℹ️ | 9 res holds: WRITTEN@1791739310.6 DELETED@1791739325.7 WRITTEN@1791739325.7 DELETED@1791739340.8  |
| ✅ | 9 res was held by one build at a time (store events alternate, two holds) |
| ℹ️ | 9 the waiter got res 0.0 s after its release |
| ✅ | 10 one upstream build on B triggered exactly one downstream build |
| ✅ | 10 one more on A triggered exactly one more |
| ✅ | 10 build(wait: true) from A completes |
| ✅ | 10 the callee ran on A, where the caller waits |
| ℹ️ | 11 BackgroundGlobalBuildDiscarder allowed for this period: A=true B=false |
| ℹ️ | 11 claims for this period: 1 (the discarder may already have run after startup and claimed it) |
| ✅ | 11 at most one controller runs it this period |
| ✅ | 11 and exactly one claim is recorded for it |
| ✅ | 11 per-controller work is not coordinated |

Failures: 0

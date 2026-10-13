## ET7 results (2026-10-13T18:01Z)

| | Check |
|---|---|
| ✅ | agent1 is connected to A |
| ✅ | 2 the build page answers on B |
| ✅ | 2 its REST API on B says it is building |
| ✅ | 2 answered by A (forwarded) |
| ✅ | 2 through rr too |
| ℹ️ | 1 last tick seen on B 12, on A 12; polls within 2 ticks of A: 10/10 |
| ✅ | 1 B's live console keeps up with A (within 2 s on every poll) |
| ✅ | 1 B's live console has every line so far, in order |
| ✅ | 3 ask is waiting for input (seen from B) |
| ℹ️ | 3 form submit through rr answered 302 |
| ✅ | 3 approved through the form, the build continues on A with the value |
| ✅ | 3 and completes |
| ✅ | 3 ask2 is waiting for input |
| ✅ | 3 aborted from B (REST) |
| ✅ | 4 stopped from B |
| ℹ️ | 5 replay from B answered 302 |
| ✅ | 5 a new build ran the edited script |
| ✅ | 6 the live freestyle console answers on B |
| ✅ | 6 and grows |
| ✅ | 7 on B, agent1's page says online (answered by A) |
| ✅ | 7 forwarded to A |
| ✅ | 7 its log answers on B |
| ✅ | 7 its script console works from B |
| ✅ | 7 the channel table names A |
| ✅ | 8 late is waiting for input |
| ✅ | 8 B adopted the build |
| ✅ | 8 approved on B after adoption, the build completes |
| ✅ | 8 with the line after the input |
| ✅ | 7 agent1 reconnected to B |
| ℹ️ | 7 agent1 online on B 7 s after A was killed |
| ✅ | 7 the channel table names B |
| ✅ | 10 no 502 for a dead owner's build (answered locally) |
| ✅ | 9 notifyCommit to A: exactly one build for the commit (1 -> 2) |
| ✅ | 9 notifyCommit to B: exactly one build for the commit (2 -> 3) |
| ✅ | 9 notifyCommit to both: exactly one build for the commit (3 -> 4) |
| ✅ | 9 a multibranch push event to B: exactly one branch build (1 -> 2) |
| ✅ | 10 requests for a completed build are not forwarded (21 -> 21) |
| ✅ | 10 an already forwarded request is not forwarded again (answered by B itself: 404) |
| ℹ️ | forwarded build requests: A=0 B=22; computer requests: A=0 B=4 |
| ✅ | 10 no routing warnings in the logs |

Failures: 0

## ET4 results (2026-10-14T11:56Z)

| | Check |
|---|---|
| ✅ | agent1 is connected to A (the primary) |
| ✅ | agent1 is not connected to B |
| ✅ | 3 B shows ctl #1 running on jenkins-a |
| ✅ | 3 B adopts and resumes it |
| ℹ️ | 3 adopted 6 s after the kill |
| ✅ | 3 B logs the resumption |
| ✅ | 3 the build completes on B with SUCCESS |
| ✅ | 3 every step ran (step 1..30) |
| ✅ | 3 steps before the kill were not re-run from the start (step 1 once) |
| ℹ️ | 3 steps repeated after the resume:  |
| ✅ | 3 fresh A lists the build with the same result |
| ✅ | 3 fresh A shows the same console |
| ℹ️ | 1 agent1 connected to: B |
| ✅ | 1 agent sh is running on A |
| ✅ | 1 B adopts the build |
| ℹ️ | 1 adopted 6 s after the kill |
| ✅ | 1 agent1 reconnects to B |
| ✅ | 1 the sh step continues to tick 60 |
| ✅ | 1 every tick is in the log (1..60) |
| ✅ | 1 the sh step was not restarted (tick 1 once) |
| ✅ | 1 the build waits at input on B |
| ✅ | 1 approved on B: the build completes with SUCCESS |
| ✅ | 1 post-input agent step ran |
| ✅ | 1 the log has the pre-kill output, the resumption, and the rest |
| ✅ | 1 fresh A lists it with the same number and result |
| ✅ | 1 fresh A shows the same console |
| ✅ | 2 A adopts the build waiting at input |
| ℹ️ | 2 adopted 14 s after the kill |
| ✅ | 2 A has the pending input |
| ✅ | 2 approved on A: the build completes with SUCCESS |
| ✅ | 2 the agent step after input ran |
| ✅ | 2b the other controller adopts it, and agent1 reconnects there |
| ✅ | 2b the read stage finds the file the write stage left: the build completes |
| ✅ | 2b ... printing it after the resumption |
| ✅ | 4 B adopts while A is paused |
| ℹ️ | 4 adopted 20 s after the pause |
| ✅ | 4 A notices it lost the build |
| ✅ | 4 the build completes on B with SUCCESS |
| ✅ | 4 only B published the record |
| ✅ | 4 A lists B's build (its own copy discarded) |
| ✅ | 4 A's console is B's (no appends from A after the fence) |
| ✅ | 4 every beat is in the log |
| ✅ | 5 completes on A with SUCCESS |
| ✅ | 5 B never showed it as anything but running on A |
| ✅ | 5 B never adopted it |
| ✅ | 6 B removes its placeholder |
| ✅ | 6 B does not run it: it is closed as aborted (Target 12) |
| ✅ | 6 ... keeping what it printed on A (Target 15) |
| ℹ️ | 7 200 echo steps: 677 ms; placeholder writes (save point commits + start/end): 3 |
| ℹ️ | 7 the same with PERFORMANCE_OPTIMIZED (no save points): 183 ms |
| ℹ️ | 7 builds with save points left in blob storage: none |
| ✅ | 7 no save points left in blob storage after completion |

Failures: 0

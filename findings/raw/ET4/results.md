## ET4 results (2026-10-11T12:47Z)

| | Check |
|---|---|
| ✅ | agent1 is connected to A (the primary) |
| ✅ | agent1 is not connected to B |
| ✅ | 3 B shows ctl #1 running on jenkins-a |
| ✅ | 3 B adopts and resumes it |
| ℹ️ | 3 adopted 27 s after the kill |
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
| ℹ️ | 1 adopted 49 s after the kill |
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
| ℹ️ | 2 adopted 24 s after the kill |
| ✅ | 2 A has the pending input |
| ✅ | 2 approved on A: the build completes with SUCCESS |
| ✅ | 2 the agent step after input ran |

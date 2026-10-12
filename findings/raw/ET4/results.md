## ET4 results (2026-10-12T06:45Z)

| | Check |
|---|---|
| ✅ | agent1 is connected to A (the primary) |
| ✅ | agent1 is not connected to B |
| ✅ | 3 B shows ctl #1 running on jenkins-a |
| ✅ | 3 B adopts and resumes it |
| ℹ️ | 3 adopted 7 s after the kill |
| ✅ | 3 B logs the resumption |
| ✅ | 3 the build completes on B with SUCCESS |
| ✅ | 3 every step ran (step 1..30) |
| ✅ | 3 steps before the kill were not re-run from the start (step 1 once) |
| ℹ️ | 3 steps repeated after the resume:  |
| ✅ | 3 fresh A lists the build with the same result |
| ✅ | 3 fresh A shows the same console |

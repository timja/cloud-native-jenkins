## ET12 results (2026-10-12T15:32Z)

| | Check |
|---|---|
| ✅ | lin is connected to B only |
| ✅ | 1 lab1 completes |
| ✅ | 1 on lin, by B (handed over from A) |
| ✅ | 1 lab2 (first on A's controller, then on lin) completes with both parts |
| ℹ️ | 1 A logged: Handing lab1 #1 over;Handing lab2 #1 over; |
| ✅ | 1b lab3 completes, with both branches |
| ℹ️ | 1b A handed lab3 over 1 time(s) |
| ✅ | 1b handed over once, after the branch on A's node finished (not back and forth) |
| ✅ | 1c up1 completes with the downstream result |
| ✅ | 1c ... the downstream build ran on B, not pinned to A |
| ✅ | 1d B adopts up2, which completes with the downstream result |
| ✅ | 2 B's copy of the build gets the file |
| ✅ | 4 a request for a build of the old controller is answered (not forwarded to its namesake: 404) |
| ✅ | 3 the Pipeline is resumed and completes |
| ✅ | 3 the freestyle build is closed as aborted |
| ✅ | 3 with a console saying why, on both controllers |
| ✅ | 4 no routing warnings after the restart |

Failures: 0

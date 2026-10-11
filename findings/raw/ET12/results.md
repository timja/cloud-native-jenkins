## ET12 results (2026-10-11T05:53Z)

| | Check |
|---|---|
| ✅ | lin is connected to B only |
| ❌ | 1 lab1 completes |
| ❌ | 1 on lin, by B (handed over from A) |
| ✅ | 1 lab2 (first on A's controller, then on lin) completes with both parts |
| ℹ️ | 1 A logged: Handing lab2 #1 over; |
| ✅ | 2 B's copy of the build gets the file |
| ✅ | 4 a request for a build of the old controller is answered (not forwarded to its namesake: 200) |
| ✅ | 3 the Pipeline is resumed and completes |
| ✅ | 3 the freestyle build is closed as aborted |
| ❌ | 3 with a console saying why |
| ✅ | 4 no routing warnings after the restart |

Failures: 3

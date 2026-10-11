## ET15 results (2026-10-11T15:13Z)

| | Check |
|---|---|
| ✅ | 1 while it runs, B's API shows it as the last build, building |
| ✅ | 1 ... and first in builds[] |
| ✅ | 1 ... and B's /lastBuild/api/json is the running build (redirected and forwarded to A) |
| ✅ | 1 completes |
| ℹ️ | 1 log segments: 6 |
| ✅ | 1 the log was rolled over into several segments |
| ✅ | 1 B shows the whole log, in order |
| ❌ | 1 after completion, B's API shows the completed build |
| ✅ | 2 B adopts within 30 s |
| ℹ️ | 2 adopted 21 s after the kill |
| ✅ | 2 completes on B |
| ✅ | 2 the whole log, in order, across the segments of both owners |
| ℹ️ | 2 log segments: 7 |
| ❌ | 3 the renamed build completes |
| ❌ | 3 its whole log, in order, on B |
| ✅ | 3 no failed appends after the rename |
| ℹ️ | 4 run 1: adopted 18 s after the kill |
| ℹ️ | 4 run 2: adopted 22 s after the kill |
| ℹ️ | 4 run 3: adopted 21 s after the kill |
| ✅ | 4 every adoption within 30 s (worst case was ~50 s) |
| ✅ | 4 every adopted build completed |

Failures: 3

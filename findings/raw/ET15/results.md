## ET15 results (2026-10-11T21:29Z)

| | Check |
|---|---|
| ✅ | 1 while it runs, B's API shows it as the last build, building |
| ✅ | 1 ... and first in builds[] |
| ✅ | 1 ... and B's /lastBuild/api/json is the running build (redirected and forwarded to A) |
| ✅ | 1 completes |
| ℹ️ | 1 log segments: 6 |
| ✅ | 1 the log was rolled over into several segments |
| ✅ | 1 B shows the whole log, in order |
| ✅ | 1 after completion, B's API shows the completed build |
| ✅ | 2 B adopts within 30 s |
| ℹ️ | 2 adopted 9 s after the kill |
| ✅ | 2 completes on B |
| ✅ | 2 the whole log, in order, across the segments of both owners |
| ℹ️ | 2 log segments: 8 |
| ✅ | 3 renaming the job from B is refused (400) |
| ✅ | 3 renaming its folder from B is refused (400) |
| ✅ | 3 the build completes, with its whole log on B |
| ✅ | 3 ... in order |
| ℹ️ | 4 SIGKILL run 1: adopted 6.1 s after the kill |
| ℹ️ | 4 SIGKILL run 2: adopted 6.3 s after the kill |
| ℹ️ | 4 SIGKILL run 3: adopted 5.0 s after the kill |
| ✅ | 4 every adoption after a SIGKILL within 12 s (the session lock is freed with the process) |
| ✅ | 4 every adopted build completed |
| ℹ️ | 4 clean stop: adopted 6.5 s after docker stop began |
| ✅ | 4 after a clean stop, adopted within 30 s of the stop beginning (shutdown included) |
| ✅ | 4 ... and completed |
| ✅ | 4 a 12 s pause is not taken for a death (no adoption) |
| ✅ | 4 ... and the build completes on A |
| ✅ | 5 the build is closed as aborted on B |

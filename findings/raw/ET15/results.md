## ET15 results (2026-10-12T06:23Z)

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
| ℹ️ | 2 adopted 7 s after the kill |
| ✅ | 2 completes on B |
| ✅ | 2 the whole log, in order, across the segments of both owners |
| ℹ️ | 2 log segments: 7 |
| ✅ | 3 renaming the job from B is refused (400) |
| ✅ | 3 renaming its folder from B is refused (400) |
| ✅ | 3 the build completes, with its whole log on B |
| ✅ | 3 ... in order |

## ET8 results (2026-10-14T11:51Z)

| | Check |
|---|---|
| ✅ | both controllers use the store for fingerprints |
| ✅ | agent1 is connected to A |
| ℹ️ | 1 fingerprint 94dcd26d8b74fe93fa8e9dad6f46f7fa: A sees [fpa:1-1 fpb:1-1], B sees [fpa:1-1 fpb:1-1] |
| ✅ | 1 A lists both usages |
| ✅ | 1 B lists both usages |
| ✅ | 1 nothing under fingerprints/ in either home |
| ℹ️ | 2 usages after: fpa:1-6 fpb:1-6; builds: fpa [1 2 3 4 5 6] fpb [1 2 3 4 5 6] |
| ✅ | 2 every build is a usage (fpa:1-6 fpb:1-6) |
| ✅ | 2 every round's builds ran (6 each) |
| ✅ | 2 B agrees |
| ✅ | 3 B shows the report within 2 s of completion (3 tests, 1 failed) |
| ℹ️ | 3 report on B after 0.1 s |
| ✅ | 3 deleted on B, its results are gone on A |
| ✅ | 3 and on B |
| ✅ | 3b the coverage build completes on A |
| ✅ | 3b B shows the same coverage (line coverage 50%) |
| ✅ | 3b ... and its coverage page's class tree (fetched as the page's script does) |
| ℹ️ | 4 keep builds: A [6 7 8] B [6 7 8] |
| ✅ | 4 A keeps exactly the last 3 (6 7 8) |
| ✅ | 4 B keeps exactly the last 3 (6 7 8) |
| ✅ | 4 no discarder errors in the logs |
| ✅ | 5 B's view lists both controllers alive |
| ✅ | 5 B's view lists dk #1 running on A |
| ✅ | 5 B's view says agent1 is connected to A |
| ✅ | 5 A's view is the same apart from 'self' |
| ✅ | 4b dk #1, running on A while B discarded, completed on A: its record published, or its placeholder removed by A |
| ✅ | 4b then, the oldest of three, it was discarded: both keep [2 3] |
| ℹ️ | 4b dk builds after: A [2 3] B [2 3] |
| ✅ | 5 after A dies, B's view shows only B alive (within the membership timeout) |
| ✅ | 1 the fingerprint and its usages survive (fresh B) |

Failures: 0

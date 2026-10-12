## ET18 results (2026-10-12T18:58Z)

| | Check |
|---|---|
| ✅ | 1 the WAL is archived |
| ℹ️ | 1 at T: 36 keys; blobs 132K |
| ✅ | 1 after T: doomed is gone, created exists |
| ✅ | 2 Postgres recovered to T and was promoted |
| ✅ | 2 doomed is back (its configuration and records, as at T) |
| ✅ | 2 created is not there (made after T) |
| ✅ | 2 edited has its configuration as at T |
| ✅ | 2 kept #1 is there, with its console |
| ℹ️ | 2 without blob versioning: doomed #1's record is back, its console answers 200 (its snapshot was deleted after T) |
| ✅ | 2 ... which is what blob versioning is needed for: doomed #1's snapshot is missing |
| ℹ️ | 2 longp #1 (running at T): ? |
| ✅ | 3 doomed #1's console is back |
| ✅ | 3 the Pipeline running at T is resumed from its save point (or closed as aborted), not lost |
| ℹ️ | 3 longp #1: SUCCESS |
| ✅ | 3 its console has what it printed before T |

Failures: 0

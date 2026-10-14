## ET3 results (2026-10-14T11:50Z)

| | Check |
|---|---|
| ✅ | both controllers use the blob store |
| ✅ | 1 B lists the freestyle build made on A |
| ✅ | 1 same result on both (UNSTABLE) |
| ✅ | 1 same duration on both |
| ✅ | 1 console text is identical on B |
| ✅ | 1 test report has the same counts on B |
| ℹ️ | 1 test report on B: {"_class":"hudson.tasks.junit.TestResult","failCount":1,"passCount":1,"skipCount":0} |
| ✅ | 2 B lists the Pipeline build made on A |
| ✅ | 2 consoleText is byte-identical on B |
| ✅ | 2 flow graph and step logs are identical on B |
| ℹ️ | 2 flow nodes on B: 19, step logs: 7 |
| ℹ️ | 2 blobs of pl #1: jobs/pl/builds/1/files.tgz jobs/pl/builds/1/log jobs/pl/builds/1/log-index |
| ✅ | 2 the log is in log storage (append blobs), next to the snapshot |
| ✅ | 3 A lists builds 1..20 |
| ✅ | 3 B lists builds 1..20 |
| ✅ | 3 permalinks agree |
| ℹ️ | 3 permalinks on A: lastBuild=20 lastSuccessfulBuild=20 lastFailedBuild=14; on B: lastBuild=20 lastSuccessfulBuild=20 lastFailedBuild=14 |
| ✅ | 3 lastFailedBuild is the failed build, on A |
| ✅ | 3 lastFailedBuild is the failed build, on B |
| ✅ | 4 a description set on B for a build that ran on A is visible on A |
| ✅ | 4 B materialized #2 locally before the delete |
| ✅ | 4 a build deleted on A is gone on B |
| ✅ | 4 and its local directory on B is deleted |
| ✅ | 4 B's history is [1] |
| ✅ | 5 fresh A has no local build directories to start with |
| ✅ | 5 fresh A lists all builds of il |
| ✅ | 5 B still lists all builds of il |
| ✅ | 5 fresh A opens the console of a build A ran before |
| ✅ | 5 fresh A opens the console of a build B ran |
| ✅ | 5 fresh A opens the console of a freestyle build |
| ✅ | 5 fresh A shows the test results |
| ✅ | 5 fresh A shows the flow graph and step logs |
| ✅ | 6 B shows a placeholder for #1 running on jenkins-a |
| ℹ️ | 6 placeholder visible on B after 2 s |
| ✅ | 6 A shows no placeholder for its own build |
| ✅ | 6 B's read-only console shows the output so far |
| ✅ | 6 placeholder replaced by the full build on B when it completes |
| ✅ | 6 no placeholder left on B |
| ✅ | 6 completed console identical on B |
| ✅ | 6 B removes the stale placeholder within 90 s |
| ℹ️ | 6 stale placeholder removed 4 s after the kill |
| ✅ | 7 A sees the rename |
| ✅ | 7 history intact on A under the new name |
| ✅ | 7 history intact on B under the new name |
| ✅ | 7 console of an old build readable on A |
| ✅ | 7 console of an old build readable on B |
| ✅ | 7 nothing left under the old name in the blob store |
| ✅ | 7 numbering continues after the rename (#21) |
| ✅ | 8 the job is gone on B |
| ✅ | 8 first build of the recreated job is numbered after the old ones (#2) |
| ✅ | 8 no old build in its history on A |
| ✅ | 9 no snapshots or logs in jenkins_store |
| ✅ | 9 largest jenkins_store value is under 1 MB (3329 bytes) |
| ✅ | 9 snapshots and logs are in the blob container (103 blobs) |

Failures: 0

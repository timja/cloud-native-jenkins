## ET3 results (2026-10-12T17:21Z)

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
| ❌ | 2 flow graph and step logs are identical on B |
| ℹ️ | 2 flow nodes on B: 135, step logs: 0 |
| ℹ️ | 2 blobs of pl #1: jobs/pl/builds/1/log jobs/pl/builds/1/log-index jobs/pl/builds/1/running/build.xml.56.1 jobs/pl/builds/1/running/files/workflow-completed/flowNodeStore.xml jobs/pl/builds/1/running/program.dat.56.1 jobs/pl/builds/1/running/workflow/10.xml jobs/pl/builds/1/running/workflow/11.xml jobs/pl/builds/1/running/workflow/12.xml jobs/pl/builds/1/running/workflow/13.xml jobs/pl/builds/1/running/workflow/14.xml jobs/pl/builds/1/running/workflow/15.xml jobs/pl/builds/1/running/workflow/16.xml jobs/pl/builds/1/running/workflow/17.xml jobs/pl/builds/1/running/workflow/2.xml jobs/pl/builds/1/running/workflow/3.xml jobs/pl/builds/1/running/workflow/4.xml jobs/pl/builds/1/running/workflow/5.xml jobs/pl/builds/1/running/workflow/6.xml jobs/pl/builds/1/running/workflow/7.xml jobs/pl/builds/1/running/workflow/8.xml jobs/pl/builds/1/running/workflow/9.xml |
| ❌ | 2 the log is in log storage (append blobs), next to the snapshot |
| ✅ | 3 A lists builds 1..20 |
| ✅ | 3 B lists builds 1..20 |

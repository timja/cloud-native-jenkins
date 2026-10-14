## ET1 results (2026-10-14T11:47Z)

| | Check |
|---|---|
| ✅ | store plugin active on A and B |
| ✅ | A and B decrypt each other's secrets (S0 seed) |
| ✅ | 1a freestyle created on A appears on B |
| ✅ | 1b folder + child created on A appear on B |
| ✅ | 1c config edited on B is seen on A |
| ✅ | 1d folder renamed on A: new path on B |
| ✅ | 1d folder renamed on A: old path gone on B |
| ✅ | 1e job deleted on B is gone on A |
| ✅ | 2a parameter defined by a build on A is seen on B |
| ✅ | 2b cron trigger defined by a build on A is in B's config |
| ✅ | 2c no item config.xml written to A's local JENKINS_HOME (found: 0) |
| ✅ | 3a branches indexed on A appear on B |
| ✅ | 3b branch added and indexed on A appears on B |
| ✅ | 3c branch deleted and pruned on A is pruned on B |
| ✅ | 3d indexing on B: B still has exactly the branches |
| ✅ | 3d indexing on B: A still has exactly the branches |
| ℹ️ | conflicts detected during scenario 4: A=0 B=0 (warn policy: logged, last writer wins) |
| ✅ | 4 mb/main keeps its properties() parameter in the store |
| ✅ | 4 mb/main shows the parameter on A and B |
| ✅ | 4 mb/feature2 keeps its properties() parameter in the store |
| ✅ | 4 mb/feature2 shows the parameter on A and B |
| ℹ️ | scenario 4b: 80 builds rewrote jobs/race/config.xml; store version 586 → 1772; conflicts detected A=3 B=12 |
| ℹ️ | scenario 4b: final default parameter value: store=a39 A=a39 B=a39 |
| ✅ | 4b A, B and the store converge on the same job config |
| ℹ️ | scenario 4b: 80 writes in the event log; the store holds the last one. Builds reach properties() out of trigger order, so it need not be a40/b40; a stale overwrite would have shown as a conflict |
| ✅ | 6 re-index with no SCM changes bumps no stored versions (0 changed) |
| ✅ | 5 fresh B loads /job/f2/job/inner from the store |
| ✅ | 5 fresh B loads /job/props from the store |
| ✅ | 5 fresh B loads /job/mb/job/main from the store |
| ✅ | 5 fresh B loads /job/mb/job/feature2 from the store |
| ✅ | 5 fresh B does not resurrect deleted items |

Failures: 0

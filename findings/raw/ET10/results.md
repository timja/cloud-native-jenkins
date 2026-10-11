## ET10 results (2026-10-11T18:05Z)

| | Check |
|---|---|
| ✅ | 1 controllers were replaced one at a time (both images ran at once) |
| ✅ | 1 the rollout completes with both controllers on the new image |
| ✅ | 1 every running build completed |
| ✅ | 1 a build started after the rollout ran |
| ℹ️ | 1 live members after the rollout: ops-0:1 ops-1:1 |
| ✅ | 2 a controller requiring protocol 99 refuses to start, saying why |
| ℹ️ | 3 the old controller's quiet period: 3 |
| ℹ️ | 3 the old home: oldp#1 team/fs#2 |
| ℹ️ | 3 the importer's quiet period: 3; stored: <quietPeriod>3</quietPeriod> |
| ℹ️ | 3 import: files imported 13, kept 0; builds imported 3, kept 0 |
| ✅ | 3 the folder's job and its builds are there (team/fs: 2) |
| ✅ | 3 an old console |
| ✅ | 3 an old Pipeline console |
| ✅ | 3 old test results |
| ✅ | 3 the old user's API token works |
| ✅ | 3 the old credential decrypts in a build |
| ℹ️ | 3 node and quiet period on ops-1: true/3 |
| ✅ | 3 the old node and setting |
| ℹ️ | 3 import again: files imported 0, kept 13; builds imported 0, kept 4 |
| ✅ | 3 importing again changes nothing |
| ℹ️ | 4 backed up cnj-ops/ops to /Users/timja/code/jenkins/cloud-native-jenkins-t14/validation/out/backup:  24K database, 104K blobs |
| ✅ | 4 the restored cluster comes up |
| ✅ | 4 every job is there (during-backup,oldp,team,usecred) |
| ✅ | 4 history and consoles (team/fs #1) |
| ✅ | 4 users (the old token) |
| ✅ | 4 credentials and nodes |
| ℹ️ | 4 the build running at backup time, after restore: {} |

Failures: 0

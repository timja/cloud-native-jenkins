# ET10: Target 10 acceptance (rolling upgrades, importing a home, backup and restore)

**Result: PASS**, 18/18 checks, in two consecutive runs on the final build. The plan's point-in-time item for builds running at backup time is not met (below). Raw evidence from the last run: `raw/ET10/`
- `results.md`
- the import reports
- `config-events.txt`
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET10-operations.sh`, on the kind cluster of ET9.
- **Build:**
  - core `cnj/t10-m1-import-api`, the top of the core stack
  - kubernetes-plugin `cnj/t9-m2-cluster-cap`
  - plugin on `cnj/t10-m1-operations`
  - the "next" image is the same build under another image id

| Criterion | Result |
|---|---|
| 1 Rolling upgrade | ✅ `helm upgrade` to a new image while two 90 s Pipelines ran, one on each controller. The StatefulSet replaced the controllers one at a time (both images ran at once), and the rollout completed. Both running builds completed: each was adopted by the controller still up while its own was replaced. A build started after the rollout ran. Both members report protocol 1 in the cluster view |
| 2 Refused older peer | ✅ a controller started with `-Dio.jenkins.plugins.cnj.store.minimumPeerProtocol=99` refused to start while protocol-1 peers were alive, logging "needs every peer to speak cluster protocol 99" |
| 3 Import | ✅ a plain Jenkins (no store) was imported by starting one controller on its home with `-Dio.jenkins.plugins.cnj.store.importHome=true`. Its contents: a folder with a freestyle job and 2 builds with test results, a Pipeline with a build, a user with an API token, a credential, a node and a quiet period of 3. The cluster's controllers, seeded with the old home's keys, then had all of it: the builds and both consoles, the test results, the old token, the credential (decrypted in a new build), the node and the quiet period. Importing again imported nothing (`files imported 0, kept 13; builds imported 0`) |
| 4 Backup and restore | ✅ `scripts/backup.sh` (`pg_dump`, then the blob container) while a build ran. The namespace was deleted, and `scripts/restore.sh` restored into a new namespace with empty Postgres and Azurite. Every job is there, with history and consoles, the old user's token, the credential and the node |

## Not met: builds running at backup time
- **What the plan asked for:** that a build running when the backup was taken either resumes from its save point or is recorded as lost.
- **What happens:** after the restore it is simply absent: no record, no placeholder and no history entry.
  - Its running placeholder and save points are in the backup, but nothing adopts a build whose owner was never a member of the restored cluster.
  - A restored cluster needs a "restore sweep": every placeholder whose owner is not alive is adopted (Pipelines, from their last save point) or closed as aborted.
- This is a follow-up; ordinary backups of quiescent clusters are unaffected.

## How it works
- **Membership:** members publish their core and plugin version and a cluster protocol number (1). `minimumPeerProtocol` makes a version refuse to start next to older peers.
- **Import (`HomeImporter`):**
  - when the store is created, before anything reads from it, every owned key missing from the store is uploaded from the old home, and so are the fingerprints
  - once jobs are loaded, completed builds are published as T3 records and snapshots, with Pipeline logs uploaded to their blobs
  - keys already present are left alone, and a report is written to `cnj-import-report.txt`
  - runtime secrets are published by core (T6); the old home's pre-plugin keys become the seed Secret
- **Backup and restore:** chart scripts for the in-chart services. Postgres and Azurite now use persistent volumes when `persistence.enabled` is set (the default).

## Bugs found by ET10, now fixed
- **The import could lose settings** (runs 3, 5 and 6).
  - **Cause:** it ran as an initialiser before `SYSTEM_CONFIG_LOADED`, but that does not order it before core's own configuration loading, which attains that milestone. When loading won the race, the importer's Jenkins kept its defaults and saved them over the imported `config.xml` (the quiet period of 3 became 5).
  - **Fix:** the import runs while the store is being created.
- **The import and the peer check could do nothing.** Initialisers declared only after `PLUGINS_STARTED` may run before the store exists (the Target 7 finding). They now run once it exists.
- **Harness:**
  - the old controller reused the chart's JCasC with 0 executors, so its freestyle builds never ran
  - a Groovy `Boolean + String`
  - the import trace now records the setting at each step

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | 12/18 | the old controller had no executors (harness) |
| 2 | 16/18 | Groovy in two checks (harness) |
| 3-6 | 17-18/18 | the import losing the setting in 3 of 4 runs (traced, fixed) |
| 7-8 | 18/18 ×2 | final build |

# Target 10: Operating the cluster (upgrades, migration, backup and restore)

| | |
|---|---|
| **Roadmap tier** | X Ops |
| **Status** | Planned |
| **Depends on** | T1–T9 (the chart, all state in Postgres and blob storage) |

## Goal
An operator can run the cluster for years:
- **Upgrades:** upgrade it one controller at a time without stopping builds, while two versions share the store.
- **Migration:** move an existing single-controller Jenkins into it.
- **Backup:** back it up, and restore it into empty services.

## Design
### 1. Rolling upgrades with mixed versions
- **The store schema only grows.** Tables and columns are added with `if not exists`, and nothing is renamed or dropped in place. Migrations that need a rewrite are done by the new version in the background, keeping both shapes readable until every member runs it. Each member publishes its version in the membership table, and the cluster view shows it.
- **Records stay compatible:**
  - XML is read leniently, as Jenkins always has: unknown fields are ignored.
  - Queue records, placeholders and save points keep their fields; new ones are optional.
- **The rollout:** the StatefulSet's `RollingUpdate` replaces one controller at a time, gated by readiness. Running builds on the replaced controller are adopted (T4), and its queue items are claimed by others (T5).
- **Blocking mixed versions where they cannot coexist:** a version can declare a minimum peer version. A controller refuses to start while an older peer is alive, and says so in its log and the cluster view.

### 2. Migrating a file-based home (store plugin)
- **What:** a one-off import mode (`-Dio.jenkins.plugins.cnj.store.importHome=true`, or a CLI command) on one controller started with the old home. For every key the store owns, it uploads the local file if the store lacks it:
  - jobs and folders, with `nextBuildNumber`
  - users
  - nodes
  - root and descriptor XML
  - `credentials.xml`
  - runtime secrets
- **Completed builds:** become T3 records and snapshots, with their logs.
- **Idempotent:** existing keys are left alone and reported.
- **Afterwards:** the old home is no longer needed, and the S0 seed is taken from it (its `secret.key`, `secrets/` and `identity.key.enc`).

### 3. Backup and restore
- **Backup:** `pg_dump` of the database, then a copy of the blob container, in that order. Blobs are written before the records that refer to them (T3), so a dump taken first never refers to a missing blob. Blobs without a record are harmless, and are garbage collected.
- **Restore:** load the dump into an empty Postgres, copy the container into empty storage, and install the chart. The controllers start with all state.
- **Point in time:** with managed Postgres point-in-time recovery and blob versioning, restore to a time T. Running builds at T resume from their save points, or are recorded as lost.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | cloud-native-jenkins | `cnj/t10-m1-operations` | `cnj/t9-m3-chart` | membership versions and the minimum-peer check; home import; backup and restore scripts; tests |
| M2 | cloud-native-jenkins | `cnj/t10-m2-et10` | M1 | `ET10-operations.sh` (kind) |

## E-T10 acceptance experiment (kind)
1. **Rolling upgrade:** with Pipelines running on both controllers, `helm upgrade` to a new image (the same core with a newer plugin build).
   - Controllers are replaced one at a time.
   - Every running build completes.
   - During the rollout both versions are live members and the cluster view shows both.
   - Builds started during the rollout run.
2. **Refused downgrade:** a controller declaring a minimum peer version above an alive peer refuses to start, and says why.
3. **Import:**
   - A plain Jenkins (no store) with the following is imported by one controller:
     - a folder with jobs
     - completed builds with logs and test results
     - a user with an API token
     - a credential
     - a node
     - a global setting
   - A fresh second controller then has all of them, the token works, and the credential decrypts in a build.
   - Running the import again changes nothing.
4. **Backup and restore:**
   - Back up while builds run.
   - Delete the namespace.
   - Restore into a new namespace with empty Postgres and Azurite.
   - Everything up to the backup is there: jobs, history and consoles, users, credentials, nodes.

# ET18: Target 15 item 18 (point-in-time recovery)

**Result: PASS**, every check. Raw evidence: `raw/ET18/`
- `results.md`
- `restore.log`, the restore's Postgres log (`postgres.log`)
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET18-pitr.sh`
- **Setup:**
  - two tmpfs controllers and token auth
  - Postgres 16 archiving its WAL (`archive_mode=on`, `archive_timeout=5`) and Azurite, both on volumes (`compose.et18.yaml`)

| Scenario | Result |
|---|---|
| 1 A base backup (`pg_basebackup`), then work, then a named restore point T (`pg_create_restore_point`) while a Pipeline runs, then more work after T: a job deleted with its builds, another edited, a third created | ✅ the WAL is archived; at T the store had 30 keys |
| 2 Postgres restored to T into a fresh data directory (`recovery_target_name`), fresh controllers started on it, blob storage as it is now | ✅ Postgres recovered to T and was promoted. The deleted job is back with its configuration and build records, the edited job has its configuration as at T, the job created after T is not there, and a job untouched since T has its build and console. As expected without blob versioning, the deleted job's build is listed but its snapshot (and so its console) is gone |
| 3 Blob storage restored to T as well (standing in for Azure's blob versioning or point-in-time restore) | ✅ the deleted job's console is back. The Pipeline that was running at T was adopted from its save point and completed (SUCCESS), with what it had printed before T |

## What this shows
- **Postgres is restored with its own tools.** The cluster needs nothing of its own for a point-in-time restore of the database: base backup plus WAL, or a managed service's restore to a time, into a new database. Fresh controllers then start on it as on any database (JEP-255).
- **Builds running at T are not lost.** Their placeholders and save points are in the restored state, so a controller adopts them (or closes them as aborted) as after a failover (JEP-252).
- **Blob storage must be restorable to the same point.** The cluster deletes a job's blobs (snapshots, logs) when the job or build is deleted, and there is no deferred deletion (a decision recorded in Target 15). So without blob versioning, a restore to before a deletion brings back records whose files are gone. Azure Blob Storage's versioning and point-in-time restore for block blobs cover this. Azurite has neither, so the experiment restores a copy of Azurite's data taken at T.

## Bugs found while running ET18, now fixed (harness)
- **The WAL was not archived.** The archive volume belonged to root, so Postgres' `archive_command` failed and recovery ended before reaching T. The script now gives the volume to `postgres`, forces a WAL switch, and checks `pg_stat_archiver` before going on.
- **A failed recovery left the script waiting** for controllers that could not start. It now stops with the failure when Postgres did not recover to T.

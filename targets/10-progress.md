# Target 10: progress and branches

Design: [10-operations.md](10-operations.md). **Done means:** `validation/experiments/ET10-operations.sh` passes (on kind).

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t10-m1-import-api` | `cnj/t11-m1-xml-merge` | `BuildRecords.publish`/`storedNumbers`/`isShared` become Beta (for the importer) | ✅ |
| M1-M2 | cloud-native-jenkins | `cnj/t10-m1-operations` | `cnj/t9-m3-chart` | member versions and protocol, the minimum-peer check, `HomeImporter`, backup/restore scripts, persistent volumes, ET10 | ✅ ET10 18/18 ×2 |

## Log
- 2026-10-11: Target 10 planned and implemented. ET10 found the import losing settings: an initialiser racing with core's configuration loading (fixed: the import runs as the store is created). **ET10 passes 18/18** in two consecutive runs ([findings](../findings/ET10-operations.md) on `cnj/t10-m1-operations`).

## Follow-ups
- **Restore sweep:** builds running at backup time are absent after a restore; their placeholders should be adopted or closed.
- Point-in-time recovery with managed Postgres and blob versioning is documented in the plan, not exercised.
- Schema changes that need a rewrite (none so far) would need the background migration described in the plan.

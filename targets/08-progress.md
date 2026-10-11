# Target 8: progress and branches

Design: [08-history-at-scale.md](08-history-at-scale.md). **Done means:** `validation/experiments/ET8-history.sh` passes.

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | cloud-native-jenkins | `cnj/t8-m1-fingerprints-and-cluster-view` | `cnj/t7-m1-owner-routing` | `StoreFingerprintStorage` (three-way merge, remote changes applied), `ClusterView`, ET8 | ✅ ET8 22/22 ×3 |
| fix | core | `cnj/t6-m6-stateless-sessions` | | a Target 3 race: a build deleted while being published came back | ✅ 9b7309c391; `BuildRecordsTest` 8/8; ET3 48/48 |

## Log
- 2026-10-11: Target 8 planned and implemented. ET8 found:
  - the fingerprint cache not refreshed for changes made elsewhere (fixed)
  - usages lost by merging against another object's base (fixed: the base is per object)
  - a race in core where a build discarded while being published came back on the other controller (fixed in core)

  **ET8 passes 22/22** in three consecutive runs ([findings](../findings/ET8-history.md) on `cnj/t8-m1-fingerprints-and-cluster-view`).

## Follow-ups
- Per-item UUIDs (data keyed by job name across renames, e.g. in junit-sql) are not addressed; numbers are never reused (T3), which removed the E2 collision.
- The cluster view does not yet show each member's executors.

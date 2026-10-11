# Target 11: progress and branches

Design: [11-cross-cutting.md](11-cross-cutting.md). **Done means:** `validation/experiments/ET11-cross-cutting.sh` passes.

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t11-m1-xml-merge` | `cnj/t9-m1-provisioning-demand` | three-way merge on conflict (recursive into distinctly named parts); no reload while a save is pending; the version an object holds kept after a merge; raw reads only look (a Target 1 fix) | ✅ `XmlMergeTest` 9/9, `StaticStateApplierTest` 8/8, `ItemChangeApplierTest` 8/8, `XmlFileTest` 4/4 |
| fix | core | `cnj/t6-m6-stateless-sessions` | | reload an item changed elsewhere under its lock (a Target 1 fix) | ✅ c691f73648 |
| M2 | cloud-native-jenkins | `cnj/t11-m2-cross-cutting` | `cnj/t10-m1-operations` | `HomeWrites` (compatibility report at `/manage/cluster/compatibility`), ET11 | ✅ ET11 8/8 ×3 |

## Log
- 2026-10-11: Target 11 planned and implemented. ET11 found four lost-update paths:
  - a torn reload
  - a reload between a setter and its save
  - raw reads moving the version this controller holds (a Target 1 bug)
  - a second save before a merged reload

  All four are fixed in core. **ET11 passes 8/8** in three consecutive runs ([findings](../findings/ET11-cross-cutting.md) on `cnj/t11-m2-cross-cutting`).

## Follow-ups
- **Residual risk:** a field assigned during a reload (core setters do not lock); not observed in 30 rounds.
- A GitHub Actions workflow running the inventory check, once the branches are pushed.

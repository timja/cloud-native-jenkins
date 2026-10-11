# Target 12: progress and branches

Design: [12-remainder.md](12-remainder.md). **Done means:** `validation/experiments/ET12-remainder.sh` passes.

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t12-m1-remainder` | `cnj/t10-m1-import-api` | `RunningBuilds.handOver`/`takeOver`; builds nothing can resume closed as aborted (`BuildRecords.publishLost`), listed by the closing controller too; `BuildRecords.LateFiles` (completed builds whose directory changed are republished; other controllers re-materialise them); snapshots of saved objects without materialising them (`XmlFile.getPath`, a Target 11 fix) | ✅ `BuildRecordsTest` 10/10, `ItemChangeApplierTest` 8/8, `StaticStateApplierTest` 8/8, `XmlFileTest` 4/4, `XmlMergeTest` 9/9 |
| M2 | cloud-native-jenkins | `cnj/t12-m2-remainder` | `cnj/t11-m2-cross-cutting` | `Handover` (Pipelines waiting for a label only another controller has); owner routing by origin; forwarded requests bounded at 20 s; save point clean-up races (Target 4 fixes); ET2 and ET4 brought up to date with shared builds; ET12 | ✅ ET12 10/10 ×2 |

## Log
- 2026-10-11: Target 12 implemented. **ET12 passes 10/10** in two consecutive runs ([findings](../findings/ET12-remainder.md) on `cnj/t12-m2-remainder`). The final regression sweep on the same build found:
  - a forwarded request hanging on a paused owner (T7)
  - save points left behind by two races (T4)
  - item `config.xml` copies left in the home by the T11 snapshot listener (T11)
  - two harnesses (ET2, ET4) that predated shared builds and the restore sweep

  All are fixed.
- 2026-10-11 (after manual testing on kind): a build running on another controller was missing from the Builds card ("No builds") and the job's icon was not animated; both now show it (core 64036f3468, `RunningBuildsTest`).
- 2026-10-11 (manual testing on kind): a plugin installed from the Plugins page went to one controller only, so a Mock Agent form broke when its requests reached the other. The plugin set is now enforced (`FixedPluginSet`, 409 with a message pointing to the image), and mock-slave is in the kind image.

## Follow-ups
- Placement by label moves the whole build. Running only a `node` body on another controller's agent would need remote executors.
- The late-files sweep looks at builds this controller published in the last hour and still holds in memory (by default every minute); files added to older builds are republished only when the build is saved again.

# ET11: Target 11 acceptance (concurrent edits merged; nothing unshared under the homes)

**Result: PASS**, 8/8 checks, in three consecutive runs on the final build. Raw evidence from the last run: `raw/ET11/` (`results.md`, controller logs, `BUILD`).

- **Script:** `validation/experiments/ET11-cross-cutting.sh`
- **Setup:** two tmpfs controllers, S0, Postgres, Azurite, API-token auth.
- **Build:**
  - core `cnj/t10-m1-import-api`, the top of the core stack, including `cnj/t11-m1-xml-merge`
  - plugin on `cnj/t11-m2-cross-cutting`

| Criterion | Result |
|---|---|
| 1 Concurrent edits (ET6 10, strictly) | ✅ in 10 rounds per run, A and B saved different global settings at the same moment. Both survived every round, on both controllers: 30/30 rounds over the three final runs, almost every write merged. The stored `config.xml` has both. ET6 met this criterion only partly |
| 2 The same setting | ✅ A and B set the system message at the same moment. Both controllers agree on one of the two values, and the conflict is counted |
| 3 Job configuration | ✅ in 3 rounds per run, A changed a job's description while B added a trigger. Both survived every round, on both controllers |
| 4 Inventory check | ✅ after a standard scenario (freestyle builds with artifacts, fingerprints and JUnit results, a Pipeline, a user, a credential, a node), neither home holds anything the cluster does not share |
| 5 Compatibility report | ✅ a deliberate write under A's home outside the store (`foo-bad/state.xml`) is reported at `/manage/cluster/compatibility`, and it is the only finding. *Deviation from the plan:* naming the responsible plugin works only for root `<class>.xml` files, all of which are per-controller by design, so the report gives the path pattern and count |

## How it works
- **Three-way merge (core, `XmlMerge`):**
  - a routed write whose compare-and-set fails is merged against the content the object was last loaded from or saved as, part by part (the root's children)
  - parts changed on both sides are merged recursively when they are made of distinctly named parts (e.g. `<properties>`); otherwise this side's version is kept and a conflict counted
  - the object is then reloaded with both changes
- **Reloading only what is in sync (core):**
  - objects are snapshotted (their own serialisation) when saved or reloaded
  - a change from elsewhere to an object that differs from its snapshot (a save is pending) is put off, so that the pending save conflicts and merges first
- **`HomeWrites` (store plugin):** files written under the home since startup that the store does not own, outside what is local by design, grouped by path pattern. The page is at `/manage/cluster/compatibility`, with an API; it serves as the inventory check and the ecosystem report.

## Bugs found by ET11, now fixed (core)
- **A raw read hid a change made elsewhere** (a Target 1 bug).
  - **Cause:** every routed read recorded the stored version as the one this controller holds, including reads that do not load the object (REST `GET config.xml`, `getFile` snapshots, encoding sniffing). The next save of the stale object did not conflict and overwrote the other change, and the merge base moved too.
  - **Fix:** raw reads only look; `read()` and `unmarshal()` record the version.
- **A save during a reload wrote a mix of the two versions** (a Target 1 bug). The applier reloaded an item into the live object without its lock. It now holds the item's lock, as `AbstractItem.save` does.
- **A reload between a setter's change and its save lost the change.** It is now put off while a save is pending; for `config.xml`, the check and the reload happen under the `Jenkins` lock.
- **A second save before a merged reload wrote the stale part back.** After a merge, the controller now keeps the version and content its object has, so that save conflicts and merges again.
- **Merging parts changed on both sides:** recursion into distinctly named parts, found by ET9 (the shared admin user).
- **Plugin:**
  - the compatibility page was not routable (`@StaplerAccessibleType`)
  - per-controller API token statistics and core's default `userContent/readme.txt` are local by design

## Residual risk
- Core's setters assign fields without a lock, then save. A field assigned *during* a reload (milliseconds) can still be lost: the reload overwrites it, and the save then writes the reloaded value.
- In 30 rounds of deliberately simultaneous edits over the final runs it did not occur; earlier builds lost 1-2 rounds in 10 to the wider windows fixed above.
- Closing it fully would need setters that save under the object's lock.

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | stopped at 3 | a lost trigger: a torn reload (fixed); the compatibility page not routable (fixed) |
| 2-3 | 3-4 | per-controller token statistics and readme (allowed); reloads during setters (fixed: snapshots); scenario 1 at 8/10 |
| 4-7 | 7/8 | stale descriptions: raw reads moving the seen version (fixed) |
| 8-9 | 7-8/8 | the second save after a merge (fixed); scenario 1 at 9/10 (check and reload under one lock) |
| 10-12 | 8/8 ×3 | final build |

## Target 15
- **Scenario 3 lost a description in one run** (core, fixed). A merge that wrote nothing new kept the version it had read, but the object in memory still had only its own side, so its next save overwrote the other change. A no-op merge now keeps the version and base read before, so the next save merges again. Deterministic test: `ItemReloadTest.twoSavesHereWhileAChangeElsewhereIsNotAppliedYet`.
- **Field-by-field reloads (item 8)**: an item changed elsewhere is reloaded into a scratch instance, and only the changed fields are copied over, except those changed here since. The core setters on `Jenkins` assign under its lock.
- Scenario 3 now passes 3/3: the description set on A and the trigger set on B both survive on both controllers.

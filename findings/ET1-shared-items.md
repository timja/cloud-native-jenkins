# ET1: Target 1 acceptance (shared item configuration)

**Result: PASS.** 0 failures on two consecutive runs. Raw evidence: `raw/ET1/`
- `results.md`
- `store.txt`, `events.txt`
- controller logs
- `BUILD` (the exact commits under test)

- **Script:** `validation/experiments/ET1-shared-items.sh`
- **Build:** `validation/experiments/ET1-build.sh` builds everything from local branches:
  - core `cnj/t1-m4-item-change-applier`
  - cloudbees-folder `cnj/t1-m2-folder-children`
  - `impl/persistence-store-postgresql`
- **Setup** (`validation/compose.et1.yaml`):
  - two controllers with **tmpfs** `JENKINS_HOME`s
  - byte-identical JCasC (`jenkins/casc-et1`) and seeded key material (S0, [E8](E8-casc-secret-sync.md))
  - Postgres 16 as the item store

| Scenario | Checks | Result |
|---|---|---|
| 1 Admin lifecycle: create (job, folder + child), edit on the other controller, rename folder, delete | 7 | ✅ |
| 2 Runtime config: `properties()` in a build on A defines a parameter and a cron trigger; B sees both. A's local home has **0** `config.xml` files | 3 | ✅ |
| 3 Multibranch: index on A → branches on B; add branch → B; delete branch → pruned on B; index on B → no duplicates or spurious deletes on either | 5 | ✅ |
| 4 Indexing on A for 2 minutes while builds with `properties()` run on B | 4 | ✅, 0 conflicts (all writes were identical content, deduplicated) |
| 4b 80 builds of one job, split across A and B, each rewriting its config with a different value | 1 | ✅ A, B and the store converge on the same value; 79 version bumps; 0 conflicts |
| 6 Re-index with no SCM change | 1 | ✅ **0** stored versions bumped (the no-op write works) |
| 5 Destroy B with its tmpfs home and start a fresh B: all items load from the store, the deleted item stays deleted | 5 | ✅ |

## Observations
- **No conflicts in practice.** The applier, driven by NOTIFY, applies a change on the other controller well within the 0.5 s between conflicting writes, so CAS never saw a stale version. Conflict *detection* is covered deterministically by core `PersistenceStoreItemConfigTest.conflictingWriteIsDetected` and plugin `PostgresPersistenceStoreTest.compareAndSet`. A forced-latency variant (e.g. a delay in the applier) would exercise the `warn` path in the harness; that belongs with M5.
- **Write amplification is handled.** E9 showed indexing rewrites every branch config each scan. With hash-based no-op writes, a 2-minute indexing loop plus builds produced 25 `WRITTEN` events in total, and an unchanged re-index produced none.
- **Out of scope here, but visible:**
  - **Build numbers diverge** (R2): `race` ran on both controllers with independent `nextBuildNumber`s.
  - **Build history is local** (R5): each controller lists only its own builds of shared jobs.
  - **3d spurious builds** (R2): indexing on B rebuilt branches that A had already built, because SCM revision state is per controller.

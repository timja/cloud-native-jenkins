# ET8: Target 8 acceptance (build history and records at cluster scale)

**Result: PASS**, 22/22 checks, in three consecutive runs on the final build. Raw evidence from the last run: `raw/ET8/`
- `results.md`
- `cluster-*.json`: both controllers' cluster views
- controller logs from before and after the controllers were replaced, `BUILD`

The ET3 regression passes on the same build (48/48).

- **Script:** `validation/experiments/ET8-history.sh`
- **Setup:**
  - two tmpfs controllers, S0, Postgres, Azurite, API-token auth
  - `agent1` connected to A
- **Build:**
  - core `cnj/t6-m6-stateless-sessions` (`9b7309c391`, including the build-records fix below)
  - plugin on `cnj/t8-m1-fingerprints-and-cluster-view`

| Criterion | Result |
|---|---|
| 1 Fingerprints | ✅ the same file fingerprinted by a job on A and a job on B: both controllers list both usages. Nothing is written under `fingerprints/` in either home. After both controllers were destroyed, a fresh one lists both usages |
| 2 Concurrent updates | ✅ builds of both jobs on both controllers at once all recorded the same fingerprint, and every build is a usage on both controllers (e.g. fpa:1-6, fpb:1-4). *Deviation from the plan:* the plan asked for 10 builds per job, but requests without parameters fold in the queue (as on one controller), so the check is that every build which ran is a usage |
| 3 Test results | ✅ a JUnit build completing on A shows the same report on B 0.1-0.2 s after completion (3 tests, 1 failed). A build deleted on B has its results gone on both controllers |
| 4 Discarders | ✅ `numToKeep=3`, with 8 builds alternating between the controllers: both keep exactly #6-8, and no discarder errors are logged. A build running on A while B discarded (`numToKeep=2`) completed, and its record was published. Then, as the oldest of three, it was discarded, and both keep #2-3 |
| 5 Cluster view | ✅ `/manage/cluster/api/json` on either controller lists both members alive, the build running on A with its owner, and agent1's channel on A, and the two views agree. After A was killed, B's view showed only B alive within the membership timeout |

## How it works (store plugin)
- **`StoreFingerprintStorage`:** fingerprints are entries in the store (`fingerprints/<md5>.xml`, the same XML as files), selected automatically when the store is active (unless another storage was configured).
  - **A save** is a three-way merge of usages against what the saving object last read, written with compare-and-set and retried. Usages added elsewhere are kept, and usages trimmed here stay trimmed.
  - **A change made elsewhere** updates the object Jenkins holds for that fingerprint.
- **`ClusterView`:** a management link, reading members, placeholders and channels from the store.

## Bugs found by ET8, now fixed
- **Fingerprints were stale on the other controller** (run 1, scenario 1). Jenkins caches fingerprint objects, and nothing told the cache about changes made elsewhere. A `RemoteChangeHandler` now updates the held object.
- **Usages were lost under concurrency** (run 1, scenario 2). The merge base was kept per fingerprint id, but Jenkins can hold several objects for one fingerprint (its cache is weak). An object merged against another object's newer base dropped that object's additions as if they had been removed. The base is now per object.
- **A deleted build came back on the other controller** (runs 1, 5, 6; a Target 3 race in core).
  - **Cause:** a build discarder runs as soon as builds complete, so it could delete a build whose record and snapshot were still being published. The late write brought the record back, and the other controller listed a build the deleting controller no longer had.
  - **Fix (core, on `cnj/t6-m6-stateless-sessions`):** deleted builds are remembered, and a publication finding its build deleted, before or after writing, removes what it wrote. `BuildRecordsTest` has a test for it.
- **Harness:**
  - the scenario 2 expectation (folding)
  - scenario 4b checked that a build survived its own discarder (with `numToKeep=1` it is rightly deleted on completion)
  - the logs are now kept before the controllers are replaced

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | 15/20 | the stale fingerprint cache and the per-id merge base (fixed); the resurrected build; 2 harness checks |
| 2-6 | 21-22/22 | the resurrected build in 3 of 5 runs, then caught with logs (fixed in core) |
| 7-9 | 22/22 ×3 | final build; ET3 48/48 |

# Target 8: Build history and records at cluster scale (R5 remainder)

| | |
|---|---|
| **Roadmap tier** | R5 (fingerprints, test results, build discarders, cluster-wide views and REST) |
| **Status** | **Done** (2026-10-11): ET8 passes 22/22, see [08-progress](08-progress.md) |
| **Depends on** | T3 (build records), T5 (cluster singletons), T7 (owner routing) |
| **Evidence** | [E2](../findings/E2-disposable-controller.md) (junit-sql identity collision), [E7](../findings/E7-ecosystem-scan.md), ET3 |

## Goal
Everything a user can see about past builds is the same on every controller, and the housekeeping of that history runs once:
- fingerprints, from builds anywhere
- test results, as soon as a build completes
- build discarding, per job and global
- a cluster view of members, executors and running builds, in the UI and in REST

## Today
- **Fingerprints:** `FileFingerprintStorage` writes through `AtomicFileWriter` directly, so they are not routed to the store and stay on the controller that recorded them. Usages recorded on B are invisible on A, and fingerprints are lost with the controller.
- **Test results:** junit-sql works (T3). Its per-controller cache is invalidated on publish and finalisation, and with T7 a running build is only ever viewed on its owner. What remains is validating freshness on the other controller right after completion.
- **Build discarders:**
  - A job's discarder runs on the controller completing a build, so builds of one job on A and B both run it against the shared history.
  - The global discarder is a cluster singleton (T5).
  - Concurrent discarding, and builds still running elsewhere, have not been tested.
- **Cluster views:** the executor widget, `/computer` and the load statistics show only the local controller. Nothing shows the members.

## Design
### 1. Fingerprints in the store (store plugin)
- `StoreFingerprintStorage`, a `FingerprintStorage` keyed `fingerprints/<md5>.xml` in `jenkins_store`. The XML is the same as `FileFingerprintStorage`'s.
- **Save:** read the stored version, merge, then compare-and-set; retry on conflict.
  - The merge unions the usages (`RangeSet` per job) and keeps the earliest timestamp and the original.
  - Two builds on two controllers recording the same file both keep their usage.
- **Cleanup (`iterateAndCleanupFingerprints`):** a cluster singleton. It writes with compare-and-set too, so a usage added meanwhile is not lost.
- **Selection:** the storage is selected automatically when the store is active, and JCasC can still choose another.

### 2. Test results (validation)
- On the other controller, a build's test report and the job's trend are correct within 2 s of completion.
- Results of a deleted build are gone from both controllers.

### 3. Build discarders
- **Per-job discarders** run where the build completes (as today). Deletes are idempotent: deleting a build already deleted elsewhere is a no-op, not an error.
- **Builds running elsewhere** are never discarded: placeholders are not runs. The test is that a discarder on B with `numToKeep=2` does not touch A's running build.
- **The global discarder** stays a cluster singleton.

### 4. Cluster view (store plugin)
- `/manage/cluster` (an administrative monitor link) and `/cluster/api/json` list:
  - the live members, with name, internal URL, heartbeat and start time
  - each member's executors, read from the queue claims and placeholders
  - running builds, with their owner
  - agent channels (T7), with their holder
- The data comes from the store (members, leases, placeholders, channels), so any controller answers.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | cloud-native-jenkins | `cnj/t8-m1-fingerprints-and-cluster-view` | `cnj/t7-m1-owner-routing` | `StoreFingerprintStorage`, cluster view, discarder idempotence (core change only if needed); tests |
| M2 | cloud-native-jenkins | `cnj/t8-m2-et8` | M1 | `ET8-history.sh` |

## E-T8 acceptance experiment
1. **Fingerprints:** a job on A and a job on B both fingerprint the same file. On both controllers, the fingerprint page lists both usages. Both survive destroying the controllers.
2. **Concurrent fingerprint updates:** 10 builds on A and 10 on B record the same file at once. All 20 usages are present.
3. **Test results:** a JUnit build completing on A shows the same report on B within 2 s. A build deleted on B has its results gone on A.
4. **Discarders:**
   - `numToKeep=3`, with builds alternating between controllers: exactly the last 3 remain on both controllers, and no errors are logged.
   - A build running on A is not discarded by B.
5. **Cluster view:** on either controller, `/cluster/api/json` lists both members, the build running on A, and agent1's holder. After A dies, it is gone within the membership timeout.

# ET2: Target 2 acceptance (cluster-wide build numbers, exactly-once triggers)

**Result: PASS**, 0 failures. Raw evidence: `raw/ET2/`
- `results.md`
- `builds.jsonl`: every build seen on either controller
- `claims.txt`, `build-numbers.txt`
- controller logs, `BUILD`

The ET1 setup was re-run on the same build: 27/27 pass (`raw/ET1/`).

- **Script:** `validation/experiments/ET2-scheduling.sh`
- **Setup:** two tmpfs controllers, S0, Postgres, API-token auth
- **Build:** core `cnj/t2-m2-trigger-coordination`, cloudbees-folder `cnj/t1-m2-folder-children`, plugin on `cnj/t2-m3-postgres-coordination`

| Scenario | Result |
|---|---|
| 2 20 + 20 concurrent manual builds on A and B | ✅ numbered exactly 1..40, none duplicated (20 ran on each) |
| 1 `* * * * *` cron with both controllers up | ✅ one build per claimed tick (5/5); all builds on the owner |
| 3 SCM polling every minute | ✅ initial build, then exactly one build per commit, none without changes; all on the owner |
| 4 multibranch with 1-minute periodic indexing | ✅ existing branches never rebuilt; a new branch built exactly once |
| 5 SIGKILL of the cron owner | ✅ the survivor took over (2 builds within 2.5 min); still one build per tick |
| 6 fresh controller rejoins | ✅ cron kept firing; one build per tick across failover and rejoin (21 builds, 21 ticks) |
| 7 continuity | ✅ a fresh controller continued at #41; after a rename, #42 |

## Observations
- **Cron catch-up.** `Trigger.Cron` runs any minute it has not yet processed. A job created seconds before a minute boundary can therefore fire two ticks within one wall-clock minute. This is stock Jenkins behaviour; each tick still ran once cluster-wide. The invariant checked is one build per claimed tick, recorded in `jenkins_trigger_claims`.
- **Failover gap.** With a 20 s membership timeout, about one tick is missed after an ungraceful kill, as designed: a missed tick is preferred over a doubled one. A graceful stop removes the member row immediately.
- **Shared indexing state.** Periodic indexing can run on a different controller from the one that did the initial index without rebuilding branches, because `scm-revision-hash.xml` lives in the store. This fixes the ET1 observation "indexing on B rebuilt branches".
- **Harness bugs found while running ET2:**
  - Concurrent POSTs sharing one cookie jar broke crumbs. Fixed by switching to API-token auth.
  - A global `core.fsmonitor` setting hung `git` in non-interactive runs. Disabled in the scripts.

## Target 15 (item 22): the SCM polling log
| Check | Result |
|---|---|
| The polling log reads the same on both controllers | ✅ requests for a job's polling log go to the controller that last polled it (from the trigger claims), which has the log |
| Polling never built the same change twice, and stayed on one controller | ✅ |

**Fixed:** the polling log was on whichever controller polled, so the other showed none.

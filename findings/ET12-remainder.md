# ET12: Target 12 acceptance (placement by label, files added to completed builds, the restore sweep)

**Result: PASS**, 10/10 checks, in two consecutive runs on the final build. Raw evidence from the last run: `raw/ET12/` (`results.md`, controller and agent logs, `BUILD`).

- **Script:** `validation/experiments/ET12-remainder.sh`
- **Setup:**
  - two tmpfs controllers, Postgres, Azurite, API-token auth
  - an inbound agent `lin` (`jenkins/inbound-agent:latest-jdk21`) connected to B only
  - the late-files sweep every 5 s (`-Djenkins.persistence.BuildRecords.lateFilesPeriodSeconds=5`; default 60)
- **Build:**
  - core `cnj/t12-m1-remainder`, the top of the core stack
  - plugin on `cnj/t12-m2-remainder`

| Criterion | Result |
|---|---|
| 1 Placement by label | ✅ `node('lin') { sh … }` started on A completed on `lin`, by B: A handed it over after 15 s of waiting for a label only B has. A Pipeline that first ran a `node('built-in')` block on A and then needed `lin` also completed on B, with both parts in one console. *Deviation from the plan:* the plan expected that second case only to be reported; the handover resumes it from its save point just as well |
| 2 Late build files | ✅ a file written into a completed build's directory on A (as a plugin adding an attachment would, without saving `build.xml`) appeared in B's copy of that build within the sweep period |
| 3 Restore sweep | ✅ both controllers were SIGKILLed while a Pipeline slept and a freestyle build ran, and fresh controllers (new members, empty homes) started after the membership timeout, as after a restore. The Pipeline was adopted from its save point and completed, with "after its controller was gone" in its console. The freestyle build was closed as ABORTED, with the reason in its console, on both controllers. *Deviation from the plan:* the experiment replaces the backup and restore by killing the whole cluster: both leave placeholders whose owner is no live member, which is what the sweep acts on. ET10's backup and restore is unchanged |
| 4 Routing after the restart | ✅ requests for the old controller's builds were answered locally, not forwarded to the new controller of the same name, and no routing warnings were logged |

## How it works
- **Handover (store plugin, `Handover`):**
  - every 5 s a controller looks at its Pipeline `node` queue items (`PlaceholderTask`) waiting for at least 15 s (`io.jenkins.plugins.cnj.store.Handover.waitSeconds`)
  - it hands one over if no executor or cloud here can serve its label, but an agent with that label is connected to another live controller (the T7 channel table)
  - to hand over, it ships the latest save point, writes `<job>/running/<N>.handover` naming the target, and releases its lease (`RunningBuilds.handOver`)
  - the target takes the build over (`RunningBuilds.takeOver`) and resumes it from the save point, where the `node` step asks for an executor again, now one it has
- **Late files (core, `BuildRecords.LateFiles`):** every minute, completed local builds whose directory changed since they were published are republished. A controller holding a materialised copy of a build that was republished elsewhere drops it and materialises it again.
- **Restore sweep (core, `RunningBuilds.removeStale`):**
  - a placeholder whose owner is not a live member, and which no controller adopts, is closed as aborted (`BuildRecords.publishLost`): a minimal ABORTED record and a console saying why
  - Pipelines with a save point are adopted as in T4
  - this covers builds of a restored cluster, whose owners never were members of it
- **Routing by origin (store plugin):** owner routing now looks up the live member with the placeholder's origin, not a controller with the owner's name. After a restore, or a restart with the same pod name, a request for a dead incarnation's build is answered locally instead of being forwarded to its namesake.

## Bugs found while running ET12 and the final regressions, now fixed
- **A build closed as aborted was missing on the controller that closed it** (core). Its own write raises no event there; it now brings itself up to date as it would for a record from elsewhere.
- **A forwarded request to a paused owner hung for 5 minutes** (a Target 7 bug, found by ET4 scenario 4).
  - **Cause:** a paused JVM accepts the TCP connection but never answers.
  - **Fix:** forwarded requests now wait at most 20 s for the owner's response headers (`io.jenkins.plugins.cnj.store.Forwarding.timeoutSeconds`), then are served locally, where the build has usually been adopted by then. Streamed bodies (live consoles) are not limited.
- **Save points left in blob storage** (Target 4 bugs, ET4 scenario 7):
  - a fenced owner woken from a pause could upload flow nodes after the new owner had completed the build and deleted its save points; it now deletes them itself once the build has no running placeholder
  - a save point shipping while the build completed (an adopted build approved a second after adoption) was uploaded after the cleanup; shipping and the cleanup now exclude each other
- **Every save of a job left its `config.xml` in the home** (a Target 11 bug, ET1 2c). The listener recording what an object was saved as asked `XmlFile.getFile()` for the store key, which materialises the stored content to the local file. It now uses `XmlFile.getPath()`, which does not copy. ET11's inventory check did not see it because materialised snapshots are local by design.
- **Harness:**
  - ET2 predated Target 3: with shared build records every controller lists every build, so ET2 counted each build twice. Builds are now attributed to the controller that wrote their record.
  - ET4 scenario 6 now expects the `PERFORMANCE_OPTIMIZED` build to be closed as aborted rather than vanish.
  - `wait_up` writes container logs and a thread dump on a boot timeout.

## Regression sweep on the final build
ET1-ET12 all pass on this build (core `cnj/t12-m1-remainder` 1f5de855f3, plugin `cnj/t12-m2-remainder`):

| Experiment | Result | Note |
|---|---|---|
| ET1 | ✅ | 2c failed twice before the snapshot listener fix |
| ET2 | ✅ (rerun) | harness: shared builds attributed to their writer; multibranch records are under `branches/` |
| ET3, ET5-ET8, ET11 | ✅ | |
| ET4 | ✅ | after the forwarding timeout and the two save point fixes; scenario 6 now expects the closed build |
| ET9 | ✅ | |
| ET10 | ✅ (rerun) | the old plain controller's built-in node went offline for low disk on the kind node (96% used), so its second freestyle build never ran. The harness now sets low disk thresholds on it; the disk was also freed |

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | 7/10 | a Groovy quoting error in the harness's `lab1` script; the closing controller not listing its aborted build (fixed) |
| 2-3 | 10/10 ×2 | final build |

## Target 15
| Scenario | Result |
|---|---|
| 1b (item 13) Parallel branches, one on A's built-in node and one on an agent only B has | ✅ completes with both branches; handed over once, after the branch on A finished, not back and forth |
| 1c (item 14) `build(wait: true)` on A for a downstream build that runs on B | ✅ the downstream build ran on B (not pinned to A), and A's build got its result |
| 1d ... and A killed while it waits | ✅ B adopts the waiting build, which completes with the downstream result |

**Fixed:** in 1c, the waiting build failed with "Build of down was cancelled". Its local queue item left as cancelled when B took the downstream build. `SharedQueue.takenElsewhere` (core) lets pipeline-build-step tell "taken by another controller" from "cancelled", and the result comes through the delivery.

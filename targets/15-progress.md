# Target 15: progress and branches

Design: [15-remaining.md](15-remaining.md). **Done means:** every item is implemented, its experiment or tests pass, and the whole suite (ET1–ET22 on Docker Compose, ET9, ET10 and ET13 on kind) passes in one sweep.

## Branch stack
| Repo | Branch | Based on | Scope |
|---|---|---|---|
| core | `cnj/t15-remaining` | `cnj/t14-m1-resilience` | Items 1–3 (plugin-manager hooks), 6–9, 11, 12, 14, 17 (`deleteAll`, stored permalinks), 20 (`strict-ui`), 21, 27, and the fixes found by the experiments (see below) |
| cloud-native-jenkins | `cnj/t15-remaining` | `cnj/t14-m1-resilience` | Store plugin: items 1–5, 13, 16–20, 22, 26; chart (18, 27); experiments ET17–ET22 and the new scenarios of ET2, ET4, ET8, ET9, ET12, ET15, ET16 |
| cloudbees-folder | `cnj/t15-remote-child` | `cnj/t1-m2-folder-children` | Item 21: `RemoteItemGroup` |
| lockable-resources | `cnj/t15-cluster-ui` | `cnj/t5-m4-cluster-locks` | Item 15: reservations and resources made on the resources page |
| pipeline-build-step | `cnj/t15-remote-completion` | upstream `master` | Item 14: `BuildTriggerDelivery`; a downstream item taken by another controller is not a cancellation |
| configuration-as-code | `cnj/t15-keep-undeclared-nodes` | upstream `master` | Item 10: `keepUndeclaredNodes` |
| junit-sql-storage | `cnj/t15-follow-renames` | `cnj/t3-m5-remote-delete` | Item 9: results follow their job when it is renamed or moved |

## Items
| # | Item | Status | Evidence |
|---|---|---|---|
| 1 | Install progress through a load balancer | ✅ | ET19 |
| 2 | Cluster notice on the plugin manager's pages | ✅ | ET19, ET21 |
| 3 | CLI `install-plugin` | ✅ | ET19 |
| 4 | WebSockets; CLI following builds elsewhere | ✅ | ET20 |
| 5 | Failover time | ✅ 5.0–5.6 s after `SIGKILL`, 5.2–5.4 s after a clean stop; a 12 s pause not taken for a death | ET15 4 |
| 6 | Plugin files in item directories (write-back, sub-items, opt-in sync) | ✅ | `LocalStateTest`, `ItemFilesTest` |
| 7 | `@PerController` / `LocalFiles` | ✅ | `LocalStateTest` |
| 8 | Field-by-field reloads | ✅ | `ItemReloadTest`, ET11 |
| 9 | Item UUIDs; junit-sql follows renames | ✅ | `ItemUuidTest`, junit-sql test |
| 10 | JCasC `keepUndeclaredNodes` | ✅ | `KeepUndeclaredNodesTest`, ET6 |
| 11 | Files added to old builds | ✅ | `BuildRecordsTest`, ET12 |
| 12 | Freestyle console kept when its controller dies | ✅ | ET15 5, ET4 6, `BuildRecordsTest` |
| 13 | Handover of builds with several `node` blocks | ✅ handed over once | ET12 1b |
| 14 | `build(wait: true)` across controllers | ✅ | ET12 1c, 1d |
| 15 | lockable-resources reservations | ✅ | `ClusterLocksTest` |
| 16 | Executors and queue on the cluster page | ✅ | ET19 4, ET21 |
| 17 | Scale: three controllers, 200,000 build records, an indexing storm | ✅ started together in 18 s; a job page with 20,000 builds 0.6–0.8 s cold; an indexing storm with a controller killed: every branch built once | ET17 |
| 18 | Managed services, point-in-time recovery | ✅ | ET18 |
| 19 | Background schema migrations | ✅ | `SchemaMigrationsTest` |
| 20 | `podTemplate` Pipelines under caps; conflict monitor and `strict-ui`; browser tests | ✅ | ET9 4b, `StrictConflictsTest`, ET21 |
| 21 | A folder child created elsewhere added without reloading the folder | ✅ | `RemoteChildTest` |
| 22 | SCM polling log routed to the controller that polled | ✅ | ET2 |
| 23 | Workspaces across an adoption | ✅ | ET4 2b |
| 24 | Workspaces on Kubernetes pods (`stash`) | ✅ stashes through the artifact manager (azure-artifact-manager, configured by the chart and the harness), found by the adopter on a new pod | ET9 4b |
| 25 | Coverage reports on every controller | ✅ | ET8 3b |
| 26 | A controller without the cluster's keys refuses to start | ✅ | ET20 4 |
| 27 | Plugin files in user directories (the avatar plugin) | ✅ | `ItemFilesTest`, ET22 |

## Bugs found by the experiments, fixed
- **Core:**
  - `build -s -v` lost the end of a freestyle log when its shipped copy was deleted first (ET20).
  - `build(wait: true)` failed as "cancelled" when another controller took the downstream build: `SharedQueue.takenElsewhere` lets pipeline-build-step tell (ET12 1c).
  - A fresh controller resolved permalinks by going through every build: 90 s for a job page with 20,000 builds, more than 180 s on a busy controller. Permalinks are now cached in the store (`StoredPermalinks`): 1.2–1.5 s (ET17).
  - A user changed in memory and never saved never got changes made elsewhere again. JCasC re-hashing a declared password on each controller caused it. It is now put off a few times, then applied, as items are (ET17).
  - A merge that wrote nothing new let the next save from the stale object overwrite the other change (ET11 3).
  - The CLI could ask to follow a build another controller had just started before that build's placeholder arrived, and printed no log (ET20 3).
  - A controller restarted with its home resumed a build another controller had adopted while it restarted, and both ran it. With failover in about 5 s, adoption beats a restart. The reclaim now asks the store, not its cache (ET13 2).
- **Harness and chart:** no artifact manager was configured, so stashes stayed in the build directory and an adopted Pipeline lost them ("No such saved stash", ET9 4b). The chart and the harness now configure azure-artifact-manager (its credential made once at startup: declaring it in JCasC would replace the cluster's credentials on every start). An artifact manager is required: a controller without one shows an administrative monitor and logs an error at startup (store plugin, `ArtifactManagerRequired`).
  - Multibranch children were not recognised as items by the applier (ET1).
- **Store plugin:** an owner dying between uploading a save point's flow nodes and committing it left an adopter nodes newer than the program it resumed (a `ClassCastException` on resume, ET9 4b). Nodes are now kept per save point, and an adopter applies only those up to the committed one.
- **Store plugin, regressions caught by the experiments the same day:**
  - The first per-save-point nodes change listed `running/workflow`, which returns only `workflow/…`, never `workflow.<epoch>.<seq>/…`. Adopted Pipelines found no nodes and did not resume (ET4 1 and 3, ET12 1 and 3). The adopter now lists `running/`.
  - The `ArtifactManagerRequired` startup check ran after `COMPLETED`, so the reactor never reached it (JENKINS-37759). It now runs after `JOB_CONFIG_ADAPTED`.
- **Core:** a build whose controller died after it completed, but before it published its record, was adopted and resumed from an early save point. It never ran on, and its branch had no record (ET17, 1 to 11 of 300 branches when C is killed mid-storm). The owner now records the result in the placeholder on completion, and such a build is closed with its result, updating the shared permalinks as its owner would have.
- **Store plugin:** concurrent scans of one multibranch project on several controllers built its new branches twice. Folder computations now run on one controller at a time per folder (`ClusterComputations`, ET17).
- **Core, a regression from the fix above, caught the same day:** recording the result raced with a Pipeline's last save point, which updates the same placeholder at completion. Both compare-and-set the same version, and the save point, failing, took the build for lost: the controller that still owned it stopped it and never published it (ET17, `mb24/main`). A build's placeholder is now updated by one thread at a time.
- **branch-api (new fork branch `cnj/t15-first-build-revision`):** an indexing schedules a branch's build, then records its revision as built. An indexing on a controller that died in between left the next one (on another controller) to find changes and build the branch again (ET17, `mb16/b7` built twice). The shared queue keeps the scheduled build alive, so the cluster meets this far more often than one controller after a crash. A running automatic build now records the revision it builds, if none is, and an indexing finding none takes the last automatic build's.
- **Store plugin:** a deleted job's started items stayed in the shared queue, and a job created again under its name had them requeued as "started by a gone controller", building a stray #3 (ET3 8). The started items of a deleted or renamed item, and of the items inside it, are now cancelled.
- **cloudbees-folder:** the load progress of a fresh home printed percentages like 27,600 %.
- **Harness:** many. Also: ET8 4b required a published record for a build its own discarder deletes on completion (deleted while publishing, so rightly never written); it now also accepts its owner removing the placeholder. The sweep now deletes the kind release before the Compose experiments: with it running, the Docker VM's swap filled, and ET17's restarted C did not come up in 600 s (its thread dump: 97 s of CPU in 1,600 s, loading bouncycastle). ET17 records the VM's swap use. Most were `docker logs … | grep` failing under `pipefail`, checks that matched the shell's trace of a command as well as its output, compose `extends` bringing a port along, and a script edited while bash was running it.

## Follow-ups
- A Pipeline save point can capture the program just after a block step's body ended but before the step heard of it (ET17: `stage` waiting, with the body's end node present). A Pipeline resumed from such a save point waits forever. ET17 met it only for builds that had already completed; those are now closed with their result instead of resumed. A running build adopted at such a save point would still hang. Needs a look at when workflow-cps writes `program.dat` relative to body callbacks.
- Speeding up job loading on a fresh controller (ROADMAP "Future"). ET17's restart of C (330 jobs, 200,000 build records) took 37 s to 281 s on a quiet machine, nearly all of it loading jobs, and more than 600 s while other builds and tests ran.
- A deleted job's kept `nextBuildNumber` makes a restarting controller try to load it as an item ("Failed to load item h10"); harmless, but noisy.
- One ET17 run lost four branches of one project (`mb22`) before the regression above was fixed; its evidence was overwritten by the next run. The harness now keeps each run's and each lost project's details. Not seen in the runs since.
- A `sleep` step resumed after an adoption logs "cannot start writing logs to a finished node" (seen since Target 14; the build carries on).
- One ET15 scenario 2 run showed the adopting controller's part of a log missing. Not reproduced in four runs since; ET15 now keeps controller logs on every exit to diagnose it if it recurs.

## Log
- 2026-10-11: planned with the user (items 1–21); items 22–26 added from the pluggable-storage review.
- 2026-10-12: item 27 added (the avatar plugin), and the job-loading note, at the user's request.
- 2026-10-13: every experiment passed on the latest fixes (ET4, ET12, ET17 and the kind runs ET9, ET10, ET13 rerun after the two regressions above); a full sweep on the final build started. ET17 runs while the Docker VM was overloaded by other work (load about 150, swap full) failed to restart C in 600 s; not counted as product failures.
- 2026-10-13 (later): the sweep failed only ET17 at the default kill time. Two causes found and fixed (the placeholder race in core, the indexing window in branch-api); ET17 then passed three runs in a row, and a full sweep on that build started.
- 2026-10-13 (evening): that sweep failed ET17 again, both ways, on the swapping Docker VM (above). With the kind release out of the way, the next sweep failed ET3 8 (a product bug, fixed) and ET8 4b (a strict check). **The sweep after passed everything: ET1–ET22, ET17 also with C killed at 36 s, and ET9, ET10, ET13 on kind. Target 15 is done.** The kind release `cnj` was reseeded.

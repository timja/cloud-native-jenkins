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
| 24 | Workspaces on Kubernetes pods (`stash`) | ✅ | ET9 4b |
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
- **Store plugin:** save points carried only `program.dat`, the record and the flow nodes, so an adopted Pipeline lost its stashes ("No such saved stash"). They now carry the build directory's other files, changed ones only (ET9 4b).
  - Multibranch children were not recognised as items by the applier (ET1).
- **Store plugin:** concurrent scans of one multibranch project on several controllers built its new branches twice. Folder computations now run on one controller at a time per folder (`ClusterComputations`, ET17).
- **cloudbees-folder:** the load progress of a fresh home printed percentages like 27,600 %.
- **Harness:** many. Most were `docker logs … | grep` failing under `pipefail`, checks that matched the shell's trace of a command as well as its output, compose `extends` bringing a port along, and a script edited while bash was running it.

## Follow-ups
- Speeding up job loading on a fresh controller (ROADMAP "Future").
- A `sleep` step resumed after an adoption logs "cannot start writing logs to a finished node" (seen since Target 14; the build carries on).
- One ET15 scenario 2 run showed the adopting controller's part of a log missing. Not reproduced in four runs since; ET15 now keeps controller logs on every exit to diagnose it if it recurs.

## Log
- 2026-10-11: planned with the user (items 1–21); items 22–26 added from the pluggable-storage review.
- 2026-10-12: item 27 added (the avatar plugin), and the job-loading note, at the user's request.

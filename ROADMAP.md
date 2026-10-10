# Roadmap: multi-controller, cloud-native Jenkins

**Goal.** N Jenkins controllers run active/active with **ephemeral `JENKINS_HOME`s**. Durable state lives in external services (a database plus object storage). Any controller can be killed or added without losing data, double-running work, or users noticing.

**Evidence.** Every gap below is backed by an experiment run against real Jenkins 2.580.1 instances (`validation/`, findings E1–E9). Status values: ✅ exists and works · 🟡 partial / Beta / workaround · ❌ missing.

## How state is split (decided 2026-10-10)
Jenkins state falls into two kinds, and they need different treatment:

| | **Static state**: changed by admins | **Runtime state**: changed by builds and events |
|---|---|---|
| Examples | global config, system credentials, security realm, static nodes, plugin set, key material | jobs created by indexing, job config rewritten by `properties()`, the queue, triggers, build numbers, running builds, Pipeline program state, build history |
| Rate of change | rare, reviewed | continuous, on whichever controller is doing the work |
| Now | **S0 workaround:** identical JCasC on every controller plus a seeded key set ([E8](findings/E8-casc-secret-sync.md)) | **Focus of this roadmap: R1–R5** |
| Later | Track S: move into the store using the R1 SPI ([targets/90](targets/90-static-state-store.md)) | |

Why the split: E8 showed identical JCasC plus a synced key set gives two controllers with separate homes the same config, secrets, agent secrets and instance identity. E9 showed job definitions are *runtime* state: indexing creates, deletes and rewrites them, and a plain build rewrote its own `config.xml`, including a cron trigger. No amount of config syncing covers that.

**North-star metric:** the number of **runtime** state categories still written to a controller's local `JENKINS_HOME` during the E1 + E9 scenarios. Static state may stay local under S0.

---

## S0: Static state workaround (now)
| Item | Status | Notes / evidence |
|---|---|---|
| Global config, system credentials, security realm, tool installers, clouds | 🟡 | Identical JCasC on every controller. E2: survives a fresh controller. **Config must be byte-identical** (no per-controller values), and the UI must be treated as read-only (E3: UI saves land on one controller) |
| Key material | 🟡 | E8: sharing `master.key` alone is **not enough**. Seed the full set (`secret.key*`, all of `secrets/*`, `identity.key.enc`) into each home before first boot, from a k8s Secret via an init container. Re-seed after plugin changes; alert when `secrets/` drifts from the seed |
| Static agent definitions | 🟡 | JCasC. Connection state is runtime, see R3 |
| Plugin set | 🟡 | Baked into the image; `UpdateCenter.never` (harness). No runtime installs |
| User state: API tokens, user properties, UI-created views | ❌ | E5/E8: a token made on A gives `401` on B. Mitigation: SSO, plus tokens from an external IdP or a token plugin backed by the store (Track S) |
| HTTP sessions and crumbs | 🟡 | E5: fail across nodes. **Sticky sessions** at the LB are the short-term answer (the crumb salt is already synced by S0). Stateless sessions come later (X) |

## R1: Item configuration created and changed at runtime ← **Target 1: done (ET1 passes)**
Plan: [targets/01-shared-item-config.md](targets/01-shared-item-config.md)

| Item | Status | Gap / evidence |
|---|---|---|
| Persistence SPI (key → bytes, CAS, listing, change events, hybrid routing) | ✅ branch | `XmlFile` is `final` and file-bound (`hudson/XmlFile.java:122`). The foundation for R1 and every later tier |
| Item config in the store (`jobs/**/config.xml` at any depth, multibranch `branches/*`) | ✅ branch | E9: indexing creates, rewrites and deletes branch jobs; `properties()` rewrites job config on every build. E3: jobs created on A are invisible on B |
| Item enumeration from the store, not a directory scan | ✅ branch | `Jenkins.loadTasks` lists `jobs/` (`Jenkins.java:3432-3439`). `AbstractFolder.loadChildren` lists subdirectories (`cloudbees-folder AbstractFolder.java:338-347`). Multibranch uses `branches/` (`branch-api MultiBranchProject.java:869`) |
| Cross-controller item events (create, update, rename/move, delete) → in-memory update | ✅ branch | Listeners are in-JVM only. Reuse `AbstractItem.updateByXml` (`AbstractItem.java:917-929`) for updates |
| No-op writes when content is unchanged | ✅ branch | E9: every index rewrites every branch's config, and every `properties()` build rewrites its own |
| Concurrent writers to one job (indexing on A while a build on B runs `properties()`) | 🟡 branch | CAS + warn policy; ET1 4/4b converge with 0 conflicts. Strict/writer-aware retries deferred (M5) |
| Plugin code reading item files directly | 🟡 | E7: 88 plugins use item-dir `File` APIs; 42 call `XmlFile#getFile` |

## R2: Build scheduling ← **Targets 2 and 5 done (ET2, ET5 pass): build numbers, exactly-once triggers, a shared queue**, [targets/02](targets/02-cluster-scheduling.md)
| Item | Status | Gap / evidence |
|---|---|---|
| Triggers fire once per cluster (cron, SCM polling, `upstream`) | ✅ branch (cron-style; `upstream` not covered) | **E3: cron fired 8 times in 3 minutes on 2 controllers.** E9: triggers are *defined by builds* via `properties()`, so the scheduler must follow R1 events |
| Branch indexing and org scanning run once, with indexing state shared | ✅ branch (periodic; manual scan runs where requested) | E9: `state.xml`, `indexing/indexing.xml`, `scm-*-revision-hash.xml` are written by whoever indexes |
| Webhook fan-in (an event delivered to any controller is handled once) | ❌ | GitHub/GitLab/Bitbucket hooks arrive at one controller through the LB |
| Atomic, cluster-wide build numbers | ✅ branch | `Job.BuildNumberAssigner` (Beta) exists; the default is `synchronized(job)` + file (`Job.java:378`). E3: only the JENKINS-23152 directory check avoided a collision |
| Queue: shared or partitioned, cluster-unique ids | ✅ branch (T5: mirrored shared queue with claims, folding, cancel and `queue/item` anywhere; [targets/05](targets/05-shared-queue.md)) | In-memory under one lock (`Queue.java:352`); `queue.xml` is written only at shutdown (E1). `QueueIdStrategy` (Beta) exists |
| Placing builds on a controller (load balancing, affinity to connected agents) | 🟡 branch (T5: the requesting controller goes first, then whoever can run it; label-bound jobs go where the agent is. Open: Pipeline `node` labels) | E5: an agent connects to one controller only |
| Quiet period, `build` step, upstream/downstream, `lock` (lockable-resources) | ✅ branch (T5: cluster-wide quiet period and folding, blocking, `milestone`, `lock`; `build(wait)` pinned to its caller) | Cross-build coordination happens in one JVM |
| Cluster-singleton `PeriodicWork` (declared, not blanket leader election) | ✅ branch (T5: `@ClusterSingleton` + `PeriodicWorkCoordinator`) | E7: 131 plugins (37 BOM) define periodic work, and much of it is legitimately per-JVM |

## R3: Running builds ← **Target 4 done (ET4 passes): leases, fencing, adoption**, [targets/04](targets/04-running-build-failover.md)
| Item | Status | Gap / evidence |
|---|---|---|
| Build ownership lease plus fencing | ✅ branch (T4: epoch-fenced leases; a partitioned owner stops itself, ET4 scenario 4) | E4: B adopted A's builds after a SIGKILL, but nothing stops a *partitioned* A from carrying on |
| Adoption of a dead controller's builds | ✅ branch (T4: from store save points, no shared disk; ET4 scenarios 1-3) | E4: works today **only** via a shared disk (`program.dat`, flow nodes, `build.xml`). Must keep working via R4 |
| Agent connections: which controller holds the channel, reconnecting after failover | ❌ | E5: one controller only. E4: needed a stable DNS alias moved by hand |
| Cloud provisioning across controllers (k8s, EC2, Docker) | ❌ | `NodeProvisioner` per JVM over-provisions. Orphan cleanup kills other controllers' agents |
| Routing requests to the owner (live console, `input`, abort, replay) | ❌ | |
| Executors and built-in node | 🟡 | Set built-in executors to 0; agents only (E1: built-in workspaces are local) |

## R4: Pipeline data ← **Target 4 done (ET4 passes): save points in the store, resume elsewhere**, [targets/04](targets/04-running-build-failover.md)
| Item | Status | Gap / evidence |
|---|---|---|
| `program.dat` (CPS continuation) in the store | ✅ branch (T4: save points via `ProgramSavedListener`, blob storage) | `CpsFlowExecution.java:1038`. E4: the resume source |
| Flow nodes (`workflow/*.xml` → `workflow-completed/flowNodeStore.xml`) | ✅ branch (T4: shipped with save points; completed nodes in the T3 snapshot) | `FlowNodeStorage` is abstract, but `createStorage()` hard-codes the file implementations. E9: written continuously during each build |
| Running-build index (`FlowExecutionList.xml`) | ✅ branch (T4: not needed; running placeholders and leases replace it, and the adopter registers the build) | E1, E4 |
| Pipeline logs | 🟡 | `LogStorageFactory` (Beta). The only reader/writer, pipeline-cloudwatch-logs, has had no commits since 2020. E1: falls back to local `log` + `log-index` |
| Artifacts and stashes | ✅ | azure-artifact-manager against Azurite (E1, E2); artifact-manager-s3. Gap: identity collision (R5) |
| Durability hints | 🟡 | `PERFORMANCE_OPTIMIZED` skips `program.dat` saves, so failover loses the build. Force `MAX_SURVIVABILITY` in HA |
| `milestone`, `lock`, `input` approvals, `build` step state | ❌ | Held in memory or XML on the owning controller |

## R5: Build records and history ← **Target 3 done (ET3 passes): shared completed builds, logs, running placeholders**, [targets/03](targets/03-shared-build-records.md)
| Item | Status | Gap / evidence |
|---|---|---|
| Globally unique build identity | 🟡 | T3: cluster-wide numbering plus `nextBuildNumber` kept on delete, so `(name, #)` is never reused (ET3 scenario 8). Originally found in **E2: junit-sql showed 4 tests instead of 2** after a recreated job reused `(name, #1)`. Open: a per-item UUID for data keyed by name across renames |
| `build.xml`, `RunMap` index, permalinks, `nextBuildNumber` | ✅ | T3: records in Postgres and snapshots in blob storage. Index merge, lazy materialisation, remote events and permalink hints ([ET3](findings/ET3-build-records.md) on `cnj/t3-m6-blob-and-log-storage`) |
| Freestyle and other non-Pipeline logs | 🟡 | Completed freestyle logs travel in the snapshot (ET3 scenario 1). Pipeline logs go through `LogStorageFactory` into append blobs. Open: a live freestyle console needs a core log SPI (`Run.getLogFile` hard-codes files, `Run.java:1409`; JEP-207 deferred) |
| Test results | 🟡 | junit-sql-storage works (E1). Per-controller cache, 10 min, no invalidation; identity collision |
| Per-build plugin data in the build dir | 🟡 | T3: the snapshot carries the build dir unchanged, so plugins keep working (junit file storage, ET3). Open: files rewritten after completion without a `build.xml` save. E7: 212 plugins (32 BOM) use build-dir `File` APIs |
| Fingerprints | 🟡 | `FingerprintStorage` (Beta), with no maintained external implementation |
| Item deletion vs history (branch pruning) | 🟡 | T3: deleting a job deletes its stored history but keeps its numbers. E9: pruning a branch still deletes its history (by design, as on one controller) |
| Build discarders, cluster-wide history views and REST | ❌ | |

## X: Cross-cutting
| Item | Status | Notes |
|---|---|---|
| Stateless sessions and crumbs | ❌ | E5. Sticky sessions are fine short-term (S0) |
| Inventory regression check in CI | ❌ | `analyze_inventory.py` over E1 + E9 |
| Ecosystem migration (E7 hit lists) | 🟡 measured | 513 / 2,117 plugins (99 of 358 BOM) |
| Ops: helm/operator, rolling upgrades with mixed versions on one store, home → store migration, backup/PITR | ❌ | |

## Track S: Static state in the store ← **Target 6 (planned)**, [targets/06](targets/06-static-state.md)
Replaces S0 once the R1 SPI exists: global config, descriptors, key material, users/tokens, credentials and nodes, plus cross-controller reload. Plan kept at [targets/90-static-state-store.md](targets/90-static-state-store.md). Findings E1, E3 and E6 still apply.

---

## Order of work
1. **R1** (Target 1): the persistence SPI plus shared item config. Every runtime tier keys off "which jobs exist and what they say".
2. **R2 build numbers + single scheduler.** This removes E3's duplicate cron builds and gives atomic numbering. It needs R1 because triggers come from job config (E9).
3. **R5 identity + build records.** This removes the E2 collisions and gives every controller the full history.
4. **R4 + R3:** Pipeline state in the store, ownership leases and adoption. This preserves E4's failover without a shared disk.
5. **X and Track S**, in parallel as capacity allows.

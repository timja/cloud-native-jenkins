# Roadmap: multi-controller, cloud-native Jenkins

**Goal.** N Jenkins controllers run active/active with **ephemeral `JENKINS_HOME`s**, and all durable state lives in external services (a database plus object storage). Any controller can be killed or added without losing data, double-running work, or users noticing.

**Decision.** Fully externalised storage. No shared filesystem. See [README](README.md#decision-record).

**Evidence.** Every gap below is backed by an experiment run against real Jenkins 2.580.1 instances (`validation/`). Status values: ✅ exists and works · 🟡 partial / Beta / unmaintained · ❌ missing.

**North-star metric.** The number of durable state categories written to `JENKINS_HOME` during the E1 scenario, measured by `analyze_inventory.py`. Today: 18 durable state categories (E1, `findings/raw/E1/analysis.md`). Target: 0, with only `war/`, `plugins/` and caches left.

---

## T0: Measure (done)
| Item | Status | Notes |
|---|---|---|
| Docker Compose harness: 2 controllers, Postgres, Azurite, non-sticky nginx, inbound agent | ✅ | `validation/compose.yaml`. `JENKINS_WAR_DIR` swaps in a locally built core war |
| `JENKINS_HOME` inventory (inotify + init-milestone correlation) | ✅ | [E1](findings/E1-home-inventory.md) |
| Experiments E1–E7 | ✅ | `validation/experiments/` |
| Inventory regression check in CI (fail if a category comes back) | ❌ | wire `analyze_inventory.py` into a GitHub Action once T1 lands |

## T1: Foundation: persistence SPI, secrets, global config ← **first target**
Plan: [targets/01-persistence-spi-global-config.md](targets/01-persistence-spi-global-config.md)

| Item | Status | Gap / evidence |
|---|---|---|
| Storage SPI: key → bytes, versions/CAS, listing, change events, hybrid routing | ❌ | `XmlFile` is `final` and wraps `java.io.File` (`hudson/XmlFile.java:122`); there is no persistence extension point. JEP-213 stalled here |
| Boot-time selection of the store | ❌ | E6: `--commonLibFolder` → `NoClassDefFoundError`; only `WEB-INF/lib` works. E1: `master.key`/`secret.key` are read before plugins are listed |
| Shared secrets (`secrets/*`, `secret.key`, instance identity) | 🟡 | `ConfidentialStore` is `ServiceLoader`-pluggable, but `secureRandom()` is package-private (E6). `secret.key` bypasses it. E2: UI-entered secrets and API tokens are lost on a new controller |
| Global config (`config.xml`, descriptor XML) in the store | ❌ | E3: lost update between controllers |
| Cross-controller change propagation and targeted reload | ❌ | E3: only a full stop-the-world `/reload` makes B see changes |
| Idempotent writes (CasC re-apply on every boot) | ❌ | E1 finding 2: a plain restart rewrites 9+ durable files |
| `DoubleLaunchChecker` replaced | ❌ | E3: never fires inside 60–90 min |

## T2: Configuration entities
| Item | Status | Gap / evidence |
|---|---|---|
| Items (jobs, folders, multibranch children) | ❌ | `jobs/*/config.xml`; loaded by **directory scan** of `jobs/` (`Jenkins.loadTasks`). Needs `list(prefix)` plus rename/move as a store transaction. E3: new jobs invisible on other controllers |
| Nodes | ❌ | `nodes/*/config.xml` (E1). Separate *definition* (shared) from *connection* (per controller), see T5 |
| Users, `UserIdMapper`, API tokens, `apiTokenStats.xml` | ❌ | E5: a token created on A gets `401` on B. `User.AllUsers` is an in-memory map (`hudson/model/User.java:1103`) |
| System credentials (`credentials.xml`), folder credentials | ❌ | Read while plugins load (E1). External providers (Vault, Key Vault, AWS SM) cache per controller with 10 min TTLs |
| Views | ❌ | Inside `config.xml` (moves with T1) and folder `config.xml` (T2) |
| Plugin-owned `XmlFile`s and `getFile()` users | 🟡 | [E7](findings/E7-ecosystem-scan.md): 171 plugins use `XmlFile` directly (covered by path-based routing); **42 call `XmlFile#getFile`** (10 in BOM, e.g. github, job-dsl, junit, jobConfigHistory, saml), which bypasses the store |
| JCasC as a store writer (apply once per cluster, not per boot) | ❌ | `ConfigurationAsCode` applies in an `@Initializer` on every controller; reload is per controller |

## T3: Build records and outputs
| Item | Status | Gap / evidence |
|---|---|---|
| Stable, globally unique build identity | ❌ | **E2: junit-sql shows stale results** (4 tests instead of 2), and Azure artifact blob keys (`<job>/<#>/…`) collide when a recreated job reuses `(name, #)`. Needs a UUID or never-resetting numbers |
| Build numbering from one atomic source | 🟡 | `Job.BuildNumberAssigner` (Beta) exists; the default is `synchronized(job)` + file (`Job.java:378`). E3: only the JENKINS-23152 directory check avoided a collision |
| Build records (`build.xml`) and the `RunMap` index | ❌ | Lazy-load index is `dir.list()` of `builds/` (`AbstractLazyLoadRunMap.java:175`). Beta hooks `recognizeNumber`/`allowLoad` exist |
| Permalinks | ❌ | `builds/permalinks` (E1) |
| Build logs: core SPI for Freestyle and all `Run`s | ❌ | `Run.getLogFile` hard-codes files (`Run.java:1409`). JEP-207 deferred |
| Build logs: Pipeline | 🟡 | `LogStorageFactory` (Beta, workflow-api). Only reader/writer is pipeline-cloudwatch-logs (last commit 2020). The Azure one is write-only. E1: with nothing installed, `log` + `log-index` stay local |
| Artifacts and stashes | ✅ | azure-artifact-manager works end to end against Azurite (E1, E2); artifact-manager-s3 exists. Gap: identity collision (above) |
| Test results | 🟡 | junit-sql-storage works (E1). Gaps: identity collision (E2); per-controller Caffeine cache, 10 min, no cross-controller invalidation |
| Fingerprints | 🟡 | `FingerprintStorage` (Beta) in core; no maintained external implementation. E1: `fingerprints/**` written even with external artifacts |
| Task logs (agent logs, polling logs) | ❌ | `logs/slaves/*` (E1); category "not started" on jenkins.io |
| Per-build plugin data written into the build dir | ❌ | [E7](findings/E7-ecosystem-scan.md): **212 plugins** (32 in BOM, incl. workflow-api, workflow-job, junit, warnings-ng, coverage) use build-dir `File` APIs. Needs a per-build attachment/blob API and a hybrid migration |

## T4: Pipeline runtime state
| Item | Status | Gap / evidence |
|---|---|---|
| `program.dat` (CPS continuation) in the store | ❌ | `CpsFlowExecution.java:1038`. E4: resume works **only** because the new controller can read it from disk |
| Flow nodes (`workflow/*.xml` → `workflow-completed/flowNodeStore.xml`) | ❌ | `FlowNodeStorage` is abstract, but `createStorage()` hard-codes the file implementations |
| Running-build index (`FlowExecutionList.xml`) | ❌ | E1, E4 |
| Build ownership lease, fencing, adoption on controller death | ❌ | E4: B adopted A's builds, but nothing would stop a *partitioned* A carrying on in parallel |
| Durability hints | 🟡 | `PERFORMANCE_OPTIMIZED` skips `program.dat` saves, so failover loses the build. Document or force `MAX_SURVIVABILITY` in HA |

## T5: Multi-controller coordination
| Item | Status | Gap / evidence |
|---|---|---|
| Leader election or sharding for `PeriodicWork`/`AperiodicWork`, `Trigger.Cron`, SCM polling | ❌ | **E3: cron fired 8 times in 3 minutes on 2 controllers.** Every periodic task runs per JVM. E7: 131 plugins (37 BOM) define periodic work, much of it legitimately per-JVM. Needs a way to *declare* cluster-singleton work |
| `NodeProvisioner` / cloud provisioning across controllers | ❌ | Would over-provision. CloudBees HA notes that k8s concurrency limits and orphan cleanup break |
| Queue: shared or partitioned, `QueueIdStrategy` | 🟡 | Queue is in memory under one lock (`Queue.java:352`); `queue.xml` is written on shutdown (E1). `QueueIdStrategy` (Beta) exists |
| Agent channel ownership and routing (console follow, `input`, `FilePath`) | ❌ | E5: an agent through the LB attaches to one controller; the other waits forever. E4: reconnects need a stable endpoint |
| Stateless sessions and crumbs | ❌ | E5: login and crumb POST fail whenever requests cross nodes; sessions are in-memory Jetty |
| HTTP routing to a build's owner | ❌ | Needed for live console, `input`, stop/abort |
| Cross-controller events (`ItemListener`, `RunListener`, `SaveableListener`) | ❌ | Listeners are in-JVM only. Generalise the T1 change dispatcher |
| Webhook fan-in (deliver once) | ❌ | GitHub/GitLab webhooks via the LB hit one controller, which must own or forward |
| Immutable plugin set (no UC installs or dynamic loading at runtime) | 🟡 | Harness already uses `UpdateCenter.never`. Make it a supported mode |

## T6: Ecosystem
| Item | Status | Gap / evidence |
|---|---|---|
| Plugins using file-storage or single-JVM APIs | 🟡 measured | [E7](findings/E7-ecosystem-scan.md): **513 / 2,117 plugins (99 of 358 BOM)**. `Jenkins#getRootDir` 166, build dirs 212, item dirs 88, `getLogFile` 24, periodic work 131 (37 BOM) |
| Aggregated UI and REST (build history, queue, executors across controllers) | ❌ | E3: each controller lists only its own builds |
| `build` step, `lock` step (lockable-resources state), `copyArtifact` | ❌ | Cross-build, in-JVM coordination |
| Built-in node workspaces | 🟡 | E1: built-in node writes `workspace/`. Set built-in executors to 0 in HA |
| Metrics per controller vs cluster | ❌ | |

## T7: Operations
| Item | Status | Notes |
|---|---|---|
| Helm chart / operator for N controllers + store | ❌ | `helm-charts` checkout available to extend |
| Rolling upgrades (core and plugins) with mixed versions on one store | ❌ | Needs schema/format versioning per key |
| Migration of an existing file-based `JENKINS_HOME` into the store | ❌ | `FilesystemPersistenceStore` → external copy tool |
| Backup/restore and point-in-time recovery | ❌ | Store-native (Postgres PITR + blob versioning) |
| Store schema migrations | ❌ | |

---

## Order of work
1. **T1** builds the primitive every later tier needs and fixes secrets.
2. **T2** items/nodes/users and **T3** build identity + numbering + records can then proceed in parallel. Both widen T1's set of owned keys.
3. **T3 logs** and **T4** follow. Together with T3 they make cold failover work with no shared disk, preserving the E4 behaviour.
4. **T5** coordination comes next: leader election first, because cron double-firing (E3) is the most visible failure, then sessions, then agent routing.
5. **T6** ecosystem work is continuous, driven by E7, and **T7** comes once T1–T4 are stable.

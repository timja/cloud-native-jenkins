# Upstreaming plan

All prototype work is pushed as stacked pull requests to **forks** (`timja/*`) for review first. Nothing has been opened against `jenkinsci/*`. This document tracks every PR, and the order and priority for taking each one upstream.

**Layout: cnj2.** After the round-4 review ([reviews/agents-2026-10-17.md](reviews/agents-2026-10-17.md)) the core stack was re-cut from `fork/cnj/base` (`cd2a077`) as new `cnj2/*` branches, following [targets/17-architecture.md](targets/17-architecture.md) (DESIGN-17): cluster engines out of core into the `cluster` plugin behind a few core extension points, build data keyed by item identity instead of by path, and no migration of stored data. The frozen API is in [targets/17-api.md](targets/17-api.md). The old stack (tag `cnj-rev3-final` = `0a47f66bc6`, PRs timja/jenkins #139–#162) is kept as an archive; **#139–#162 will be closed, each with a link to its cnj2 successor, once the cnj2 PRs are up.**

**Status key:** 🟡 review PR open on the fork · 🔵 branch ready, PR not opened yet · ⬜ not pushed · ✅ upstreamed · ⛔ superseded (to be closed)

**Stacks.** The fork PRs are real GitHub stacks now: cluster-plugin stack #8 (cluster-plugin#1–#7), jenkins stack #178 (#171–#174) and jenkins stack #179 (#176–#177). Groups A and C1 stay single PRs on `fork/cnj/base` and B1.

## Getting started (minimum first step)
Open these together, at once:
1. **The JEPs**: one PR to `jenkinsci/jep` with all of them (the content of [timja/jep#4](https://github.com/timja/jep/pull/4)), JEP-250 marked as the one to review first, since everything depends on it. With it, a post to the developer list / Discourse giving the big picture: why several controllers on one shared store, what goes in core (mechanisms) and what in the `cluster` plugins (policy), and the order of what follows.
2. **B1a (the store SPI)** as the first core PR, since it is what JEP-250 describes. (B1 #171 is being split into B1a–B1c.)
3. **Small genuine fixes that need no cluster context**: branch-api 1.5 (interrupted indexing no longer builds a branch twice), core A7 [#170](https://github.com/timja/jenkins/pull/170), and the lockable-resources lock-step fix (the step returns false after queueing) if it is split out of L1.

Then, after a first round of reviews: A1 [#164](https://github.com/timja/jenkins/pull/164) and the Stapler prefix 1.2 (needed before B4), then B1b, B1c, B2 onwards, **one core PR open at a time**.

Hold the other extension points (group C, L1/S1, workflow-cps 1.1) and the cluster plugins until JEP-250 is accepted or close to it.

Consider splitting two pieces into PRs of their own: workflow-cps `CpsFlowExecution.halt` (independent of `ProgramSavedListener`), and the lockable-resources lock-step fix (a plain bug fix, reachable without a coordinator).


## Principles
1. **The JEPs come first.** JEP-250 (the persistence SPI) is the foundation, and every core PR in groups B and C refers to it. All the JEPs go to `jenkinsci/jep` together in one PR, with JEP-250 reviewed first; B1a, the store SPI JEP-250 describes, is opened at the same time. The rest of group B follows after a first round of review.
2. **Core gets mechanisms, the cluster plugin gets policy.** A core PR must be justifiable on its own: useful without a store, or the narrowest seam the plugin cannot get from an existing extension point. Each new extension point carries a fake implementation in its tests and a link to its user.
3. **No store means upstream behaviour, provably.** Every hook defaults to the old code path. No extra saves, no new XML elements, no new REST fields, and no core extension registered at a high ordinal to replace a default. Each core PR has a test for this where it touches it.
4. **Changes that stand alone go first.** They need no new SPI, and are small: they build trust and shorten the later stacks. Group A in core and the standalone plugin and Stapler PRs can be opened in any order.
5. **Plugins keep their baselines where possible.** lockable-resources and pipeline-build-step get small extension points of their own (L1, S1) and the store-facing code lives in the `cluster` plugin, so neither depends on a new core.
6. **Plugins that use the new core APIs wait** for a core release that has them, then raise their `jenkins.version`.
7. **"Revert before upstreaming" commits** point a build at locally installed snapshots. Drop them before opening an upstream PR.
8. **No data migration.** Nothing has been released; a cluster store in the old path-keyed layout is refused, and is wiped and started fresh (DESIGN-17 P7 dropped).
9. **The security review runs separately.** No PR describes vulnerabilities; the JEPs' security sections stay generic.

## Order and priority

### Wave 0: proposals
All in 🟡 [timja/jep#4](https://github.com/timja/jep/pull/4) (branch `cluster-jeps`), rewritten for cnj2: 250 shrunk to the key-value SPI, 255 split into 255a and 255b, engines described as implemented by the cluster plugin.

| # | JEP | Type | Requires | Priority | Notes |
|---|---|---|---|---|---|
| 0.1 | JEP-250 Pluggable persistence store for `JENKINS_HOME` configuration | Standards (core) | none | P0 | Draft first; everything depends on it |
| 0.2 | JEP-255a Cluster coordination: membership, leases and the cluster store | Standards (cluster plugin) | 250 | P1 | The plugin SPI its backends implement; the identity-keyed layout |
| 0.3 | JEP-251 Shared build records, logs and history | Standards | 250, 255a | P1 | `BuildRecordSource` |
| 0.4 | JEP-252 Ownership of running builds | Standards | 251, 255a | P1 | `ExternalBuilds`; deliveries; `BuildTriggerRelay` |
| 0.5 | JEP-253 Cluster scheduling | Standards | 250, 252, 255a | P1 | Coordinators, `QueueStartGate`, queue hand-off; `LockCoordinator` |
| 0.6 | JEP-254 Request routing and stateless sessions | Standards | 250, 252, 255a | P1 | `LoginRevocation`; launch lease for outbound agents |
| 0.7 | JEP-256 Cluster-wide plugin management | Standards | 250, 255a | P2 | `PluginArchiveListener`, `PluginManagerNotice` |
| 0.8 | JEP-257 Cloud provisioning across controllers | Standards | 253, 255a | P2 | |
| 0.9 | JEP-255b Operating a cluster | Informational | all | P3 | May become chart documentation instead |

Submission order: one PR to `jenkinsci/jep` with all of them (the content of timja/jep#4). 250 is the one to review first, since everything depends on it; then 255a → 251 → 252 → 253 → 254 → 256 → 257 → 255b.

### Wave 1: changes that stand alone (no new core SPI)

**Standalone plugin and Stapler PRs.** Each got its round-4 fixes as a new commit on the existing fork PR's branch (local clones `~/code/jenkins/<name>-cnj`) and was pushed to the fork PR, whose description was rewritten.

| # | Repo | Branch | Fork PR | Priority | Notes |
|---|---|---|---|---|---|
| 1.1 | workflow-cps | `cnj/t4-m1-program-saved-listener` | 🟡 [workflow-cps-plugin#1](https://github.com/timja/workflow-cps-plugin/pull/1) | P1 | `ProgramSavedListener`. Round 4 (`53e6d332`): saves `build.xml` before notifying, a throwing listener is contained, the temp-file leak on the skipped save is fixed. Round 5 (`0097d29e`): `CpsFlowExecution.halt` (Beta) stops an execution without running more of its program (no `catch`/`finally`/`post`, flow graph not ended); used by P2 to stop the copy of a Pipeline taken over elsewhere; `CpsFlowExecutionHaltTest`. Candidate for a PR of its own. JEP-252 |
| 1.2 | stapler | `cnj/t6-m5-bound-id-prefix` | 🟡 [stapler#1](https://github.com/timja/stapler/pull/1) | P1 | Optional bound-id prefix. Round 4 (`b03add716`): prefix validated (`[A-Za-z0-9_.-]{1,64}`), `getIdPrefix()`/`serverIdOf(id)` instead of a public mutable field. Already on current `master`. Commit titles reworded without target numbers. JEP-254 |
| 1.3 | configuration-as-code | `cnj/t15-keep-undeclared-nodes` | 🟡 [configuration-as-code-plugin#116](https://github.com/timja/configuration-as-code-plugin/pull/116) | P2 | `keepUndeclaredNodes`. Round 4 (`d282fe8c`): README and javadoc, test on JUnit 5 |
| 1.4 | kubernetes | `cnj/t9-m2-cluster-cap` | 🟡 [kubernetes-plugin#1](https://github.com/timja/kubernetes-plugin/pull/1) | P2 | `ClusterProvisioningLimits` (Beta). Round 4 (`55fd97ec`): template cap keyed by template name, remote call out of the lock, local release never skipped. JEP-257 |
| 1.5 | branch-api | `cnj/t15-first-build-revision` | 🟡 [branch-api-plugin#1](https://github.com/timja/branch-api-plugin/pull/1) | P1 | Interrupted indexing no longer builds a branch twice. Round 4 (`4495c92`): a cheap existence check before any read on every save, tighter tests. JEP-253 |
| 1.6 | lockable-resources | `cnj2/lock-coordinator` (L1) | 🟡 [lockable-resources#4](https://github.com/timja/lockable-resources-plugin/pull/4) | P1 | `LockCoordinator` extension point at today's 13 cluster call sites, plus the genuine fixes (`LockStepExecution` returns false after queueing; the "internal error" WARNING becomes FINE). No core dependency; baseline unchanged. Round 5 (`3daeecd`): remote lock API locks go through the coordinator too (`tryLockForRemote`/`releasedForRemote`/`newRemoteLockId`, `releaseRemoteLock`), so a remote lock waits for a resource held elsewhere. Head `3daeecd`. The lock-step fix (`50d8c2f`, first commit) can be split out as a plain bug fix. Supersedes lockable-resources#1 and #2. JEP-253 |
| 1.7 | pipeline-build-step | `cnj2/build-trigger-relay` (S1) | 🟡 [pipeline-build-step#2](https://github.com/timja/pipeline-build-step-plugin/pull/2) | P1 | `BuildTriggerRelay` extension point, `waitForStart` carried in the trigger, inbound `deliver`, result from the message as fallback, unit tests with a fake relay. No core dependency; baseline stays 2.479. Round 5: `deliver` and `BuildTriggerRelay.Event` are `@Restricted(Beta.class)` (squashed; head `b552fd3`). Supersedes pipeline-build-step#1. JEP-252 |

**Core group A: standalone hooks** (`timja/jenkins`, each off `fork/cnj/base`, independent, can go to jenkinsci early). A1 lands before B2, which rebases on it.

| # | Branch | Fork PR | Priority | Size (main/test) | What | From (rev3) |
|---|---|---|---|---|---|---|
| A1 | `cnj2/a1-item-uuid` | 🟡 [#164](https://github.com/timja/jenkins/pull/164) | P1 | ~300 / 250 | `Item.getItemUuid`, `ItemUuids` (+`pin`), identity persisted only when pinned, injected into the bytes on create/copy (no second save; `ItemGroupMixInTest` assertion restored) | #162 (t15) hunks, reworked |
| A2 | `cnj2/a2-trigger-coordinator` | 🟡 [#165](https://github.com/timja/jenkins/pull/165) | P1 | ~90 / 180 | `TriggerCoordinator` and the `Trigger`/`SafeTimerTask` calls | #144 |
| A3 | `cnj2/a3-periodic-coordinator` | 🟡 [#166](https://github.com/timja/jenkins/pull/166) | P2 | ~140 / 170 | `PeriodicWorkCoordinator`, `@ClusterSingleton` on three works | #152 minus `FingerprintCleanupThread.Shared` |
| A4 | `cnj2/a4-queue-hooks` | 🟡 [#167](https://github.com/timja/jenkins/pull/167) | P1 | ~200 / 200 | `QueueStartGate` (+`countsForProvisioning`), `QueueListener.onFolded`, `Queue.LOADED` | #150, #157 hunks |
| A5 | `cnj2/a5-runmap-hooks` | 🟡 [#168](https://github.com/timja/jenkins/pull/168) | P1 | ~100 / 80 | `knows`/`unload`/`forget`/`proposeSharedNumber`, `storedNumbers` hook, `Run.isPendingDelete`; the `ItemListener` ordinal fix as its own commit | #145–#147, #162 |
| A6 | `cnj2/a6-plugin-archive-listener` | 🟡 [#169](https://github.com/timja/jenkins/pull/169) | P2 | ~110 / 100 | `PluginArchiveListener` (CLI and `doUploadPlugin`), `PluginManagerNotice` | #162 |
| A7 | `cnj2/a7-small-fixes` | 🟡 [#170](https://github.com/timja/jenkins/pull/170) | P2 | ~50 / 60 | `Items.move` into itself as `IOException`, `User.delete` fires `SaveableListener.onDeleted`, `Nodes` name validation (not done: it is B3's code, and B3 keeps the guard) | #162, #153 |

### Wave 2: core, JEP-250 (group B, strictly ordered: B1 → B2 → B3 → B4)
B1 no longer waits for JEP-250 feedback: B1a (the store SPI) is opened with the JEP PR (see Getting started). B1b, B1c and B2–B4 follow after a first round, one at a time. Stacked on GitHub as jenkins stack #178 (#171–#174).

| # | Branch | Fork PR | Priority | Size (main/test) | What | From (rev3) |
|---|---|---|---|---|---|---|
| B1 | `cnj2/b1-store-spi-routing` | 🔵 [#171](https://github.com/timja/jenkins/pull/171) | P0 | ~1,500 / 900 | Key-value `PersistenceStore` (+ provider, `move` with preconditions), `StoreRouting` + `LocalFiles`/`SharedFiles`, `XmlFile`/`TextFile` routing with the version table, `exists()` errors, create-only unread writes, refuse-and-reload with `StoredConfigReconciler`, `materializedCopies`; `InMemoryPersistenceStore` in test sources. Being split into B1a–B1c | #139, #153 (routing), #158 (hook only), #161, #162 (`getFile`) |
| B2 | `cnj2/b2-items` | 🟡 [#172](https://github.com/timja/jenkins/pull/172) | P0 | ~1,200 / 1,300 | Store-only enumeration, lifecycle with `move`+preconditions (no move claims), `ItemChangeApplier` for items, `RemoteItemGroup`, `RemoteChangeHandler` dispatch, the generic sub-item rule | #140–#142, #148 (item parts), #162 |
| B3 | `cnj2/b3-static-state` | 🟡 [#173](https://github.com/timja/jenkins/pull/173) | P1 | ~600 / 900 | Users, nodes (cloud agents local), global configuration and descriptors, `EarlyDescriptors`, setters under the lock, `unmarshalNullingOut` reload. Round 5 (head `bcd2d80012`): early descriptors read from the store at startup again (`StaticStateApplier` was not accessible to its initializer); a refused save while starting logs instead of failing the boot (controllers started together with JCasC); `User.loadStored` keeps API tokens working after a reload; descriptors reload through their own `load()`; a node reloaded after a refused save rebinds its computer | #153, #154, #162 |
| B4 | `cnj2/b4-keys-sessions` | 🟡 [#174](https://github.com/timja/jenkins/pull/174) | P1 | ~450 / 500 | Shared confidential keys, stateless sessions, `LoginRevocation` (sign-out everywhere). Round 5 (head `43da520da7`): a sign-in clears the login id the session carried (`forgetLogin` in `AuthenticationProcessingFilter2`), tested in `StatelessSessionsTest`; implemented by P5's `ClusterLoginRevocation` | #155, #156, new |

### Wave 2b: core, build and queue extension points (group C, each on B1, mutually independent)
C2 and C3 are stacked on GitHub as jenkins stack #179 (#176–#177). Held until JEP-250 is accepted or close.

| # | Branch | Fork PR | Priority | Size (main/test) | What | From (rev3) |
|---|---|---|---|---|---|---|
| C1 | `cnj2/c1-build-record-source` | 🟡 [#175](https://github.com/timja/jenkins/pull/175) | P1 | ~130 / 200 | `BuildRecordSource` + `RunMap`/`AbstractLazyLoadRunMap` wiring + fake-source test | #145 `RunMap` hunks |
| C2 | `cnj2/c2-external-builds` | 🟡 [#176](https://github.com/timja/jenkins/pull/176) | P1 | ~280 / 300 | `ExternalBuilds`/`ExternalBuild`: `Job`, `HistoryPageFilter` + jelly include, `getDynamic`, `ConsoleCommand`, `BuildCommand`. Round 5 (head `55654ef059`): javadoc allows one store read on the caller's thread when nothing is cached, for answers that decide (rename, lookup by number) | #160, #162 CLI hunks, reworked |
| C3 | `cnj2/c3-queue-handoff` | 🟡 [#177](https://github.com/timja/jenkins/pull/177) | P1 | ~250 / 350 | `scheduleShared`/`updateShared`/`handOff`/`LeftItem.isTakenElsewhere`, futures completed by the remote build; `RemoteItem` removed | #150 `Queue` hunks, new |

Core total: about 5.3k main and 6.0k test lines in 14 PRs (largest B1 at ~1.5k, against #162's 3.5k). **`cnj2/all`** merges A* + B* + C* for the plugins to build against as an incremental; it is a test branch, not a PR. Full core suite on `cnj2/all` (`577a5e37e9`) on the build machine: core module 23,177 tests, test module 2,669 tests across all 503 classes, 0 failures.

### Wave 3: plugins using the new core APIs (after a core release containing them)
| # | Repo | Branch | Fork PR | Needs | Priority | Notes |
|---|---|---|---|---|---|---|
| 3.1 | cloudbees-folder | `cnj/t1-m2-folder-children` | 🟡 [cloudbees-folder-plugin#1](https://github.com/timja/cloudbees-folder-plugin/pull/1) | B2 | P0 | Folder children only in the store; to call the Beta `storedItemNames` |
| 3.2 | cloudbees-folder | `cnj/t15-remote-child` | 🟡 [cloudbees-folder-plugin#2](https://github.com/timja/cloudbees-folder-plugin/pull/2) | B2 (`RemoteItemGroup`) | P2 | Stacked on 3.1; the `addLoadedChild` key fix is a plain upstream fix. Round 5 (`e459a05`, also branch `cnj2/remote-child`): tests use a store kept in the test sources (core's test module publishes no test jar), pom on `2.586-cnj2-all-SNAPSHOT`; 183 tests pass |
| 3.3 | credentials | `cnj/t6-m4-credentials-reload` | 🟡 [credentials-plugin#1](https://github.com/timja/credentials-plugin/pull/1) | B3 | P1 | Has a "revert before upstreaming" commit |
| 3.4 | junit-sql-storage | `cnj/t3-m5-remote-delete` | 🟡 [jenkins-junit-postgresql-plugin#52](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/52) | B2 (`isApplyingRemoteChange`) | P1 | Has a "revert before upstreaming" commit |
| 3.5 | junit-sql-storage | `cnj/t15-follow-renames` | 🟡 [jenkins-junit-postgresql-plugin#53](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/53) | A1 (`getItemUuid`) | P2 | Split as round 4 suggested: the rename fix needs no core API and can go to jenkinsci now; the cluster guard waits |
| 3.6 | lockable-resources | `cnj/t5-m4-cluster-locks`, `cnj/t15-cluster-ui` | ⛔ [#1](https://github.com/timja/lockable-resources-plugin/pull/1), [#2](https://github.com/timja/lockable-resources-plugin/pull/2) | — | — | Superseded by L1 (1.6) and P6; close with a link once those are up |
| 3.7 | pipeline-build-step | `cnj/t15-remote-completion` | ⛔ [#1](https://github.com/timja/pipeline-build-step-plugin/pull/1) | — | — | Superseded by S1 (1.7) and P6; close with a link once those are up |

### Wave 4: the Cluster plugins, chart and validation
The code is in [timja/cluster-plugin](https://github.com/timja/cluster-plugin) (public): the four plugins at its root, the chart in `charts/`, the experiments in `validation/`. One hosting request per plugin, splitting that repository then. The names `cluster`, `cluster-postgresql`, `cluster-azure-blob` and `cluster-s3` were free on the update site on 2026-10-14; `jenkins-cluster` is the fallback for `cluster`. They are released after the core stack is in a weekly.

**Plugin series (cnj2).** One stacked series on `main`, a GitHub stack (#8, cluster-plugin#1–#7), built against `cnj2/all` incrementals (until then, the locally installed `2.586-cnj2-api-SNAPSHOT` from `cnj2/api`). The engines leave core by `git show cnj-rev3-final:<path>`, one commit per engine, ported first with path keys behind `Keys` and then switched to the identity layout in one commit. The whole series is formatted with Spotless: a first commit "Format the sources with Spotless" on `cnj2/api`, then each commit replayed formatted. SpotBugs fixes (store absence treated as no store, map iteration, statics for the script console) were squashed into the commits that introduced the code; only the 15 pre-series findings in `cnj2/api` remain. Plugin suite at the harness tip (`9373ecf`): cluster 188, cluster-postgresql 35, cluster-azure-blob 17, cluster-s3 13 tests, 0 failures.

| # | Branch | Fork PR | Needs | What |
|---|---|---|---|---|
| P0 | `cnj2/p2-running` (with P1, P2) | 🟡 [cluster-plugin#1](https://github.com/timja/cluster-plugin/pull/1) | B1 | SPI: `BuildLeases` (abstract atomic `writeIfHeld`), `ClusterQueue`, `LaunchLeases`, `ClusterBlobStore` (no `move`) and `Keys` in `spi`; `owns`/`storesBuildRecords` dropped; Postgres `move` with preconditions; routing patterns become `SharedFiles`/`LocalFiles` contributions; protocol and layout bump (an old-layout store is refused) |
| P1 | `cnj2/p2-running` | 🟡 [cluster-plugin#1](https://github.com/timja/cluster-plugin/pull/1) | P0, C1 | Builds: `BuildRecords`, `StoredPermalinks`, `StoredBuildNumberAssigner` on `_items/<uuid>`; `BuildRecordSource`; `RemoteChangeHandler` |
| P2 | `cnj2/p2-running` | 🟡 [cluster-plugin#1](https://github.com/timja/cluster-plugin/pull/1) | P1, C2, workflow-cps 1.1 | Running: `RunningBuilds`, adoption (`PipelineAdopter`, `SavePoints`), `ExternalBuilds` and its views, `RunningElsewhereAction`. Round 5 (head `e284e90`): next build number follows builds completed elsewhere; an outdated copy never writes over an edit elsewhere; handover 2 s after a `node` step waits (was 15–20 s); a taken-over copy is halted through workflow-cps `halt` (no `finally`/`post`, no completion), others end as NOT_BUILT; `isBuilding`/lookup/`lastBuild` read the store when nothing is cached; a record without its snapshot is skipped; `ClusterPluginManagerNotice` is a `PluginManagerNotice`; `HomeWrites` allows `jobs/*/nextBuildNumber` |
| P3 | `cnj2/p3-queue` | 🟡 [cluster-plugin#2](https://github.com/timja/cluster-plugin/pull/2) | P2, C3, A4 | Queue: `SharedQueue` (gate, ids, reservations, publisher on `handOff`), `SharedBlocking`, `SharedQueueMaintenance`. Round 5 (head `6b307c4`): a hand-off future is not completed by a stopped copy |
| P4 | `cnj2/p4-files` | 🟡 [cluster-plugin#3](https://github.com/timja/cluster-plugin/pull/3) | P0 | Files: `ItemFiles`, write-back of materialised copies |
| P5 | `cnj2/p5-config` | 🟡 [cluster-plugin#4](https://github.com/timja/cluster-plugin/pull/4) | P0, B4 | Config: `XmlMerge` + `ItemReload` as a `StoredConfigReconciler`; `ConflictMonitor` counts refusals. Round 5 (head `162df6c`): `ClusterLoginRevocation` (sign-out and user deletion end sessions on every controller); a reload of a job no longer deadlocks with a build of it starting |
| P6 | `cnj2/p6-integrations` | 🟡 [cluster-plugin#5](https://github.com/timja/cluster-plugin/pull/5) | P3, L1, S1 | Integrations: locks (`LockCoordinator` for lockable-resources), the pipeline-build-step relay. Round 5 (head `284ee72`): a stopped copy keeps the new owner's lock; remote lock API locks exclusive across controllers (`RemoteLockForwarder`, needs L1 `3daeecd`); a stopped downstream copy does not complete the build step |
| P8 | `cnj2/p8-launch-lease` | 🟡 [cluster-plugin#6](https://github.com/timja/cluster-plugin/pull/6) | P0, A4 | Launch leases for outbound static agents (JEP-254). Round 5 (head `b16279a`): the lease holder provisions for its labels' work; `/computer/<name>/` forwarded to the holder (Overall/Read only, path checks); "connected elsewhere" offline cause; takeover on the next 15 s lease check once the holder is gone |

P7 (a layout migration tool) was dropped: there is no data to migrate.

| # | What | Where | Priority | Notes |
|---|---|---|---|---|
| 4.1 | `cluster` plugin | new repository (hosting request) | P1 | Everything backend-neutral, the engines moved out of core, and the cluster SPI (`io.jenkins.plugins.cluster.spi`) the backends implement (JEP-255a) |
| 4.2 | `cluster-postgresql` plugin | new repository (hosting request) | P1 | The reference store; uses `postgresql-api` for its driver |
| 4.3 | `cluster-azure-blob` plugin | new repository (hosting request) | P1 | Blobs in Azure Blob Storage (append blobs for logs); uses `azure-sdk` |
| 4.4 | `cluster-s3` plugin | new repository (hosting request) | P1 | Blobs in S3 or an S3-compatible store; uses `aws-java-sdk2-s3` |
| 4.5 | Helm chart | `jenkinsci/helm-charts`, or with the plugins | P2 | `blob.type: azure\|s3` (JEP-255b) |
| 4.6 | Acceptance experiments (`validation/`) | with the plugins | P3 | Docker Compose and kind; an S3 variant on Garage. Harness [cluster-plugin#7](https://github.com/timja/cluster-plugin/pull/7) (head `9373ecf`) ported to the identity layout. Round 5 on the build machine: ET1–ET8, ET11, ET12, ET14–ET22 and S3 ET3/ET4/ET8/ET15/ET19 all 0 failures; kind pod-delete failover and cross-controller pod agents pass. Seven product bugs found and fixed in B3, P2, P5. Not run: ET9, ET10, ET13 |

### Old stack (to be closed)
The rev3 PRs and where their content went. Each is closed with a link to its successors once those PRs are up.

| Old PR | Branch | Goes to |
|---|---|---|
| [#139](https://github.com/timja/jenkins/pull/139) | `cnj/t1-m1-persistence-spi` | B1 |
| [#140](https://github.com/timja/jenkins/pull/140) | `cnj/t1-m2-item-enumeration` | B2 |
| [#141](https://github.com/timja/jenkins/pull/141) | `cnj/t1-m3-item-lifecycle` | B2 (move claims replaced by `move` preconditions) |
| [#142](https://github.com/timja/jenkins/pull/142) | `cnj/t1-m4-item-change-applier` | B2 (build branches to P1) |
| [#143](https://github.com/timja/jenkins/pull/143) | `cnj/t2-m1-stored-build-numbers` | P1 (`StoredBuildNumberAssigner`), A5 (`proposeSharedNumber`) |
| [#144](https://github.com/timja/jenkins/pull/144) | `cnj/t2-m2-trigger-coordination` | A2 |
| [#145](https://github.com/timja/jenkins/pull/145) | `cnj/t3-m1-build-snapshots` | C1, A5, P1 |
| [#146](https://github.com/timja/jenkins/pull/146) | `cnj/t3-m2-remote-build-events` | A5, P1 |
| [#147](https://github.com/timja/jenkins/pull/147) | `cnj/t3-m3-running-placeholders` | C2, P2 |
| [#148](https://github.com/timja/jenkins/pull/148) | `cnj/t3-m4-explicit-numbering` | P1 (a job made again now starts afresh) |
| [#149](https://github.com/timja/jenkins/pull/149) | `cnj/t4-m2-build-leases` | P0 (leases), P2 |
| [#150](https://github.com/timja/jenkins/pull/150) | `cnj/t5-m1-cluster-queue` | A4, C3, P3 |
| [#151](https://github.com/timja/jenkins/pull/151) | `cnj/t5-m2-cluster-blocking` | P3 |
| [#152](https://github.com/timja/jenkins/pull/152) | `cnj/t5-m3-cluster-periodic-work` | A3 (`FingerprintCleanupThread.Shared` to the plugin) |
| [#153](https://github.com/timja/jenkins/pull/153) | `cnj/t6-m1-static-routing` | B1 (routing), B3 |
| [#154](https://github.com/timja/jenkins/pull/154) | `cnj/t6-m2-static-applier` | B3 |
| [#155](https://github.com/timja/jenkins/pull/155) | `cnj/t6-m3-shared-keys` | B4 |
| [#156](https://github.com/timja/jenkins/pull/156) | `cnj/t6-m6-stateless-sessions` | B4 (needs Stapler 1.2 released) |
| [#157](https://github.com/timja/jenkins/pull/157) | `cnj/t9-m1-provisioning-demand` | A4 (`countsForProvisioning`), B3 (cloud nodes local) |
| [#158](https://github.com/timja/jenkins/pull/158) | `cnj/t11-m1-xml-merge` | B1 (`StoredConfigReconciler` hook), P5 (merge) |
| [#159](https://github.com/timja/jenkins/pull/159) | `cnj/t10-m1-import-api` | P1 (the import uses the plugin's build records directly) |
| [#160](https://github.com/timja/jenkins/pull/160) | `cnj/t12-m1-remainder` | C2, P1, P2 |
| [#161](https://github.com/timja/jenkins/pull/161) | `cnj/t14-m1-resilience` | B1, B2, P* (store-outage fixes go with the code they fix) |
| [#162](https://github.com/timja/jenkins/pull/162) | `cnj/t15-remaining` | A1, A5, A6, A7, B1–B3, C2, P1–P5 |

## Pull requests
All on forks, as drafts. Bodies follow each repository's PR template (what, how, testing) and link the plan row; no commit lists.

| Repo | Stack | PRs |
|---|---|---|
| [timja/jenkins](https://github.com/timja/jenkins) | core cnj2: A1–A7 off `fork/cnj/base`; B1 → B2 → B3 → B4; C1–C3 on B1 | [#164](https://github.com/timja/jenkins/pull/164) … [#177](https://github.com/timja/jenkins/pull/177) (14 PRs); GitHub stacks #178 (#171–#174) and #179 (#176–#177); old [#139](https://github.com/timja/jenkins/pull/139) … [#162](https://github.com/timja/jenkins/pull/162) closed |
| [timja/cluster-plugin](https://github.com/timja/cluster-plugin) | P0–P6, P8 on `main` (P0 and P1 inside P2, which is where they first compile); harness and kind seeding on top | [cluster-plugin#1](https://github.com/timja/cluster-plugin/pull/1) … [cluster-plugin#6](https://github.com/timja/cluster-plugin/pull/6); harness [cluster-plugin#7](https://github.com/timja/cluster-plugin/pull/7); GitHub stack #8 (#1–#7) |
| [timja/jep](https://github.com/timja/jep) | JEPs 250–257 with 255a/255b (Wave 0) | [#4](https://github.com/timja/jep/pull/4) |
| [timja/lockable-resources-plugin](https://github.com/timja/lockable-resources-plugin) | L1 (1.6) | [#4](https://github.com/timja/lockable-resources-plugin/pull/4); old [#1](https://github.com/timja/lockable-resources-plugin/pull/1), [#2](https://github.com/timja/lockable-resources-plugin/pull/2) closed |
| [timja/pipeline-build-step-plugin](https://github.com/timja/pipeline-build-step-plugin) | S1 (1.7) | [#2](https://github.com/timja/pipeline-build-step-plugin/pull/2); old [#1](https://github.com/timja/pipeline-build-step-plugin/pull/1) closed |
| [timja/stapler](https://github.com/timja/stapler) | 1.2 | [#1](https://github.com/timja/stapler/pull/1) |
| [timja/workflow-cps-plugin](https://github.com/timja/workflow-cps-plugin) | 1.1 | [#1](https://github.com/timja/workflow-cps-plugin/pull/1) |
| [timja/configuration-as-code-plugin](https://github.com/timja/configuration-as-code-plugin) | 1.3 | [#116](https://github.com/timja/configuration-as-code-plugin/pull/116) |
| [timja/kubernetes-plugin](https://github.com/timja/kubernetes-plugin) | 1.4 | [#1](https://github.com/timja/kubernetes-plugin/pull/1) |
| [timja/branch-api-plugin](https://github.com/timja/branch-api-plugin) | 1.5 | [#1](https://github.com/timja/branch-api-plugin/pull/1) |
| [timja/cloudbees-folder-plugin](https://github.com/timja/cloudbees-folder-plugin) | 3.1, 3.2 | [#1](https://github.com/timja/cloudbees-folder-plugin/pull/1), [#2](https://github.com/timja/cloudbees-folder-plugin/pull/2) |
| [timja/credentials-plugin](https://github.com/timja/credentials-plugin) | 3.3 | [#1](https://github.com/timja/credentials-plugin/pull/1) |
| [timja/jenkins-junit-postgresql-plugin](https://github.com/timja/jenkins-junit-postgresql-plugin) | 3.4, 3.5 | [#52](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/52), [#53](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/53) |

## Reviews
Nothing was replied to on GitHub in any round.
- Round 1, Copilot, 2026-10-14 (#139–#162): 101 comments, six design decisions. [reviews/copilot-2026-10-14.md](reviews/copilot-2026-10-14.md)
- Round 2, Copilot, 2026-10-15 (with timja/jep#4): 134 comments, four design decisions. [reviews/copilot-2026-10-15.md](reviews/copilot-2026-10-15.md)
- Round 3, Copilot, 2026-10-17 (with timja/jep#4): 107 comments, six design decisions. [reviews/copilot-2026-10-17.md](reviews/copilot-2026-10-17.md)
- Round 4, independent agents (several models), 2026-10-17: the whole stack, the plugin and Stapler PRs and the JEPs; eight user decisions, which led to the cnj2 re-cut. [reviews/agents-2026-10-17.md](reviews/agents-2026-10-17.md)

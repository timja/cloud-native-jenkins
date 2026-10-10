# Target 1: progress and branches

Design: [01-shared-item-config.md](01-shared-item-config.md).

**Done means:** `validation/experiments/ET1-shared-items.sh` passes.

> **Status: DONE (2026-10-10).** ET1 passes, 0 failures on two consecutive runs. Evidence is on branch `cnj/t1-m6-postgres-store`: `findings/ET1-shared-items.md` and `findings/raw/ET1/`. It was built from core `cnj/t1-m4-item-change-applier` @ `9caafa1e30`, cloudbees-folder `cnj/t1-m2-folder-children` @ `7699d44`, and the M6 plugin.

**Policy:**
- Everything stays **local; nothing is pushed.**
- Milestones are **stacked branches**: each branch starts from the previous milestone in the same repo.
- Integration branches are temporary, for running E-T1 across repos.

## Branch stack

### jenkins core (`/Users/timja/code/jenkins/jenkins`), base `upstream/master` @ `cd2a077aed` (2.586-SNAPSHOT)
| # | Branch | Based on | Scope | Status |
|---|---|---|---|---|
| M1 | `cnj/t1-m1-persistence-spi` | `upstream/master` | `jenkins.persistence` SPI (`PersistenceStore`, `PersistenceStoreProvider`, `PersistenceStores`, `InMemoryPersistenceStore`); `XmlFile` routing for owned keys; client-side versions + conflict policy (warn); `XmlFile.writeRaw`/`readRawBytes`; `ItemGroupMixIn` create-from-XML / copy go through `XmlFile`. 3 commits (SPI; deadlock fix; keep local dirs) | ✅ 8 new tests + 61 regression tests green |
| M2 | `cnj/t1-m2-item-enumeration` | M1 | `PersistenceStore.children`; `Jenkins.loadTasks` loads store-only items (extra reactor task, since the job list is built before plugins load); `ItemGroupMixIn.loadChildren` unions store children | ✅ 2 new tests + regression green |
| M3 | `cnj/t1-m3-item-lifecycle` | M2 | `PersistenceStore.move`; rename (`AbstractItem.renameTo`) and move (`Items.move`) move owned keys first (conflict if target exists in store); delete already routed via `XmlFile.delete` | ✅ 4 new tests + regression green |
| M4 | `cnj/t1-m4-item-change-applier` | M3 | `getOrigin`/`subscribe` + `ChangeEvent`; `ItemChangeApplier` (create / update / move / delete, origin suppression, parking); `PersistenceStores.applyingRemoteChange`; `Items.renameToMatchStore` | ✅ 8 new tests green |
| M5 | `cnj/t1-m5-conflict-policy` | M4 | conflict metrics + admin monitor; strict mode for UI/REST saves; writer-aware retries | deferred: not needed for ET1 (0 conflicts observed, see ET1 observations); branch not created |

### cloudbees-folder (`/Users/timja/code/jenkins/cloudbees-folder-plugin`), base `origin/master` @ `3ad9a6d`
| # | Branch | Based on | Scope | Status |
|---|---|---|---|---|
| M2f | `cnj/t1-m2-folder-children` | `origin/master` | `DefaultChildLoader` (folder's `ChildLoader` extension point) unions store children; `PersistenceStoreChildLoaderTest`. `jenkins.version` → local `2.586-SNAPSHOT`, `useBeta` (revert before upstreaming) | ✅ 26 tests green (new + ChildLoaderTest + FolderTest); exercised by ET1 |
| M4f | `cnj/t1-m4-folder-remote-children` | M2f | anything `ComputedFolder` needs to accept remote-created children | not needed: core reloads the parent folder, and ET1 scenario 3 passes without folder changes; branch not created |

### cloud-native-jenkins (this repo), base `main`
| # | Branch | Based on | Scope | Status |
|---|---|---|---|---|
| M6 | `cnj/t1-m6-postgres-store` (worktree `../cloud-native-jenkins-m6`) | `main` | `impl/persistence-store-postgresql` plugin (event log table + LISTEN/NOTIFY); `compose.et1.yaml` (tmpfs homes, S0 seed, local plugins); `ET1-build.sh`; `ET1-shared-items.sh`; ET1 evidence | ✅ 9 store tests green (Testcontainers); **ET1 PASS** |

### Integration (temporary)
| Branch | Repo | Composition | Purpose |
|---|---|---|---|
| (none needed) | | | The core stack is linear, so its top (`cnj/t1-m4-item-change-applier`) *is* the integration point. `ET1-build.sh` combines it with the folder and M6 branches without merging. |

## Worktrees
- `/Users/timja/code/jenkins/jenkins-cnj-test`: detached core checkout used to build/test any branch without disturbing the main checkout.
- `/Users/timja/code/jenkins/cloudbees-folder-plugin-cnj-build`: detached folder checkout used by `ET1-build.sh`.
- `/Users/timja/code/jenkins/cloud-native-jenkins-m6`: M6 branch.

## Log
- 2026-10-10: Stack planned. Core M1 branch created from `upstream/master` `cd2a077aed`. A baseline war build is running to warm the cache.
- 2026-10-10: M1 found and fixed a lock-order deadlock (store resolution vs. extension loading) that hung ordinary startup even with no store configured; caught by the regression run.
- 2026-10-10: M1–M4 core, M2f folder, M6 plugin + harness committed. Design deltas from the target doc: no `FilesystemPersistenceStore` indirection (unrouted `XmlFile` keeps its code path); routed writes keep the local item directory; `children()` and `move()` instead of a generic `apply(ops)`; remote changes reuse core rename/move/delete under `applyingRemoteChange`.
- 2026-10-10: ET1 harness bugs fixed along the way: `docker cp` cannot read tmpfs (seed via `docker exec tar`); the image must be rebuilt for entrypoint changes; plugins need `Files.createDirectories` (`Util.createDirectories` is restricted).
- 2026-10-10: **ET1 PASS** (0 failures, two consecutive runs). Added scenario 4b (80 builds across A and B rewriting one job's config with different values → converged, 0 conflicts). M5 and M4f deferred as unnecessary for the acceptance criterion.

## Review and merge order
1. core `cnj/t1-m1-persistence-spi` (3 commits) → `m2` → `m3` → `m4`: each reviewable on its own, stacked.
2. cloudbees-folder `cnj/t1-m2-folder-children` (needs core M2 released; drop the `jenkins.version` SNAPSHOT override).
3. cloud-native-jenkins `cnj/t1-m6-postgres-store` → `main`: the plugin, harness and evidence.

## Follow-ups (not required for Target 1)
- M5: admin monitor for conflict counts; strict policy for UI/REST saves; writer-aware retry for `JobPropertyStep` and indexing; a harness variant with injected applier latency to exercise the `warn` path.
- Remote creation in folders reloads the whole parent folder (O(children) per created child). Use `AbstractFolder.addLoadedChild` through a core hook if large multibranch projects show it.
- `getFile()` callers (E7: 42 plugins) get a read-only snapshot; direct writes are lost. Audit the BOM ones.

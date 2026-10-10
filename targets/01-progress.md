# Target 1: progress and branches

Design: [01-shared-item-config.md](01-shared-item-config.md).

**Done means:** `validation/experiments/ET1-shared-items.sh` passes.

**Policy:**
- Everything stays **local; nothing is pushed.**
- Milestones are **stacked branches**: each branch starts from the previous milestone in the same repo.
- Integration branches are temporary, for running E-T1 across repos.

## Branch stack

### jenkins core (`/Users/timja/code/jenkins/jenkins`), base `upstream/master` @ `cd2a077aed` (2.586-SNAPSHOT)
| # | Branch | Based on | Scope | Status |
|---|---|---|---|---|
| M1 | `cnj/t1-m1-persistence-spi` | `upstream/master` | `jenkins.persistence` SPI (`PersistenceStore`, `PersistenceStoreProvider`, `PersistenceStores`); `XmlFile` routing for owned keys; client-side versions + conflict policy (warn); `XmlFile.writeRaw`; `ItemGroupMixIn` create-from-XML / copy go through `XmlFile` | in progress |
| M2 | `cnj/t1-m2-item-enumeration` | M1 | `PersistenceStore.children`; `Jenkins.loadTasks` loads store-only items; `ItemGroupMixIn.loadChildren` unions store children | not started |
| M3 | `cnj/t1-m3-item-lifecycle` | M2 | `PersistenceStore.move`; rename (`AbstractItem.renameTo`) and move (`Items.move`) move owned keys; delete is already routed via `XmlFile.delete` | not started |
| M4 | `cnj/t1-m4-item-change-applier` | M3 | `subscribe` + `ChangeEvent`; `ItemChangeApplier` (create / update / move / delete, origin suppression, parking) | not started |
| M5 | `cnj/t1-m5-conflict-policy` | M4 | conflict metrics + admin monitor; strict mode for UI/REST saves | not started |

### cloudbees-folder (`/Users/timja/code/jenkins/cloudbees-folder-plugin`), base `origin/master` @ `3ad9a6d`
| # | Branch | Based on | Scope | Status |
|---|---|---|---|---|
| M2f | `cnj/t1-m2-folder-children` | `origin/master` | `AbstractFolder.loadChildren` unions store children (needs core M2) | not started |
| M4f | `cnj/t1-m4-folder-remote-children` | M2f | anything `ComputedFolder` needs to accept remote-created children (from E-T1 scenario 3) | not started |

### cloud-native-jenkins (this repo), base `main`
| # | Branch | Based on | Scope | Status |
|---|---|---|---|---|
| M6 | `cnj/t1-m6-postgres-store` | `main` | `impl/persistence-store-postgresql` plugin; harness: tmpfs homes, extra plugins, core war; `ET1-shared-items.sh` | not started |

### Integration (temporary)
| Branch | Repo | Composition | Purpose |
|---|---|---|---|
| (none yet) | | | |

## Log
- 2026-10-10: Stack planned. Core M1 branch created from `upstream/master` `cd2a077aed`. A baseline war build is running to warm the cache.

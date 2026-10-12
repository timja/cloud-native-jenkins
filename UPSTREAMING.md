# Upstreaming plan

All prototype work is pushed as stacked pull requests to **forks** (`timja/*`) for review first. Nothing has been opened against `jenkinsci/*`. This document tracks every PR, and the order and priority for taking each one upstream.

**Status key:** 🟡 review PR open on the fork · ⬜ not pushed · ✅ upstreamed

## Principles
1. **The JEPs come first.** JEP-250 (the persistence SPI) is the foundation, and every core PR refers to it. Submit JEP-250 to `jenkinsci/jep` as a draft and get feedback before any core PR is opened. JEPs 251–257 follow it, in number order.
2. **Changes that stand alone go first.** They are useful on one controller, need no new SPI, and are small: they build trust and shorten the later stacks.
3. **The core stack lands in order.** Each branch builds on the one before it. Upstream it as smaller PRs where a branch is large; see "Splitting" below.
4. **Plugins that use the new core APIs wait** for a core release that has them, then raise their `jenkins.version`.
5. **"Revert before upstreaming" commits** point a build at locally installed snapshots. Drop them before opening an upstream PR.
6. **The security review runs separately.** No PR describes vulnerabilities; the JEPs' security sections stay generic.

## Order and priority

### Wave 0: proposals
| # | What | Where | Priority | Notes |
|---|---|---|---|---|
| 0.1 | JEP-250 Persistence SPI | `jenkinsci/jep` | P0 | Draft; everything depends on it |
| 0.2 | JEPs 251–257 | `jenkinsci/jep` | P1 | One at a time, after 250 has had a first round |

### Wave 1: changes that stand alone (no new core SPI)
| # | Repo | Branch | Fork PR | Priority | Notes |
|---|---|---|---|---|---|
| 1.1 | workflow-cps | `cnj/t4-m1-program-saved-listener` | ⬜ | P1 | `ProgramSavedListener`, a durability hook. Useful for any external `LogStorage` (JEP-252) |
| 1.2 | stapler | `cnj/t6-m5-bound-id-prefix` | ⬜ | P1 | An optional server-id prefix for bound object ids. Rebase onto the current `master` first: the local base was stale |
| 1.3 | configuration-as-code | `cnj/t15-keep-undeclared-nodes` | ⬜ | P2 | `keepUndeclaredNodes`, also useful with one controller (nodes added by other means) |
| 1.4 | kubernetes | `cnj/t9-m2-cluster-cap` | ⬜ | P2 | `ClusterProvisioningLimits` (Beta) extension point. No core dependency |
| 1.5 | core | split out of `cnj/t15-remaining` | ⬜ | P2 | Hooks useful without a store: `PluginManager/notices.jelly`, `PluginArchiveListener`, `Item.getItemUuid`, `Jenkins` setters under the lock. Cherry-pick into standalone PRs |

### Wave 2: core, the persistence stack (`timja/jenkins`, base `cd2a077`, linear)
| # | Branch | Fork PR | Priority | Size | What |
|---|---|---|---|---|---|
| 2.1 | `cnj/t1-m1-persistence-spi` | ⬜ | P0 | 8 files, +866 | SPI, `XmlFile` routing |
| 2.2 | `cnj/t1-m2-item-enumeration` | ⬜ | P0 | +194 | Items only in the store |
| 2.3 | `cnj/t1-m3-item-lifecycle` | ⬜ | P0 | +175 | Rename and move |
| 2.4 | `cnj/t1-m4-item-change-applier` | ⬜ | P0 | +662 | Applying others' changes |
| 2.5 | `cnj/t2-m1-stored-build-numbers` | ⬜ | P1 | +277 | Cluster build numbers |
| 2.6 | `cnj/t2-m2-trigger-coordination` | ⬜ | P1 | +159 | `TriggerCoordinator` |
| 2.7 | `cnj/t3-m1-build-snapshots` | ⬜ | P1 | +907 | Build records and snapshots |
| 2.8 | `cnj/t3-m2-remote-build-events` | ⬜ | P1 | +349 | Remote build events |
| 2.9 | `cnj/t3-m3-running-placeholders` | ⬜ | P1 | +691 | Running elsewhere |
| 2.10 | `cnj/t3-m4-explicit-numbering` | ⬜ | P1 | +263 | History deletion keeps numbers |
| 2.11 | `cnj/t4-m2-build-leases` | ⬜ | P1 | +798 | Leases, fencing, adoption |
| 2.12 | `cnj/t5-m1-cluster-queue` | ⬜ | P1 | +1263 | Shared queue |
| 2.13 | `cnj/t5-m2-cluster-blocking` | ⬜ | P1 | +480 | Stop and blocking across controllers |
| 2.14 | `cnj/t5-m3-cluster-periodic-work` | ⬜ | P2 | +242 | `@ClusterSingleton` |
| 2.15 | `cnj/t6-m1-static-routing` | ⬜ | P1 | +219 | Users, nodes, descriptors |
| 2.16 | `cnj/t6-m2-static-applier` | ⬜ | P1 | +414 | Static state applier |
| 2.17 | `cnj/t6-m3-shared-keys` | ⬜ | P1 | +262 | Shared runtime keys |
| 2.18 | `cnj/t6-m6-stateless-sessions` | ⬜ | P1 | +239 | Sessions across controllers. Has a "revert before upstreaming" Stapler snapshot commit; needs 1.2 released |
| 2.19 | `cnj/t9-m1-provisioning-demand` | ⬜ | P2 | +296 | Provisioning demand |
| 2.20 | `cnj/t11-m1-xml-merge` | ⬜ | P1 | +658 | Three-way merge |
| 2.21 | `cnj/t10-m1-import-api` | ⬜ | P2 | +3 | Import API |
| 2.22 | `cnj/t12-m1-remainder` | ⬜ | P1 | +444 | Handover, aborted closes, late files |
| 2.23 | `cnj/t14-m1-resilience` | ⬜ | P1 | +412 | Store outages |
| 2.24 | `cnj/t15-remaining` | ⬜ | P1 | 45 files, +3284 (18 commits) | Target 15 (split before upstreaming) |

**Splitting.** Upstream reviewers will want PRs of a few hundred lines. In particular:
- `t5-m1-cluster-queue`: the id strategy, then mirroring, then claiming.
- `t15-remaining` (one commit per topic already): the plugin-manager hooks; local state (reloads, late files, write-back, `@PerController`, sub-items, folder children); item UUIDs; deliveries and `SharedQueue.takenElsewhere`; `LostLog`; CLI follow; `deleteAll`; `strict-ui`; `ItemFiles` (items and users); stored permalinks; the fixes to the merge (fold into 2.20), to user reloads (fold into 2.16) and to the restart reclaim (fold into 2.22).
- The fixes made in later branches (marked "Target N fix" in commit messages) should be folded into the branch that introduced the code.

### Wave 3: plugins using the new core APIs (after a core release containing them)
| # | Repo | Branch | Fork PR | Needs | Priority | Notes |
|---|---|---|---|---|---|---|
| 3.1 | cloudbees-folder | `cnj/t1-m2-folder-children` | ⬜ | 2.2 | P0 | Folder children only in the store |
| 3.2 | cloudbees-folder | `cnj/t15-remote-child` | ⬜ | 2.24 (`RemoteItemGroup`) | P2 | Stacked on 3.1 |
| 3.3 | credentials | `cnj/t6-m4-credentials-reload` | ⬜ | 2.16 | P1 | Has a "revert before upstreaming" commit (core snapshot). Accidental reformatting was removed from the branch before the push (+126 lines) |
| 3.4 | lockable-resources | `cnj/t5-m4-cluster-locks` | ⬜ | 2.12 | P1 | Has a "revert before upstreaming" commit |
| 3.5 | lockable-resources | `cnj/t15-cluster-ui` | ⬜ | 3.4 | P2 | Stacked on 3.4 |
| 3.6 | junit-sql-storage | `cnj/t3-m5-remote-delete` | ⬜ | 2.8 | P1 | Fork `timja/jenkins-junit-postgresql-plugin`. Has a "revert before upstreaming" commit. Accidental reformatting was removed before the push (+31 lines) |
| 3.7 | junit-sql-storage | `cnj/t15-follow-renames` | ⬜ | 2.24 (`getItemUuid`) | P2 | Stacked on 3.6 |
| 3.8 | pipeline-build-step | `cnj/t15-remote-completion` | ⬜ | 2.24 (deliveries, `SharedQueue.takenElsewhere`) | P2 | No `timja` fork exists yet. No unit tests yet (covered by ET12 1c/1d): add some before upstreaming |

### Wave 4: the store plugin, chart and validation
| # | What | Where | Priority | Notes |
|---|---|---|---|---|
| 4.1 | `persistence-store-postgresql` plugin | new repository (hosting request) | P1 | Reference implementation of every SPI; release after 2.x is in a weekly |
| 4.2 | Helm chart | `jenkinsci/helm-charts`, or with the plugin | P2 | |
| 4.3 | Acceptance experiments (`validation/`) | with the plugin | P3 | Docker Compose and kind |

## Pull requests
Filled in as they are opened.

| Repo | Branch | PR |
|---|---|---|

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
| 0.1 | JEP-250 Persistence SPI | `jenkinsci/jep` | P0 | Draft; everything depends on it. With 251–257 as AsciiDoc in 🟡 [timja/jep#4](https://github.com/timja/jep/pull/4) |
| 0.2 | JEPs 251–257 | `jenkinsci/jep` | P1 | In the same fork PR as 250; to jenkinsci one at a time, after 250 has had a first round |

### Wave 1: changes that stand alone (no new core SPI)
| # | Repo | Branch | Fork PR | Priority | Notes |
|---|---|---|---|---|---|
| 1.1 | workflow-cps | `cnj/t4-m1-program-saved-listener` | 🟡 [workflow-cps-plugin#1](https://github.com/timja/workflow-cps-plugin/pull/1) | P1 | `ProgramSavedListener`, a durability hook. Useful for any external `LogStorage` (JEP-252) |
| 1.2 | stapler | `cnj/t6-m5-bound-id-prefix` | 🟡 [stapler#1](https://github.com/timja/stapler/pull/1) | P1 | An optional server-id prefix for bound object ids. Rebase onto the current `master` first: the local base was stale |
| 1.3 | configuration-as-code | `cnj/t15-keep-undeclared-nodes` | 🟡 [configuration-as-code-plugin#116](https://github.com/timja/configuration-as-code-plugin/pull/116) | P2 | `keepUndeclaredNodes`, also useful with one controller (nodes added by other means) |
| 1.4 | kubernetes | `cnj/t9-m2-cluster-cap` | 🟡 [kubernetes-plugin#1](https://github.com/timja/kubernetes-plugin/pull/1) | P2 | `ClusterProvisioningLimits` (Beta) extension point. No core dependency |
| 1.5 | core | split out of `cnj/t15-remaining` | ⬜ | P2 | Hooks useful without a store: `PluginManager/notices.jelly`, `PluginArchiveListener`, `Item.getItemUuid`, `Jenkins` setters under the lock. Cherry-pick into standalone PRs |
| 1.6 | branch-api | `cnj/t15-first-build-revision` | 🟡 [branch-api-plugin#1](https://github.com/timja/branch-api-plugin/pull/1) | P1 | An indexing that stopped after scheduling a branch's build, before recording its revision, no longer leaves the next one to build the branch again. A plain crash fix too; no core dependency. Built with spotless checks skipped (`spotless:apply` reformats the module at this base) |

### Wave 2: core, the persistence stack (`timja/jenkins`, base `cd2a077`, linear)
| # | Branch | Fork PR | Priority | Size | What |
|---|---|---|---|---|---|
| 2.1 | `cnj/t1-m1-persistence-spi` | 🟡 [jenkins#139](https://github.com/timja/jenkins/pull/139) | P0 | 8 files, +866 | SPI, `XmlFile` routing |
| 2.2 | `cnj/t1-m2-item-enumeration` | 🟡 [jenkins#140](https://github.com/timja/jenkins/pull/140) | P0 | +194 | Items only in the store |
| 2.3 | `cnj/t1-m3-item-lifecycle` | 🟡 [jenkins#141](https://github.com/timja/jenkins/pull/141) | P0 | +175 | Rename and move |
| 2.4 | `cnj/t1-m4-item-change-applier` | 🟡 [jenkins#142](https://github.com/timja/jenkins/pull/142) | P0 | +662 | Applying others' changes |
| 2.5 | `cnj/t2-m1-stored-build-numbers` | 🟡 [jenkins#143](https://github.com/timja/jenkins/pull/143) | P1 | +277 | Cluster build numbers |
| 2.6 | `cnj/t2-m2-trigger-coordination` | 🟡 [jenkins#144](https://github.com/timja/jenkins/pull/144) | P1 | +159 | `TriggerCoordinator` |
| 2.7 | `cnj/t3-m1-build-snapshots` | 🟡 [jenkins#145](https://github.com/timja/jenkins/pull/145) | P1 | +907 | Build records and snapshots |
| 2.8 | `cnj/t3-m2-remote-build-events` | 🟡 [jenkins#146](https://github.com/timja/jenkins/pull/146) | P1 | +349 | Remote build events |
| 2.9 | `cnj/t3-m3-running-placeholders` | 🟡 [jenkins#147](https://github.com/timja/jenkins/pull/147) | P1 | +691 | Running elsewhere |
| 2.10 | `cnj/t3-m4-explicit-numbering` | 🟡 [jenkins#148](https://github.com/timja/jenkins/pull/148) | P1 | +263 | History deletion keeps numbers |
| 2.11 | `cnj/t4-m2-build-leases` | 🟡 [jenkins#149](https://github.com/timja/jenkins/pull/149) | P1 | +798 | Leases, fencing, adoption |
| 2.12 | `cnj/t5-m1-cluster-queue` | 🟡 [jenkins#150](https://github.com/timja/jenkins/pull/150) | P1 | +1263 | Shared queue |
| 2.13 | `cnj/t5-m2-cluster-blocking` | 🟡 [jenkins#151](https://github.com/timja/jenkins/pull/151) | P1 | +480 | Stop and blocking across controllers |
| 2.14 | `cnj/t5-m3-cluster-periodic-work` | 🟡 [jenkins#152](https://github.com/timja/jenkins/pull/152) | P2 | +242 | `@ClusterSingleton` |
| 2.15 | `cnj/t6-m1-static-routing` | 🟡 [jenkins#153](https://github.com/timja/jenkins/pull/153) | P1 | +219 | Users, nodes, descriptors |
| 2.16 | `cnj/t6-m2-static-applier` | 🟡 [jenkins#154](https://github.com/timja/jenkins/pull/154) | P1 | +414 | Static state applier |
| 2.17 | `cnj/t6-m3-shared-keys` | 🟡 [jenkins#155](https://github.com/timja/jenkins/pull/155) | P1 | +262 | Shared runtime keys |
| 2.18 | `cnj/t6-m6-stateless-sessions` | 🟡 [jenkins#156](https://github.com/timja/jenkins/pull/156) | P1 | +239 | Sessions across controllers. Has a "revert before upstreaming" Stapler snapshot commit; needs 1.2 released |
| 2.19 | `cnj/t9-m1-provisioning-demand` | 🟡 [jenkins#157](https://github.com/timja/jenkins/pull/157) | P2 | +296 | Provisioning demand |
| 2.20 | `cnj/t11-m1-xml-merge` | 🟡 [jenkins#158](https://github.com/timja/jenkins/pull/158) | P1 | +658 | Three-way merge |
| 2.21 | `cnj/t10-m1-import-api` | 🟡 [jenkins#159](https://github.com/timja/jenkins/pull/159) | P2 | +3 | Import API |
| 2.22 | `cnj/t12-m1-remainder` | 🟡 [jenkins#160](https://github.com/timja/jenkins/pull/160) | P1 | +444 | Handover, aborted closes, late files |
| 2.23 | `cnj/t14-m1-resilience` | 🟡 [jenkins#161](https://github.com/timja/jenkins/pull/161) | P1 | +412 | Store outages |
| 2.24 | `cnj/t15-remaining` | 🟡 [jenkins#162](https://github.com/timja/jenkins/pull/162) | P1 | 48 files, +3538 (24 commits) | Target 15 (split before upstreaming) |

**Splitting.** Upstream reviewers will want PRs of a few hundred lines. In particular:
- `t5-m1-cluster-queue`: the id strategy, then mirroring, then claiming.
- `t15-remaining` (one commit per topic already): the plugin-manager hooks; local state (reloads, late files, write-back, `@PerController`, sub-items, folder children); item UUIDs; deliveries and `SharedQueue.takenElsewhere`; `LostLog`; CLI follow; `deleteAll`; `strict-ui`; `ItemFiles` (items and users); stored permalinks; the fixes to the merge (fold into 2.20), to user reloads (fold into 2.16) and to the restart reclaim (fold into 2.22).
- The fixes made in later branches (marked "Target N fix" in commit messages) should be folded into the branch that introduced the code.

### Wave 3: plugins using the new core APIs (after a core release containing them)
| # | Repo | Branch | Fork PR | Needs | Priority | Notes |
|---|---|---|---|---|---|---|
| 3.1 | cloudbees-folder | `cnj/t1-m2-folder-children` | 🟡 [cloudbees-folder-plugin#1](https://github.com/timja/cloudbees-folder-plugin/pull/1) | 2.2 | P0 | Folder children only in the store |
| 3.2 | cloudbees-folder | `cnj/t15-remote-child` | 🟡 [cloudbees-folder-plugin#2](https://github.com/timja/cloudbees-folder-plugin/pull/2) | 2.24 (`RemoteItemGroup`) | P2 | Stacked on 3.1 |
| 3.3 | credentials | `cnj/t6-m4-credentials-reload` | 🟡 [credentials-plugin#1](https://github.com/timja/credentials-plugin/pull/1) | 2.16 | P1 | Has a "revert before upstreaming" commit (core snapshot). Accidental reformatting was removed from the branch before the push (+126 lines) |
| 3.4 | lockable-resources | `cnj/t5-m4-cluster-locks` | 🟡 [lockable-resources-plugin#1](https://github.com/timja/lockable-resources-plugin/pull/1) | 2.12 | P1 | Has a "revert before upstreaming" commit |
| 3.5 | lockable-resources | `cnj/t15-cluster-ui` | 🟡 [lockable-resources-plugin#2](https://github.com/timja/lockable-resources-plugin/pull/2) | 3.4 | P2 | Stacked on 3.4 |
| 3.6 | junit-sql-storage | `cnj/t3-m5-remote-delete` | 🟡 [jenkins-junit-postgresql-plugin#52](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/52) | 2.8 | P1 | Fork `timja/jenkins-junit-postgresql-plugin`. Has a "revert before upstreaming" commit. Accidental reformatting was removed before the push (+31 lines) |
| 3.7 | junit-sql-storage | `cnj/t15-follow-renames` | 🟡 [jenkins-junit-postgresql-plugin#53](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/53) | 2.24 (`getItemUuid`) | P2 | Stacked on 3.6 |
| 3.8 | pipeline-build-step | `cnj/t15-remote-completion` | 🟡 [pipeline-build-step-plugin#1](https://github.com/timja/pipeline-build-step-plugin/pull/1) | 2.24 (deliveries, `SharedQueue.takenElsewhere`) | P2 | Fork `timja/pipeline-build-step-plugin` created. Its `pom.xml` points at a local core snapshot. No unit tests yet (covered by ET12 1c/1d): add some before upstreaming |

### Wave 4: the Cluster plugins, chart and validation (Target 16 split the store plugin)
The code is in [timja/cluster-plugin](https://github.com/timja/cluster-plugin) (public; moved from this repository on 2026-10-14, with its history): the four plugins at its root, the chart in `charts/`, the experiments in `validation/`. One hosting request per plugin, splitting that repository then. The names `cluster`, `cluster-postgresql`, `cluster-azure-blob` and `cluster-s3` were free on the update site on 2026-10-14; `jenkins-cluster` is the fallback for `cluster`. They are released after the core stack is in a weekly.

| # | What | Where | Priority | Notes |
|---|---|---|---|---|
| 4.1 | `cluster` plugin | new repository (hosting request) | P1 | Everything backend-neutral, and the cluster SPI (`io.jenkins.plugins.cluster.spi`) the backends implement |
| 4.2 | `cluster-postgresql` plugin | new repository (hosting request) | P1 | The reference store; uses `postgresql-api` for its driver |
| 4.3 | `cluster-azure-blob` plugin | new repository (hosting request) | P1 | Blobs in Azure Blob Storage (append blobs for logs); uses `azure-sdk` |
| 4.4 | `cluster-s3` plugin | new repository (hosting request) | P1 | Blobs in S3 or an S3-compatible store; uses `aws-java-sdk2-s3` |
| 4.5 | Helm chart | `jenkinsci/helm-charts`, or with the plugins | P2 | `blob.type: azure\|s3` |
| 4.6 | Acceptance experiments (`validation/`) | with the plugins | P3 | Docker Compose and kind; an S3 variant on Garage (`experiments/with-s3.sh`) |

## Pull requests
All on forks, as drafts, each stack based on a `cnj/base` branch (the upstream commit the work started from). Bodies list the commits and the plan row.

| Repo | Stack | PRs |
|---|---|---|
| [timja/jenkins](https://github.com/timja/jenkins) | core, 24 branches (rows 2.1–2.24), base `cd2a077` | [#139](https://github.com/timja/jenkins/pull/139) … [#162](https://github.com/timja/jenkins/pull/162) |
| [timja/cluster-plugin](https://github.com/timja/cluster-plugin) | the Cluster plugins, chart and experiments (rows 4.1–4.6), on `main` | none (moved there from the closed #1–#15 and #17 of timja/cloud-native-jenkins, whose branches are kept) |

**Reviews:** Copilot reviewed #139–#162 on 2026-10-14. Its 101 comments, what came of each (82 fixed on their own branch, 11 already fixed later in the stack, the rest documented, partly fixed or answered) and the six design decisions, now implemented, are in [reviews/copilot-2026-10-14.md](reviews/copilot-2026-10-14.md). Nothing was replied to on GitHub. Each branch passes the stack's tests on its own. A second round on 2026-10-15 (134 comments including timja/jep#4: 109 fixed, 11 fixed later, the rest partly fixed, documented or answered; four more design decisions) is in [reviews/copilot-2026-10-15.md](reviews/copilot-2026-10-15.md). A third round on 2026-10-17 (107 comments including timja/jep#4: 86 fixed, the rest partly fixed, documented or answered; six more design decisions) is in [reviews/copilot-2026-10-17.md](reviews/copilot-2026-10-17.md).
| [timja/jep](https://github.com/timja/jep) | JEPs 250–257 (rows 0.1, 0.2) | [#4](https://github.com/timja/jep/pull/4) |
| [timja/stapler](https://github.com/timja/stapler) | 1.2 | [#1](https://github.com/timja/stapler/pull/1) |
| [timja/workflow-cps-plugin](https://github.com/timja/workflow-cps-plugin) | 1.1 | [#1](https://github.com/timja/workflow-cps-plugin/pull/1) |
| [timja/configuration-as-code-plugin](https://github.com/timja/configuration-as-code-plugin) | 1.3 | [#116](https://github.com/timja/configuration-as-code-plugin/pull/116) |
| [timja/kubernetes-plugin](https://github.com/timja/kubernetes-plugin) | 1.4 | [#1](https://github.com/timja/kubernetes-plugin/pull/1) |
| [timja/branch-api-plugin](https://github.com/timja/branch-api-plugin) | 1.6 | [#1](https://github.com/timja/branch-api-plugin/pull/1) |
| [timja/cloudbees-folder-plugin](https://github.com/timja/cloudbees-folder-plugin) | 3.1, 3.2 | [#1](https://github.com/timja/cloudbees-folder-plugin/pull/1), [#2](https://github.com/timja/cloudbees-folder-plugin/pull/2) |
| [timja/credentials-plugin](https://github.com/timja/credentials-plugin) | 3.3 | [#1](https://github.com/timja/credentials-plugin/pull/1) |
| [timja/lockable-resources-plugin](https://github.com/timja/lockable-resources-plugin) | 3.4, 3.5 | [#1](https://github.com/timja/lockable-resources-plugin/pull/1), [#2](https://github.com/timja/lockable-resources-plugin/pull/2) |
| [timja/jenkins-junit-postgresql-plugin](https://github.com/timja/jenkins-junit-postgresql-plugin) | 3.6, 3.7 | [#52](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/52), [#53](https://github.com/timja/jenkins-junit-postgresql-plugin/pull/53) |
| [timja/pipeline-build-step-plugin](https://github.com/timja/pipeline-build-step-plugin) | 3.8 | [#1](https://github.com/timja/pipeline-build-step-plugin/pull/1) |

Row 1.5 (standalone core hooks) is not a branch of its own yet: it is to be cherry-picked out of `cnj/t15-remaining` (#162).

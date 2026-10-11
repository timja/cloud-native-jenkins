# Target 9: progress and branches

Design: [09-kubernetes.md](09-kubernetes.md). **Done means:** `validation/experiments/ET9-kubernetes.sh` passes on kind.

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t9-m1-provisioning-demand` | `cnj/t6-m6-stateless-sessions` | provisioning demand only from items this controller is expected to run; reservations while agents start; cloud agents stay with their controller | ✅ 63d73a7a14, c9d2dc7de7, df261ecd76; `SharedQueueTest` 11/11, `NodeProvisionerTest` 7/7, `StaticStateApplierTest` 6/6, `NodesTest` 16/16 |
| M2 | kubernetes-plugin | `cnj/t9-m2-cluster-cap` | `master` | `ClusterProvisioningLimits` extension point (worktree `../kubernetes-plugin-cnj`) | ✅ 600159e5; `ClusterProvisioningLimitsTest` + `KubernetesProvisioningLimitsTest` green |
| M3 | cloud-native-jenkins | `cnj/t9-m3-chart` | `cnj/t8-m1-…` | helm chart, k8s image, seed Secret, `ClusterAgentLimits` (Postgres counters), ET9 | ✅ ET9 30/30 ×2 on kind |

## Log
- 2026-10-11: Target 9 planned and implemented (the `kindest/node:v1.37.0` image was provided, unblocking it). ET9 found:
  - every controller launching every other controller's Kubernetes agents: cloud agents were shared as nodes (fixed in core)
  - agents started for items another controller then took (fixed: reservations)
  - the shared admin user losing its token or password when controllers boot at once (fixed by the Target 11 merge, run on the top of the core stack)

  **ET9 passes 30/30** in two consecutive runs ([findings](../findings/ET9-kubernetes.md) on `cnj/t9-m3-chart`).

## Follow-ups
- The chart's in-chart Postgres and Azurite are for development; production values for managed services are documented in `values.yaml` but not exercised.
- Pod templates declared in Pipelines (`podTemplate`) behave like JCasC templates; not separately tested.

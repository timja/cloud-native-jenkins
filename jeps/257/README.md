# JEP-257: Cloud provisioning across controllers

| | |
|---|---|
| **JEP** | 257 |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-253, JEP-255 |

## Abstract
Several controllers provision agents from the same clouds (Kubernetes, and by extension EC2, Docker and others) without over-provisioning, exceeding cluster-wide caps, or killing each other's agents.
- Demand is counted once, by the controller that will run the build.
- Agents being provisioned are reserved for the item they are for.
- Caps are enforced through counters in the store.
- Cloud agents stay with the controller that provisioned them.

## Specification
- **Demand counted once.** `NodeProvisioner` counts a queue item's demand only on the controller that will start it: the one that requested it, or that claimed it in the shared queue (`QueueStartGate.countsForProvisioning`). Mirrored items elsewhere do not provision.
- **Reservations.** An agent provisioned for a queue item reserves the item (`queue/reserved/<id>`). Other controllers wait for it instead of starting the item on an agent they would provision too.
- **Caps** across the cluster, through counters in the store (`ClusterAgentLimits`, `jenkins_counters`): each controller's share of a cloud's and template's cap. The kubernetes plugin consults them (`ClusterProvisioningLimits`). A dead controller's share lapses with its membership, and its counter rows are deleted. A counter back at zero is deleted too, so Pipelines declaring their own `podTemplate` (a template per build) leave no rows behind; an hourly pass prunes any left.
- **Cloud agents are local.** Nodes created by a cloud are not shared through the store (`PersistenceStores.keepLocal`). Each controller manages and cleans up only its own. Static nodes are shared (JEP-250).
- **Placement.** A Pipeline whose `node` label only another controller can serve is handed over to it (JEP-252).
- **Workspaces on pods.** An agent pod belongs to the controller that started it and goes with it. A build adopted after its controller died gets a new pod with an empty workspace, so files a later stage needs are carried with `stash`/`unstash`, which are kept in the blob store (JEP-252).

## Motivation
- `NodeProvisioner` runs per JVM: with a mirrored queue, every controller would provision for the same item.
- Orphan cleanup in cloud plugins deletes agents it does not know, so one controller kills another's.

## Reasoning
- **Counting demand where the build will start** keeps each controller's `NodeProvisioner` unchanged in spirit, with no global planner.
- **Keeping cloud agents local** matches how they are connected (one JVM holds the channel) and avoids shared mutable node state for short-lived agents.

## Backwards compatibility
- Without a store, nothing changes.
- Clouds other than kubernetes need to consult the cluster caps to respect them, and their orphan cleanup must skip other controllers' agents.

## Security
Unchanged: clouds use their configured credentials on each controller.

## Testing
- **Store plugin:** `PostgresPersistenceStoreTest` (caps across controllers).
- **Acceptance:** `validation/experiments/ET9-kubernetes.sh` in the cloud-native-jenkins repository runs the Helm chart with two controllers on a local kind cluster and checks:
  - 6 Pipelines started at once through the Service with a cap of 4: at most 4 agent pods at once, one pod per build, each owned by the controller that ran the build
  - the controller running a build on a Kubernetes agent deleted: the build is adopted and retried on a new pod, and the old pod garbage-collected
  - Pipelines declaring their own `podTemplate`, under the cap and across a failover: the retried stage runs on a new pod and gets an earlier stage's files from its stash, and the cluster's counters are back to zero, with no rows left, once the pods are gone
  - no controller deleted another's pods

## Prototype implementation
- **Core:** prototype branch of Jenkins core `cnj/t9-m1-provisioning-demand`.
- **kubernetes plugin:** prototype branch `cnj/t9-m2-cluster-cap`.
- **Store plugin** (`impl/persistence-store-postgresql` in the cloud-native-jenkins repository): `ClusterAgentLimits`.
- **Findings:** `findings/ET9-kubernetes.md`, a document in the cloud-native-jenkins repository on its prototype branches.

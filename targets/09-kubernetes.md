# Target 9: On Kubernetes (helm chart, cloud agents across controllers)

| | |
|---|---|
| **Roadmap tier** | X Ops (helm), R3 (cloud provisioning across controllers) |
| **Status** | **Done** (2026-10-11): ET9 passes 30/30, see [09-progress](09-progress.md) |
| **Depends on** | T1–T8 (everything a controller needs is in Postgres and blob storage) |
| **Environment** | `kind` (v0.23.0 CLI) with the local `kindest/node:v1.37.0` image. Images are loaded with `kind load docker-image`, never pulled: `cnj/jenkins:*`, `postgres:16`, `azurite:3.34.0`, `jenkins/inbound-agent:latest-jdk21` |

## Goal
`helm install` gives N controllers behind one Service, sharing Postgres and blob storage, each in a pod with an `emptyDir` home. Pipelines get Kubernetes agents from whichever controller schedules them:
- no controller over-provisions because the other sees the same queue
- the container cap holds for the cluster
- no controller deletes another's pods

## Design
### 1. Chart (`charts/cloud-native-jenkins` in this repo)
- **Controllers:**
  - a `StatefulSet` of N controllers (stable names give `CNJ_NAME`, and a headless Service gives `CNJ_INTERNAL_URL`)
  - a `Deployment` would also work, since homes are disposable, but stable names make logs and the cluster view readable
  - `emptyDir` home
  - readiness from `/login` (the stateless sessions make any ready pod servable)
  - liveness from the T2 health check
- **Services:**
  - **`jenkins`:** a ClusterIP Service in front of all controllers, with no session affinity (T6). Agents and browsers use it.
  - **`jenkins-headless`:** for the controllers to reach each other (T6/T7 forwarding).
- **Keys:** a seed `Secret` with the S0 keys (pre-plugin keys), produced by the chart's `seed` Job on first install (the same steps as the ET seed), and mounted read-only.
- **Configuration:**
  - a JCasC `ConfigMap`, identical for every controller
  - store and blob settings from a `Secret`
- **Dependencies:** Postgres and Azurite are optional in-chart StatefulSets for development; production points at managed services.
- **RBAC:** for the kubernetes cloud, a ServiceAccount that may manage pods in the agent namespace.

### 2. Kubernetes agents across controllers (kubernetes-plugin and core)
- **Pods belong to their controller:**
  - `KUBERNETES_JENKINS_URL` is set per pod to the controller's internal URL, and the cloud's `jenkinsUrl` is left empty in the shared configuration.
  - Pods are then labelled per controller (`kubernetes.jenkins.io/controller`), connect back to it, and its garbage collection only considers its own pods.
- **Provisioning once (core):**
  - `NodeProvisioner` counts the queue's demand per label (`LoadStatistics`). With T5 every controller's queue holds every waiting item (mirrors), so each controller provisions for all of them.
  - **Change:** only items this controller may claim *now* count. That is its own items, and other controllers' items once their origin-first grace has passed (T5). An item whose origin is alive and within its grace is that controller's to provision.
- **A cluster-wide container cap (kubernetes-plugin):** the cap counts the cloud's pods across all controllers, by the cloud's name label rather than this controller's URL label.
- **A pod's executor is on its controller:** that controller claims the item (T5 placement), so a pod is never used from elsewhere.

### 3. Failover
- **Controller pod deleted:**
  - Its running builds are adopted (T4).
  - Its agent pods lose their controller and are deleted by their own TTL garbage collection, or reused if the T4 agent-reconnect applies to them. Kubernetes agents are usually one-shot, so a build on one is resumed by `node` retry: `retry(conditions: [kubernetesAgent()])`.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | core | `cnj/t9-m1-provisioning-demand` | `cnj/t6-m6-stateless-sessions` | provisioning demand only from claimable items; tests |
| M2 | kubernetes-plugin | `cnj/t9-m2-cluster-cap` | `master` (+ local-core pom bump) | cluster-wide container cap |
| M3 | cloud-native-jenkins | `cnj/t9-m3-chart` | `cnj/t8-m1-…` | the helm chart, seed Job, `ET9-kubernetes.sh` on kind |

## E-T9 acceptance experiment
On a kind cluster, `helm install` with 2 controllers.
1. **Install:** both controllers become ready, are live members, and the cluster view lists both.
2. **One Service:** through the `jenkins` Service, a browser journey (T6) works, and builds started through it are spread over both controllers.
3. **Kubernetes agents:** 6 Pipelines with `podTemplate`/`node(POD_LABEL)` started at once through the Service, with a container cap of 4:
   - every build completes
   - at most 4 agent pods exist at any time
   - each pod ran exactly one build, on the controller that created it
   - no pod was created for an item another controller ran
4. **Controller loss:** delete one controller pod while it runs a Pipeline on a Kubernetes agent. The build completes on the other controller (adopted; `node` retried on a new pod), and the old agent pod is garbage collected.
5. **No foreign GC:** over 3 minutes, no controller deletes a pod labelled for the other.
6. **Scale:** `helm upgrade --set controllers=3`. The third controller joins, sees all jobs and history, and runs builds.
7. **Disposable:** `kubectl delete pod` on every controller at once. They come back with all state, and the Service serves again.

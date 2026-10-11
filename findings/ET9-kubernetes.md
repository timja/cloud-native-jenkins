# ET9: Target 9 acceptance (the helm chart on kind; Kubernetes agents across controllers)

**Result: PASS**, 30/30 checks, in two consecutive runs on the final build. Raw evidence from the last run: `raw/ET9/`
- `results.md`
- `pods.txt`: agent pods sampled every 2 s, with their controller label and phase
- `journey.txt`
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET9-kubernetes.sh`
- **Setup:**
  - a kind cluster (`kind` v0.23.0, `kindest/node:v1.37.0`); every image is loaded with `kind load`, nothing is pulled
  - the chart `charts/cloud-native-jenkins` with 2 controllers (a StatefulSet with `emptyDir` homes), in-chart Postgres and Azurite, and the S0 seed as a Secret (`scripts/make-seed-secret.sh`)
  - the kubernetes cloud with a container cap of 4
  - the `cnj` Service as a NodePort on `localhost:8095`, which kube-proxy balances across the controllers
- **Build:**
  - core `cnj/t10-m1-import-api`, the top of the core stack (it includes the Target 11 merge, needed when controllers booting at once all write the shared admin user)
  - kubernetes-plugin `cnj/t9-m2-cluster-cap`
  - plugin on `cnj/t9-m3-chart`
  - other plugins as ET8

| Criterion | Result |
|---|---|
| 1 Install | ✅ both controllers become ready, and through the Service the cluster view lists both as live members |
| 2 One Service | ✅ 10 builds through the Service ran on both controllers (5/5). The browser journey through the Service passed every step (login, 10/10 signed in, bound objects 12/12, form save, progressive rendering, password change). A freestyle build ran on a Kubernetes agent. HtmlUnit's kept-alive connections mostly reached one controller, so "requests reached both controllers" is recorded as information |
| 3 Kubernetes agents | ✅ 6 Pipelines started at once through the Service, with a cap of 4: all completed; at most 4 agent pods existed at once; 6 pods for 6 builds; each build on its own pod, which belonged to the controller that ran it |
| 4 Controller loss | ✅ the pod of the controller running a Pipeline on a Kubernetes agent was deleted. The build completed: adopted, and retried on a new agent pod (`retry(conditions: [kubernetesAgent(), nonresumable()])`). The old agent pod was garbage collected |
| 5 No foreign GC | ✅ no controller deleted a pod labelled for another |
| 6 Scale | ✅ `helm upgrade --set controllers=3`: the third controller is ready, a live member, sees every job and the history, and runs builds |
| 7 Disposable | ✅ deleting every controller pod at once: they come back with every job and build, and a user created at runtime still signs in |

## How it works
- **Chart:**
  - a StatefulSet of controllers with `emptyDir` homes
  - a Service with no session affinity, plus a headless Service giving each controller its internal URL
  - pods connect back through `KUBERNETES_JENKINS_URL`, set per pod to the controller's internal URL
  - the seed Secret; JCasC in a ConfigMap; RBAC for agent pods
  - optional in-chart Postgres and Azurite, with persistent volumes
- **Core (`cnj/t9-m1-provisioning-demand`):**
  - **Provisioning demand:** only the controller expected to run an item counts it as demand: its claimer, the controller it is pinned to, or else its origin while alive.
  - **Reservations:** a controller starting cloud agents reserves as many of its waiting items, so others wait for the agents (`queue/reserved/<id>` in the store, ignored once its holder is dead or after 5 minutes).
  - **Cloud agents stay with their controller** (below).
- **kubernetes-plugin (`cnj/t9-m2-cluster-cap`):** a `ClusterProvisioningLimits` extension point. The store plugin implements it with Postgres counters per controller (`jenkins_counters`), so container and instance caps hold for the cluster, and a dead controller's share stops counting.

## Bugs found by ET9, now fixed
- **Every controller launched every other controller's Kubernetes agents.**
  - **Cause:** cloud agents (`AbstractCloudSlave`) are saved under `nodes/` like permanent agents, and Target 6 shares `nodes/*/config.xml`. The other controllers loaded each new agent and launched it too: they tried to create the same pod, failed, and terminated it, killing the original.
  - **Fix (core):** their configuration stays on the controller whose cloud launched them, and one found only in the store is never loaded. `StaticStateApplierTest` has a test for it.
- **Agents started for an item another controller then ran** (9 and 14 pods for 6 builds). Starting a pod took longer than the origin's head start in the shared queue, after which a controller with a free executor took the item. Fixed by the reservations above.
- **The harness admin lost its API token or its password when controllers booted at once.** Each controller's JCasC and token script rewrote the shared user record, and the last writer won. Fixed by the Target 11 merge on the core stack, with recursion into `<properties>` added for this. The harness token script also re-checks the stored record.
- **Chart and harness:**
  - the seed maker had no admin user
  - `alwaysPullImage`, not `imagePullPolicy`, in the pod template
  - Groovy quoting in the test Pipelines
  - builds without parameters folding in the queue
  - HtmlUnit cannot submit a Pipeline's configure form, so the journey uses a freestyle job

## Run history
| Run | Result | What it found |
|---|---|---|
| 1-3 | stopped in setup | the seed maker; the pod template field |
| 4-5 | stopped at 2 | the admin token, then the password, lost in concurrent boots (fixed by the merge) |
| 6 | 29/30 | 9 pods for 6 builds (reservations added) |
| 7 | 29/30 | 14 pods: every controller launched the others' agents (fixed) |
| 8-9 | 30/30 ×2 | final build |

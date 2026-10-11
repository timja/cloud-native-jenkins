## ET9 results (2026-10-11T12:59Z)

| | Check |
|---|---|
| ✅ | 1 both controllers become ready |
| ✅ | 1 both are live members (cluster view through the Service) |
| ℹ️ | 2 10 builds through the Service ran on: cnj-0=6 cnj-1=4  |
| ✅ | 2 builds through the Service ran on both controllers |
| ✅ | 2 a freestyle build on a Kubernetes agent completes |
| ✅ | 2 journey: logged in through the load balancer |
| ✅ | 2 journey: signed in on every request (10/10) |
| ℹ️ | 2 journey: fail requests reached both controllers [cnj-1] (kube-proxy balances connections, and HtmlUnit keeps them alive) |
| ℹ️ | 2 journey: bound objects on the configure page: 79 e.g. cnj-1-8f63cbd6-3b3f-4567-ada8-5b6565db10f7 |
| ✅ | 2 journey: bound objects answer through either controller (12/12 [200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200]) |
| ✅ | 2 journey: configure form saved (200) |
| ✅ | 2 journey: the saved description is there |
| ✅ | 2 journey: progressive renderer's proxy script served (200) |
| ✅ | 2 journey: build time trend rendered progressively (start 200 , news 200 {"data":[{"iconName":"blue","iconColorOrdinal":4,"iconColorDescription":"Success","number":1,"displayName":"#1","dur) |
| ℹ️ | 2 journey: after changing the password, the browser which changed it is signed in: false |
| ℹ️ | 2 journey: login answered 401 |
| ✅ | 2 journey: the old password no longer works |
| ✅ | 2 journey: logged in with the new password |
| ℹ️ | 2 journey: upstreams seen: [cnj-1] |
| ✅ | 3 all 6 builds complete |
| ℹ️ | 3 agent pods seen: 6, at most 4 at once |
| ✅ | 3 at most 4 agent pods at any time (the cap holds for the cluster) |
| ✅ | 3 each build ran on its own pod |
| ✅ | 3 each pod belonged to the controller that ran its build |
| ✅ | 3 no pod was created for an item another controller ran (6 builds, 6 pods) |
| ℹ️ | 4 survive #1 running on cnj-0, agent pod default-1flb6 |
| ✅ | 4 the build completes after its controller's pod was deleted |
| ✅ | 4 it was retried on a new agent pod |
| ✅ | 4 the old agent pod is garbage collected |
| ✅ | 5 no controller garbage collected another's pod |
| ✅ | 6 three controllers ready |
| ✅ | 6 the third is a live member |
| ✅ | 6 it sees every job (flyweight,fs9,kagent,survive) |
| ✅ | 6 and the history (kagent: 6 builds) |
| ✅ | 6 and runs builds |
| ✅ | 7 they come back |
| ✅ | 7 with every job and build (flyweight: 11) |
| ✅ | 7 the user created at runtime still signs in |

Failures: 0

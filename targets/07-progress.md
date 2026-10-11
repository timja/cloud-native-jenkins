# Target 7: progress and branches

Design: [07-owner-routing.md](07-owner-routing.md). **Done means:** `validation/experiments/ET7-owner-routing.sh` passes.

**Decisions (2026-10-11, taken without asking, as the roadmap is to be completed without stopping):**
- forward requests to the owner rather than read state remotely
- agent channels stay on the controller they connect to

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | cloud-native-jenkins | `cnj/t7-m1-owner-routing` | `cnj/t6-m7-static-state` | `OwnerRoutingFilter` (builds running elsewhere, agents connected elsewhere), `jenkins_channels`, shared `Forwarding`, ET7 | ✅ ET7 35/35 ×2 |
| fix | core | `cnj/t6-m6-stateless-sessions` | | a Target 5 race: the same job scheduled on two controllers at once could build twice | ✅ ab593f3b4e; `SharedQueueTest` 9/9, `ClusterCoordinationTest` 6/6; ET5 35/35 |

## Log
- 2026-10-11: Target 7 planned and implemented. ET7 run 1 found the double build from a webhook delivered to both controllers (fixed in core). Runs 2-4 found the routing filter missing on some boots: it was installed only if the store already existed, which depends on initialisation order (fixed). **ET7 passes 35/35** in two consecutive runs ([findings](../findings/ET7-owner-routing.md) on `cnj/t7-m1-owner-routing`).

## Follow-ups
- The Pipeline graph view (pipeline-graph-view plugin) is not in the harness; its pages are build URLs and are forwarded the same way.
- WebSocket requests (some console and agent paths) are not forwarded; the HTTP fallbacks are.

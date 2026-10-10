# Target 6: progress and branches

Design: [06-static-state.md](06-static-state.md).

**Done means:** `validation/experiments/ET6-static-state.sh` passes (the E-T6 criteria).

**Decisions (2026-10-10):**
- fully stateless browser sessions
- pre-plugin keys stay seeded; keys first used at runtime go to the store
- JCasC wins for what it declares

**Policy:** same as Targets 1-5:
- local only, nothing pushed
- stacked branches

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| M1 | core | `cnj/t6-m1-static-routing` | `cnj/t5-m3-cluster-periodic-work` | users and nodes from the store; routed node deletes; early descriptors re-loaded | 🚧 in progress |
| M2 | core | `cnj/t6-m2-static-applier` | T6 M1 | `reloadGlobalConfig`, handlers, `RemoteChangeHandler` | ⏳ |
| M3 | core | `cnj/t6-m3-shared-keys` | T6 M2 | runtime keys in the store, first use wins | ⏳ |
| M4 | credentials | `cnj/t6-m4-credentials-reload` | `origin/master` | reload and a handler | ⏳ |
| M5 | stapler | `cnj/t6-m5-bound-id-prefix` | release `2123.v539a_775c45c3` | controller prefix for bound ids | ⏳ |
| M6 | core | `cnj/t6-m6-stateless-sessions` | T6 M3 | `alwaysRemember`, password field, built on the M5 Stapler | ⏳ |
| M7 | cloud-native-jenkins | `cnj/t6-m7-static-state` | `cnj/t5-m5-shared-queue` | owned keys, `ClusterCrumbIssuer`, bound forwarding, harness, ET6 | ⏳ |

## Log
- 2026-10-10: Target 6 planned and started.

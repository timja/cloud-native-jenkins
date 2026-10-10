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
| M1 | core | `cnj/t6-m1-static-routing` | `cnj/t5-m3-cluster-periodic-work` | users and nodes from the store; routed node deletes; early descriptors re-loaded | ✅ 08c8deb691; `StaticStateRoutingTest` 3/3 + User, Nodes, Descriptor, HudsonPrivateSecurityRealm green |
| M2 | core | `cnj/t6-m2-static-applier` | T6 M1 | `reloadGlobalConfig`, handlers, `RemoteChangeHandler` | ✅ c497194cb5; `StaticStateApplierTest` 5/5 + Jenkins, ComputerSet, NodeListener green |
| M3 | core | `cnj/t6-m3-shared-keys` | T6 M2 | runtime keys in the store, first use wins (`loadOrCreate`) | ✅ 6d896cae85; `SharedConfidentialKeysTest` 4/4 + key, Secret, ApiTokenProperty suites green |
| M4 | credentials | `cnj/t6-m4-credentials-reload` | `origin/master` | reload and a handler (worktree `../credentials-plugin-cnj`) | ✅ 9245ca7 (local-core pom bump, revert before upstreaming) + 9f8514a; new test + SystemCredentialsProviderTest green |
| M5 | stapler | `cnj/t6-m5-bound-id-prefix` | release `2123.v539a_775c45c3` | controller prefix for bound ids (worktree `../stapler-cnj`) | ✅ 1a98d6c07; BoundObjectTableTest green; installed as 999999-SNAPSHOT |
| M6 | core | `cnj/t6-m6-stateless-sessions` | T6 M3 | `alwaysRemember`, password field, built on the M5 Stapler | ✅ dd304d1e54 (Stapler bom bump, revert before upstreaming) + ff03a152bb; `StatelessSessionsTest` 2/2 + HudsonPrivateSecurityRealm, remember-me, user seed, RenderOnDemand green |
| M7 | cloud-native-jenkins | `cnj/t6-m7-static-state` | `cnj/t5-m5-shared-queue` | owned keys, `ClusterCrumbIssuer`, bound forwarding, harness, ET6 | 🚧 in progress |

## Log
- 2026-10-10: Target 6 planned and started.
- 2026-10-10: T6 M1-M3 (core) committed. M3 needed a `loadOrCreate` step in `ConfidentialStore`: key classes generated, then stored and used their own value, so the loser of a create-only race would have kept a different key.
- 2026-10-10: T6 M4-M6 committed. The password field on a user's security page was bound to the session id, so a form rendered by one controller and saved on another replaced the password with its protected text. With a store, it is now bound to the user and their seed.

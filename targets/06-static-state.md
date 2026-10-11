# Target 6: Static state in the store (Track S)

| | |
|---|---|
| **Roadmap tier** | Track S plus X (sessions) |
| **Status** | **Done** (2026-10-11): ET6 passes 44/44, with criterion 10 only partly met (completed by Target 11); see [06-progress](06-progress.md) |
| **Depends on** | T1 (store, `XmlFile` routing, change applier), T2 (membership) |
| **Supersedes** | [targets/90](90-static-state-store.md). Its SPI and routing landed in T1; its bootstrap findings (E6) still hold |
| **Evidence** | [E2](../findings/E2-disposable-controller.md) (UI-entered secrets lost), [E3](../findings/E3-shared-home.md) (lost update on `config.xml`), [E5](../findings/E5-stateless-http.md) (a token created on A gets `401` on B; sessions and crumbs per controller), [E6](../findings/E6-bootstrap-seam.md), [E8](../findings/E8-casc-secret-sync.md) (S0: seeded keys) |

## Goal
State an administrator or user changes at runtime is stored once and seen by every controller without a restart:
- users, passwords, API tokens, user seeds and user-scoped credentials
- system credentials
- global configuration: root `config.xml` and `GlobalConfiguration`/`Descriptor` files
- permanent agents (`nodes/`)
- confidential keys created at runtime

Controllers still start with empty homes. **Browser users and API clients work through a non-sticky load balancer**: consecutive requests from one browser may reach different controllers (decision 2026-10-10: fully stateless, not affinity), and a session survives losing its controller.

**Out of scope:**
- Pre-plugin key material stays seeded (S0): `secret.key`, `secrets/master.key`, `identity.key.enc` and the keys used during plugin start. The Jenkins constructor fixes these before any plugin, and so before the store, exists (research below and E6). On Kubernetes they come from a Secret, which is the normal practice.
- Per-controller state stays local: `queue.xml`, install/upgrade wizard state, `updates/`, `logs/`, node monitor results, and `apiTokenStats.xml` (see API tokens).

## What research found
| Area | Today | Gap |
|---|---|---|
| Root `config.xml` | Loaded by the "Loading global config" task, which requires `EXTENSIONS_AUGMENTED`, so the store is already available (`Jenkins.java:3445`). `load()` unmarshals into the live `Jenkins` (`:3351`) | Only stop-the-world `Jenkins.reload()` re-reads it (`:4420`). A partial reload has to redo parts of "Finalizing set up": labels, `resetFilter`, crumb issuer |
| `Descriptor` / `GlobalConfiguration` `<id>.xml` | `load()` in the constructor. Some are created **before** `EXTENSIONS_AUGMENTED` (e.g. by `PLUGINS_STARTED` initializers) and so read the local file | Nothing re-loads them. Those created early miss the store's copy |
| Users | `users/<prefix>_<hmac>/config.xml`. `AllUsers.scanAll` (after `JOB_CONFIG_ADAPTED`) lists the local directory. Lazy load on a cache miss | A user changed elsewhere stays stale in `AllUsers.byName` and `UserDetailsCache`. Users that exist only in the store are not scanned |
| API tokens | Hashes in the user's `config.xml`; validation uses the in-memory user | Same as users. `apiTokenStats.xml` is rewritten on every authenticated request |
| Passwords and seeds | `HudsonPrivateSecurityRealm.Details` and `UserSeedProperty` in the user's `config.xml` | Same as users |
| System credentials | `credentials.xml`, loaded only in the `SystemCredentialsProvider` constructor | No reload method |
| Folder credentials | Inside the folder's `config.xml` | Already shared (T1) |
| Nodes | `nodes/<name>/config.xml`. `Nodes.load()` lists local directories; `load(name)` and `unload` exist. `removeNode` deletes with `Util.deleteRecursive`, bypassing `XmlFile.delete` | Nodes that exist only in the store are not listed; deletions are not routed |
| Confidential keys | Created lazily on first use, last writer wins, no CAS. `ConfidentialStore` is fixed in the Jenkins constructor (`Jenkins.java:980`) | A key first used at runtime gets different values on different controllers, breaking secrets, user directory names and remember-me |
| JCasC | Applies on every boot and always saves (`BulkChange.commit`) `config.xml` and each configured descriptor. Its `nodes:` setter deletes nodes that aren't in the YAML | N controllers mean N identical writes. Nodes declared in JCasC would be deleted cluster-wide by any controller whose YAML lacks them |
| Sessions | In-memory Jetty session. The cookie name is random per controller. Crumbs hash the session id. Remember-me (`username:expiry:HMAC`) validates anywhere with shared users and keys | No shared session store, and bound objects can't be shared |

## Design

### 1. Ownership and routing (store plugin, core)
- The store plugin's owned-key pattern grows to cover:
  - root `config.xml`
  - root `*.xml`, except the per-controller files: `queue.xml`, `nodeMonitors.xml`, `hudson.model.UpdateCenter.xml`, `jenkins.install.*`, `*.FlowExecutionList.xml`
  - `users/*/config.xml`
  - `nodes/*/config.xml`
- `XmlFile` routing from T1 does the rest.
- **Identical writes are already no-ops in the store.** That defuses JCasC's per-boot writes: no new version, no events.
- **Core changes, as T1 made for items:**
  - `AllUsers.scanAll` and `Nodes.load()` also take the store's children.
  - `Nodes.removeNode` / `setNodes` delete through `XmlFile`, so deletions reach the store.
  - After `EXTENSIONS_AUGMENTED`, descriptors already loaded from local files are loaded again if the store has their key.

### 2. Applying changes made elsewhere (core, plus a credentials hook)
A `StaticStateApplier` extends the T1 change applier with handlers keyed by path. Each runs under `applyingRemoteChange`, so nothing is written back.

| Key | Action |
|---|---|
| `config.xml` | `Jenkins.reloadGlobalConfig()` (new): `load()`, then re-derive labels (`trimLabels`), `resetFilter`, re-install the crumb issuer and authorization strategy, then `updateComputers`. Under the `Jenkins` monitor, so it cannot interleave with a local `save()` |
| `<id>.xml` | `load()` on the `Descriptor` / `GlobalConfiguration` with that id, if one exists |
| `users/*/config.xml` | `User.load()` on the cached user, or add a new one to `AllUsers`; `UserDetailsCache.invalidate`. Deleted: remove from `AllUsers` |
| `nodes/*/config.xml` | `Nodes.load(name)` (replacing the Computer if the node changed); deleted: `unload` + `updateComputers`. `NodeListener` events fire |
| other keys | `RemoteChangeHandler` extension point (Beta), e.g. credentials below |

**Credentials plugin:** a `SystemCredentialsProvider.reload()` (unmarshal and migrate, as SYSTEM) and a `RemoteChangeHandler` for `credentials.xml` and `credentials-configuration.xml`.

### 3. Confidential keys created at runtime (core)
- `DefaultConfidentialStore` gains a second tier, so `ConfidentialStore.get()` and every key class stay unchanged:
  - **load:** local `secrets/<id>` (seeded), then the persistence store's `secrets/<id>`, once the store exists.
  - **store** of a new key: a **create-only** write to the persistence store. If another controller created it first, read theirs and use it. Then write the local file.
- So the first use of a key anywhere decides its value for the whole cluster. Values are encrypted with `master.key`, as today, which is identical everywhere (seeded).
- Keys used before the store exists must be seeded, as today. A key first used then on a controller missing the seed is logged loudly, because it will diverge.

### 4. API tokens and users
- Token creation, revocation, passwords and seeds live in the user's `config.xml`, so they are shared by (1) and applied by (2). A token created on A works on B within the applier latency, and through a non-sticky load balancer.
- `apiTokenStats.xml` (usage counters) **stays local**. It is written on every authenticated request, and sharing it would turn each request into a store write with lost-update races. Each controller shows the usage it saw. A follow-up could aggregate counters.

### 5. Stateless browser sessions (decision: fully stateless)
**Constraint.** Configure pages ("Add" buttons, dropdown sections via `renderOnDemand`), build history, trend tables and the People page use **Stapler bound objects**: live Java objects held in the HTTP session and called back at `/$stapler/bound/<uuid>/…`. They cannot be serialized into a shared session store. Instead, a callback is **forwarded to the controller that created the object**.

| Piece | Change |
|---|---|
| Authentication on every request | An always-on signed login cookie: remember-me (`username:expiry:HMAC(seed, keys)`), issued at every login and valid for the session lifetime. Its validation needs only the shared user, seed and keys, so any controller authenticates the request and creates a local session if needed. Core gets a Beta option to always issue it, `TokenBasedRememberMeServices2.alwaysRemember`, because no switch exists today |
| CSRF crumbs | `ClusterCrumbIssuer` (store plugin, a `CrumbIssuer` extension selected by JCasC). The crumb is a hash of the user, a binding token and the shared `crumbSalt`. The token is the login cookie when signed in; before login it is a signed nonce cookie set when the crumb is issued. Crumbs, including the login form's, validate on any controller |
| Bound objects | **Stapler:** bound ids carry a controller prefix (`BoundObjectTable` id generation, configurable). **Store plugin:** a servlet filter forwards `/$stapler/bound/<prefix>-…` requests for another controller's objects to that controller's internal URL, published in the membership table. Method, headers, body and the browser's cookies are forwarded, so the owner finds its own session (each controller's session cookie name is distinct: `JSESSIONID.<random>`) |
| Password field | `HudsonPrivateSecurityRealm` round-trips the password as `sessionId:hash` (`:748`, `:887`). Saved on another controller, that is read as a literal new password. Core change: protect it with a shared confidential key and the user name instead of the session id |
| Cosmetic session state | The "Saved" notification after Apply (`FormApply`) and pane collapse state for anonymous users may be lost when the next request lands elsewhere; documented |
| Out of scope | Live interaction with builds *running* on another controller (console follow over WebSocket, `input` and abort of running builds): R3 routing. The same forwarding filter is the building block. Stop already works through T5 stop requests |

### 6. JCasC on N controllers
- JCasC stays authoritative for what it declares, re-applied on every boot. Thanks to no-op identical writes, an unchanged YAML causes no store churn.
- Settings JCasC does not declare are editable in the UI and shared.
- **Hazard: the `jenkins.nodes` setter deletes nodes absent from the YAML.** In HA, declare nodes in JCasC either everywhere or nowhere. ET6 checks that a node added in the UI survives another controller's boot with a YAML that declares no nodes.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | core | `cnj/t6-m1-static-routing` | `cnj/t5-m3-cluster-periodic-work` | `AllUsers` and `Nodes` include store children; node deletes routed; early-loaded descriptors re-loaded from the store; tests |
| M2 | core | `cnj/t6-m2-static-applier` | M1 | `Jenkins.reloadGlobalConfig()`; handlers for root config, descriptors, users, nodes; `RemoteChangeHandler` SPI; tests |
| M3 | core | `cnj/t6-m3-shared-keys` | M2 | second-tier `DefaultConfidentialStore` (create-only, first use wins); tests |
| M4 | credentials | `cnj/t6-m4-credentials-reload` | `origin/master` (+ local-core pom bump) | `reload()` plus a `RemoteChangeHandler` |
| M5 | stapler | `cnj/t6-m5-bound-id-prefix` | the release core uses (`2123.v539a_775c45c3`) | controller prefix for bound object ids |
| M6 | core | `cnj/t6-m6-stateless-sessions` | M3 | `alwaysRemember`; password field protected without the session id; core built against the M5 Stapler; tests |
| M7 | cloud-native-jenkins | `cnj/t6-m7-static-state` | `cnj/t5-m5-shared-queue` | owned-key pattern for static state; `ClusterCrumbIssuer`; bound-request forwarding with internal URLs in membership; harness (non-sticky browser LB, credentials and Stapler branches); `ET6-static-state.sh` |

## E-T6 acceptance experiment (definition of done)
ET setup: two tmpfs controllers, S0 seed (pre-plugin keys only), Postgres, Azurite, and a **non-sticky round-robin load balancer** used for every browser and API scenario.
1. **API token:** a token created for user `u1` on A (REST) authenticates on B, and through the non-sticky LB (10/10 `200`). Revoked on A, it gets `401` on B.
2. **Users and passwords:** a user created on A logs in on B. After a password change on A, the old password fails on B and the new one works.
3. **Global config:** a system message and the number of executors set on A appear on B within 5 s, without a restart. B's next save of something else does not revert them.
4. **System credential:** a secret-text credential created on A is listed on B, and a build on B (`withCredentials`) reads its value.
5. **Node:** a permanent agent added on A appears on B and connects through B. Deleted on A, it is gone on B.
6. **Runtime key:** a confidential key first used at runtime on A has the same value on B. A `Secret` encrypted with it on A decrypts on B.
7. **Disposable cluster:** destroy both controllers and start fresh ones. All of the above survive.
8. **Stateless browser journey through the round-robin LB** (a scripted browser with a cookie jar, so consecutive requests alternate controllers):
   - log in: the form is fetched from one controller and posted to the other;
   - open a job's configure page and use an "Add" (`renderOnDemand`) section;
   - save the form;
   - load the build history (progressive rendering);
   - change the user's own password and log in with the new one.

   All succeed, and the forwarded bound-object requests are counted. Then kill one controller and continue the journey without logging in again.
9. **JCasC coexistence:** restarting B (JCasC re-applies) bumps no store versions for unchanged config, and does not remove the UI-added node.
10. **Concurrent edits:** A and B save different global settings at nearly the same time. Both survive, and the conflict counter shows any overwrite.

## Risks
- **Partial reload of `Jenkins`.**
  - Views are recreated (old references go stale) and clouds are replaced (in-flight provisioning keeps old objects).
  - The security realm is assigned without `setSecurityRealm`'s side effects.
  - The handler is explicit about each step, and ET6 exercises the common fields.
- **Descriptors with side-effecting `load()`** (`JenkinsLocationConfiguration` migrates Mailer settings) run again on remote changes. These are idempotent in the cases found.
- **Plugins reading their files directly** (`getConfigFile().getFile()`) see T1's read-only snapshot. Writes outside `XmlFile` are not shared (E7 sizes this).
- **Early descriptors:** a descriptor used before `EXTENSIONS_AUGMENTED` runs briefly with its local defaults, until the re-load.
- **Seeded keys remain an operational step,** documented as a Kubernetes Secret.

# E8: Short-term workaround for static state: identical JCasC plus synced keys

- **Script:** `validation/experiments/E8-casc-secret-sync.sh`
- **Raw evidence:** `raw/E8/results.md`

**Setup:** two controllers with **separate** `JENKINS_HOME`s and no shared filesystem, both configured from the same JCasC (`validation/jenkins/casc/base.yaml`).

## Variant M: share only `master.key`
Uses `-Djenkins.security.DefaultConfidentialStore.file=/sync/master.key` and `.readOnly=true` (`DefaultConfidentialStore.java:29-31`).

| Check | Result |
|---|---|
| `Secret` encrypted on A decrypts on B | **no** (`null`) |
| agent inbound secret identical | **no** |
| instance identity identical | **no** |
| legacy `secret.key` identical | **no** |
| API token created on A works on B | **401** |
| `secrets/*` files that differ | `hudson.util.Secret`, `JnlpSlaveAgentProtocol.secret`, `Jenkins.crumbSalt`, `InstanceIdentity.KEY`, `User.DIRNAMES`, `csp.ReportingContext.key` |

`master.key` only *wraps* the real keys. Each `ConfidentialKey` is generated randomly the first time it is used, per controller, and written to `secrets/<id>`. So sharing `master.key` alone shares nothing.

## Variant S: seed a full key set into every home before first boot
1. Boot one controller and exercise the known keys (a build, the agent secret, instance identity, a `Secret`).
2. Copy out:
   - `secret.key`, `secret.key.not-so-secret`
   - `identity.key.enc`
   - `secrets/*` (9 files: `ConsoleNote.MAC`, `Job.serverCookie`, `User.DIRNAMES`, `hudson.util.Secret`, `crumbSalt`, `csp.ReportingContext.key`, `JnlpSlaveAgentProtocol.secret`, `master.key`, `InstanceIdentity.KEY`)
3. Copy them into both homes before first boot. This is what a k8s init container would do from a Secret.

| Check | Result |
|---|---|
| `Secret` encrypted on A decrypts on B | **yes** (`e8-value`) |
| agent inbound secret identical | **yes** |
| instance identity identical | **yes** |
| legacy `secret.key` identical | **yes** |
| `secrets/*` files that differ | none |
| new keys generated on B by a build | none |
| API token created on A works on B | **401**: tokens are user state, not key material |

## Verdict
The workaround is viable for static state. Caveats to document:
1. **Sync the whole key set, not `master.key`.** Use an init container that copies a k8s Secret into the home; `secrets/` can't be read-only, because Jenkins writes new keys into it.
2. **A plugin that introduces a new `ConfidentialKey` creates it per controller,** so the controllers silently diverge. Re-seed after plugin changes, and alert when `secrets/` differs from the seed (the `sha256sum` diff in this script).
3. **The configuration UI must be treated as read-only.** A change saved in the UI lands on one controller only (E3).
4. **JCasC must be byte-identical across controllers.** The harness's own `systemMessage: ... (${CNJ_NAME})` is a counter-example.
5. **Not covered by this workaround:**
   - users' API tokens and other user properties (E5, E8). Prefer SSO plus a token mechanism that doesn't live in `users/`.
   - UI-created credentials, views and nodes
   - **anything created by builds**, see [E9](E9-runtime-job-state.md)

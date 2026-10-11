# ET6: Target 6 acceptance (static state in the store, stateless browser sessions)

**Result: PASS, with one criterion only partly met (10, concurrent edits; see below).** 44/44 checks, in two consecutive runs on the final build. Raw evidence from the last run: `raw/ET6/`
- `results.md`
- `journey.txt`, `journey-failover.txt`: the scripted browser's output
- `static-keys.txt`: every non-job key in the store, with its version
- controller logs, `BUILD`

The regressions pass on the same build: ET3 48/48, ET4 39/39 (three runs), ET5 35/35. In a fourth ET4 run, controller B did not finish booting within the setup's 300 s (no logs were kept); it was not reproduced.

- **Script:** `validation/experiments/ET6-static-state.sh`. The browser is `validation/journey` (HtmlUnit, JavaScript on, a cookie jar).
- **Setup:**
  - two tmpfs controllers, S0 (pre-plugin keys only), Postgres, Azurite
  - `rr`: an nginx round-robin load balancer with no stickiness, used by the browser and the token checks
  - the controllers' JCasC is `base.yaml` alone, which declares no nodes (see 9)
  - `crumbIssuer: cluster`, and `-Dhudson.security.TokenBasedRememberMeServices2.alwaysRemember=true`
  - each controller's bound-object prefix and internal URL come from `CNJ_NAME` and `CNJ_INTERNAL_URL`
- **Build:**
  - core `cnj/t6-m6-stateless-sessions`, against Stapler `cnj/t6-m5-bound-id-prefix`
  - credentials `cnj/t6-m4-credentials-reload`
  - lockable-resources `cnj/t5-m4-cluster-locks`
  - workflow-cps `cnj/t4-m1-program-saved-listener`
  - cloudbees-folder `cnj/t1-m2-folder-children`
  - plugin on `cnj/t6-m7-static-state` (built from `8cb0859`, the same code as `f171eab`)

| Criterion | Result |
|---|---|
| 1 API token | ✅ `u1` created a token on A over REST, with its password and a cluster crumb. It works on B, and 10/10 through `rr`. Revoked on A over REST, it gets `401` on B |
| 2 Users and passwords | ✅ a user created on A logs in on B. After a password change on A, the old password gets `401` on B and the new one works |
| 3 Global config | ✅ set at runtime on A, visible on B within 5 s without a restart: the quiet period (`config.xml`) and the admin address (`JenkinsLocationConfiguration`, a descriptor). B's next save of something else (the SCM checkout retry count) did not revert them. *Deviation from the plan:* the plan named the system message and the executor count, but the baseline JCasC declares both. They would propagate at runtime too, but JCasC rewrites them on every boot (criterion 9), so ET6 uses settings JCasC does not declare |
| 4 System credential | ✅ a secret-text credential created on A is listed on B. A Pipeline on B read it with `withCredentials` |
| 5 Node | ✅ a permanent agent added on A appears on B, with the same agent secret on both. A real inbound agent (`jenkins/inbound-agent`, WebSocket) connected through B. Deleted on A, the node is gone on B and B dropped the connection |
| 6 Runtime key | ✅ an `HMACConfidentialKey` first used at runtime on A has the same value on B. Data encrypted on A with a `CryptoConfidentialKey` first used at runtime decrypts on B |
| 7 Disposable cluster | ✅ both controllers destroyed and recreated. The following all survive: users and passwords, global settings, the descriptor setting, the credential, the UI-added node (with the deleted one still deleted), and the runtime key |
| 8 Stateless browser journey through `rr` | ✅ every step below, and the requests reached both controllers |
| 9 JCasC coexistence | ✅ restarting B (JCasC re-applies) bumped no store versions of unchanged config. The undeclared setting, a runtime user and the UI-added node survive B's boot, and the declared system message is still JCasC's |
| 10 Concurrent edits | ⚠️ partial. A and B saved different fields of `config.xml` at the same moment. Both survived (9/4) in 8 of the 13 runs that reached it; in the other 5, A's change was lost (7/4). Every time, the store's version check detected the overwrite (the conflict counter rose) and A and B converged on the same result. See below |

### Criterion 8: the browser journey
- **Login:** the form came from one controller and was posted to the other. The browser was then signed in on 10/10 requests, whichever controller answered.
- **Bound objects:** the job's configure page holds 79 bound objects (`renderOnDemand` sections and others) with prefixed ids. Calls to three of them, made 4 times each the way Stapler's JavaScript proxy makes them, all answered `200` (12/12) through `rr`. The controllers counted 8 forwarded requests; the other 4 landed on the owner.
- **The form saved,** and the description was there afterwards.
- **Progressive rendering (build-time trend):**
  - the renderer's proxy script was served (the `script` endpoint, forwarded by the fix below)
  - `start()`, then `news()` until done, returned the builds
  - HtmlUnit has no `fetch()`, so the journey drives these calls itself instead of running the page's JavaScript.
- **Password change:** the user changed their own password on the security page. The new password logs in and the old one gets `401`. The password field round-trips across controllers (core M6).
- **Losing a controller:** A was killed with SIGKILL. The browser that signed in with the new password stayed signed in, 5/5 requests, now all on B.
- **Logged out by the change, as on one controller:** the browser that changed the password is signed out. The password change renews the user's seed, which ends every session, including that one.

## Bugs found by ET6 and the regressions, now fixed
- **The bound-object script endpoint was not forwarded** (journey, build-time trend).
  - **Cause:** its URL is `/$stapler/bound/script/$stapler/bound/<id>`, and the forwarder took its id from the first `/$stapler/bound/`.
  - **Fix (M7):** take the last one, with a test.
- **lockable-resources state shared as configuration** (ET5 regression, scenario 9: one lock held forever).
  - **Cause:** `LockableResourcesManager.xml` is a root XML file, so the static-state pattern shared it. Besides the declared resources, it holds each controller's waiting builds. B loaded A's waiting `lk1#1`, and the reload overwrote lock state that Target 5 manages through `ClusterLocks`.
  - **Fix (M7):** the file stays per-controller, with a test.
- **A Pipeline resumed elsewhere could skip log lines** (Target 4 race; ET4 scenarios 1 and 4, found by the regression runs).
  - **Cause:** a build's log and its save points were shipped independently. A save point could commit a `sh` step's output offset past lines not yet shipped, and they were lost when the owner died.
  - **Fix:** the log is shipped before each save point is committed; a save point whose log cannot be shipped is retried.
- **A fenced controller could still upload a save point** (Target 4 race; ET4 scenario 7, made more likely by the fix above).
  - **Cause:** the save point's blobs were uploaded before the commit that would have told it the build was lost. They stayed after completion, and could overwrite the new owner's flow nodes.
  - **Fix:** ownership is checked before uploading, and a save point whose commit conflicts is deleted.
- **Harness:**
  - Admins cannot create another user's token over REST. The token is now created by the user.
  - The journey called bound objects without the content type and crumbs that Stapler's proxy sends (`404`/`403`).
  - HtmlUnit's classpath lacked commons-logging, commons-io, commons-text and commons-codec.
  - The JCasC node list deleted the UI-added node. The controllers' YAML now declares no nodes, as the plan says.
  - The older experiments' seed controllers did not load the store plugin that `crumbIssuer: cluster` needs.

## Open: concurrent edits of one file
- **What happens:** `Jenkins.save()` writes the whole of `config.xml`. When A and B save at once, the store's compare-and-set detects the conflict.
- **Why a change is lost:** the store runs in Target 1's lenient mode, which logs the conflict and lets the last writer win. The winner's file then reaches the other controller, so the cluster converges, but one change is lost.
- **What surviving would take:** retry on conflict. Reload the file, re-apply the change, and write again. That needs the change itself, not the serialised object. It suits setters such as Jenkins' own fields and `Descriptor.save()` with form data, but not arbitrary `save()` callers.
- **Scope:** this is the same behaviour as T1 for descriptors, now also for users and nodes. A single admin is unaffected; two admins editing global configuration within the same second are the case.

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | stopped at 1 | admins cannot create another user's token over REST (harness) |
| 2 | 19/21, stopped at 8 | the JCasC node list deleted the UI-added node; the journey's classpath |
| 3 | 23/23, stopped at 8 | the journey's bound-object calls (harness); the script endpoint not forwarded (fixed) |
| 4 | 40/41 | the browser that changed its own password was signed out (as on one controller; the journey now continues with the new login). |
| 5 | 42/42 | clean |
| 6–8 | all passed ×3 | ET6 brought in line with the plan (token over REST by the user, a real agent through B, JCasC without nodes); regressions: ET5 9 failed (lockable-resources, fixed), ET4 and ET3 seeds (harness, fixed) |
| 9–10 | all passed ×2 | ET5 35/35, ET4 39/39, ET3 48/48 |
| 11–14 | 44/44 ×4 | regressions: ET4 lost log lines after failover in 1 of 3 runs (fixed), then left a save point of a fenced owner (fixed) |
| 15–16 | 44/44 ×2 | final build; ET4 39/39 ×3, ET5 35/35, ET3 48/48 |

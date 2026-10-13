# Target 15: everything still outstanding

| | |
|---|---|
| **Roadmap tier** | All tiers (the follow-ups of Targets 1–14) |
| **Status** | Done 2026-10-13 (planned 2026-10-11): every experiment passed in one sweep, see [15-progress](15-progress.md) |
| **Depends on** | Targets 1–14 |
| **Out of scope** | Upstreaming (not started, at the user's request). The security review is handled through a separate process |

**Done means:** every item below is implemented, its experiment or tests pass, and ET1–ET22 pass in one sweep (ET9, ET10 and ET13 on kind).

## Decisions (user, 2026-10-11)
- **Failover:** session-lock liveness plus a 2 s heartbeat and a 15 s timeout.
- **JCasC nodes:** a flag. There is none upstream (only the per-node `CascKeepNode` property), so it goes on a local branch.
- **Storage:**
  - The generic sync of plugin files in item directories is opt-in. Write-back of `getFile()` snapshots, and the promoted-builds and matrix sub-items, are always on.
  - There is no deferred blob deletion. Point-in-time recovery is documented as needing blob versioning (real Azure). ET18 restores Postgres and checks what is missing.
- **Tooling:** Playwright may be installed from npm locally, for real-browser tests. WebSockets are proxied to the owner.

## Items
| # | Item | Plan |
|---|---|---|
| 1 | Install progress page alternates between controllers | Route `pluginManager/updates/…` and `updateCenter/{body,installStatus,incompleteInstallStatus,api}` to the controller running the install: a cookie, then a correlation id in the store, then the member publishing active jobs. In cluster mode, hide or refuse the per-controller "restart when done" checkbox |
| 2 | Plugins pages don't say changes are cluster-wide | A core include hook on the plugin manager pages. A plugin decorator shows "applies to all N controllers" and each controller's status (what it is applying, failures), refreshed |
| 3 | CLI `install-plugin` from a file or URL stays local | A core listener hook where the CLI moves the archive into place: it is recorded in the cluster set, and `fixed` mode and the store-plugin guard are enforced |
| 4 | WebSockets not forwarded | A WebSocket proxy in the routing filter (backend leg `java.net.http.WebSocket`, frames relayed both ways). CLI `build -s` and `console -f` follow builds running elsewhere. Agents keep connecting per controller |
| 5 | Failover 16–23 s | Watch on the next member's deadline, not a 5 s tick. NOTIFY on leave. A session advisory lock per controller on a direct connection: a dead process frees it at once (crash failover ~6–8 s), while pauses and partitions wait the timeout. Heartbeat 2 s, timeout 15 s. A switch turns the locks off for poolers |
| 6 | Plugins writing files in item directories | (a) Write back changed `getFile()` snapshots, CAS-merged. (b) Share the promoted-builds `promotions/*/config.xml` and matrix `configurations/**/config.xml` sub-items. (c) An opt-in generic item-files sync (manifest plus per-file blobs, 1 MiB cap) |
| 7 | Per-controller files are a hand-kept list | A core `@PerController` annotation and a `LocalFiles` extension point. Core's own entries move into core. The store's list remains for third-party plugins |
| 8 | A field set during a reload can be lost | Reload into a scratch instance, then apply only the changed parts, under the lock, skipping fields with local pending changes. Core setters save under the object's lock. A deterministic latch test |
| 9 | Data keyed by job name lost on rename | A per-item UUID in core (`config.xml`; v5 from the full name for legacy items; regenerated on copy). junit-sql-storage keyed by it |
| 10 | JCasC `jenkins.nodes` deletes undeclared nodes cluster-wide | JCasC branch: `-Dio.jenkins.plugins.casc.core.JenkinsConfigurator.keepUndeclaredNodes=true` keeps nodes the YAML does not declare. The chart sets it |
| 11 | Files added to old builds | A published-files manifest with each record. The republisher publishes when files changed. A scan of loaded builds replaces the 1 h in-memory window. A local copy with newer files is published, not wiped |
| 12 | Freestyle console lost when its controller dies | Ship freestyle logs to the build's blob segments while they run. A build closed as aborted keeps its output |
| 13 | `node` label only on another controller | Keep whole-build handover and document why (remote executors would mean re-implementing Remoting across JVMs). Fix the bounce of builds with nested or parallel `node` blocks on different controllers (hand over only with no other executors held, hop limit) |
| 14 | `build(wait: true)` pinned | Remote step completion through the store: a delivery mailbox consumed by the owner or an adopter. Also `waitForBuild`. Stop reaches builds and queue items elsewhere. Pins to a dead controller lapse |
| 15 | lockable-resources reserve, steal, … local | A holder record in the store (build or reservation): reserve, unreserve, steal, unlock, reassign and reset are cluster-wide. Notes and UI-created resources are shared |
| 16 | Executors on the cluster page | Busy/total per controller and per agent, and the shared queue length, published with the heartbeat |
| 17 | Scale | Fixes first: indexed `parent`/`name` columns for `children()` (the first background migration), bulk deletes, NOTIFY per batch. Then ET17: 3 controllers, 200k synthetic builds, a branch-indexing storm of 50 projects with a kill midway |
| 18 | Managed services, PITR | Chart: `existingSecret`, TLS, pool size. ET18: external postgres:16 with WAL archiving, a restore to a named point, fresh controllers; documents the blob-versioning requirement |
| 19 | Background schema migrations | `jenkins_schema_migrations`: expand at startup, backfill in batches with a cursor (leased, resumable), contract once no older peer remains. Indexes built concurrently |
| 20 | Untested paths | Pipelines declaring `podTemplate` under cluster caps and failover (and pruning `jenkins_counters`). A conflict-count admin monitor and a strict policy for UI saves. Playwright tests: console across the load balancer, save conflict, cluster view, the plugins banner |
| 21 | Creating a child in a folder reloads the whole folder | A core hook adds just the remotely created child. cloudbees-folder implements it (fixing `addLoadedChild`'s naming) |
| 22 | Task logs: the SCM polling log is on whichever controller polled | Requests for a job's polling log go to the controller that last polled it (from the trigger claims) |
| 23 | Workspaces across an adoption (static and inbound agents) | ET4: a later stage reads what an earlier stage left in the workspace, with the controller killed between them |
| 24 | Workspaces on Kubernetes pods | ET9: `stash`/`unstash` carry files across the pod replaced after an adoption; documented that pod workspaces are otherwise lost |
| 25 | Code coverage (coverage plugin) | ET8: a coverage report published on one controller, shown in full (summary, tree, source) on another |
| 26 | A controller started without the cluster's seeded keys | Detect it: a check value encrypted with the cluster's key, refusing to start (or a monitor) when it cannot be decrypted |
| 27 | Plugin files in user directories (avatar plugin: the image is written to `users/<id>/avatar.<ext>` with file APIs, so other controllers and fresh homes have the property but no image) | Extend the opt-in sharing of plugin files (`ItemFiles`) to user directories, so no plugin change is needed. Acceptance: an avatar uploaded on A is served by B and by a fresh controller (added 2026-10-12, at the user's question) |
| — | Housekeeping | `compose.yaml` pins azurite 3.34.0; the plugin tests run with Testcontainers' ryuk container disabled (it is not an approved image) |

> Approved 2026-10-17: re-cut as `cnj2/*` and close #139–#162 once the new PRs are up; launch lease for outbound agents (§5); no migration of stored data (P7 dropped: wipe the store). Decisions 1–8 of round 4 are in [reviews/agents-2026-10-17.md](../reviews/agents-2026-10-17.md).

# DESIGN-17: Round-4 architecture — cluster engines out of core, build data keyed by item UUID

Status: design for decision. Inputs: DECISIONS.md (1, 2, 6, 7; 3 and 4 constrain the design), BRIEF.md, core-arch-opus, core-compat-sonnet, core-b-fable M1, plugins-w3-fable (design question), jep-fable (M5, Size). Code: core `0a47f66bc6` (= `fork/cnj/t15-remaining`; base `fork/cnj/base` = `cd2a077`), cluster-plugin `84ac070`, plugin clones `*-cnj` and `lockable-resources-plugin` `cnj/t15-cluster-ui`.

Notation: **[Beta]** = `@Restricted(Beta.class)`; **[NEU]** = `@Restricted(NoExternalUse.class)`. "Upstream" = behaviour at `cnj/base`.

---

## 0. Principles

1. **Core gets mechanisms, the plugin gets policy.** A core change must be justifiable on its own: either it is useful without a store, or it is the narrowest seam the plugin cannot get from an existing extension point.
2. **No store means upstream behaviour, provably.** Each hook's default is the old code path, or it is gated at a single seam. No `@Extension` in core is a wrapper on an upstream extension point.
3. **Build data has no path.** Anything that outlives a request and belongs to an item is keyed by the item's UUID. Paths are kept only for files that core routes from `JENKINS_HOME` (config).
4. **Reuse upstream extension points before adding new ones.** Every new extension point lists the call sites that need it.

---

## 1. What stays in core and what moves

### 1.1 Stays in core (mechanisms)

| Area | Classes (at 0a47f66bc6) | Change from today |
|---|---|---|
| Store SPI | `PersistenceStore`, `PersistenceStoreProvider`, `ConflictException`, `ChangeEvent`/`Entry`/`Moved` | Becomes key-value plus events only (§2). Drops `owns`, `storesBuildRecords`, `liveOrigins`, `leases`, `writeIfHeld`, `queue`. Keeps `storesConfidentialKeys` as a capability of the store. `move` gains preconditions (§3.3). |
| Routing | `PersistenceStores` (routing and version-table part), `LocalFiles`, **new** `SharedFiles`, **new** `StoreRouting` (single policy class) | `@PerController` and `declare()` are removed, and so is `LocalFiles.declares` (dead). `keepLocal` stays, but as a `StoreRouting` input for cloud agents only. |
| Files | `XmlFile`, `TextFile` routing; `XmlFile.getFile()` materialises a read-only copy | The write-back moves out (§1.2). `exists()` now throws `UncheckedIOException` on a store error (core-arch M3). A write to a key never read is create-only (`ABSENT`). `TextFile` loses the `StoredBuildNumberAssigner.writeNoLower` call. |
| Conflicts | `Route.write` conflict path | Default (no reconciler): refuse with `ConflictException`, then reload the stored version. Never last-writer-wins (decision 4). The `warn` and `strict-ui` modes and the `CONFLICTS` property are deleted. The merge is pluggable (§1.3 D). |
| Items | store-only enumeration (`storedItemNames`, `Jenkins`/`ItemGroupMixIn`), lifecycle (`AbstractItem.renameTo`, `Items.move`, create, delete through the store), `RemoteItemGroup`, `ItemChangeApplier` (item config only), `ItemUuids` | The build, permalink, delivery, stop and placeholder branches of `ItemChangeApplier.build()/moved()` go. Keys outside routed space are sent to `RemoteChangeHandler`. `SubItems` becomes generic: a routed key below an item directory that is not another item reloads the nearest owning item, with no plugin names. The move claims (`.moving`, `lapsed`, `checkNoOverlappingMove`, about 250 lines) are deleted (§3.3). |
| Static state | `StaticStateApplier`, `EarlyDescriptors`, `Jenkins.reloadGlobalConfig`, `Nodes.changedElsewhere` (and fix: validate `name`), `User.changedElsewhere`, setters under the lock | `clearRemovedFields` (reflection) is replaced by upstream `XmlFile.unmarshalNullingOut`, the same semantics as `updateByXml`. |
| Keys and sessions | `DefaultConfidentialStore.reconcile` and the `*ConfidentialKey` changes, `Protector`, `HudsonPrivateSecurityRealm` replay guard, `TokenBasedRememberMeServices2.alwaysRemember`, plus the decision-5 logout revocation hook | Unchanged in scope; these belong to JEP-254. |
| Coordination hooks | `TriggerCoordinator`, `PeriodicWorkCoordinator`, `@ClusterSingleton`, `QueueStartGate` (with `provisionsFor`), `QueueListener.onFolded`, `Queue.LOADED`, `Queue.isLockHeldByCurrentThread`, `PluginArchiveListener` (plus `doUploadPlugin`) | `FingerprintCleanupThread.Shared` moves to the plugin (it only matters with `ClusterFingerprintStorage`). |
| Queue hand-off | `Queue.scheduleShared`, `updateShared`, **new** `Queue.handOff(id)` and `LeftItem.isTakenElsewhere()` (decision 3) | `cancelLocal`, `RemoteItem`, `RemoteExecutable` and the remote branch of `getItem(long)` are deleted. |
| Run map hooks | `AbstractLazyLoadRunMap.knows/unload/forget/proposeSharedNumber` (NEU→Beta), `Run.isPendingDelete`, the `ItemListener` ordinal fix | These are the hooks the plugin calls. |
| New extension points | `ExternalBuilds` + `ExternalBuild`, `BuildRecordSource`, `StoredConfigReconciler` | §1.3 |
| Item identity | `Item.getItemUuid`, `ItemUuids` (+`pin`) | Persistence changes (§3.5). |

### 1.2 Moves to cluster-plugin (new module `cluster`, package `io.jenkins.plugins.cluster.builds|queue|files|config`)

| Engine (lines) | Registered in the plugin on | Notes |
|---|---|---|
| `SharedQueue` (1607), `ClusterQueue` (154), `SharedBlocking` (86) | `QueueIdStrategy` (existing), `QueueStartGate`, `QueueListener` (incl. `onFolded`), `QueueTaskDispatcher` (existing), `RunListener`, `CloudProvisioningListener`, `ItemListener`; uses `Queue.scheduleShared/updateShared/handOff` | The hard-coded `BUILD_TRIGGER_ACTION` / `<waitForStart>` sniffing goes. Pinning becomes a question to `BuildTriggerRelay` (§4.2). `ClusterQueue.claim(long)` overload is removed. Records are keyed by item UUID. |
| `RunningBuilds` (1983), `BuildAdopter` (73), `BuildLeases` (65), `LeaseLostException` (46), `BuildElsewhere`/`running-elsewhere*` jelly | `ExternalBuilds`, `TransientActionFactory<Job>` (existing; `RunningElsewhereAction` carries the exported `buildsElsewhere`), `RunListener`, `AsyncPeriodicWork`, `RemoteChangeHandler` | `owned`/`lost` are keyed by `Run`, not path (core-b M1 fix, and simpler with UUID keys). `Delivery` becomes plugin-internal. |
| `BuildRecords` (1544), `StoredPermalinks` (301), `StoredBuildNumberAssigner` (178) | `BuildRecordSource`, `PeepholePermalink.Cache` (existing), `Job.BuildNumberAssigner` (existing), `RunListener`, `SaveableListener`, `ItemListener`, `RemoteChangeHandler`, `AsyncPeriodicWork` | The Pipeline paths in `DEFAULT_EXCLUDES` become the plugin's `Exclusion` for workflow. Snapshot staging moves out of tmpdir (core-b m5). `PeepholePermalink.Cache.isShared` and `PeepholePermalink.changedElsewhere` move too, if the plugin can do it with public API; otherwise keep `changedElsewhere` as [Beta]. |
| `ItemFiles` (596), `MaterializedFiles` write-back part (≈180) | `ItemListener`, `AsyncPeriodicWork`, `SaveableListener.onDeleted` for users | Core keeps a list of materialised copies as [Beta] `PersistenceStores.materializedCopies()` (key, file, version). A one-line generic core hook is added: `User.delete` fires `SaveableListener.fireOnDeleted`, as `Nodes` already does. |
| `XmlMerge` (363), `ItemReload` (189) | `StoredConfigReconciler` | Field-by-field reload stays available, but as plugin policy. |
| `BlobStore` (73), `BlobStoreProvider` (47), `InMemoryBlobStore` (107) | plugin SPI (`ClusterBlobStore` already extends it) | Core has no blob user left. |
| `InMemoryPersistenceStore` (564) | core test sources (`test/src/test/java/jenkins/persistence`), cut to the key-value contract (≈300); plugin keeps its own in-memory `ClusterStore` in a test-jar | This removes the fixture from the production jar (core-arch M8). |
| `SubItems` (106) | — | Deleted; replaced by the generic owner-reload rule. |
| `FingerprintCleanupThread.Shared` | `AsyncPeriodicWork` + `@ClusterSingleton` | |

Totals: about 8.4k core main lines leave, and core main ends at about 5.3k.

### 1.3 New core extension points (the ones that replace direct calls)

The engines call core through the hooks in §1.1. Core calls the engines only through these. Every default is today's upstream behaviour.

**A. `jenkins.model.ExternalBuilds` [Beta] — "builds of this job not on this controller"** (replaces 13 call sites into `RunningBuilds` / `BuildRecords.hasUnpublished`)

```java
public interface ExternalBuilds extends ExtensionPoint {
    /** Running builds of the job held by other controllers, newest first. Must not do remote I/O on the caller's
     *  thread (answer from a cache, refresh asynchronously): called under the Queue lock and per job per page render. */
    @NonNull List<? extends ExternalBuild> running(@NonNull Job<?, ?> job);
    /** One of them by number, or null. Same no-I/O rule. */
    @CheckForNull default ExternalBuild get(@NonNull Job<?, ?> job, int number) { /* scan running() */ }
    /** Whether the job is to be considered building because of builds elsewhere (incl. completed here, not yet published). */
    default boolean isBuilding(@NonNull Job<?, ?> job) { return !running(job).isEmpty(); }
    static ExternalBuilds get() /* first registered, or NONE */;
}
@Restricted(Beta.class)
public interface ExternalBuild extends ModelObject {   // Stapler object: the plugin supplies index/console views
    int getNumber(); long getStartTimeInMillis(); long getEstimatedDuration();
    @NonNull String getLocation();           // controller display name
    boolean isBuilding(); @CheckForNull Result getResult();
    @NonNull String getUrl();                // relative, under the job
    /** Copies the log from offset; returns the new offset; -1 once complete and fully copied. */
    long writeLogTo(long offset, @NonNull OutputStream out) throws IOException;
    /** Asks the owner to stop it; caller has checked Item.CANCEL. */
    void requestStop(@NonNull String by) throws IOException;
}
```

Core call sites:
- `Job.isBuilding()`: `|| ExternalBuilds.get().isBuilding(this)`. The `isLockHeldByCurrentThread` branch goes away, because the contract forbids I/O.
- `Job.getIconColor()`: anime if `!running(this).isEmpty()`.
- `Job.getDynamic(token)`: when `getBuildByNumber(n) == null`, return `get(this, n)`.
- `HistoryPageFilter.getRunningElsewhere()` returns `List<? extends ExternalBuild>`. The two jelly files become `<st:include it="${b}" page="historyEntry.jelly" optional="true"/>`, so the markup lives in the plugin. This also fixes the drift risk core-compat raised about the duplicated history-entry markup.
- `ConsoleCommand`: run null → `get(job, n)` → `writeLogTo` loop. For `lastBuild`, compare with the newest of `running(job)`.
- `BuildCommand -s/-f/-v`: no special case. It waits on the future (decision 3). If the start future's executable is an `ExternalBuild` view (see C), it follows `writeLogTo`, and on interrupt calls `requestStop`.
- `Job.getBuildsElsewhereForApi` is deleted. The exported data moves to the plugin's `RunningElsewhereAction`, which fixes core-compat M4.

Default `NONE`: empty list, null, false. That is upstream behaviour exactly.

Why it is needed: no upstream extension point contributes to `isBuilding`, the icon, URL dispatch of a build number, or the build history widget.

**B. `jenkins.model.lazy.BuildRecordSource` [Beta] — build records that exist outside this controller's disk** (replaces `RunMap` → `BuildRecords`)

```java
public interface BuildRecordSource extends ExtensionPoint {
    /** Numbers of builds of the job stored elsewhere (added to the lazy index; may have no local directory yet). */
    @NonNull Set<Integer> numbers(@NonNull Job<?, ?> job);
    /** Loads a build directory: may first create it from the stored record, and wraps the load (e.g. so saves made
     *  while loading are not taken for changes). Default: load.get(). */
    <R extends Run<?, ?>> R retrieve(@NonNull Job<?, ?> job, @NonNull File dir, @NonNull IOSupplier<R> load) throws IOException;
    /** Whether build numbers of this job are assigned from a sequence shared with others (RunMap.put then only checks
     *  the number is unused: proposeSharedNumber). */
    boolean sharesNumbers(@NonNull Job<?, ?> job);
}
```

Call sites:
- `AbstractLazyLoadRunMap.loadIdOnDisk`: union with `numbers()`, through the existing `storedNumbers()` protected hook.
- `RunMap.retrieve`: the `d.getParentFile().equals(job.getBuildDir())` check stays in core; then `source.retrieve(job, d, …)`.
- `RunMap.put`: `sharesNumbers` → `proposeSharedNumber`.

At most one source is consulted, the first registered. Default: empty set, direct load, false. That is upstream exactly.

Why it is needed: `RunMap` is final and has no extension point for where builds come from. `ArtifactManager` and `LogStorage` cover only files under a build.

**C. Queue hand-off (core API on `Queue`, [Beta]; decision 3)** — not an extension point, but the narrowest seam for the shared queue

```java
public @NonNull WaitingItem scheduleShared(Task, int quietPeriod, List<Action>);      // exists
public boolean updateShared(long id, Calendar due, List<Action>);                       // exists
/** The item was taken by another controller: it leaves this queue as a LeftItem that is not cancelled
 *  (isTakenElsewhere() true) and its future stays pending. Null if no such waiting/blocked/buildable item. */
public @CheckForNull HandOff handOff(long id);
public interface HandOff {
    void started(@NonNull Queue.Executable view);   // completes getStartCondition(); view is a Run if loadable here
    void completed(@NonNull Queue.Executable run);   // completes the future (Run loaded from the shared record, or the view)
    void cancelled();                                // only if the remote item was cancelled
}
// LeftItem: public boolean isTakenElsewhere();  getExecutable() returns the view once started
```

`QueueListener.onLeft` fires for the hand-off as it does today for a cancellation, and `isTakenElsewhere()` tells the two apart. The external executable is the plugin's `ExternalBuild` implementing `Queue.Executable`, with `run()` throwing. The compatibility page must list one consequence: a caller that casts the *start* value to `Run` gets a `ClassCastException`, whereas the *completion* value is a real `Run` when build records are shared.

No `RemoteQueueItems` extension point is needed. Each controller mirrors every shared item, so `queue/item/<id>` is served from the local LeftItem, which is retained for the usual 5 minutes. Cancel works the same way: the local mirror is cancelled and the plugin's `QueueListener` publishes it. `getItem(long)` stays local, which fixes core-arch M5 and core-compat m7.

Default: no caller, so upstream.

**D. `jenkins.persistence.StoredConfigReconciler` [Beta] — what to do on a conflicting save, and how to reload** (replaces `XmlMerge`, `ItemReload`, `strict-ui`)

```java
public interface StoredConfigReconciler extends ExtensionPoint {
    /** Merged content to write over `theirs`, or null to refuse (the save fails with ConflictException and the
     *  object is reloaded from `theirs`). Called only when a base is known. */
    @CheckForNull default byte[] merge(@NonNull String key, @NonNull byte[] base, @NonNull byte[] mine, @NonNull byte[] theirs) { return null; }
    /** Brings a live object up to date with stored content; false to leave it to the default. */
    default boolean reload(@NonNull XmlFile file, @NonNull Saveable o, @NonNull PersistenceStore.Entry entry) throws IOException { return false; }
}
```

Core calls it from `Route.write` on `ConflictException` (merge), and from `ItemChangeApplier` and `StaticStateApplier` (reload). Defaults:
- merge: refuse and reload, which is decision 4 when no merge is possible;
- reload: `unmarshalNullingOut` followed by `onLoad`, under the object's monitor, the same as `updateByXml`.

Without a store, neither is ever called. The plugin implements the three-way merge, refuses when the same part changed on both sides, and does the field-by-field reload.

**E. Routing extension points.** `LocalFiles` exists already. The new one is `SharedFiles` [Beta]: `Collection<String> patterns()`, for extra per-item files that a plugin wants shared, such as `jobs/**/scm-revision-hash.xml`. The cluster plugin declares these on behalf of plugins that do not, as it does for FlowExecutionList and the LockableResourcesManager XML. This keeps the other plugins' baselines.

**F. Existing, kept:** `RemoteChangeHandler` and `RemoteItemGroup`. The plugin now uses `RemoteChangeHandler` for all `_items/…`, `_locks/…` and similar keys.

The total is three new extension points (A, B, D), one Beta API cluster (C), and one pattern extension point (E). Each has core call sites that no upstream extension point reaches.

### 1.4 Restriction levels

- `PersistenceStores` becomes [NEU] at class level. Only these members are [Beta]: `get`, `isApplyingRemoteChange`, `applyingRemoteChange`, `keyFor`, `fileFor`, `storedItemNames`, `materializedCopies`. `seen`, `lastSeen`, `move`, `moveBack` and the public `Route` record become [NEU] (core-arch M8).
- [Beta] on all new extension points and on `Queue.scheduleShared`, `updateShared`, `handOff`, `LeftItem.isTakenElsewhere`, `AbstractLazyLoadRunMap.knows`, `unload`, `forget`, `Item.getItemUuid`, `ItemUuids.get` and `ItemUuids.pin`.
- `Run.isPendingDelete` and `Queue.isLockHeldByCurrentThread` stay [NEU] unless the plugin needs them; if it does, [Beta].

---

## 2. SPI split

**Decision: core gets a plain versioned key-value store with events; the cluster facets go in the plugin.**

Core `PersistenceStore` [Beta]:
- `read`, `write(key, bytes, expected)`, `delete(key, expected)`, `children`, `deleteAll`;
- `move(fromPrefix, toPrefix, List<Precondition>)`, see §3.3;
- `subscribe`, `getOrigin`;
- `default boolean storesConfidentialKeys()`. This is a statement about the store's security posture, not about file layout, so it may stay.

Contract additions for JEP-250:
- **Total order.** Events are delivered in a single total order, the commit order, at least once. A `MOVED` event is ordered before any later event under the destination prefix.
  - Decision: keep total order as a hard requirement rather than relax it. The applier's ordering of moves and parking depends on it.
  - Postgres (sequence), etcd (global revision) and any log-backed store provide it. A store that cannot provide it is out of scope for v1.
- **Reserved prefix.** Keys whose first segment starts with `_` are never produced by routing. They are reserved for direct users (plugins).

Routing lives in one place: **`jenkins.persistence.StoreRouting`** [NEU], which owns `route(File) → Route|null`.
- **Inputs:** core defaults, `SharedFiles` extensions, `LocalFiles` extensions, and the per-node cloud-agent mark that `Nodes` already sets.
- **Default shared set** (taken from today's `PostgresPersistenceStoreProvider.OWNED`/`STATIC`, minus plugin names):
  - `jobs/**/config.xml`, excluding `jobs/**/builds/**`. This generically covers folders, multibranch `branches/`, matrix `configurations/`, promotions and maven modules.
  - root `config.xml` and root `*.xml`;
  - `users/*/config.xml`, `nodes/*/config.xml`, `log/*.xml`;
  - `secrets/*`, only if `storesConfidentialKeys()`.
- **Default local set:** `LocalFiles.Core`, unchanged.
- **Optional:** the property `jenkins.persistence.StoreRouting.staticState=false` replaces the backend's `SHARE_STATIC`.
- **Invariant written down:** routing changes only when the `LocalFiles`/`SharedFiles` extension lists change (plugin install) or a cloud node is added or removed. The applier re-evaluates routing on those events.
- `nextBuildNumber` is no longer routed (§3.2).

Facets in the plugin SPI (`io.jenkins.plugins.cluster.spi`, already partly there: `ClusterStore extends PersistenceStore, Membership, Claims, Channels, Counters, QueueMaintenance`). Add:
- `BuildLeases`: `acquire`, `holds`, `release`, and an **abstract** `writeIfHeld`. This removes the unsafe check-then-write default (core-compat m6).
- `ClusterQueue`, moved from core;
- `LaunchLeases` (§5);
- `ClusterBlobStore` absorbs `BlobStore`.

The backend (cluster-postgresql) implements `ClusterStore`. `ClusterStore.get()` casts `PersistenceStores.get()`. No `facet(Class)` is needed in core.

Reasons:
- Only plugin code uses the facets once the engines move.
- Core reviewers then review a key-value contract, not a coordination service.
- The Beta key-value API can settle while the facets keep changing.
- Other backends (S3 for blobs, a future etcd) implement the plugin SPI, which is versioned by the cluster plugin's own protocol (JEP-255).
- This also answers the jep-fable question ("one interface or a key-value store plus a coordination interface"): the second option, with the coordination interface in the plugin.

---

## 3. Item-UUID keying

### 3.1 Key scheme

| Data | Today (path) | New |
|---|---|---|
| Item config | `jobs/a/jobs/b/config.xml` | unchanged (routed file) |
| Build record | `jobs/…/builds/<N>/build.xml` | `_items/<uuid>/builds/<N>/build.xml` |
| Permalinks | `jobs/…/builds/permalinks` | `_items/<uuid>/builds/permalinks` |
| Running placeholder | `jobs/…/running/<N>.xml` | `_items/<uuid>/running/<N>.xml` |
| Stop request / delivery | `…/running/<N>.stop`, `…/running/<N>.deliver-<id>` | `_items/<uuid>/running/<N>.stop`, `…/<N>.deliver-<id>` |
| Build lease | placeholder path | lease key `_items/<uuid>/running/<N>` (lease rows keyed by that string) |
| Build counter | `jobs/…/nextBuildNumber` (routed TextFile) | `_items/<uuid>/nextBuildNumber`, written only by the plugin's `BuildNumberAssigner` (raise-only CAS loop). The local `nextBuildNumber` file stays local. The assigner takes `max(stored, job.getNextBuildNumber())`, so `updateNextBuildNumber` on any controller still raises it. |
| Snapshot / log blobs | `jobs/…/builds/<N>/files.tgz`, log | `items/<uuid>/builds/<N>/files.tgz`, `items/<uuid>/builds/<N>/log` |
| ItemFiles (plugin files in item dirs) | path-keyed entries and blobs | `_items/<uuid>/files/<rel>`, blobs `items/<uuid>/files/<rel>`; users: `_users/<id>/files/…` |
| Shared queue record task | full name | item UUID (plus full name for display only); `cancelStartedOf(fullName)` → `(uuid)` |
| lockable-resources holder | `build=<fullName>#N` | `build=<uuid>#N` (plus display name) |
| pipeline-build-step return address | upstream full name | opaque, from the relay (§4.2); the plugin encodes `uuid#N` |

UUID → item resolution uses core `ItemUuids.get(uuid)`, the index that already exists. An event for an unknown UUID is ignored, as "not loaded (yet)" is today: the job lists its numbers from the store when it loads.

### 3.2 Renames and moves

- `AbstractItem.renameTo` / `Items.move` call `PersistenceStores.move(oldRoot, newRoot)`. This moves only the routed entries under the item's path, that is its config and its descendants' configs plus `SharedFiles`. It is one atomic store operation (a single `UPDATE … SET key = …` in Postgres). The cost is O(items in the subtree), with no blobs and no build records involved.
- **Blobs never move.** `AzureBlobStore.move` / `S3BlobStore.move` lose their only caller.
- The O(builds) rename under the `Jenkins` monitor (core-arch M2) is gone.
- **core-b M1 (rename or move while a build runs) is dissolved.** The placeholder, lease, `owned`/`lost` and delivery keys do not change on rename. `owned`/`lost` are keyed by `Run` anyway (`WeakHashMap<Run, Placeholder>`), and `update()` throws if a run is untracked.
- **Local build directories** still move with the job on each controller. That is upstream's own `onLocationChanged` handling, plus the ordinal fix.

### 3.3 What replaces the `.moving` claims

The claims existed because blobs and entries moved in separate steps, and to stop overlapping moves (A into B while B moves into A). With entries only, the store does it atomically:

```java
List<Moved> move(String fromPrefix, String toPrefix, List<Precondition> preconditions) throws IOException;
record Precondition(String key, long expectedVersion /* ANY, ABSENT, or a version; PRESENT = -2 */) {}
```

Core passes:
- the source item's `config.xml` at the version it holds;
- the destination parent's `config.xml` as `PRESENT`, when the parent is not root;
- nothing under `toPrefix`, as `ABSENT` on `toPrefix + "/config.xml"`.

The store checks all of these in the same transaction. A failure raises `ConflictException`, which becomes the existing `Failure` messages.

Overlapping moves now serialise: the second move finds its destination parent gone and fails. `moveBack` remains for the local-filesystem failure path (same preconditions, reversed). About 250 lines are deleted: `claim`, `lapsed`, `checkNoOverlappingMove`, `checkNothingClaimedBelow`, `raiseTo`, `collectBuildNumbers`, `CLAIM_LAPSE_MILLIS`.

### 3.4 Delete and copy

**Delete.** On the controller where the delete was made (not when `isApplyingRemoteChange`), the plugin's `ItemListener.onDeleted` asynchronously runs:
- `deleteAll("_items/<uuid>")`
- `blobs.delete("items/<uuid>")`

Afterwards:
- A job made again with the same name gets a fresh UUID, so it inherits nothing. That is upstream semantics, and simpler than today's `DeletedJobNumbering` handling.
- If the delete fails, orphans are left. They are swept by a `@ClusterSingleton` job that removes `_items/<uuid>` entries with no item.

**Copy.** A copy gets a new UUID and no builds, which is also upstream.

### 3.5 `getItemUuid` persistence (fixes core-compat M1/M2, core-arch M4)

- **The field is serialised only when pinned.** `itemUuid` stays a plain field, and XStream omits it when it is null.
- **When an item is pinned:**
  - when a store is active, a new item is pinned at construction, and an item that is created, copied or imported is pinned before its first write;
  - when a consumer calls [Beta] `ItemUuids.pin(item)`; this saves once and keeps the current derived value, so the identity is continuous.
- **When it is not pinned,** `getItemUuid()` returns the name-derived UUID. That value is stable across restarts but not across renames. The Javadoc says this, and also says the UUID is not unique across instances (core-arch m6).
- **No second save.**
  - `createProjectFromXML`/`copy` inject `<itemUuid>` into the bytes before the first `writeRaw`, only when pinning applies.
  - The `ItemGroupMixInTest` assertion is restored.
  - `ItemUuids.Listener.claim` (a clash after posted XML) still re-saves, but only in the clash case, which is rare and intended.
- **`keepDerivedUuids` is deleted.** When a store is active, every stored item has a UUID, either from creation or from the import or migration tool. Without a store nothing depends on stability unless something pinned it.
- **Downgrade:** a plain install's disk is unchanged. A cluster or pinned install shows OldDataMonitor "unknown field" until each item is re-saved, which is documented. Downgrading a cluster is not supported anyway (the cluster plugin's operations guide, Rolling back).

### 3.6 Migration of existing stored data

**Decision: an offline one-shot tool, not an online Expand/Backfill/Contract.** Nothing has been released, and the only stores holding data are validation clusters. That saves about a week and a whole class of dual-read bugs.

1. Bump the plugin's store protocol and schema version. A controller refuses to join a store with the old layout and says to run the tool.
2. `ClusterLayoutMigration`, run with every controller stopped, as a CLI main in cluster-postgresql or `-D…migrateLayout=true` on one controller before it joins. For each `jobs/**/config.xml`:
   - inject `itemUuid` if it is missing (the derived value, so it stays continuous with what consumers like junit-sql-storage may have recorded);
   - copy `builds/*`, `running/*`, `permalinks` and `nextBuildNumber` to `_items/<uuid>/…` in one transaction per job;
   - copy blobs to `items/<uuid>/…`, then delete the old ones;
   - rewrite lease rows and queue-record tasks;
   - drop the `.moving` entries.
   - It is idempotent and resumable (it skips jobs already migrated).
3. If real data ever exists before release, use the JEP-255 migration framework (`SchemaMigrations`) instead.

---

## 4. Plugin extension points (decision 7)

### 4.1 lockable-resources: `org.jenkins.plugins.lockableresources.LockCoordinator` [Beta in the plugin]

```java
public interface LockCoordinator extends ExtensionPoint {
    /** All or none; false = some resource is held elsewhere (the context stays queued and is retried). */
    boolean tryLock(@NonNull List<LockableResource> resources, @NonNull Run<?, ?> build);
    void released(@NonNull LockableResource resource, @NonNull Run<?, ?> build);
    boolean tryReserve(@NonNull List<LockableResource> resources, @NonNull String user, @CheckForNull String reason);
    void reservedForcibly(@NonNull LockableResource r, @NonNull String user, @CheckForNull String reason, boolean stolen);
    void unreserved(@NonNull LockableResource r);
    /** Unlocked or reset from the resources page: frees it everywhere. */
    void freed(@NonNull LockableResource r);
    boolean isHeldElsewhere(@NonNull LockableResource r);
    void definitionChanged(@NonNull LockableResource r);
    void definitionDeleted(@NonNull String name);
}
```

**Stays in lockable-resources** (no core dependency, baseline unchanged), about 120 lines:
- the extension point and its calls at today's 13 `ClusterLocks.*` sites;
- `LockableResource.remoteLockedBy`, generalised to "held elsewhere by…" for display;
- an inbound API for the coordinator: the existing public `unlockResources`, setters, and `proceedNextContext`;
- the genuine fixes: `LockStepExecution` returns `false` after queuing, and the "internal error" WARNING becomes FINE.

**Moves to cluster-plugin** (`io.jenkins.plugins.cluster.locks`, optional dependency on lockable-resources):
- `ClusterLocks` (526 lines, keys `_locks/<resource>`, `_locks-meta/<name>`), with the plugins-w3 fixes:
  - `release` and `free` use a versioned delete;
  - a DELETED event re-reads the key before unlocking a local build;
  - the orphan sweep uses `ExternalBuilds`/UUID keys and is `@ClusterSingleton`;
  - the listener hands its work off the store's notification thread;
- the `LocalFiles` declaration for the manager XML.

### 4.2 pipeline-build-step: `org.jenkinsci.plugins.workflow.support.steps.build.BuildTriggerRelay` [Beta in the plugin]

```java
public interface BuildTriggerRelay extends ExtensionPoint {
    /** At scheduling: an opaque address to reach this upstream step from wherever the downstream runs; null = local only. */
    @CheckForNull String returnAddress(@NonNull Run<?, ?> upstream, @NonNull String nodeId);
    /** On the downstream's controller: deliver the event if the upstream is not here. true = relayed (skip local path). */
    boolean relay(@NonNull BuildTriggerAction.Trigger trigger, @NonNull Run<?, ?> downstream, @NonNull Event event);
    /** The queue item left this controller because another took it (decision 3: core LeftItem.isTakenElsewhere()). */
    default boolean isTakenElsewhere(@NonNull Queue.LeftItem item) { return false; }
    /** Stop a downstream not running here; false if unknown. */
    default boolean requestStop(@NonNull Job<?, ?> job, int number, @NonNull String by) { return false; }
    /** Whether a queue item must start where it was scheduled (waitForStart). */
    default boolean pinsToOrigin(@NonNull Queue.Item item) { return false; }
    enum Event { STARTED, COMPLETED, DELETED }
}
// Inbound, public static: BuildTriggerStepExecution.deliver(Run<?, ?> upstream, String nodeId, Event e,
//   String downstreamExternalizableId, @CheckForNull Result result, boolean waitForStart)
//   -> completes the step context (STARTED + waitForStart: onSuccess with RunWrapper by id, or a clear failure)
```

**Stays in pipeline-build-step** (no core dependency; baseline stays 2.479):
- `Trigger.returnAddress` (replaces `upstreamJob`/`upstreamNumber`);
- `waitForStart` in the payload;
- the inbound `deliver`, which fixes plugins-w3 finding 1 (the `waitForStart` hang);
- the falls-back-to-payload-result path (finding 2);
- unit tests with a fake relay.

**Moves to cluster-plugin** (`io.jenkins.plugins.cluster.pipeline`, optional dependency): `BuildTriggerDelivery` (the deliveries over `_items/<uuid>/running/<N>.deliver-<id>`), `takenElsewhere`, stop and pinning. Core's `SharedQueue.BUILD_TRIGGER_ACTION` string sniffing goes, replaced by `pinsToOrigin`.

With decision 3 the step does not need the `QueueTaskFuture`: the future does not survive a restart or an adoption, but deliveries do.

---

## 5. SSH/outbound agents: launch lease (sketch, decision 6 pending)

**Goal:** one controller holds a static node's outbound connection; the other controllers never launch it.

1. **Lease:** plugin facet `LaunchLeases` (Postgres: row `(node, holder_origin, epoch)`), key `_nodes/<name>/launch`. It is acquired create-only, or taken over if the holder is not in `aliveMembers()`. It is released on `ComputerListener.onOffline` (a local disconnect) and on node deletion. It is not renewed per lease; liveness comes from membership heartbeats.
2. **Gate, with no core change:** the plugin's `ComputerListener.preLaunch(c, listener)` applies to nodes that are not cloud agents, whose `getLauncher()` is not `JNLPLauncher`/inbound. If the lease is held by another live member it throws `AbortException("Connected on <controller>")`. The computer stays offline on non-holders.
   - `RetentionStrategy.Always` retries about every 60 s, and each retry re-checks the lease. That retry is the lapse detection: when the holder dies, membership times out (15 s), and the next retry anywhere acquires and launches.
   - The cost is one "launch failed" line per minute per node on non-holders. If that is too noisy, add a tiny core `LaunchGate` (`@CheckForNull OfflineCause mayLaunch(SlaveComputer)`, checked in `SlaveComputer._connect`), which shows a quiet offline cause.
3. **Builds:** non-holders have no online executors on the node, so a shared-queue item for its label can only be claimed by the holder (the existing claim on start). The plugin's `QueueStartGate.countsForProvisioning` returns false for labels the node serves while its holder is live and has idle capacity (it uses `Channels`), so clouds elsewhere do not provision for it.
4. **UI:** `/computer/<name>/` on a non-holder is forwarded to the holder, using JEP-254 `Forwarding` with `Channels.channelHolder`. Node configuration stays shared. An edit applies on the holder at the next connect, as it does upstream.
5. **Fencing:**
   - A partitioned holder fences itself: when it loses its heartbeat (T14) it disconnects its outbound channels. The overlap is bounded by the member timeout.
   - During that window a second agent JVM may start against the same remote root. This is documented. The remoting work-dir lock (`-workDir`) catches it where it is enabled.
6. **Inbound and cloud agents** are unchanged (inbound: wherever the load balancer sends; cloud: `keepLocal`).
7. **Size:** about 300 plugin lines plus tests. JEP-254 and JEP-257 get a section. Alternative: declare outbound agents unsupported in v1 and warn on such a node.

---

## 6. New stack layout

### 6.1 Core PRs (in order; new branches `cnj2/*` from `fork/cnj/base`)

**A. Standalone hooks.** Each is useful without a store, they can be opened in any order, and they can go to jenkinsci early.

| # | Branch | Scope | Main/test lines | From (0a47f66bc6) |
|---|---|---|---|---|
| A1 | `cnj2/a1-item-uuid` | `Item.getItemUuid`, `ItemUuids` (+`pin`), pinned-only persistence, byte injection in create/copy | ~300 / 250 | t15 (AbstractItem, Item, ItemUuids, ItemGroupMixIn hunks), reworked |
| A2 | `cnj2/a2-trigger-coordinator` | `TriggerCoordinator` + `Trigger`/`SafeTimerTask` call | ~90 / 180 | t2-m2 |
| A3 | `cnj2/a3-periodic-coordinator` | `PeriodicWorkCoordinator`, `@ClusterSingleton` on 3 works | ~140 / 170 | t5-m3 minus `FingerprintCleanupThread.Shared` |
| A4 | `cnj2/a4-queue-hooks` | `QueueStartGate` + `provisionsFor`, `QueueListener.onFolded` (+test), `Queue.LOADED`, `isLockHeldByCurrentThread` | ~200 / 200 | t5-m1, t9-m1 hunks |
| A5 | `cnj2/a5-runmap-hooks` | `knows`/`unload`/`forget`/`proposeSharedNumber`, `storedNumbers` hook, `Run.isPendingDelete`, ItemListener ordinal fix (separate commit, own test) | ~100 / 80 | t3-m1..m3, t15 |
| A6 | `cnj2/a6-plugin-archive-listener` | `PluginArchiveListener` + `doUploadPlugin` + test; notices extension point (a dedicated small one, not the `PageDecorator` overload) | ~110 / 100 | t15 |
| A7 | `cnj2/a7-small-fixes` | `Items.move` into-itself (as `IOException`), `User.delete` fires `SaveableListener.onDeleted`, `Nodes` name validation | ~50 / 60 | t15, t6 |

**B. JEP-250 core.** Strictly ordered.

| # | Branch | Scope | Main/test | From |
|---|---|---|---|---|
| B1 | `cnj2/b1-store-spi-routing` | Key-value `PersistenceStore`, provider, `StoreRouting` + `LocalFiles`/`SharedFiles`, `XmlFile`/`TextFile` routing with version table, `exists()` error semantics, create-only unread writes, refuse+reload default, `StoredConfigReconciler`, `materializedCopies`, `InMemoryPersistenceStore` in test sources | ~1,500 / 900 | t1-m1, t6-m1 (routing parts), t11 (hook only), t14 (store-outage fixes), t15 (getFile) |
| B2 | `cnj2/b2-items` | store-only enumeration, lifecycle with `move`+preconditions (no claims), `ItemChangeApplier` for items, `RemoteItemGroup`, `RemoteChangeHandler` dispatch, generic sub-item rule | ~1,200 / 1,300 | t1-m2..m4, t3-m4 (item parts), t15 |
| B3 | `cnj2/b3-static-state` | users, nodes (cloud keepLocal), global config/descriptors, `EarlyDescriptors`, setters under lock, `unmarshalNullingOut` reload | ~600 / 900 | t6-m1, t6-m2, t15 |
| B4 | `cnj2/b4-keys-sessions` | shared confidential keys, stateless sessions, logout revocation hook (decision 5) | ~450 / 500 | t6-m3, t6-m6, + new |

**C. Build and queue extension points.** These come after B1, are independent of B2–B4, and are mutually independent.

| # | Branch | Scope | Main/test | From |
|---|---|---|---|---|
| C1 | `cnj2/c1-build-record-source` | `BuildRecordSource` + RunMap/AbstractLazyLoadRunMap wiring + fake-source test | ~130 / 200 | t3-m1 RunMap hunks |
| C2 | `cnj2/c2-external-builds` | `ExternalBuilds`/`ExternalBuild`, Job/HistoryPageFilter/jelly include/getDynamic/ConsoleCommand/BuildCommand | ~280 / 300 | t12-m1, t15 CLI hunks, reworked |
| C3 | `cnj2/c3-queue-handoff` | `scheduleShared`/`updateShared`/`handOff`/`LeftItem.isTakenElsewhere`, future completion (decision 3); remove `RemoteItem` | ~250 / 350 | t5-m1 Queue hunks, + new |

Core total: about 5.3k main and about 6.0k tests, in 14 PRs. The largest is B1 at about 1.5k, compared with today's #162 at 3.2k.

Merged-together test branch `cnj2/all` = A* + B* + C*, for the plugin to build against as an incremental.

### 6.2 Plugin-side counterparts

**cluster-plugin** (one stacked series, against `cnj2/all` incrementals):

| # | Scope |
|---|---|
| P0 | SPI: move `BlobStore`/`BuildLeases`(abstract `writeIfHeld`)/`ClusterQueue` into `spi`; drop `owns`, `storesBuildRecords`; Postgres `move` with preconditions; routing patterns removed from `PostgresPersistenceStoreProvider` (now a `SharedFiles`/`LocalFiles` contribution); protocol bump |
| P1 | builds: `BuildRecords`, `StoredPermalinks`, `StoredBuildNumberAssigner` on `_items/<uuid>`; `BuildRecordSource`; `RemoteChangeHandler` |
| P2 | running: `RunningBuilds`, `BuildAdopter` (+`PipelineAdopter`, `SavePoints`), `ExternalBuilds` + views, `RunningElsewhereAction` |
| P3 | queue: `SharedQueue` (+gate, ids, reservations, publisher on `handOff`), `SharedBlocking`, `SharedQueueMaintenance` |
| P4 | files: `ItemFiles`, materialised write-back |
| P5 | config: `XmlMerge` + `ItemReload` as `StoredConfigReconciler`; `ConflictMonitor` adjusts |
| P6 | integrations: locks (lockable-resources), pipeline relay (pipeline-build-step) |
| P7 | layout migration tool |
| P8 | launch leases (if approved) |

**lockable-resources:** L1, the `LockCoordinator` extension point and the fixes (no core dependency). **pipeline-build-step:** S1, the `BuildTriggerRelay` extension point, `waitForStart` and tests (no core dependency). **folder, credentials, junit-sql-storage:** unchanged. The folder PR now calls Beta `storedItemNames`. junit-sql-storage#53 is split as plugins-w3 suggested.

### 6.3 Mechanics

**Decision: new branches from `fork/cnj/base`; do not rewrite the 24.** Interactive rebase is unavailable, and 242 commits with fixups across layers would cost more to untangle than to re-cut.

1. Tag `cnj-rev3-final` = `0a47f66bc6` and keep the old branches and PRs #139–#162 as an archive. Close them with a link once the new ones are up.
2. For each new branch:
   - `git switch -c cnj2/<x> fork/cnj/base`
   - `git checkout cnj-rev3-final -- <files wholly in scope>`
   - hand-edit the shared files: Job, Queue, AbstractItem, PersistenceStores, ItemChangeApplier, XmlFile;
   - commit as 1–5 commits per PR with upstream-style messages, and no "Target N"/ET jargon (core-arch m4).
   - Stack with gh-stack: A* are independent off base; B1 → B2 → B3 → B4; C* on B1.
3. Engines go to the plugin through `git show cnj-rev3-final:<path>` into the new package, then repackaged and rewired. The plugin keeps a commit per engine.
4. The JEP branch is rewritten in place (it is documentation).

### 6.4 Test strategy

- **Core keeps and adapts** (about 5.5k lines):
  - InMemoryPersistenceStoreTest (key-value contract, plus `move` preconditions), PersistenceStoreItem{Config,Enumeration,Lifecycle}Test, ItemChangeApplierTest (minus the build cases), StaticStateApplierTest, StaticStateRoutingTest, LocalStateTest;
  - StrictConflictsTest becomes RefuseAndReloadTest;
  - SharedConfidentialKeysTest, StatelessSessionsTest, TriggerCoordinatorTest, PeriodicWorkCoordinatorTest, ItemUuidTest, NodesTest;
  - new: tests with fake extension-point implementations for C1/C2/C3, `onFolded`, `PluginArchiveListener`, a test that the disk is unchanged without a store (no `itemUuid`, no `buildsElsewhere`), and a test that `getFile()` reads the store at most once.
- **Move to cluster-plugin** (about 6.3k lines): SharedQueueTest, BuildLeasesTest, BuildRecordsTest, ClusterCoordinationTest, RunningBuildsTest, RemoteBuildEventsTest, ItemFilesTest, DeletedJobNumberingTest, StoredPermalinksTest, StoredBuildNumberAssigner*Test, ItemReloadTest, XmlMergeTest (unit).
  - They run against the plugin's in-memory `ClusterStore` with `as("other")`, plus a delayed-delivery wrapper for the stale-event cases.
  - The core stub `org.jenkinsci.plugins.workflow.support.steps.build.BuildTriggerAction` is deleted; use pipeline-build-step in test scope.
- **New tests for the open findings:** rename/move while building (now trivial), a record written elsewhere for a build running here (core-b M2), queue hand-off futures (decision 3), launch lease.
- **Remove fixed sleeps** while moving the tests: poll with deadlines on counters (core-compat). PeriodicWorkCoordinatorTest:58-67 goes first.
- **Gate:** the validation ET suites on kind after P3 and again after P6. Postgres Testcontainers tests stay in cluster-postgresql.

---

## 7. JEP impact

- **JEP-250 "Pluggable persistence for configuration" (Standards, core)** specifies:
  - the key-value SPI (all methods listed, `move` preconditions, CAS versions, total-order at-least-once events, the reserved `_` prefix, `storesConfidentialKeys`);
  - routing owned by core (default shared set, `LocalFiles`/`SharedFiles`, the invariant);
  - "no provider, no change", with the list of the only seams;
  - conflict detection with refuse-and-reload as the only core behaviour, and `StoredConfigReconciler` (merge is an implementation's choice; the text states that lists are one unit for the reference merge);
  - applying remote changes for items, static state, users and nodes, plus `RemoteChangeHandler`/`RemoteItemGroup`;
  - confidential key sharing; and item identity (`getItemUuid`, pinning) as a prerequisite section. A1 needs no JEP of its own.
  - Removed from 250: `ItemFiles`, write-back, merge algorithm, `@PerController`, `storesBuildRecords`, leases, queue, `DoubleLaunchChecker` sentence (jep-fable m1).
- **JEP-255 "Cluster coordination" (Standards, cluster plugin; was 255a)** specifies:
  - the plugin coordination SPI: membership and liveness (database clock), leases with fenced writes, claims, counters, shared-queue record contract, channels, launch leases;
  - the UUID-keyed build-data layout (§3.1);
  - protocol and schema versions with the migration rule; outage behaviour.
- **Operating a cluster** (was JEP-255b, Informational; now documentation in the plugin repository, `docs/operating-a-cluster.md`, cluster-plugin#7): chart, import, backup/PITR, rollback window, sizing (laptop measurements as a table).
- **251, 252, 253** each specify the core extension points they rely on (251: `BuildRecordSource` and permalinks/number assigner use; 252: `ExternalBuilds`, adoption is plugin-side; 253: queue hand-off and decision 3 future semantics, plus `QueueStartGate`) and the plugin behaviour.
  - Requires: 251 → 250, 255; 252 → 251, 255 (deliveries move into 252); 253 → 250, 255, 252.
  - This removes the circular 252↔255 dependency (jep-fable M5): liveness is defined in 255, which needs only 250.
- **254/257** gain the outbound-agent launch lease, or the "unsupported in v1" statement.
- **Submission order:** 250 → 255 → 251 → 252 → 253 → 254 → 256 → 257 (the operations guide is plugin documentation, not a JEP).

---

## 8. Effort, risks, parallelism

### 8.1 Effort (agent-days)

| Step | Days |
|---|---|
| S0 API freeze: write the Java signatures of §1.3/§2/§3.3/§4 into one stub branch (`cnj2/api`) that every agent compiles against | 0.5 |
| A1–A7 | 2.5 |
| B1 | 3 |
| B2 | 2.5 |
| B3 + B4 | 2 |
| C1 + C2 | 2 |
| C3 (decision 3) | 2 |
| P0 SPI + Postgres move preconditions + protocol | 2 |
| P1 builds on UUID layout | 3 |
| P2 running/adoption/ExternalBuilds views | 3 |
| P3 shared queue on hand-off | 2.5 |
| P4 + P5 | 1.5 |
| P6 + L1 + S1 | 2.5 |
| P7 migration tool | 1 |
| P8 launch lease (if approved) | 2 |
| Test moves and de-sleeping (overlaps P*) | 3 |
| JEP rewrite | 1.5 |
| Validation (ET on kind, 2 rounds) | 2 |
| **Total** | **≈ 40 gross; ≈ 30 net of overlap**; ~2.5 weeks wall with 4–5 agents |

### 8.2 Parallel agents (no file conflicts)

- **Agent 1 (core A):** A1–A7, branches off base. It touches AbstractItem (A1), which B2 also touches, so A1 lands first and B2 rebases on it. A1 is small.
- **Agent 2 (core B):** B1 → B2 → B3 → B4. It owns PersistenceStores, XmlFile, ItemChangeApplier, Nodes, User and Jenkins.
- **Agent 3 (core C):** C1, C2, C3 on B1. It owns Job, RunMap, HistoryPageFilter, Queue, the CLI and the jelly. It does not touch Agent 2's files, except PersistenceStores (read-only use).
- **Agent 4 (cluster plugin engines):** P0–P5 against the S0 stub, then against real incrementals. It is the only writer in cluster-plugin `cluster/` and `cluster-postgresql/`.
- **Agent 5 (integrations):** L1 in lockable-resources and S1 in pipeline-build-step, then P6 in a separate cluster-plugin package (`locks`, `pipeline`) to avoid Agent 4's files; later P8 (`agents` package).
- **Agent 6 (docs):** JEP rewrite and UPSTREAMING.md, from this document; no code.
- **Serialisation points:**
  - S0 before everyone;
  - B1 before C* and P1+;
  - C3 before P3;
  - the final validation after all.

### 8.3 Risks

1. **Decision 3 queue hand-off** (C3/P3) changes `QueueTaskFuture` semantics, and the start value is not a `Run` (ClassCastException for some callers).
   - Mitigation: list it on the compatibility page.
   - Test it with Parameterized Trigger–style `future.get()` and with CLI `-s -v`.
   - Fallback: also pin awaited items to the origin.
2. **Big move plus layout change together.** Regressions get hidden behind the repackaging.
   - Mitigation: P1/P2 first port with the old path keys behind a `Keys` helper, run the moved tests, then switch `Keys` to the UUID layout in one commit.
3. **Plugin tests lose the core harness.** Some core tests relied on package-private engine access.
   - Mitigation: the plugin engines expose `@VisibleForTesting` package-private hooks; the in-memory `ClusterStore` goes in the test-jar.
4. **Reload semantics change:** `unmarshalNullingOut` replaces `clearRemovedFields`, and the default reload replaces field-by-field.
   - Mitigation: the plugin's reconciler keeps field-by-field. Core tests cover the default with Jenkins root, descriptor, user and folder.
   - Watch the Jenkins root: keep the existing `reloadGlobalConfig` subset rather than nulling out the root object.
5. **Refuse-by-default for background saves** (decision 4) can drop automated saves, such as multibranch indexing.
   - Mitigation: the plugin's merge handles disjoint parts. The ConflictMonitor counts refusals. ET11 is re-run.
6. **Total-order requirement** may block future non-SQL backends. Accepted for v1 and stated in the JEP.
7. **Upstream acceptance of C2/C3:** these extension points have no in-tree user.
   - Mitigation: each PR carries a fake implementation in tests and a link to the plugin PR. JEP-251/252/253 justify them.
8. **Stale incrementals:** the plugin builds against a moving core stack.
   - Mitigation: pin to `cnj2/all` incrementals, rebuilt by Agent 2/3 on each merge. Plugin CI runs nightly against them.

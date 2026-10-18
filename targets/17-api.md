# DESIGN-17 S0: the frozen API

The signatures every round-4 agent builds against. Design: [17-architecture.md](17-architecture.md). Javadoc in the branches is the contract; this page is the index.

## Branches and commits

| Repo | Branch | Commit | Base | Worktree |
|---|---|---|---|---|
| timja/jenkins (core) | `cnj2/api` | `9ab92f4e3c` "Freeze the core API that the cnj2 branches and the cluster plugin build against" | `fork/cnj/base` (`cd2a077`) | `~/code/jenkins/jenkins-cnj-test` |
| timja/cluster-plugin | `cnj2/api` | `87dce90` "Freeze the cluster SPI that the ported engines build against" | `main` (`67c9e63`) | `~/code/jenkins/cluster-plugin-cnj2` |

Nothing is pushed.

Stub bodies:
- where the no-store answer is trivial, the body is that answer (`PersistenceStores.get()` returns null, `materializedCopies()` returns an empty list, `ExternalBuilds.NONE`, `BuildRecordSource.get()` returns null, the run map hooks, `isApplyingRemoteChange`);
- everything else throws `UnsupportedOperationException("cnj2/api stub")`.

Each `cnj2/*` branch is cut from `fork/cnj/base` (§6.3). It copies the API files it owns from `cnj2/api` unchanged and fills in the bodies. A signature change needs agreement, and the change is made on `cnj2/api` first.

Who replaces which stub:

| Owner | Stubs |
|---|---|
| A1 | `ItemUuids` |
| A2 | `TriggerCoordinator` call sites |
| A3 | `PeriodicWorkCoordinator` call sites |
| A4 | `QueueStartGate` and `onFolded` call sites |
| A5 | run map hooks |
| A6 | `PluginArchiveListener`, `PluginManagerNotice` |
| B1 | `PersistenceStore*`, `StoreRouting`, `SharedFiles`, `LocalFiles`, `StoredConfigReconciler`, `materializedCopies` |
| B2 | `move`/`moveBack`, `storedItemNames`, `RemoteChangeHandler`/`RemoteItemGroup` dispatch |
| B4 | `LoginRevocation` call sites |
| C1 | `BuildRecordSource` wiring |
| C2 | `ExternalBuilds` wiring |
| C3 | `scheduleShared`, `updateShared`, `handOff`, `isTakenElsewhere` |

## Compile status

- **Core:** `mvn -B -ntp -q -o -pl core -am compile -DskipTests -Dspotbugs.skip -Dcheckstyle.skip -Denforcer.skip -Djacoco.skip -Dlicense.skip -Dmaven.javadoc.skip` passes. `spotless:apply` was run.
- **Cluster plugin `spi` package:** compiles with `javac --release 17` against `jenkins-core-2.586-cnj2-api-SNAPSHOT.jar` and the module's compile classpath, with no errors.
  - `ClusterMember` was left out of that run. It is unchanged, and it depends only on plugin classes (`ClusterPlugins`, `MemberStatus`).
- **The rest of the `cluster` module does not compile yet**, as expected. About 11 files still use the core engines that have moved out (`PipelineAdopter`, `BlobLogStorage`, `ClusterPlugins`, `OwnerRoutingFilter`, …). P0–P5 port them.
- **`cluster-postgresql`, `cluster-s3` and `cluster-azure-blob`** still implement the old core `BlobStore*` and the old `PersistenceStore` methods. P0 fixes them; they were not built.

## How plugins build against the core API

- The core build is installed **locally only**, under its own version **`2.586-cnj2-api-SNAPSHOT`**. Nothing overwrites `2.586-SNAPSHOT` or `999999-SNAPSHOT`.
  - Command, in `jenkins-cnj-test` on `cnj2/api`: `mvn -o -pl war -am install -Dchangelist=-cnj2-api-SNAPSHOT -DskipTests` (plus the skips above). The version is set on the command line, so no pom is edited.
  - `bom` was installed too (`-pl bom`), because the plugin pom imports `jenkins-bom:${jenkins.version}`.
  - Installed artifacts: `jenkins-parent`, `jenkins-bom`, `websocket-spi`, `cli`, `jenkins-core` and `jenkins-war`, all `2.586-cnj2-api-SNAPSHOT`.
- `cluster/pom.xml` on cluster-plugin `cnj2/api` sets `<jenkins.version>2.586-cnj2-api-SNAPSHOT</jenkins.version>`, with a comment giving the install command.
  - Another plugin can build against it the same way: `-Djenkins.version=2.586-cnj2-api-SNAPSHOT`, with `useBeta=true`.
- **Later:** when `cnj2/all` exists (§6.1), the plugin moves to its incrementals (`2.586-rcNNNNN.<hash>`), as §8.3 risk 8 says.
  - If an agent needs a refreshed API build before then, it re-runs the install command after `cnj2/api` changes.
  - If a build machine is used instead (192.168.4.91), use a fresh `~/code/cnj2-api` directory there.
- lockable-resources and pipeline-build-step do not depend on core changes; their baselines stay as they are.

## Core (`cnj2/api`), in brief

**[Beta]** = `@Restricted(Beta.class)`; **[NEU]** = `@Restricted(NoExternalUse.class)`.

### Store SPI (`jenkins.persistence`)

- `PersistenceStore` [Beta]:
  - constants: `ANY=-1`, `ABSENT=0`, `PRESENT=-2` (`PRESENT` is for preconditions only);
  - `Entry read(key)`, `long write(key, bytes, expected)`, `void delete(key, expected)`, `Set<String> children(parent)`, `default int deleteAll(prefix)`;
  - `List<Moved> move(from, to, List<Precondition>)`;
  - `String getOrigin()`, `AutoCloseable subscribe(Consumer<ChangeEvent>)`, `default boolean storesConfidentialKeys()`;
  - records: `Precondition(key, expectedVersion)`, `Moved(from, fromVersion, to, version)`, `ChangeEvent(kind WRITTEN|DELETED|MOVED, key, oldKey, version, origin)`, `Entry(content, version)`;
  - the Javadoc states the contract: total commit order, at-least-once delivery, `MOVED` ordered before later events under the destination, and the reserved `_` prefix.
- `PersistenceStoreProvider` [Beta]: `PersistenceStore create()`.
- `ConflictException` [Beta]: `(key, expectedVersion, actualVersion)` plus getters.
- `PersistenceStores` [NEU class]:
  - [Beta] members: `get()`, `applyingRemoteChange(ThrowingCallable)`, `isApplyingRemoteChange()`, `keyFor(File)`, `fileFor(String)`, `storedItemNames(File[, Set skip])`, `materializedCopies()`, and the record `MaterializedCopy(key, file, content, version, modified)`;
  - [NEU] members: `move(File, File)`, `moveBack(File, File)`, `seen(key, v)`, `lastSeen(key)`, and the record `Route(store, key)`.
- `StoreRouting` [NEU]: `Route route(File)`, `keepLocal(key, boolean)`, `STATIC_STATE` (`-Djenkins.persistence.StoreRouting.staticState`).
- `SharedFiles` [Beta]: `Collection<String> patterns()`. The Javadoc lists core's default shared set.
- `LocalFiles` [Beta]: `Collection<String> patterns()` and the nested `Core` [NEU]. `declares` was removed.
- `StoredConfigReconciler` [Beta]:
  - `default byte[] merge(key, base, mine, theirs)` returns null (refuse);
  - `default boolean reload(XmlFile, Saveable, Entry)` returns false (use the default reload, `unmarshalNullingOut` + `onLoad`);
  - reconcilers are consulted in ordinal order.
- `RemoteChangeHandler` [Beta]: `boolean handles(key)`, `void changed(key, deleted)`. It receives every key core does not route.
- `RemoteItemGroup` [Beta]: `void addRemotelyCreated(TopLevelItem)`.

### Builds

- `jenkins.model.ExternalBuilds` [Beta]:
  - `List<? extends ExternalBuild> running(Job)`, `default ExternalBuild get(Job, int)`, `default boolean isBuilding(Job)`;
  - `static ExternalBuilds get()`, which returns the first registered or `NONE`;
  - the Javadoc forbids remote I/O.
- `jenkins.model.ExternalBuild` [Beta], `extends ModelObject`: `getNumber`, `getStartTimeInMillis`, `getEstimatedDuration`, `getLocation`, `isBuilding`, `getResult`, `getUrl`, `long writeLogTo(offset, out)` (returns -1 when done), `requestStop(by)`.
- `jenkins.model.lazy.BuildRecordSource` [Beta]:
  - `Set<Integer> numbers(Job)`, `<R extends Run> R retrieve(Job, File dir, ThrowingCallable<R, IOException> load)`, `boolean sharesNumbers(Job)`;
  - `static BuildRecordSource get()`, which returns the first registered or null.
- `AbstractLazyLoadRunMap`:
  - [Beta]: `knows(n)`, `unload(n)`, `forget(n)`;
  - [NEU]: `protected Set<Integer> storedNumbers()`, already wired into `loadIdOnDisk` with an empty default, and `protected proposeSharedNumber(n)`.
- `Run.isPendingDelete()` [Beta].

### Queue

- `Queue` [Beta] methods:
  - `WaitingItem scheduleShared(Task, int quietPeriod, List<Action>)`;
  - `boolean updateShared(long id, Calendar due, List<Action>)`;
  - `HandOff handOff(long id)`;
  - the constant `String LOADED`, which `init` now attains.
- `Queue.HandOff` [Beta]:
  - `started(Executable view)`, `completed(Executable run)`, `cancelled()`;
  - decision 3 semantics are in the Javadoc: the future stays pending; at start the value is a `Run` or a view (the `ClassCastException` caveat); at completion it is a `Run` when records are shared.
- `Queue.LeftItem.isTakenElsewhere()` [Beta]. `isCancelled()` is false for such an item.
- `Queue.isLockHeldByCurrentThread()` [NEU].
- `QueueListener.onFolded(Queue.Item, List<Action>)` [Beta]. `onLeft` also fires for a hand-off.
- `jenkins.model.queue.QueueStartGate` [Beta]: `CauseOfBlockage mayStart(BuildableItem)`, `default boolean countsForProvisioning(BuildableItem)`, `static provisionsFor(item)`, `static check(item)`. Taken from rev3 unchanged.

### Items

- `Item.getItemUuid()` [Beta]: the default is null. The Javadoc states the pinned/derived contract and that the UUID is not unique across instances.
- `jenkins.model.ItemUuids` [Beta]: `Item get(String uuid)` (in-memory, visibility-checked) and `String pin(Item)` (saves once).

### Coordination and other hooks (taken from rev3 unchanged)

- `jenkins.triggers.TriggerCoordinator` [Beta]: `boolean shouldRun(TriggeredItem, Trigger, Calendar tick)`, `static allows(…)`.
- `jenkins.util.PeriodicWorkCoordinator` [Beta]: `boolean shouldRun(PeriodicWork, long period)`, `static allows(work)`.
- `@jenkins.util.ClusterSingleton` [Beta].
- `jenkins.plugins.PluginArchiveListener` [Beta]: `default beforeInstall(shortName, File)` (throws `AbortException`), `default installed(shortName, File)`, `static all()`.

### New in S0

- `jenkins.plugins.PluginManagerNotice` [Beta]: an abstract class (`ExtensionPoint`) with `boolean isVisible()` and `static all()`. Its view is `notice.jelly`. It replaces the `PageDecorator`/`pluginManager.jelly` overload (A6).
- `jenkins.security.LoginRevocation` [Beta], for decision 5:
  - `void revoke(String loginId, String username, long expiresAt)`, `boolean isRevoked(String loginId)` (in memory), `static anyRevoked(loginId)`;
  - the login id is the Base64url SHA-256 of the remember-me cookie value;
  - core records the login id in every session made from that cookie, and calls `revoke` on sign-out;
  - core rejects revoked cookies and invalidates sessions whose login is revoked;
  - the user's other browsers stay signed in.

## Cluster plugin SPI (`io.jenkins.plugins.cluster.spi`, `cnj2/api`)

- `ClusterStore extends PersistenceStore, Membership, Claims, Channels, Counters, QueueMaintenance`: adds `BuildLeases leases()`, `ClusterQueue queue()` and `LaunchLeases launchLeases()`, plus `static get()`.
- `BuildLeases`: `long acquire(leaseKey)`, `boolean holds(leaseKey, epoch)`, `void release(leaseKey, epoch)`, and an abstract `long writeIfHeld(leaseKey, epoch, key, bytes, expected)`, which must be atomic and throws `LeaseLostException`.
- `LeaseLostException(leaseKey, epoch)`, `extends IOException`.
- `ClusterQueue`:
  - moved from core; `Record(id, itemUuid, task /* display */, dedupKey, actions, dueAt, state, origin, pinnedTo, claimedBy, buildNumber)`;
  - `nextId`/`nextIds`, `publish`, `update(…, pinTo)`, `claim(id, expectedPin)`, `started`, `cancel`, `get`, `pending`, `subscribe`.
- `QueueMaintenance`: `cancelStartedOf(String itemUuid)`, and `StartedItem(id, itemUuid, task, buildNumber)`. The other methods are unchanged.
- `LaunchLeases` (§5): `long acquire(node)` (create-only, or take over from a dead holder), `Holder holder(node)`, `void release(node, epoch)`, and the record `Holder(origin, epoch)`.
- `ClusterBlobStore` is standalone now; it absorbs core's `BlobStore`:
  - `put`, `get`, `delete(prefix)`, `list`, `download`, `size`, `sizes`, `logs()`;
  - it has no `move`;
  - `static get()` (stub).
- `ClusterBlobStoreProvider` (`ExtensionPoint`): `ClusterBlobStore create()`.
- `Keys`, the only place keys are built (§3.1, §8.3 risk 2):
  - `LAYOUT` (`PATH` while porting, `UUID` by default);
  - `item(Item)` and `itemByUuid(uuid)` (`_items/<uuid>`), `blobs(Item)` (`items/<uuid>`);
  - per item prefix: `builds`, `buildRecord(p, n)`, `permalinks`, `nextBuildNumber`, `running`, `placeholder(p, n)`, `stopRequest`, `delivery(p, n, id)`, `buildLease(p, n)`, `itemFile(p, rel)`;
  - `userFile(id, rel)`;
  - blobs: `snapshotBlob`, `logBlob`, `itemFileBlob`;
  - `lock(resource)`, `lockMeta(name)`, `launchLease(node)`;
  - `ItemKey parse(key)`, with the record `ItemKey(uuid, rest)`.
- Unchanged: `Membership`, `Claims`, `Channels`, `Counters`, `EarlyStoreReader`, `LogBlobs`, `ChunkedLogBlobs`, `ClusterMember`.

## Plugin extension points (§4), signatures only

No repository edits were made for these. Agent 5 creates them in L1 and S1.

### lockable-resources: `org.jenkins.plugins.lockableresources.LockCoordinator` [Beta in the plugin]

```java
package org.jenkins.plugins.lockableresources;

/** Coordinates resources with other controllers (implemented by the cluster plugin). With none: local only, as today. */
@Restricted(Beta.class)
public interface LockCoordinator extends ExtensionPoint {
    /** All or none; false = some resource is held elsewhere (the context stays queued and is retried). */
    boolean tryLock(@NonNull List<LockableResource> resources, @NonNull Run<?, ?> build);
    void released(@NonNull LockableResource resource, @NonNull Run<?, ?> build);
    /** All or none; false = some resource is locked or reserved elsewhere. */
    boolean tryReserve(@NonNull List<LockableResource> resources, @NonNull String user, @CheckForNull String reason);
    void reservedForcibly(@NonNull LockableResource r, @NonNull String user, @CheckForNull String reason, boolean stolen);
    void unreserved(@NonNull LockableResource r);
    /** Unlocked or reset from the resources page: frees it everywhere. */
    void freed(@NonNull LockableResource r);
    /** From memory, no remote I/O (called while rendering and under the manager's lock). */
    boolean isHeldElsewhere(@NonNull LockableResource r);
    void definitionChanged(@NonNull LockableResource r);
    void definitionDeleted(@NonNull String name);

    /** The first registered coordinator, or null: every call site does what it does today. */
    static @CheckForNull LockCoordinator get() { /* first of ExtensionList.lookup(LockCoordinator.class) */ }
}
// Inbound for the coordinator (existing public API): LockableResourcesManager.unlockResources(...), the LockableResource
// setters, LockableResourcesManager.proceedNextContext(); LockableResource.getHeldElsewhereBy() (was remoteLockedBy) for display.
```

The call sites are today's 13 `ClusterLocks.*` sites, mapped as follows:

| Today | `LockCoordinator` |
|---|---|
| `acquire` | `tryLock` |
| `release` | `released` |
| `reserve` | `tryReserve` |
| `reserveForcibly` | `reservedForcibly` |
| `unreserve` | `unreserved` |
| `free` | `freed` |
| `isHeldElsewhere` | `isHeldElsewhere` |
| `publishDefinition` | `definitionChanged` |
| `deleteDefinition` | `definitionDeleted` |

`isActive()` becomes `get() != null`.

### pipeline-build-step: `org.jenkinsci.plugins.workflow.support.steps.build.BuildTriggerRelay` [Beta in the plugin]

```java
package org.jenkinsci.plugins.workflow.support.steps.build;

/** Relays downstream build events to an upstream step on another controller (implemented by the cluster plugin). */
@Restricted(Beta.class)
public interface BuildTriggerRelay extends ExtensionPoint {
    /** At scheduling: an opaque address to reach this upstream step from wherever the downstream runs; null = local only. */
    @CheckForNull String returnAddress(@NonNull Run<?, ?> upstream, @NonNull String nodeId);
    /** On the downstream's controller, for a trigger carrying a return address: deliver the event if the upstream is
     *  not here. true = relayed (skip the local path). */
    boolean relay(@NonNull String returnAddress, boolean waitForStart, @NonNull Run<?, ?> downstream, @NonNull Event event);
    /** The queue item left this controller because another took it (core LeftItem.isTakenElsewhere()). */
    default boolean isTakenElsewhere(@NonNull Queue.LeftItem item) { return false; }
    /** Stop a downstream not running here; false if unknown. */
    default boolean requestStop(@NonNull Job<?, ?> job, int number, @NonNull String by) { return false; }
    /** Whether a queue item must start where it was scheduled (waitForStart). */
    default boolean pinsToOrigin(@NonNull Queue.Item item) { return false; }
    enum Event { STARTED, COMPLETED, DELETED }
}
// Inbound, public static:
// BuildTriggerStepExecution.deliver(@NonNull Run<?, ?> upstream, @NonNull String nodeId, @NonNull BuildTriggerRelay.Event e,
//     @NonNull String downstreamExternalizableId, @CheckForNull Result result, boolean waitForStart)
//   -> completes the step context (STARTED + waitForStart: onSuccess with RunWrapper by id, or a clear failure)
// BuildTriggerAction.Trigger gains: @CheckForNull String returnAddress (replaces upstreamJob/upstreamNumber).
```

## Deviations from the design, with reasons

1. **`BuildRecordSource.retrieve` takes `jenkins.util.ThrowingCallable<R, IOException>`** rather than a new `IOSupplier`. Core already has this type (since 2.534), so no new functional interface is needed.
2. **`PersistenceStores.MaterializedCopy` has `(key, file, content, version, modified)`**, not just (key, file, version). The plugin's write-back needs the content the copy was made from as its merge base, and the modification time it was given to tell whether it changed locally. Rev3's `MaterializedFiles.Copy` held both.
3. **Restriction levels where §1.4 is silent:**
   - `Queue.LOADED` and `Run.isPendingDelete` are [Beta], because the moved engines use them (`SharedQueue` and `StoredPermalinks`). This follows §1.4's "Beta if the plugin needs it".
   - `Queue.isLockHeldByCurrentThread` stays [NEU]: only `Job` used it, and that use goes away.
   - `proposeSharedNumber` and `storedNumbers` are [NEU]. Only `RunMap` calls them, through `BuildRecordSource.sharesNumbers`/`numbers`. §1.1 said NEU→Beta for `proposeSharedNumber`, but §1.4 does not list it.
4. **`Queue.getLocalItem`, `cancelLocal`, `RemoteItem` and `RemoteExecutable` are not in the API.** The design deletes them: `getItem(long)` is local again, and `handOff` replaces `cancelLocal`.
5. **`HandOff.completed` also completes the start condition** if `started` was never called, so that a fast build that finishes before its start is seen still releases `-s` waiters. The calls are once-only and in order, and later calls are ignored.
6. **`LoginRevocation` (decision 5).** The design names only "the logout revocation hook". The shape chosen here is a per-login revocation keyed by a hash of the remember-me cookie. Sessions record their login id. Revoking a login keeps the user's other browsers signed in, which is the preference stated in decision 5.
7. **`PluginManagerNotice` is new.** A6 calls for "a dedicated small notices extension point" but does not give it a shape.
8. **`StoreRouting`:**
   - `route(File)` returns `PersistenceStores.Route` [NEU];
   - `keepLocal` moves from `PersistenceStores` to `StoreRouting`, because it is a routing input;
   - `STATIC_STATE` is the `staticState` property.
9. **`ClusterStore` exposes `leases()`, `queue()` and `launchLeases()` as accessors** instead of extending `BuildLeases`/`ClusterQueue`/`LaunchLeases`. Extending them would clash on method names: `acquire`/`release` exist in both lease facets, and `get`/`subscribe` exist in the queue and in `PersistenceStore`.
10. **Blob changes:**
    - `ClusterBlobStoreProvider` is new, because core's `BlobStoreProvider` leaves with `BlobStore`;
    - `ClusterBlobStore` drops `move`, because blobs never move (§3.2).
11. **Queue records:** `ClusterQueue.Record` gains `itemUuid` as its second component, and `task` stays as a display name. `QueueMaintenance.StartedItem` gains `itemUuid`. These follow §3.1 "item UUID (plus full name for display only)".
12. **`Keys` methods take the item prefix string** (from `item(Item)`), with a `Layout` switch. This makes risk 2's "port with path keys, then switch in one commit" a single constant. In the `PATH` layout, item files keep their old relative layout.
13. **`BuildTriggerRelay.relay(returnAddress, waitForStart, downstream, event)`** instead of `relay(BuildTriggerAction.Trigger, …)`. `Trigger` is package-private, so making it part of a public extension point would mean opening up its `StepContext` field.
14. **`LockCoordinator.get()` and the Javadoc's no-I/O rule on `isHeldElsewhere`** are additions. The rest is as in §4.1.
15. **Not frozen, deliberately:**
    - `LaunchGate`, the optional core hook of §5 item 2, is only to be added if the "launch failed" lines prove too noisy;
    - `PeepholePermalink.changedElsewhere`: §1.2 keeps it only if the plugin cannot do the work with public API, and P1 decides;
    - the internals of `ItemChangeApplier`, `StaticStateApplier`, `EarlyDescriptors` and `Nodes`/`User.changedElsewhere` (B2/B3);
    - the `InMemoryPersistenceStore` test fixture (B1).
    - `TriggerCoordinator` is unchanged. Decision 8 (ready members and missed-tick catch-up) is plugin policy behind the same signature.
16. **Small behaviour changes in the stub branch, both upstream-neutral:**
    - `Queue.init` attains `Queue.LOADED`;
    - `AbstractLazyLoadRunMap.loadIdOnDisk` unions `storedNumbers()`, which is empty by default.
17. **Verification of the plugin side used javac on the `spi` package, not `mvn compile` of the module.** The module cannot compile until the engines are ported, as described under Compile status.

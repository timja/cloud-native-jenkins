# JEP-250: Pluggable persistence store for `JENKINS_HOME` state

| | |
|---|---|
| **JEP** | 250 |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | none |
| **Supersedes** | the configuration-storage parts of JEP-213 (draft, never implemented) |

## Abstract
An SPI for storing what Jenkins writes to `JENKINS_HOME` (item, global, descriptor, user and node configuration, and shared keys) in an external store instead of local files. The store is versioned, shared by several controllers, and publishes changes, which every controller applies to its objects in memory. `XmlFile` routes reads and writes to the store, so existing code that persists through `XmlFile` (most of core and the plugin ecosystem) needs no change. Concurrent writes are detected by version and merged.

## Specification

### SPI (`jenkins.persistence`, `@Restricted(Beta.class)`)
- **`PersistenceStore`** is a key → bytes store.
  - Keys are paths relative to `JENKINS_HOME`, such as `config.xml` or `jobs/f/jobs/p/config.xml`.
  - Methods:
    - `read(key)` → `Entry(content, version)` or null
    - `write(key, bytes, expectedVersion)`: compare-and-set, returning the new version. It throws `ConflictException`. `ANY` and `ABSENT` are special expected versions.
    - `delete(key, expectedVersion)`
    - `children(key)`
    - `move(fromPrefix, toPrefix)`, atomic, for renames
    - `deleteAll(prefix)`: every key below a prefix (a deleted job's build records); by default one `delete` per key
    - `subscribe(Consumer<ChangeEvent>)`, where `ChangeEvent` is `(kind, key, oldKey, version, origin)`
    - `getOrigin()`, which identifies this controller's instance
  - `owns(key)` tells which keys the store holds; the others stay local.
  - Optional facets: `liveOrigins()`, `leases()` (JEP-252), `queue()` (JEP-253) and `storesBuildRecords()` (JEP-251).
- **`BlobStore`** holds large, immutable or append-only data: `put`, `get`, `delete(prefix)` and `move(prefix)`.
- **`PersistenceStoreProvider` / `BlobStoreProvider`** are extension points. `PersistenceStores.get()` resolves the first provider that returns a store, once extensions are augmented. Files read before that come from the local disk, as do the pre-plugin key material and plugin set; see *Reasoning*.
- **`RemoteChangeHandler`** is an extension point through which plugins apply changes to their own keys.
- **`RemoteItemGroup`** is implemented by item groups (folders) that can take an item another controller created in them without reloading the whole group, with its children's builds, triggers and computations.
- **Per-controller state.** `@PerController` marks a class whose saved state belongs to the controller it runs on (caches, statistics, what this controller alone is connected to). `LocalFiles` is an extension point declaring other per-controller paths. Core declares its own (`queue.xml`, `nodeMonitors.xml`, the update center's state, the setup wizard's markers), so the store keeps no list of core's files.
- **`Item.getItemUuid()`** is an identity a rename or move does not change, kept in the item's `config.xml`: a random UUID for new items (a new one for a copy), and for an item saved before, a name-based UUID derived from its full name when it was first loaded. Plugins keeping data outside `JENKINS_HOME` keyed by job name can key it by this instead.

### Routing and change propagation
- **Routing.** `XmlFile.read`, `unmarshal`, `write`, `exists`, `delete` and the raw-read methods route to the store when the store owns the key.
  - Each `XmlFile` remembers the version it last loaded, and a write expects that version.
  - Raw reads (REST `config.xml`, `getFile()` snapshots) only look; they do not move the version.
  - `getFile()` materialises a local copy for code that needs a `File`. A plugin that writes the copy with its own writer has the change written back to the store, merged against the content the copy was made from: at the next `getFile()`, and every minute.
- **`ItemChangeApplier`** subscribes to the store and applies other controllers' changes in memory:
  - items are created, updated, renamed or deleted (folders included)
  - global configuration, descriptors, users and nodes are reloaded (`StaticStateApplier`)
  - each object is reloaded under its own lock, and not while a save of it is pending here
  - an item is reloaded field by field: the stored configuration is read into a scratch instance, and only the fields that changed are copied over, except those changed locally since the item was loaded, so a field set here during a reload is not lost. Core's setters on `Jenkins` and on items assign under the object's lock
  - a change to an item's sub-items (promoted-builds' `promotions/*/config.xml`, matrix-project's `configurations/**/config.xml`) reloads the item; their directories are created when the item loads, since their owners find them by listing
  - a change that fails to apply is retried with backoff
- **Concurrent writes.**
  - A compare-and-set conflict is resolved by a three-way XML merge (`XmlMerge`): base = what this controller last loaded, theirs = the store, mine = this save. The merge recurses into distinctly named parts. A part changed on both sides keeps this controller's value, and the conflict is counted.
  - The object is then reloaded with both changes.
  - Objects are snapshotted on save and on reload, so a pending save is never overwritten by a reload.
  - A merge whose result is what is stored already writes nothing; the object then keeps the version and base it read before, so its next save merges again rather than overwriting.
  - **Policy** (`PersistenceStores.conflicts`): `warn` (the default) merges as above; `fail` refuses every conflicting save; `strict-ui` refuses only a save made by a web request (a form, a `config.xml` upload) that changed the same part as the change made meanwhile, or cannot be merged, so the user reloads instead of silently overwriting someone else's change. An administrative monitor shows the conflict count.
- **Keys created at runtime** (`secrets/*`) are shared. The first controller to create one decides its value, through a create-only write. Each controller keeps a local copy of every shared key from startup.
- **No-op writes** (content unchanged) create no version and no event.
- **The `DoubleLaunchChecker`** is disabled while a store is in use.

### What stays local
Workspaces, caches, `plugins/`, `war/`, logs of Jenkins itself, per-controller files such as API token statistics, and the pre-plugin keys (`secret.key`, `secrets/master.key`, `identity.key.enc`). The pre-plugin keys are seeded identically on every controller; the chart does this from a Kubernetes Secret.

`/manage/cluster/compatibility` lists everything a controller wrote under its home that the store does not hold, so that plugins writing state outside `XmlFile` can be found.

**Plugin files in item and user directories.** Plugins that keep files next to an item's or a user's `config.xml` with their own file APIs (the avatar plugin keeps the user's image there) can have them shared, opt-in (`-Djenkins.persistence.ItemFiles.enabled=true`; the chart's `store.sharePluginFiles`): every minute, each item's and user's directory is compared with what was last shared, and changed or removed files (up to 1 MiB each) are shared through a manifest per directory and blob storage. Files core declares per controller (such as each user's `apiTokenStats.xml`) are left out. The last writer of a file wins, and a plugin caching a file's content sees a change made elsewhere when it next reads the file. It is opt-in because it scans every item and user directory and the files' semantics are unknown to core.

## Motivation
- The pluggable-storage effort (JEP-202, 207, 210, 212, 213, 226) externalised artifacts, Pipeline logs, fingerprints, test results and credential values. Configuration was never externalised, because `File` and `XmlFile` use runs right through core.
- Without shared configuration there can be no second controller. An experiment ran two controllers on one shared home, as Jenkins is today. It gave duplicate cron builds, colliding build numbers (avoided only by a racy directory check), lost configuration updates, and new jobs invisible to the other controller.
- A second experiment, with a multibranch Pipeline, showed that job configuration is runtime state: branch indexing creates and rewrites jobs, and `properties()` rewrites a job's own `config.xml`. So configuration-as-code alone cannot keep controllers consistent.

## Reasoning
- **Routing at `XmlFile`** gives the widest coverage for the least change. A bytecode scan of every plugin in the update center found 42 that call `XmlFile#getFile`; they get read-only snapshots, and their direct writes are reported by the compatibility page.
- **A plugin, not a jar on the Winstone class path.** This was the original plan. An experiment found that such a jar has to go into the war's `WEB-INF/lib`, because Winstone's common library folder cannot see core's classes. The plan was dropped because nothing needs the store before plugins load except the pre-plugin keys and the plugin set, and those are seeded or baked into the image.
- **Versions plus merge, not locks.** Writers on different controllers rarely touch the same part of the same file. Merging keeps both changes in the common case. In an acceptance test, two controllers saved different global settings at the same moment, 30 times, and no change was lost.
- **The reference backend is Postgres**: a single table with CAS through `UPDATE … WHERE version = ?`, and an event log table plus `LISTEN/NOTIFY`. Its operational behaviour is covered in JEP-255.

## Backwards compatibility
- Without a provider, nothing changes: `XmlFile` writes files as before. Every new API is `@Restricted(Beta.class)` or `NoExternalUse`.
- With a store, plugins that persist through `XmlFile` (descriptors, `Saveable`s) are shared without changes.
- Plugins that write files directly under the home keep that state per controller. The compatibility report lists them.
- Plugin setters that assign a field without the object's lock, during the milliseconds of a field-by-field reload of that field, can still lose the assignment; core's own setters take the lock.
- `Item.getItemUuid()` is a new default method returning null; `AbstractItem` implements it.

## Security
- Encryption at rest is the backend's job; envelope or KMS encryption is a follow-up. The store holds the same data as the home, so access to the store is equivalent to access to `JENKINS_HOME`.
- Secrets stay encrypted with the shared master key, exactly as on disk.
- A security review of the cross-controller interfaces is pending, through a separate process.

## Testing
- **Core:** `XmlFileTest`, `XmlMergeTest`, `ItemChangeApplierTest`, `StaticStateApplierTest`, `PersistenceStoreItem*Test`, `StaticStateRoutingTest`, `ItemReloadTest` (two saves here while a change made elsewhere is not applied yet, deterministic), `LocalStateTest`, `ItemUuidTest`, `StrictConflictsTest` and `ItemFilesTest`, using the in-memory store.
- **cloudbees-folder:** `RemoteChildTest`.
- **Store plugin:** `PostgresPersistenceStoreTest` runs against Postgres in Testcontainers.
- **Acceptance** (scripts in `validation/experiments/` of the cloud-native-jenkins repository; each runs two controllers with empty homes against one store):
  - `ET1-shared-items.sh`: jobs and folders created, edited, renamed and deleted on either controller, including multibranch jobs and `properties()`, are the same on both, and a fresh controller loads them all
  - `ET6-static-state.sh`: global configuration, credentials, users, nodes and keys changed on one controller apply on the other without a restart
  - `ET11-cross-cutting.sh`: simultaneous edits on two controllers are merged, and nothing a controller writes under its home is left unshared
  - `ET22-avatar.sh`: an avatar (avatar plugin) set on one controller is served by the other, by a fresh controller, and changes made on either are shared

## Prototype implementation
- **Core:** prototype branches of Jenkins core, `cnj/t1-*`, `cnj/t6-*`, `cnj/t11-m1-xml-merge` and `cnj/t15-remaining`.
- **cloudbees-folder:** prototype branches `cnj/t1-m2-folder-children`, `cnj/t15-remote-child`.
- **Store plugin:** `impl/persistence-store-postgresql` in the cloud-native-jenkins repository, on its branches `cnj/t1-*` to `cnj/t11-m2-cross-cutting`.
- **Findings:** documents in the cloud-native-jenkins repository, on those branches: `findings/ET1-shared-items.md`, `ET6-static-state.md` and `ET11-cross-cutting.md`.

## References
- [Pluggable storage](https://www.jenkins.io/doc/book/using/pluggable-storage/), and JEP-202, 207, 210, 212, 213, 226
- `ROADMAP.md` in the cloud-native-jenkins repository: the sections on item configuration changed at runtime and on static state in the store

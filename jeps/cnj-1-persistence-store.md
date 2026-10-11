# JEP-CNJ-1: Pluggable persistence store for `JENKINS_HOME` state

| | |
|---|---|
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
    - `subscribe(Consumer<ChangeEvent>)`, where `ChangeEvent` is `(kind, key, oldKey, version, origin)`
    - `getOrigin()`, which identifies this controller's instance
  - `owns(key)` tells which keys the store holds; the others stay local.
  - Optional facets: `liveOrigins()`, `leases()` (JEP-CNJ-3), `queue()` (JEP-CNJ-4) and `storesBuildRecords()` (JEP-CNJ-2).
- **`BlobStore`** holds large, immutable or append-only data: `put`, `get`, `delete(prefix)` and `move(prefix)`.
- **`PersistenceStoreProvider` / `BlobStoreProvider`** are extension points. `PersistenceStores.get()` resolves the first provider that returns a store, once extensions are augmented. Files read before that come from the local disk, as do the pre-plugin key material and plugin set; see *Reasoning*.
- **`RemoteChangeHandler`** is an extension point through which plugins apply changes to their own keys.

### Routing and change propagation
- **Routing.** `XmlFile.read`, `unmarshal`, `write`, `exists`, `delete` and the raw-read methods route to the store when the store owns the key.
  - Each `XmlFile` remembers the version it last loaded, and a write expects that version.
  - Raw reads (REST `config.xml`, `getFile()` snapshots) only look; they do not move the version.
  - `getFile()` materialises a read-only local snapshot for code that needs a `File`.
- **`ItemChangeApplier`** subscribes to the store and applies other controllers' changes in memory:
  - items are created, updated, renamed or deleted (folders included)
  - global configuration, descriptors, users and nodes are reloaded (`StaticStateApplier`)
  - each object is reloaded under its own lock, and not while a save of it is pending here
  - a change that fails to apply is retried with backoff
- **Concurrent writes.**
  - A compare-and-set conflict is resolved by a three-way XML merge (`XmlMerge`): base = what this controller last loaded, theirs = the store, mine = this save. The merge recurses into distinctly named parts. A part changed on both sides keeps this controller's value, and the conflict is counted.
  - The object is then reloaded with both changes.
  - Objects are snapshotted on save and on reload, so a pending save is never overwritten by a reload.
- **Keys created at runtime** (`secrets/*`) are shared. The first controller to create one decides its value, through a create-only write. Each controller keeps a local copy of every shared key from startup.
- **No-op writes** (content unchanged) create no version and no event.
- **The `DoubleLaunchChecker`** is disabled while a store is in use.

### What stays local
Workspaces, caches, `plugins/`, `war/`, logs of Jenkins itself, per-controller files such as API token statistics, and the pre-plugin keys (`secret.key`, `secrets/master.key`, `identity.key.enc`). The pre-plugin keys are seeded identically on every controller; the chart does this from a Kubernetes Secret.

`/manage/cluster/compatibility` lists everything a controller wrote under its home that the store does not hold, so that plugins writing state outside `XmlFile` can be found.

## Motivation
- The pluggable-storage effort (JEP-202, 207, 210, 212, 213, 226) externalised artifacts, Pipeline logs, fingerprints, test results and credential values. Configuration was never externalised, because `File` and `XmlFile` use runs right through core.
- Without shared configuration there can be no second controller. Experiment E3 showed what a shared home does today: duplicate cron builds, colliding build numbers, lost configuration updates, and new jobs invisible to the other controller.
- E9 showed that job configuration is runtime state: branch indexing creates and rewrites jobs, and `properties()` rewrites a job's own `config.xml`. So configuration-as-code alone cannot keep controllers consistent.

## Reasoning
- **Routing at `XmlFile`** gives the widest coverage for the least change. E7 counted 42 plugins that call `XmlFile#getFile`; they get read-only snapshots, and their direct writes are reported by the compatibility page.
- **A plugin, not a jar on the Winstone class path.** This was the original plan (E6 validated the class-loader seam). It was dropped because nothing needs the store before plugins load except the pre-plugin keys and the plugin set, and those are seeded or baked into the image.
- **Versions plus merge, not locks.** Writers on different controllers rarely touch the same part of the same file. Merging keeps both changes in the common case. ET11 ran 30 rounds of deliberately simultaneous edits and lost none.
- **The reference backend is Postgres**: a single table with CAS through `UPDATE … WHERE version = ?`, and an event log table plus `LISTEN/NOTIFY`. Its operational behaviour is covered in JEP-CNJ-6.

## Backwards compatibility
- Without a provider, nothing changes: `XmlFile` writes files as before. Every new API is `@Restricted(Beta.class)` or `NoExternalUse`.
- With a store, plugins that persist through `XmlFile` (descriptors, `Saveable`s) are shared without changes.
- Plugins that write files directly under the home keep that state per controller. The compatibility report lists them.
- **Residual risk:** core setters assign fields without a lock and then save. A field assigned during a reload (milliseconds) can still be lost. This was not observed in 30 rounds; closing it fully needs setters that save under the object's lock.

## Security
- Encryption at rest is the backend's job; envelope or KMS encryption is a follow-up. The store holds the same data as the home, so access to the store is equivalent to access to `JENKINS_HOME`.
- Secrets stay encrypted with the shared master key, exactly as on disk.
- A security review of the cross-controller interfaces is pending, through a separate process.

## Testing
- **Core:** `XmlFileTest`, `XmlMergeTest`, `ItemChangeApplierTest`, `StaticStateApplierTest`, `PersistenceStoreItem*Test` and `StaticStateRoutingTest`, using the in-memory store.
- **Store plugin:** `PostgresPersistenceStoreTest` runs against Postgres in Testcontainers.
- **Acceptance:** ET1 (shared item configuration, 2026-10-10), ET6 (static state: global configuration, credentials, users, nodes, keys) and ET11 (concurrent edits merged, inventory check).

## Prototype implementation
- **Core:** `cnj/t1-*`, `cnj/t6-*` and `cnj/t11-m1-xml-merge`.
- **Store plugin:** `cnj/t1-*` to `cnj/t11-m2-cross-cutting` in this repository.
- **Findings:** `findings/ET1-shared-items.md`, `ET6-static-state.md` and `ET11-cross-cutting.md` on those branches.

## References
- [Pluggable storage](https://www.jenkins.io/doc/book/using/pluggable-storage/), and JEP-202, 207, 210, 212, 213, 226
- This repository's `ROADMAP.md`, R1 and Track S

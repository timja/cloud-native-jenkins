# Target 1: Shared item configuration (jobs created and changed at runtime)

| | |
|---|---|
| **Roadmap tier** | R1 |
| **Status** | Planned |
| **Depends on** | S0 workaround (identical JCasC + seeded key set, [E8](../findings/E8-casc-secret-sync.md)) |
| **Unblocks** | R2 (a single scheduler needs one view of jobs and their triggers), R5 (build records hang off items), Track S (reuses the SPI) |
| **Evidence** | [E3](../findings/E3-shared-home.md), [E6](../findings/E6-bootstrap-seam.md), [E7](../findings/E7-ecosystem-scan.md), [E9](../findings/E9-runtime-job-state.md) |

## Goal
N controllers, each with its **own local `JENKINS_HOME`** (static state via S0), share **item configuration** through an external store:
- folders, jobs, multibranch projects and their branch jobs
- organisation folders

Whoever creates, changes, renames or deletes an item — an admin, branch indexing, `properties()` in a running build, a Job DSL seed — every controller reflects it within seconds and nothing is silently lost. A controller started with an empty home loads every item from the store.

**Out of scope:**
- Build history, `nextBuildNumber` and permalinks stay local and per controller (R5).
- Triggers and indexing still run on every controller (R2). E-T1 runs with triggers disabled on all but one controller.
- Global config and secrets come from S0 (Track S later).

## Why first
- **Item config is runtime state and the root of everything else.**
  - E9: indexing creates, rewrites and deletes branch jobs, and a plain build rewrote `config.xml`, including a `TimerTrigger`.
  - E3: an item created on A doesn't exist on B.
  - The scheduler (R2) can't run a trigger it can't see, and build records (R5) can't attach to an item that only exists on one controller.
- **It forces the reusable primitive:** the persistence SPI, with CAS, listing and change events.
- **It can use hooks that already exist:**
  - `AbstractItem.load()` re-reads config (`hudson/model/AbstractItem.java:972`).
  - `Jenkins.putItem` (`jenkins/model/Jenkins.java:3187`).
  - `AbstractFolder.addLoadedChild` / `reloadThis` (Beta, added for multi-controller use, cloudbees-folder `7b6cbb3`, 2024-02).

## Design

### 1. Persistence SPI: `jenkins.persistence` (new core package, `@Restricted(Beta)`)
```java
public interface PersistenceStore {
    boolean owns(String key);                                  // hybrid routing; keys are JENKINS_HOME-relative POSIX paths
    Optional<Entry> read(String key) throws IOException;       // Entry{byte[] content, long version}
    long write(String key, byte[] content, long expectedVersion) throws IOException, ConflictException;
    //  ANY (-1) = last writer wins; ABSENT (0) = create only; n = compare-and-set
    //  identical content (sha256) is a no-op returning the current version  <- E9 write amplification
    void apply(List<Op> ops, String origin) throws IOException, ConflictException;   // atomic multi-key: rename/move/delete subtree
    List<String> list(String prefix) throws IOException;
    AutoCloseable subscribe(String prefix, Consumer<ChangeEvent> listener);          // ChangeEvent{key, version, kind, origin}
}
```
- **`FilesystemPersistenceStore`** is the default. It owns everything, maps keys 1:1 to files with today's `AtomicFileWriter` semantics, and has no events, so behaviour is unchanged when nothing is configured.
- **`RoutingPersistenceStore`** sends a key to the external store when `external.owns(key)`, and to the filesystem otherwise.
- **Version tracking lives in the store client, keyed by key.** `getConfigFile()` creates a new `XmlFile` per call (`Items.getConfigFile`, `Descriptor.getConfigFile`), so per-instance state would be lost between load and save.
- **Delivery.** An implementation may ship as a **plugin** for R1. E1 shows items are loaded at `Loaded all jobs`, after `Started all plugins` and `Augmented all extensions`. The store is resolved lazily, on first owned-key access after `EXTENSIONS_AUGMENTED`. Track S needs keys read before plugins load, so it will need the `WEB-INF/lib` route (E6). The SPI supports both.

### 2. Owned keys in R1
- Every `config.xml` of an item at any depth. The pattern is `jobs/<name>(/(jobs|branches)/<name>)*/config.xml` (folders use `jobs/`, multibranch uses `branches/`, `MultiBranchProject.java:869`).
- Nothing under any `builds/` segment.
- Not `nextBuildNumber`, `state.xml`, `indexing/**` or `scm-*-revision-hash.xml`. Those are R2/R5 state (E9 finding 4).

### 3. Core changes
| Area | Today | Change |
|---|---|---|
| `XmlFile` read/write/exists/delete | `java.io.File` (`hudson/XmlFile.java:160-260`) | route owned keys through `PersistenceStores.get()`; `getFile()` materialises a read-only cache copy and logs the caller once |
| Top-level item enumeration | `new File(root, "jobs").listFiles()` (`Jenkins.java:3432-3439`) | `ItemStorage.listChildren(parentKey)`: names from `store.list("jobs/")` when owned, else a directory listing. Create the local item dir lazily (builds and workspaces still live there) |
| Folder / multibranch enumeration | `modulesDir.listFiles(File::isDirectory)` (cloudbees-folder `AbstractFolder.java:338-347`) | same helper (**cloudbees-folder PR**) |
| Create / save | `AbstractItem.save` → `getConfigFile().write` | routed automatically |
| Rename / move | directory move (`AbstractItem.renameTo`, `Items.move` at `Items.java:610`) | also `store.apply([move subtree of owned keys])`, atomically, then the local directory move |
| Delete | `Util.deleteRecursive(getRootDir())` | also `store.apply([delete subtree])`. Remote controllers drop the item from memory and delete their local item dir: today's semantics, including history loss on branch pruning (E9 finding 5; revisit in R5) |

### 4. Applying remote changes: `ItemChangeApplier` (core)
- Subscribes to `jobs/`. Events whose `origin` is this JVM are dropped. Each key keeps its last applied version; stale events are ignored.
- Everything runs as `ACL.SYSTEM2`.
- **Event → action:**
  - **created** → resolve the parent `ItemGroup` from the key path, `Items.load(parent, localDir)`, then `Jenkins.putItem` or `AbstractFolder.addLoadedChild`, then `ItemListener.fireOnCreated`.
  - **updated** → `item.load()`, which re-unmarshals from the store and calls `onLoad` (`AbstractItem.java:972`), then `ItemListener.fireOnUpdated`.
    - Not `updateByXml`: it checks `CONFIGURE` and writes back (`AbstractItem.java:929-954`).
    - Folders use `AbstractFolder.reloadThis` so their children aren't reloaded.
  - **moved** → `onRenamed` / `Items.move` semantics without the store write.
  - **deleted** → remove from the parent map (`Jenkins.onDeleted` `:3231`, `AbstractFolder.onDeleted` `:983`), then `ItemListener.fireOnDeleted`.
- **Ordering.** A child's event can arrive before its parent's. Park it, and retry when the parent appears.
- **Multibranch.** Children of a `ComputedFolder` normally come only from computation. Remote-created branch jobs go in through `addLoadedChild`. The next local computation must treat them as existing; this needs a **branch-api / cloudbees-folder** check, covered by E-T1 scenario 3.

### 5. Concurrent writers
E9 shows real concurrent writers on the same key: indexing rewrites every branch config while builds run `properties()`.
- **Phase 1** (`-Djenkins.persistence.conflicts=warn`):
  1. CAS against the version this JVM last saw.
  2. On conflict, log a WARNING, count it in a metric, and write last-writer-wins.
  3. The no-op hash check removes the common case where indexing rewrites unchanged config.
- **Phase 2:** writer-aware retries for the two known runtime writers:
  - `JobPropertyStep` (workflow-multibranch): reload, re-apply properties, retry.
  - `MultiBranchProject` computation: recompute is idempotent, so retry.
  - Admin saves through the UI or REST get `fail`, i.e. a visible error.

### 6. Reference backend: `impl/persistence-store-postgresql` (this repo)
- Table `jenkins_store(key text pk, content bytea, sha256 bytea, version bigint, updated_at timestamptz, updated_by text)`.
- CAS via `UPDATE … WHERE version = ?`.
- `apply()` runs in one transaction. Subtree moves use `UPDATE … SET key = overlay(...) WHERE key LIKE 'prefix/%'`.
- `LISTEN/NOTIFY` for events, with a catch-up scan on reconnect.
- An `InMemoryPersistenceStore` for core tests and `RealJenkinsRule` multi-JVM tests.

## Milestones
| # | Deliverable | Upstreamable alone? | Validation |
|---|---|---|---|
| M1 | SPI, `FilesystemPersistenceStore`, `XmlFile` routing, client-side versions, no-op writes | yes (Beta) | full core test suite unchanged; E1/E9 inventories identical with no store configured |
| M2 | `ItemStorage.listChildren` in `Jenkins.loadTasks`; cloudbees-folder `loadChildren` uses it | core + folder PRs | `InMemoryPersistenceStore`: a fresh home loads top-level, folder and multibranch items from the store |
| M3 | create / rename / move / delete mapped to store ops | yes | `RealJenkinsRule` ×2 sharing a store: rename a folder with children on A, B sees the new paths, no orphans |
| M4 | `ItemChangeApplier` (create / update / move / delete, ordering, origin suppression) | yes | two JVMs: job created on A appears on B < 2 s (fixes E3 row 1); `properties()` build on A → B shows parameters and trigger |
| M5 | conflict policy phase 1 + metrics; writer-aware retries (phase 2) | core yes; workflow-multibranch / branch-api PRs | concurrent `properties()` build on B during indexing on A: no lost properties, conflicts counted |
| M6 | `impl/persistence-store-postgresql` as a plugin, harness wiring | n/a | **E-T1** |

## E-T1 acceptance experiment
`validation/experiments/ET1-shared-items.sh`, written in M6.

**Setup:**
- Two controllers, **separate tmpfs homes**, S0 (identical JCasC plus the E8 variant-S key seed), Postgres store plugin.
- The core war is built from the Target 1 branch (`JENKINS_WAR_DIR`).
- Triggers are disabled on B (R2 isn't done).

**Scenarios:**
1. **Admin item lifecycle:**
   1. Create a freestyle job and a folder with a job inside on A → both appear on B.
   2. Edit the job on B → the change appears on A.
   3. Rename the folder on A → B shows the new path.
   4. Delete on B → gone on A.
2. **Runtime config:** build a `properties()` pipeline on A → B's copy of the job shows the parameter, the build discarder and the `TimerTrigger`. Re-run E9's inventory: no `config.xml` writes under the local `jobs/`.
3. **Multibranch:**
   1. Index on A → `main` and `feature` appear on B.
   2. Add a branch and re-index on A → it appears on B.
   3. Delete a branch → it is pruned on B.
   4. Then index on B: no duplicate items, no spurious deletes.
4. **Concurrency:** index on A while builds with `properties()` run on B for 5 minutes. Check that the final configs match the Jenkinsfiles and that the conflict metric is recorded.
5. **Disposable controller:** destroy B and its home, and start a fresh B. All items load from the store, with no admin action.
6. **Write amplification:** re-index with no changes produces **0** row version bumps.

## Risks and open questions
- **Plugins touching item files directly.** E7 counts 88 plugins using item-dir `File` APIs and 42 calling `XmlFile#getFile`. Reads work through the materialised cache; direct writes are lost. Use the E7 hit list to prioritise, starting with the BOM plugins that touch item dirs: job-dsl, jobConfigHistory, promoted-builds.
- **`onLoad` side effects on remote controllers.** A reload may restart a trigger, re-register a webhook, or re-create a matrix configuration. With triggers active everywhere (R2 not done), this can duplicate work. That is why E-T1 disables triggers on B.
- **Item deletion destroys local history** on every controller (E9 finding 5). It is acceptable for R1 because it matches today's semantics, but R5 must decide whether deleted branches keep their history.
- **Ordering and partitions.** Events are at-least-once with a catch-up scan; correctness comes from per-key versions, not from delivery order.
- **Upstream path.** Develop M1–M5 on a core fork plus plugin forks until E-T1 passes, then propose a JEP (superseding JEP-213 for this scope) with the E-series evidence.

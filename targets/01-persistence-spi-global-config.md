# Target 1: Persistence SPI + externalised secrets and global configuration

| | |
|---|---|
| **Roadmap tier** | T1 (foundation) |
| **Status** | Planned |
| **Depends on** | T0 harness (done) |
| **Unblocks** | T2 (items, nodes, users, credentials), T3 fingerprints and build records, T5 change events |
| **Evidence** | [E1](../findings/E1-home-inventory.md), [E2](../findings/E2-disposable-controller.md), [E3](../findings/E3-shared-home.md), [E5](../findings/E5-stateless-http.md), [E6](../findings/E6-bootstrap-seam.md) |

## Goal
Two controllers with **empty, ephemeral `JENKINS_HOME`s** (tmpfs) share one external store and agree on:
- the cryptographic identity: `secrets/*`, `secret.key`, the instance identity key
- the global configuration: Jenkins `config.xml` plus every root-level descriptor/`GlobalConfiguration` XML

A change saved on one controller shows up on the other without a restart. Concurrent saves are detected, not silently lost. Everything else (items, builds, users, nodes) stays on the local filesystem for now. That is deliberate: the SPI must support a **hybrid** mode, because T2/T3 move state over incrementally.

## Why this first
- **Every later tier needs the same primitive:** a key → bytes store with versions, prefix listing and change events, selected before Jenkins loads anything.
- **Secrets gate everything else.** E2 showed that UI-entered `Secret`s and API tokens don't survive a new controller, and E5 showed agent secrets, crumbs and tokens all derive from `secrets/`. No other state can be shared until the key material is.
- **Global config is the smallest real state with real concurrency bugs.** E3 reproduced a lost update on `config.xml` between two controllers. It is also read late in boot (`System config loaded`, E1), so it is a gentle first consumer.
- **This is where JEP-213 stalled.** It tried to make every `XmlFile` pluggable at once. Target 1 limits the scope with an explicit set of routed keys.

## Design

### 1. SPI: `jenkins.persistence` (new package in core, `@Restricted(Beta)`)
```java
public interface PersistenceStore {
    /** Keys are JENKINS_HOME-relative POSIX paths, e.g. "config.xml", "secrets/master.key". */
    boolean owns(String key);                                   // hybrid mode: which keys this store serves
    Optional<Entry> read(String key) throws IOException;        // Entry{byte[] content, long version}
    long write(String key, byte[] content, long expectedVersion) throws IOException, ConflictException;
    //  expectedVersion: ANY (-1) = last-writer-wins; ABSENT (0) = create only; n = compare-and-set
    //  writing identical bytes is a no-op that returns the current version (E1 finding 2: boot write storms)
    void delete(String key, long expectedVersion) throws IOException, ConflictException;
    List<String> list(String prefix) throws IOException;
    AutoCloseable subscribe(String prefix, Consumer<ChangeEvent> listener);   // ChangeEvent{key, version, origin}
}
```
- **`FilesystemPersistenceStore`** is the default. It owns every key, maps them 1:1 to files, and keeps today's `AtomicFileWriter` semantics. Versions are file mtime+size hashes and `subscribe` is a no-op, so no migration is needed and behaviour is unchanged when nothing is configured.
- **`PersistenceStores.get()`** resolves the active store once per JVM.
- **Hybrid routing.**
  - A `RoutingPersistenceStore` sends a key to the external store when `external.owns(key)`, and to the filesystem otherwise.
  - The Target 1 set of owned keys is:
    - `config.xml`
    - `*.xml` at the root, excluding `queue.xml*`, `nodeMonitors.xml`, `hudson.model.UpdateCenter.xml` and the `FlowExecutionList` (those belong to T4/T5 or are ephemeral)
    - `secrets/**`
    - `secret.key*`
    - `identity.key.enc`
  - Later targets widen the set; this is how T2/T3 land without a big-bang switch.

### 2. Bootstrap (driven by E1 + E6)
- **Selection.** `-Djenkins.persistence.store=<fqcn>` selects the store, falling back to `ServiceLoader` on core's class loader. Resolution happens in the `Jenkins` constructor, before `secret.key` is read (`Jenkins.java:932`).
- **Packaging.**
  - Implementations must be on the **webapp class path**: a jar layered into `WEB-INF/lib`, through a custom war or a Dockerfile layer (E6 variant b).
  - Winstone `--commonLibFolder` cannot work, because its class loader is the parent of the webapp loader (E6 variant a).
  - Plugins are too late for secrets (E1).
  - **Follow-up:** propose a supported `--extraWebappLib`-style option to Winstone so images don't have to repack the war.
- **Configuration** of the store itself comes only from env/system properties (JDBC URL, credentials file). It can't live in the store.

### 3. `XmlFile` routing (`core/src/main/java/hudson/XmlFile.java`)
- `read`, `unmarshal*`, `write`, `exists`, `delete`, `readRaw`, `asString`, `writeRawTo` and `sniffEncoding` resolve `file` to a key when it lies under `Jenkins.getRootDir()`, and delegate to `PersistenceStores.get()`. Files outside the root (tests, plugin-private paths) are untouched.
- **Version tracking lives in the store client, keyed by key, not on the `XmlFile` instance.** `Descriptor.getConfigFile()` (`Descriptor.java:994`) and `Jenkins.getConfigFile()` (`Jenkins.java:3280`) build a fresh `XmlFile` on every call, so per-instance state is lost between `load()` and `save()`. The client records the version last read or written for each key in this JVM, and passes it as `expectedVersion`.
- **Conflict policy:**
  - `-Djenkins.persistence.conflicts=warn` (default in phase 1): write anyway, log a WARNING with both versions, and raise an `AdministrativeMonitor`.
  - `=fail`: throw `IOException(ConflictException)`, surfaced to the user as a save error.
  - The E3 lost-update scenario is the acceptance test for `fail`.
- **`getFile()`** is deprecated. With a non-filesystem owner for that key, it materialises a read-only copy under `$JENKINS_HOME/.persistence-cache/`, logs the caller once, and counts it in a telemetry/diagnostics page. E7 sizes how many plugins hit this.
- `TextFile` and `FileBoolean` for `secret.key` / `secret.key.not-so-secret` (`Jenkins.java:932-940`) are switched to the store client.

### 4. Secrets (driven by E2, E5 and E6)
- **Make `ConfidentialStore` implementable outside its package.** `secureRandom()` becomes `protected` with a default `SecureRandom` (the TODO at `ConfidentialStore.java:60`). This is a small, independently mergeable core PR.
- **`StoreBackedConfidentialStore`** (core) persists `ConfidentialKey`s under `secrets/<id>` in the store, encrypted with `master.key`, as `DefaultConfidentialStore` does today. `master.key` itself comes from the store, or from a KEK supplied by env/file (a k8s Secret) for envelope encryption. It is used automatically when the routed store owns `secrets/**`.
- **instance-identity:** add a `ConfidentialStore`-based load path so `identity.key.enc` stops being a separate file. This is a plugin PR to `instance-identity-module`. Until it lands, `identity.key.enc` is routed as an owned key.

### 5. Change propagation (driven by E3 and E5)
- A core **`PersistenceChangeDispatcher`** subscribes to the owned prefixes. Events whose `origin` is this JVM's write are dropped.
- **Key → action:**
  - `config.xml` → `Jenkins.reloadGlobalConfig()`, a new method that re-unmarshals `config.xml` into the live `Jenkins` under its monitor and re-applies the security realm, authorisation strategy and views. It is *not* the stop-the-world `Jenkins.reload()` (`Jenkins.java:4396`).
  - `<descriptorId>.xml` → `Descriptor.load()` on the matching `Descriptor`/`GlobalConfiguration`.
  - `secrets/*` → evict the `ConfidentialKey` caches.
- Reloads fire `SaveableListener.fireOnChange` with a "remote" flag, so listeners such as CasC export or config-history can tell them apart.
- **Ordering:** each key keeps its last applied version, and stale events are ignored.

### 6. `DoubleLaunchChecker`
It is disabled when `secrets/**` is owned by a non-filesystem store; it is meaningless there (E3: it never fired anyway).

### 7. Reference backend: `impl/persistence-store-postgresql` (this repo; plain jar, no plugin)
```sql
create table jenkins_store (
  key        text primary key,
  content    bytea not null,
  sha256     bytea not null,
  version    bigint not null,
  updated_at timestamptz not null default now(),
  updated_by text not null              -- controller instance id, used as event origin
);
-- trigger: pg_notify('jenkins_store', json_build_object('key',key,'version',version,'origin',updated_by)::text)
```
- **CAS:** `update … set content=?, sha256=?, version=version+1 where key=? and version=?`. A row count of 0 means a conflict.
- **Unchanged-content no-op:** writes with an identical `sha256` return early.
- **Events:** `LISTEN jenkins_store` on a dedicated connection. On reconnect, run a catch-up scan for `version > last seen`.
- **Testing:** an `InMemoryPersistenceStore` ships in `jenkins-test-harness` (or core test scope) for unit tests and `RealJenkinsRule` multi-JVM tests, which can share it over a loopback.

## Milestones
| # | Deliverable | Mergeable upstream on its own? | Validation |
|---|---|---|---|
| M0 | `ConfidentialStore.secureRandom()` → `protected` + default; `secret.key` read and written via a small seam | yes | existing `ConfidentialStoreTest`; E6 re-run with the spike moved out of `jenkins.security` |
| M1 | `jenkins.persistence` SPI, `FilesystemPersistenceStore`, `XmlFile` delegation (no behaviour change) | yes (Beta) | full core test suite; E1 inventory identical |
| M2 | `InMemoryPersistenceStore`, routing, client-side version tracking, conflict policy | yes | unit tests for CAS, no-op writes, routing; `RealJenkinsRule` with two JVMs sharing one store: system message set on A is read on B after restart |
| M3 | `StoreBackedConfidentialStore`, routed `secret.key`, instance-identity path | core yes; plugin PR separate | two JVMs: a `Secret` saved on A decrypts on B; agent secret identical |
| M4 | `PersistenceChangeDispatcher`, `Jenkins.reloadGlobalConfig()`, descriptor reload | yes | two JVMs: change visible on B in < 2 s without restart; E3 lost-update test → conflict raised under `fail` |
| M5 | `impl/persistence-store-postgresql` plus a harness image layer that injects it into `WEB-INF/lib` | n/a (this repo) | **E-T1** below |
| M6 | Write up as a JEP (supersedes JEP-213) with the E-series evidence | n/a | community review |

## E-T1 acceptance experiment (harness)
`validation/experiments/ET1-global-config.sh`, to be written in M5:
1. Build the core war from the Target 1 branch, and layer the Postgres store jar into `WEB-INF/lib` (`JENKINS_WAR_DIR`).
2. Start `jenkins-a` and `jenkins-b` with **tmpfs** homes, both pointed at Postgres via `-Djenkins.persistence.store`.
3. Pass criteria:
   - The system message, a UI-entered `Secret` (not JCasC) and a global tool config saved on A appear on B within 2 s, and the secret decrypts on B.
   - Destroy both containers and start fresh ones. All of the above survive (fixes the E2 "UI-entered secrets lost" row).
   - E3 section 3 (A saves `systemMessage`, B saves `numExecutors`): both changes survive under `warn` with an admin monitor shown, and B's save is rejected under `fail`.
   - The E1 inventory shows **no** writes to `config.xml`, root `*.xml` in scope, `secrets/`, `secret.key*` or `identity.key.enc` in `JENKINS_HOME`.
   - The agent secret matches on A and B (E5 section 4) without a shared volume.
   - A restart of A causes **no** row version bumps for unchanged config (boot write-storm check, E1 finding 2).

## Risks and open questions
- **`getFile()` consumers.** Plugins that read or write their descriptor file directly bypass versioning and events. The materialised cache keeps reads working, but direct writes are lost. E7 sizes this. Mitigation: a diagnostics page, then targeted PRs.
- **Reloading `Jenkins` in place is risky.** Fields marked `transient` and objects held by reference (views, the `SecurityRealm` filter chain) need care. Start with a hand-written field-by-field reload covering `systemMessage`, `numExecutors`, `mode`, `labelString`, `securityRealm`, `authorizationStrategy` and `views`, and test each.
- **CasC re-apply on every boot** (E1 finding 2). With N replicas, every boot re-saves global config. No-op writes on identical content handle the common case. CasC templates that differ per replica (as here: `${CNJ_NAME}` in `systemMessage`) are a real conflict and must be flagged.
- **Upstream strategy.** M0–M4 are designed as upstreamable Beta PRs, and M5 lives here. Open: land early in jenkinsci/jenkins, or iterate on a fork until E-T1 passes. The recommendation is the fork until M4, then propose.
- **Default conflict policy.** `warn` is the phase-1 default for compatibility; `fail` is the goal.

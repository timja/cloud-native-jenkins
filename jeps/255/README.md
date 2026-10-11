# JEP-255: Cluster membership, store resilience and operations

| | |
|---|---|
| **JEP** | 255 |
| **Status** | Draft |
| **Type** | Process / Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-250, JEP-252 |

## Abstract
How controllers know about each other, and how the cluster behaves when its shared services fail. Also how a cluster is installed, upgraded, migrated to, backed up and restored:
- **Membership:** heartbeats with versions and protocol numbers.
- **Resilience:** pooled database connections with timeouts; failing fast while the store is unreachable; no takeovers after an outage; waiting for the store at boot.
- **Operations:** a Helm chart; rolling upgrades with mixed versions; importing a file-based home; backup and restore.

## Specification

### Joining and leaving
There is no join step: no token, UI, CLI or API to add a controller. A controller is a member because it is configured for the cluster's stores, and it stops being one when it stops heartbeating. Adding a controller means starting another one with the same configuration (in Kubernetes, raising the StatefulSet's replicas). What a controller needs:
- **The stores:** the database's JDBC URL and credentials (`CNJ_STORE_URL`, `CNJ_STORE_USER`, `CNJ_STORE_PASSWORD`) and the blob store's connection string and container (`CNJ_BLOB_CONNECTION_STRING`, `CNJ_BLOB_CONTAINER`). Access to these is what makes a controller part of the cluster.
- **The cluster's key material:** `secret.key`, `secrets/` and the instance identity, generated once for the cluster and given to every controller (the chart's `make-seed-secret.sh` creates it as a Kubernetes Secret; the entrypoint copies it into an empty home). Keys created later at runtime are in the store (JEP-250). A controller started without this seed creates keys of its own and cannot read the cluster's secrets; nothing detects that yet.
- **An identity:** a name unique among the live controllers (`CNJ_NAME`, by default the host name, such as the pod name) and an internal URL other controllers can reach it at (`CNJ_INTERNAL_URL`, JEP-254).
- **A compatible version:** the controller image, of the same or an adjacent version (rolling upgrades are supported, see below). Plugins added at runtime are applied from the cluster's plugin set (JEP-256).

On start, a controller:
1. waits for both stores (see Resilience below)
2. creates or migrates the schema, under a lock, if it is the first of a new version
3. refuses to start if a live peer speaks a protocol older than its `minimumPeerProtocol`
4. registers itself with its first heartbeat
5. catches up: items, static configuration, build history and the plugin set are read from the stores, and update-site data is taken from the cluster
6. starts serving. Readiness probes and load balancers send it requests once it answers.

It leaves by stopping, cleanly (its row is deleted, and the others react at once) or not (its heartbeat expires). Its running builds are adopted (JEP-252). The cluster view (Manage Jenkins > Cluster) shows the members, read-only.

### Membership
- **Heartbeats.** Each controller heartbeats every 5 s into a members table. A row holds its origin (a new one per start), name, internal URL, core and plugin version, cluster protocol number, and applied plugin-set generation (JEP-256).
- **Liveness.** A member is alive while its last heartbeat is younger than the member timeout (20 s).
  - A cleanly stopped controller deletes its row, so others react at once.
  - Rows dead for a day are pruned.
- **Leaving.** A controller compares the live members at every heartbeat. When one leaves, it sweeps that controller's builds and queue items right away (JEP-252, JEP-253).
- **Protocol versions.** `minimumPeerProtocol` makes a new version refuse to start while older peers are alive, for changes that cannot coexist.

### Resilience to store outages
- **Connections.** Two connection pools (HikariCP):
  - one for ordinary work
  - one for heartbeats, leases and membership, so a busy work pool never makes a controller look gone

  Every connection has socket, connect, statement and lock timeouts and TCP keepalive. A call is retried on another connection only if the connection broke before the commit was sent, so no work is done twice.
- **Fail fast.** While the database is unreachable, ordinary store calls fail at once for 5 s at a time, instead of each waiting out the timeouts. The heartbeat keeps probing and closes the breaker when it gets through.
- **What controllers do meanwhile:**
  - Pages answer, without data from the store, and a key never used before is temporary.
  - Running builds carry on. Their logs and save points wait.
  - Configuration saves fail with an error.
  - New builds wait in the queue.
  - A cron tick is skipped, never run twice.
- **Recovery:**
  - Change events are delivered after reconnecting, including late-committed ones: a gap in event numbers is re-read for 60 s.
  - Completed builds whose publication failed are published.
  - Builds whose start could not be recorded are registered.
  - Failed applies are retried.
- **No takeovers after an outage.** Nobody takes another controller for gone until its own heartbeats have been steady for longer than the timeout (JEP-252).
- **Blob storage** calls have bounded per-try and response timeouts. Logs and save points are shipped later, and a log's tail shipped after its record is republished.
- **Boot.** A controller starting while the database or blob storage is unreachable waits, with backoff, for up to 10 minutes (configurable). Meanwhile it is not ready. It never runs half-connected, and leaves no "ghost" member behind.

### Operations
- **Helm chart** (`charts/cloud-native-jenkins`):
  - a StatefulSet of N controllers with `emptyDir` homes
  - a non-sticky Service
  - the seeded key Secret
  - JCasC, with an option for in-chart Postgres and Azurite (the Azure Storage emulator) for development; managed services in production
  - a Kubernetes cloud
  - probes: readiness reflects whether Jenkins is up and serving
- **Rolling upgrade.** `helm upgrade` to a new image replaces controllers one at a time. Running Pipelines are adopted by the controllers still up, and mixed versions coexist within one protocol number.
- **Import.** A file-based home is imported by starting one controller on it with `importHome=true`. Configuration, users, credentials, nodes, builds, logs and fingerprints are uploaded, keys already present are kept, and a report is written. Importing again imports nothing.
- **Backup and restore.**
  - `scripts/backup.sh` does a `pg_dump` and then copies the blob container; `scripts/restore.sh` restores into a new namespace.
  - After a restore, builds that were running at backup time are adopted from their save points, or closed as aborted (JEP-252).
  - Point-in-time recovery uses the managed services' own features (documented, not exercised).

## Motivation
- Every controller depends on the store. Before this work, an outage of more than 20 s aborted every running build in the cluster, and on recovery the controllers took over each other's work. This was found by reading the code, then shown by pausing the database in an acceptance test.
- Single-controller Jenkins has no equivalents for joining, upgrading or restoring a cluster.

## Reasoning
- **Membership follows from access to the stores** rather than from an explicit join. Controllers are started and stopped by the deployment (Kubernetes scales them, replaces failed pods and rolls upgrades), so a separate join step would be one more thing for it to do, and to get wrong. Whether joining should also need explicit approval between controllers is part of the separate security review.
- **The store is the source of truth for liveness:** one table, no gossip and no consensus between controllers. Controllers judge liveness only while their own view of the store is trustworthy (steady heartbeats).
- **Fail fast rather than queue up.** A degraded UI that answers is better than one that hangs, and work that needs the store is retried or waits in the queue anyway.
- **Waiting at boot rather than failing.** Kubernetes restarts would otherwise loop while the database starts.

## Backwards compatibility
- Without a store, nothing changes.
- The chart and scripts are new.
- **Known limitations:**
  - Point-in-time recovery is not exercised.
  - Schema changes that need a rewrite would need a background migration (none so far).

## Security
- The store and blob credentials come from Secrets, and connections can use TLS through the JDBC URL and connection string.
- The internal URLs used between controllers must stay inside the cluster network (JEP-254).

## Testing
- **Store plugin:** `PostgresPersistenceStoreTest`: reconnects; no takeovers before the controller is steady; late-committed events; prefix listings with wildcards.
- **Acceptance** (scripts in `validation/experiments/` of the cloud-native-jenkins repository; each runs two controllers against one store):
  - `ET9-kubernetes.sh`: the chart on a local kind cluster (scaling, agents, losing a controller)
  - `ET10-operations.sh`: rolling upgrade, refused older peer, import, backup and restore
  - `ET14-store-outage.sh`: Postgres paused for 10 s and 60 s and restarted, Azurite stopped, a controller booting while Postgres is down; no aborts, takeovers, duplicates or lost logs, and pages answering
  - `ET16-scale.sh`: 2,000 jobs and 2,000 builds on two controllers. The second controller lists the jobs 10 s after the first starts creating them, warm pages load in under 1 s, a restart takes 14 s, and 50 running Pipelines are adopted within 24 s. Prefix listings use an index (`text_pattern_ops`)

## Prototype implementation
- **Store plugin** (`impl/persistence-store-postgresql` in the cloud-native-jenkins repository): `PostgresPersistenceStore`, `PostgresPersistenceStoreProvider`, `AzureBlobStore(Provider)`, `PeerVersionCheck`, `HomeImporter`, `ClusterView`.
- **Core:** prototype branch of Jenkins core `cnj/t14-m1-resilience` (retries, temporary keys, `sweepSoon`).
- **Chart:** `charts/cloud-native-jenkins` in the cloud-native-jenkins repository.
- **Findings:** documents in the cloud-native-jenkins repository, on its prototype branches: `findings/ET9-kubernetes.md`, `ET10-operations.md`, `ET14-store-outage.md` and `ET16-scale.md`.

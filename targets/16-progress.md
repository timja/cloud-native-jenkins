# Target 16: progress and branches

## Branch
`cnj/t16-plugin-split` in this repository (worktree `../cloud-native-jenkins-t16`), on `cnj/t15-remaining`. Core and the forked plugins are unchanged from Target 15.

## Steps
| # | Step | Status | Evidence |
|---|---|---|---|
| 1 | Cluster SPI in the old plugin: `ClusterStore` (+ `Membership`, `Claims`, `Channels`, `Counters`, `QueueMaintenance`), `ClusterBlobStore`, `LogBlobs`, `ChunkedLogBlobs`, `ClusterMember`; no class outside the backends casts to them | ✅ | `74adca2`; contract test against an in-memory store and Azurite |
| 2 | Split into `plugins/cluster`, `cluster-postgresql`, `cluster-azure-blob`, `cluster-s3`; renamed packages, settings, headers, symbols; the plugin BOM | ✅ | `3e95879`; 51 + 27 + 14 + 10 tests pass (Postgres, Azurite, Garage in Testcontainers) |
| 3 | `cluster-s3`: `S3BlobStore` on the AWS SDK v2 plugins, multipart, pagination, chunked logs | ✅ | `S3BlobStoreTest` against Garage |
| 4 | Harness, image and chart (`blob.type`), the S3 variant | ✅ | `422f8b7` |
| 5 | Docs: JEPs, this target, ROADMAP, the upstreaming plan | ✅ | |
| 6 | Default stack: full sweep | ✅ | ET1–ET22, ET17 with C killed at 36 s, ET9/ET10/ET13 on kind; 6 lanes on native Docker (ET4 in a rerun after a check fix) |
| 7 | S3 variant: ET3, ET4, ET8, ET15, ET19 | ✅ | Garage; `findings/raw-s3/`, `findings/T16-plugin-split.md` |
| 8 | Reseed the kind release; push the branch to the private repository | ⬜ | |

## Found on the way
- `InjectedTest` of the old plugin failed: its test classpath mixed incompatible versions of credentials, commons-lang3 and jackson2. The sweep builds skip tests, so it went unnoticed. The plugin BOM in each module's pom fixes it.
- Making the protected plugin name a set (`ClusterPlugins.PROTECTED`) broke on a null plugin name (`Set.of(...).contains(null)` throws). The `ClusterPluginSetTest` caught it, and `isProtected` is null-safe.

- **Core (found by running six lanes at once):** a controller whose own heartbeat ran late took itself for gone and adopted, and resumed a second time, its own running build (ET15 2). Fixed in core `cnj/t15-remaining` a9515a0, to fold into the lease branch.
- **Harness:** parallel sweeps in slots (`experiments/sweep.sh`): one lane on macOS, otherwise sized to the machine. Docker Desktop on Linux needed temporary files under the repository and single-platform `kind load`; the remote machine runs native Docker Engine now.

## Log
- 2026-10-14: planned with the user (naming, depth of the split, S3 now, Garage, one repository, configuration renamed without aliases). Steps 1–5 done.
- 2026-10-14 (afternoon): the full sweep passes on the Cluster plugins, on a 24-CPU Linux machine with 6 lanes (about 55 minutes for the Compose experiments). Target 16 done but for reseeding the kind release and pushing.

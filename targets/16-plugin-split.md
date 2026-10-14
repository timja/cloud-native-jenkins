# Target 16: the Cluster plugin family

| | |
|---|---|
| **Roadmap tier** | Packaging (before hosting requests) |
| **Status** | Done 2026-10-14 (planned the same day): every experiment passed in one sweep, see [16-progress](16-progress.md) |
| **Depends on** | Target 15 |
| **Out of scope** | Hosting requests and upstreaming. Point-in-time recovery on S3 (needs bucket versioning; ET18 stays on Azure) |

**Done means:**
- the four plugins build and pass their tests, including against Postgres, Azurite and Garage in Testcontainers
- the `cluster` plugin has no Postgres or Azure dependency
- ET1–ET22 pass in one sweep on the default stack (ET9, ET10 and ET13 on kind)
- ET3, ET4, ET8, ET15 and ET19 pass on the S3 variant

## Decisions (user, 2026-10-14)
- **Name:** "Cluster", which is the JEPs' term; high availability is its benefit. No "cloud-native" in plugin names.
- **Split:** a backend-neutral plugin plus pluggable backends. Azure and AWS dependencies are large and installations rarely need both, so each SDK comes only with its plugin.
- **S3:**
  - Built in this target.
  - Tested against an S3-compatible store in Testcontainers. MinIO no longer publishes images, so this uses Garage (`dxflrs/garage:v2.4.1`).
- **Layout:** one repository, multi-module (`plugins/`), split into jenkinsci repositories at hosting time.
- **Configuration:** renamed, with no `CNJ_*` aliases.
- **Demo stack:** Postgres plus Azurite stays the default stack for tests and demos.

## The plugins
| Plugin | Holds | Depends on |
|---|---|---|
| `cluster` | Everything backend-neutral: routing, the queue's housekeeping, adoption, save points and log shipping, handover, plugin management, the cluster view, fingerprints, milestones. Also the cluster SPI. | workflow-api, workflow-cps (fork), pipeline-milestone-step, kubernetes (optional) |
| `cluster-postgresql` | The store: configuration, build records, membership, leases, claims, the shared queue, schema migrations | `cluster`, `postgresql-api` |
| `cluster-azure-blob` | Blobs in Azure Blob Storage; logs as segmented append blobs | `cluster`, `azure-sdk` |
| `cluster-s3` | Blobs in Amazon S3 or an S3-compatible store (endpoint, path style); logs as chunks; multipart uploads | `cluster`, `aws-java-sdk2-s3` |

**The cluster SPI** (`io.jenkins.plugins.cluster.spi`) sits on top of core's:
- **`ClusterStore`** extends `PersistenceStore` with `Membership`, `Claims`, `Channels`, `Counters` and `QueueMaintenance`.
- **`ClusterBlobStore`** extends `BlobStore` with `list`, `download`, `size` and `logs()`, which returns a `LogBlobs`.
- **`ChunkedLogBlobs`** serves stores that cannot append.
- **`ClusterMember`** holds this controller's identity and what it reports.

Backends implement the SPI and call `ClusterStartup` as they join.

## Configuration
| | Environment | System properties |
|---|---|---|
| Controller | `JENKINS_CLUSTER_NAME`, `JENKINS_CLUSTER_INTERNAL_URL` | `io.jenkins.plugins.cluster.*` |
| Postgres | `CLUSTER_POSTGRES_URL`, `CLUSTER_POSTGRES_USER`, `CLUSTER_POSTGRES_PASSWORD` | `io.jenkins.plugins.cluster.postgresql.{url,user,password}` |
| Azure | `CLUSTER_AZURE_BLOB_CONNECTION_STRING`, `CLUSTER_AZURE_BLOB_CONTAINER` | `io.jenkins.plugins.cluster.azure.{connectionString,container}` |
| S3 | `CLUSTER_S3_BUCKET`, `CLUSTER_S3_PREFIX`, `CLUSTER_S3_REGION`, `CLUSTER_S3_ENDPOINT`, `CLUSTER_S3_PATH_STYLE`; credentials from the AWS default chain | `io.jenkins.plugins.cluster.s3.*` |

Other renames:
- **Headers:** `X-Jenkins-Cluster-Controller` and `X-Jenkins-Cluster-Forwarded-From`.
- **JCasC symbols:** `crumbIssuer: cluster` and `clusterFingerprintStorage`.

## Harness and chart
- **Build:** `ET1-build.sh` builds `plugins/` into `.et1/plugins` (Postgres plus Azure) and `.et1/plugins-s3` (Postgres plus S3).
- **S3 variant:** `experiments/with-s3.sh` runs experiments on it.
  - It uses Garage in `compose.et1.yaml` (profile `s3`), with artifacts through artifact-manager-s3 (`jenkins/casc-s3`).
- **Chart:** `blob.type`, either `azure` (the default, with in-chart Azurite) or `s3` (an external bucket).

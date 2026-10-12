# JEP-251: Shared build records, logs and history

| | |
|---|---|
| **JEP** | 251 |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-250 |

## Abstract
Completed builds are published to the shared store, so every controller shows the full history of every job, whichever controller ran each build:
- the build's record (`build.xml`) goes to the persistence store
- a snapshot of its directory goes to blob storage
- a Pipeline's log goes to external log storage (JEP-210), written while the build runs

Builds are materialised locally only when something reads them. Builds still running elsewhere are shown in the UI and the REST API as such. Fingerprints, test results and build discarders work across controllers.

## Specification

### Records and snapshots
- **Publishing.** When a build is finalized, its controller publishes:
  - `jobs/<job>/builds/<N>/build.xml` to the persistence store
  - `jobs/<job>/builds/<N>/files.tgz` to blob storage: the build directory without the Pipeline log, written first, so a record never refers to a missing snapshot
- **Numbering.** `nextBuildNumber` is shared (JEP-253) and kept when builds or jobs are deleted. A `(job, number)` pair is never reused, so data keyed by it (for example junit-sql) never collides.
- **The index.** `RunMap` is the union of local build directories and the store's records (`BuildRecords.storedNumbers` → `AbstractLazyLoadRunMap.recognizeNumber`). A build only in the store is materialised into the home (record plus snapshot) when first loaded.
- **Changes from elsewhere.**
  - A new record becomes a recognised number.
  - A republished record drops the local copy, which is materialised again.
  - A deletion removes the local copy without firing listeners, because cleanup happened where it was deleted.
  - **Permalinks** (last successful build, last failed build, …) are cached in the store per job (`<job>/builds/permalinks`), updated by the controller completing or deleting a build, as `PeepholePermalink` does on one controller. A controller with a fresh home then resolves them without going through the job's builds: with 20,000 builds and none failed, a job page took 90 s cold before, 1.5 s now. A controller that has not seen a newer build yet never moves a target back.
- **Republishing.**
  - A build is republished when its `build.xml` is saved again (a description, keep-forever).
  - It is also republished when files are added to, or changed in, its directory later without such a save (a plugin writing a report afterwards). Each published snapshot carries a manifest of the files it holds (`.cnj-published`, their names, sizes and times). A sweep compares builds against it every minute for an hour after completion, and every 10 minutes over every build loaded on the controller (at most 1,000 per run, carrying on in the next).
  - A local copy with files newer than the published snapshot is published, not replaced by it, when the record changes elsewhere.
- **Builds nothing can resume** (their controller died, and they are not resumable Pipelines) are closed with an ABORTED record whose console has what the build had printed, then why it ended (JEP-252). A `BuildRecords.LostLog` extension supplies what was printed, from wherever the log storage shipped it.
- **Deleting a job's history** deletes every record below the job's key at once (`PersistenceStore.deleteAll`), with one change event each.
- **Failed publication.** A build whose publication fails (the store was unreachable) stays "running" for the cluster, and publication is retried until it succeeds.

### Logs
Pipeline logs use the existing external log storage API (JEP-210, `LogStorageFactory`), so any implementation can serve a cluster, for example append blobs (the reference implementation below) or CloudWatch Logs (pipeline-cloudwatch-logs). This JEP adds no log API. It states what an implementation must do for a cluster:
- **Readable from every controller**, while the build runs and after, through `LogStorageFactory.forBuild` on any controller. The console of a running build is also forwarded to its owner (JEP-254).
- **Keyed by the build, not by the controller** (for example job full name and build number), so a controller adopting a build continues the same log.
- **Durable at save points.** Everything written before a Pipeline save point is stored before the save point is (JEP-252). A controller resuming from it then continues the log where the save point left off, with no lines missing or repeated. The prototype makes the save point wait through workflow-cps' `ProgramSavedListener`. An implementation that buffers asynchronously (CloudWatch batches) must flush there. That hook is the one API addition logs need, and it belongs to JEP-252.
- **Fenced.** A controller that may have lost the build's lease stops writing (JEP-252). A store accepting writes from two controllers (CloudWatch does) relies on this.
- **Moved with the job** on a rename, or keyed so it need not move (as on a single controller).

**Reference implementation** (store plugin, `BlobLogStorage`, Azure Blob Storage):
- The build writes its local `log` and `log-index`. A shipper appends what is new to append blobs: when 32 KiB is pending, 3 s after the last shipment, before each save point and at the end of the build.
- **Segments:** an append blob takes at most 50,000 blocks, so a log is stored as `log`, `log.1`, …. A new segment starts before the limit, and readers concatenate them. This is specific to Azure append blobs; another backend has its own limits.
- The shipper checks ownership and pauses while this controller's membership may have lapsed. A new owner continues where the blobs end.

**Freestyle and other build logs** are written to the build's `log` file, as on one controller; core has no log SPI for them (`Run.getLogFile`). The reference implementation ships them to the same append-blob segments while the build runs, so that a build whose controller dies keeps what it printed (closed as aborted, above), and the CLI can follow it from another controller (JEP-254). Once the build is published, its log travels in the snapshot and the shipped copy is deleted. While such a build runs, its console page is served by the controller running it (JEP-254).

### Builds running on another controller
- Each running build has a placeholder `jobs/<job>/running/<N>.xml`, holding its owner, start time and save point (JEP-252).
- On other controllers, the job page lists placeholders under "Running on other controllers".
- The Builds card shows them above local builds, with a stop button, and the job's icon animates.
- In the REST API, `builds`, `allBuilds` and `lastBuild` include them as a stand-in with a `Run`'s properties (`number`, `url`, `building`, `timestamp`, `result`, …). Their own URLs are answered by their owner.
- `/job/x/lastBuild/…` redirects to the newest build when that build runs elsewhere.

### History at scale
- **Fingerprints.** A `FingerprintStorage` in the store, keyed by MD5. Concurrent updates of usages are merged three-way against the version each controller last read.
- **Test results.** junit-sql-storage, with its cache invalidated when a build's record changes elsewhere, so a report is right on another controller within about 0.2 s of completion. Its rows follow their job when it is renamed or moved.
- **Other per-build reports** (coverage, static analysis and so on) are files in the build directory and travel in the snapshot, unchanged: the coverage plugin's summary, tree and source views are the same on every controller.
- **Build discarders** apply to the whole history, whichever controller ran each build. A build still running elsewhere is never discarded.
- **A cluster view** (`/manage/cluster`) lists the controllers, running builds with their owners, agent channels with their holders, and plugin changes.

## Motivation
- An experiment replaced a controller and its home with a fresh one against the same external stores. All build history was lost, and recreating a job reused `(name, #1)`, so junit-sql-storage showed the old build's tests with the new one's: 4 instead of 2.
- In a cluster, each controller would otherwise show only the builds it ran.

## Reasoning
- **Snapshots** keep the build directory byte-for-byte, so plugins that store per-build files (junit file storage, coverage, badges and so on) keep working without changes.
- **Pipeline logs ship continuously** (not only at completion), so a controller adopting a build after its owner died has the log up to the save point it resumes from.
- **Lazy materialisation** keeps startup cost proportional to what is looked at, not to the size of the history.
- **A live freestyle console needs a core log SPI** (`Run.getLogFile` hard-codes files; JEP-207 was deferred). Forwarding to the owner gives the same result without one.

## Backwards compatibility
- Without a store, nothing changes.
- With one, plugins reading the build directory see a complete directory once the build is loaded.
- Plugins writing to a completed build's directory have their files republished, within a minute during the first hour after completion, and within 10 minutes later.
- Plugins keeping data keyed by job *name* outside `JENKINS_HOME` can key it by `Item.getItemUuid()` instead (JEP-250), which a rename does not change.

## Security
- Snapshots and logs have the same content as the build directory. Blob storage access is equivalent to access to `JENKINS_HOME/jobs`.
- Read permission is checked by the controller serving the build, as before.

## Testing
- **Core:** `BuildRecordsTest` (late files, a closed build keeping what it printed), `RunningBuildsTest`, `RemoteBuildEventsTest`, `DeletedJobNumberingTest`.
- **Store plugin:** `AzureBlobStoreTest` (against Azurite, the Azure Storage emulator: segments, moves keep append blobs appendable), `StoreFingerprintStorageTest`.
- **Acceptance** (scripts in `validation/experiments/` of the cloud-native-jenkins repository; each runs two controllers against one store):
  - `ET3-build-records.sh`: builds that ran on either controller are listed on both, with the same records, consoles, permalinks and placeholders for running builds; deletion and renames carry over, a fresh controller sees the whole history, and a recreated job never reuses a number
  - `ET8-history.sh`: fingerprints, test results, coverage reports and build discarders work across controllers, and the cluster view lists both
  - `ET12-remainder.sh`: a file added to a completed build's directory later is republished
  - `ET15-logs-api-failover.sh`: a Pipeline log rolls over segments and stays whole across a failover and a rename; a freestyle build whose controller is killed keeps its output; the REST API lists builds running elsewhere

## Prototype implementation
- **Core:** prototype branches of Jenkins core, `cnj/t3-*`, `cnj/t8-*`, `cnj/t12-m1-remainder`, `cnj/t14-m1-resilience` and `cnj/t15-remaining`.
- **junit-sql-storage:** prototype branches `cnj/t3-m5-remote-delete`, `cnj/t15-follow-renames`.
- **Store plugin** (`impl/persistence-store-postgresql` in the cloud-native-jenkins repository): `BuildRecords` publication hooks, `BlobLogStorage`, `StoreFingerprintStorage`, `ClusterView`.
- **Findings:** documents in the cloud-native-jenkins repository, on its prototype branches: `findings/ET3-build-records.md`, `ET8-history.md`, `ET12-remainder.md` and `ET15-logs-api-failover.md`.

## References
- JEP-207 (external build logging), JEP-210 (external log storage for Pipeline), JEP-226 (external fingerprint storage)
- [junit-sql-storage](https://plugins.jenkins.io/junit-sql-storage/)

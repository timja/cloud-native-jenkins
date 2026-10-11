# JEP-CNJ-2: Shared build records, logs and history

| | |
|---|---|
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-CNJ-1 |

## Abstract
Completed builds are published to the shared store, so every controller shows the full history of every job, whichever controller ran each build:
- the build's record (`build.xml`) goes to the persistence store
- a snapshot of its directory goes to blob storage
- a Pipeline's log goes to append blobs, shipped while the build runs

Builds are materialised locally only when something reads them. Builds still running elsewhere are shown in the UI and the REST API as such. Fingerprints, test results and build discarders work across controllers.

## Specification

### Records and snapshots
- **Publishing.** When a build is finalized, its controller publishes:
  - `jobs/<job>/builds/<N>/build.xml` to the persistence store
  - `jobs/<job>/builds/<N>/files.tgz` to blob storage: the build directory without the Pipeline log, written first, so a record never refers to a missing snapshot
- **Numbering.** `nextBuildNumber` is shared (JEP-CNJ-4) and kept when builds or jobs are deleted. A `(job, number)` pair is never reused, so data keyed by it (for example junit-sql) never collides.
- **The index.** `RunMap` is the union of local build directories and the store's records (`BuildRecords.storedNumbers` → `AbstractLazyLoadRunMap.recognizeNumber`). A build only in the store is materialised into the home (record plus snapshot) when first loaded.
- **Changes from elsewhere.**
  - A new record becomes a recognised number.
  - A republished record drops the local copy, which is materialised again.
  - A deletion removes the local copy without firing listeners, because cleanup happened where it was deleted.
  - Permalinks are updated incrementally (`PeepholePermalink.changedElsewhere`).
- **Republishing.**
  - A build is republished when its `build.xml` is saved again (a description, keep-forever).
  - It is also republished when files are added to its directory later without such a save; the sweep runs every minute for an hour after completion.
- **Builds nothing can resume** (their controller died, and they are not resumable Pipelines) are closed with an ABORTED record whose console says why (JEP-CNJ-3).
- **Failed publication.** A build whose publication fails (the store was unreachable) stays "running" for the cluster, and publication is retried until it succeeds.

### Logs
- **Pipeline logs** go through a `LogStorageFactory`. The build writes its local `log` and `log-index`, and a shipper appends what is new to append blobs:
  - when 32 KiB is pending, or 3 s after the last shipment
  - at once before each save point and at the end of the build
- **Segments.** An append blob takes at most 50,000 blocks, so a log is stored in segments `log`, `log.1`, …. A new segment starts before the limit, and readers concatenate them.
- **Ownership checks.** The shipper checks ownership and pauses while this controller's membership may have lapsed, so a fenced owner never appends (JEP-CNJ-3). A new owner continues where the blob ends.
- **Freestyle and other build logs** travel in the snapshot. While such a build runs, its console is served by the controller running it (JEP-CNJ-5).

### Builds running on another controller
- Each running build has a placeholder `jobs/<job>/running/<N>.xml`, holding its owner, start time and save point (JEP-CNJ-3).
- On other controllers, the job page lists placeholders under "Running on other controllers".
- The Builds card shows them above local builds, with a stop button, and the job's icon animates.
- In the REST API, `builds`, `allBuilds` and `lastBuild` include them as a stand-in with a `Run`'s properties (`number`, `url`, `building`, `timestamp`, `result`, …). Their own URLs are answered by their owner.
- `/job/x/lastBuild/…` redirects to the newest build when that build runs elsewhere.

### History at scale
- **Fingerprints.** A `FingerprintStorage` in the store, keyed by MD5. Concurrent updates of usages are merged three-way against the version each controller last read.
- **Test results.** junit-sql-storage, with its cache invalidated when a build's record changes elsewhere, so a report is right on another controller within about 0.2 s of completion.
- **Build discarders** apply to the whole history, whichever controller ran each build. A build still running elsewhere is never discarded.
- **A cluster view** (`/manage/cluster`) lists the controllers, running builds with their owners, agent channels with their holders, and plugin changes.

## Motivation
- E2 showed that a disposable controller loses all build history, and that recreating a job reused `(name, #1)`. junit-sql then showed 4 tests instead of 2.
- In a cluster, each controller would otherwise show only the builds it ran.

## Reasoning
- **Snapshots** keep the build directory byte-for-byte, so plugins that store per-build files (junit file storage, coverage, badges and so on) keep working without changes.
- **Pipeline logs ship continuously** (not only at completion), so a controller adopting a build after its owner died has the log up to the save point it resumes from.
- **Lazy materialisation** keeps startup cost proportional to what is looked at, not to the size of the history.
- **A live freestyle console needs a core log SPI** (`Run.getLogFile` hard-codes files; JEP-207 was deferred). Forwarding to the owner gives the same result without one.

## Backwards compatibility
- Without a store, nothing changes.
- With one, plugins reading the build directory see a complete directory once the build is loaded.
- Plugins writing to a completed build's directory have their files republished, but only within an hour. Later writes are republished only when the record is saved again.
- **Known limitation:** data keyed by job *name* across renames (for example junit-sql history) is unchanged from a single controller; it needs a per-item UUID in core.

## Security
- Snapshots and logs have the same content as the build directory. Blob storage access is equivalent to access to `JENKINS_HOME/jobs`.
- Read permission is checked by the controller serving the build, as before.

## Testing
- **Core:** `BuildRecordsTest`, `RunningBuildsTest`, `RemoteBuildEventsTest`, `DeletedJobNumberingTest`.
- **Store plugin:** `AzureBlobStoreTest` (Azurite: segments, moves keep append blobs appendable), `StoreFingerprintStorageTest`.
- **Acceptance:**
  - ET3: records, logs, placeholders, permalinks, deletion and numbering
  - ET8: fingerprints, test results, discarders and the cluster view
  - ET12: late files
  - ET15: log rollover, also across a failover and a rename, and the REST API

## Prototype implementation
- **Core:** `cnj/t3-*`, `cnj/t8-*`, `cnj/t12-m1-remainder` and `cnj/t14-m1-resilience`.
- **Store plugin:** `BuildRecords` publication hooks, `BlobLogStorage`, `StoreFingerprintStorage`, `ClusterView`.
- **Findings:** `findings/ET3-build-records.md`, `ET8-history.md`, `ET12-remainder.md` and `ET15-logs-api-failover.md`.

## References
- JEP-207 (external build logging), JEP-210 (external log storage for Pipeline), JEP-226 (external fingerprint storage)
- [junit-sql-storage](https://plugins.jenkins.io/junit-sql-storage/)

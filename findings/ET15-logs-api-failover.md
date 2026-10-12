# ET15: Target 14 M2, M4 and M5 acceptance (log segments, builds elsewhere in the REST API, faster failover)

**Result: PASS**, every check, in two consecutive runs on the final build (the final regression sweep). Raw evidence is in `raw/ET15/`:
- `results.md`
- the consoles as B shows them
- `failover-seconds.txt`
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET15-logs-api-failover.sh`
- **Setup:** two tmpfs controllers, Postgres, Azurite, token auth.
  - Log segments are 20 blocks instead of 49,000, so a few minutes of output rolls over several times.
  - Coalescing is 500 ms.

| Scenario | Result |
|---|---|
| 1 A chatty Pipeline on A, watched from B | ✅ while it runs, B's `api/json` lists it first in `builds[]` and as `lastBuild`, building. B's `/lastBuild/api/json` is redirected to the running build and forwarded to A. The log rolled over into 5–6 segments, and B shows it whole and in order. Once it completes, B's API shows the completed build |
| 2 A killed mid-log | ✅ B adopts within 20–21 s, and the build completes on B. The log is whole and in order across the segments of both owners (7 segments) |
| 3 Renames while the build runs on A | ✅ renaming the job from B, or its folder, is refused (400), as it is on A. The build completes with its whole log |
| 4 Failover time, three SIGKILLs | ✅ adopted 16–23 s after the kill (it was about 50 s worst case before), and every adopted build completed |

## What was wrong before
- **A log longer than 50,000 appends could not be written.** Azure append blobs take at most 50,000 blocks, and the shipper appended one block per flush, so a chatty multi-day Pipeline would stop shipping its log.
- **On a controller not running the build, the REST API showed no build.** `builds`, `allBuilds` and `lastBuild` showed nothing until it completed, and so did tools polling `lastBuild` (scripts, IDE plugins, dashboards).
- **Failover took up to about 50 s.** Builds were adopted only by the periodic sweep, which ran after the member timeout.
- **A rename on the controller not running the build was accepted.** It moved the running build's blobs from under its owner.

## How it works now
- **Segments (M2).**
  - A console log is `log`, `log.1`, `log.2`… Each is an append blob of at most 49,000 blocks (`io.jenkins.plugins.cnj.store.AzureBlobStore.maxSegmentBlocks`).
  - The shipper rolls over to the next segment when one is full. Readers (the console, progressive text, `consoleText`) concatenate the segments.
  - Shipping is coalesced: at most every 3 s or 32 KiB (`BlobLogStorage.coalesceMillis` and `coalesceBytes`). That means far fewer blocks per build and fewer requests to the blob store.
  - Moves (renames) copy append blobs as append blobs, so an adopted build can keep appending.
- **REST API parity (M4).**
  - A job's `builds`, `allBuilds` and `lastBuild` include builds running on other controllers. Each appears as a stand-in exported with a Run's properties (number, url, building, timestamp, estimatedDuration, result).
  - `/lastBuild/…` on a controller whose own last build is older is redirected to the newer build, which is then routed to its owner.
  - `Job.isBuilding()` counts builds elsewhere, so core refuses renames of the job and of its folders, as it does for a local build.
- **Faster failover (M5).** The heartbeat thread watches membership. When a controller leaves, the others reclaim its leases and sweep its builds at once instead of waiting for the periodic sweep. What remains is the member timeout (20 s) plus a heartbeat, which gives the 16–23 s measured. The target was about 15 s; going lower means lowering the member timeout, which would make GC pauses and slow disks look like deaths.

## Run history
| Run | Result | What it found |
|---|---|---|
| earlier | rename scenario failed | renaming a job while it builds is refused by core even on the owner, so the scenario became a folder rename. A folder rename from B was then accepted, which is the bug fixed above (`isBuilding` counts builds elsewhere). The harness also did not wait for B to see the completed build |
| sweep ×2 | all ✅ | final build |

## Target 15
| Scenario | Result |
|---|---|
| 4 (item 5) Failover time | ✅ builds adopted 5.2, 5.5 and 5.6 s after three `SIGKILL`s (the session lock is freed with the process), 5.4 s after a clean stop began; a 12 s pause is not taken for a death, and its build completes on A |
| 5 (item 12) A freestyle build whose controller is killed | ✅ closed as aborted on B, keeping what it printed (ticks 1 to 15, in order), then why it ended |

Failover was 16–23 s before session locks (Target 14).

**Harness fixes:**
- `docker logs | grep -m1` under `pipefail` ended the script with code 255 once the log grew past the pipe buffer.
- The order check of 5 also counted the shell's trace of each `echo`.
- The controller logs are now kept however the script ends.

**Not reproduced:** in one run, scenario 2 found the adopting controller's part of the log missing (lines 169 on). It passed in every run since. The logs of that run were not kept; they now are.

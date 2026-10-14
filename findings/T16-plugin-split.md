# Target 16: the Cluster plugins, the S3 variant, and parallel sweeps

**Result (2026-10-14):** with the store plugin split into `cluster`, `cluster-postgresql`, `cluster-azure-blob` and `cluster-s3`, every experiment passes:
- ET1–ET22, plus ET17 again with C killed at 36 s
- ET9, ET10 and ET13 on kind
- the S3 variant: ET3, ET4, ET8, ET15 and ET19 on Garage, an S3-compatible store

They ran in one sweep on a 24-CPU, 62 GB Linux machine with native Docker Engine: 6 Compose lanes plus a kind lane. ET4 passed in a rerun after a check was fixed (below).
- Summary: [`raw/sweep-t16/summary.txt`](raw/sweep-t16/summary.txt)
- Per-experiment evidence: `raw/ET*/` (default stack) and `raw-s3/ET*/` (S3)

## The S3 variant
- **Setup:** `experiments/with-s3.sh` runs an experiment with Garage instead of Azurite (`dxflrs/garage:v2.4.1`, one node, the bucket made at startup). It uses the plugin set with `cluster-s3` instead of `cluster-azure-blob`, and artifacts and stashes go through artifact-manager-s3 (custom endpoint, path style).
- **What it covers:** build records and snapshots (ET3), Pipeline failover with save points and logs (ET4), fingerprints (ET8), log failover (ET15) and plugin archives (ET19). All of it goes through the same code as on Azure. Only the blob store and the way logs are laid out differ.
- **Logs on S3** are chunks while the build runs (`log.d/<offset>.<length>`), compacted into one object when it completes. ET15 checks the compaction; on Azure it checks that the log rolls over into append-blob segments.
- **Not covered on S3:** point-in-time recovery (ET18). It needs bucket versioning, so it stays on the default stack.

## Running experiments side by side
- **Slots:** every experiment runs in a slot (`lib.sh`). A slot is its own Compose project (`et1sN`), host ports (`20N81`, …), container names and copy of the harness's mutable files, sharing the built war and plugins.
- **Lanes:** `experiments/sweep.sh` gives each lane a slot. A lane takes the next experiment from a queue, longest first.
  - On macOS it uses one lane, so a laptop stays usable.
  - Elsewhere it uses one lane per 4 CPUs and 8 GB that Docker has, at most 8.
  - `SWEEP_LANES` overrides both.
- **Kind:** the kind experiments have a lane of their own.
- **Time:** with 6 lanes on native Docker, the Compose experiments took about 55 minutes. The scale experiments are much faster there: ET17's restarted controller comes back in 9–15 s, against minutes on Docker Desktop (whose VM had 8 GB).

## Found and fixed
- **Core: a controller adopted its own running build (ET15 2, under load).**
  - What happened: B's own heartbeat ran late, so for a moment the store's live set lacked B. B's sweep then adopted, and resumed a second time, a build it was still running. The second resume re-staged the build directory, and the rest of the log was never shipped.
  - Fix: the sweep never takes its own origin for gone; a membership that really lapsed is fenced by the ownership check. Core `cnj/t15-remaining` a9515a0 (to fold into the lease branch), with `BuildLeasesTest.aControllerNeverTakesItselfForGone`.
- **Harness:**
  - `sweep.sh` died at the first failing command, because `lib.sh` is strict.
  - `agent1` mounted slot 0's files (its secret) in other slots.
  - ET21 looked for a CSS class that was renamed.
  - The checks in ET4 4 (who published the record) and ET15 1 on S3 (compaction) ran before the build was finalized; they now wait for it.
- **The machine:**
  - **Docker Desktop on Linux:** its VM had 8 GB, it does not share `/tmp` with containers, and its containerd image store breaks `kind load` for single-platform pulls of multi-platform images. The harness now keeps temporary files under the repository and loads images from a single-platform archive when needed. The machine was switched to native Docker Engine.
  - **Images:** a fresh Docker has none of the images ET9 loads into kind; it now pulls them.

## Earlier on this Mac (Docker Desktop, 7.8 GB VM)
Two experiments failed only because the VM was short of memory:
- ET1's fresh controller did not come up while the kind cluster ran beside it.
- A slot test took 24 minutes to boot next to a leftover stack.

This is why one lane is the macOS default.

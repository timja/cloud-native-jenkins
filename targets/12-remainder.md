# Target 12: The remainder (placement by label, late build files, restore sweep)

| | |
|---|---|
| **Roadmap tier** | R2 (placement), R5 (per-build data), X Ops (restore) |
| **Status** | Planned |
| **Depends on** | T3, T4, T5, T10 |

## Goal
The last items the roadmap leaves partly done:
1. **Placement by label:** a Pipeline whose `node` block needs an agent connected to another controller runs its `node` body there. Today, T5 places the *build* on the controller it was scheduled on (or whoever claims it), and its `node('label')` waits forever if only another controller has such an agent.
2. **Late build files:** files a plugin writes into a completed build's directory without saving `build.xml` (e.g. attachments added later) reach the other controllers.
3. **Restore sweep:** builds running when a backup was taken are, after a restore, either resumed from their last save point or closed as aborted, instead of vanishing.

## Design
### 1. Placement by label (core and store plugin)
- **The problem:** a `node` block is a queue item of its own, an `ExecutorStepExecution.PlaceholderTask`, and is not shared through the shared queue: only jobs are.
- **Detection:** when such an item stays blocked for want of an executor with its label, and another live controller has idle executors with that label (published with the agent channels, T7), the controller logs it and offers the item to that controller.
- **Remote execution is out of reach:** running a node body on another controller's agent would mean moving the build. Instead, the build is moved: a Pipeline still waiting at its first `node` step is handed over (as in adoption, T4) to the controller with the agent, which resumes it there.
- **Scope:** only builds waiting at a `node` step with no executor anywhere locally, and only to a live controller advertising a matching idle executor.

### 2. Late build files (core)
- When a completed build's directory changes without a `build.xml` save, nothing republishes its snapshot.
- `BuildRecords.Republisher` already republishes the record on `build.xml` saves. A periodic sweep of recently completed local builds (the last hour) compares their directories' contents with what was published and republishes the snapshot when they differ.

### 3. Restore sweep (store plugin)
- **On startup**, a cluster-singleton task looks for running placeholders whose owner is not a live member and was not alive in the last membership timeout: the owner disappeared, or the cluster was restored.
- **What it does with them:**
  - Pipelines with a committed save point are adopted (T4).
  - Others are closed as aborted, with their console saying why.
- T4's adoption already takes over the leases of dead owners. The sweep covers placeholders that no live controller ever leased in this cluster (after a restore).

## E-T12 acceptance experiment
1. **Placement by label:**
   - Agent `lin` is connected to B only.
   - A Pipeline `node('lin') { sh 'hostname' }` started on A completes, on B.
   - A Pipeline started on A that first runs on the built-in node and then needs `lin` waits, and is reported as waiting for a label only another controller has.
2. **Late build files:** a file added to a completed build's directory on A (as a plugin would) appears in that build on B within the sweep period.
3. **Restore sweep:**
   - Back up while a Pipeline sleeps and a freestyle build runs, then restore.
   - The Pipeline resumes from its save point and completes.
   - The freestyle build is closed as aborted, with a console line saying the cluster was restored.

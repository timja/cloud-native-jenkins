# E9: Job state written by build activity, not by admins

- **Script:** `validation/experiments/E9-runtime-job-state.sh`
- **Raw evidence:** `raw/E9/`
  - `writes-by-phase.md`
  - `jobs.txt`
  - `main-config.xml`
  - `inotify.log`

## Method
- A multibranch Pipeline (`scenarios/jobs/multibranch.xml`) points at a local git repo with branches `main` and `feature`.
- The `Jenkinsfile` uses `properties([parameters(...), buildDiscarder(...), pipelineTriggers([cron(...)])])`.
- inotify records writes under `jobs/`, grouped by the action during which they happened:
  1. create the multibranch project
  2. index (which auto-builds every branch)
  3. rebuild `main` with a parameter
  4. add branch `feature2` and re-index
  5. delete branch `feature` and re-index

## Results (excluding per-build files)
| Action | Item state written |
|---|---|
| create multibranch project (admin) | `mb/config.xml`, `mb/indexing/indexing.log` |
| index + auto-build | **creates** `branches/main/config.xml` and `branches/feature/config.xml`; `nextBuildNumber`, `permalinks`, `build.xml` for each; `mb/state.xml`; `mb/indexing/indexing.{xml,log}` |
| **rebuild `main` (a plain build, no admin)** | **`branches/main/config.xml` rewritten**, plus `nextBuildNumber`, `permalinks` |
| add branch + re-index | **creates** `branches/feature2/**`, including `scm-revision-hash.xml` and `scm-last-seen-revision-hash.xml`; **rewrites `config.xml` of `main` and `feature` too** |
| delete branch + re-index | `branches/feature/**` **deleted, including build history**; rewrites `config.xml` of the remaining branches |

`branches/main/config.xml` after the builds contains `ParametersDefinitionProperty`, `BuildDiscarderProperty` and `TimerTrigger` with `<spec>H H * * *`. So **a build defined a cron trigger.**

## Findings
1. **Job configuration is runtime state.** In a multibranch world the set of jobs and their config is produced by indexing and by `properties()` inside builds, not by admins. JCasC + Job DSL seeding can't keep N controllers in sync, because the writes happen continuously and on whichever controller ran the index or build.
2. **Scheduling config is build-defined.** `pipelineTriggers([cron])` lands in job config during a build. A single scheduler must react to job config changes made anywhere.
3. **Write amplification.** Every index rewrites every branch's `config.xml`, and every build with `properties()` rewrites its own, even when nothing changed. A shared store needs a no-op write when the content hash is unchanged. Otherwise N controllers get event storms.
4. **Indexing state** (`state.xml`, `indexing/indexing.xml`, `scm-*-revision-hash.xml`) is scheduling state owned by whoever runs the index. It belongs with build scheduling, not with item definitions.
5. **Deleting an item deletes its build history.** Branch pruning removes the whole item directory today. With shared items and per-controller build history, deletion must propagate to every controller's builds (or to the shared build store).

## Caveat
`inotifywait -r` adds watches on new directories asynchronously, so a few events in brand-new directories can be missed. For example, `branches/main/scm-*-revision-hash.xml` exist on disk (`jobs-files.txt`) but have no recorded write. Treat the tables as a lower bound.

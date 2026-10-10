# E3: Two controllers on one shared JENKINS_HOME

- **Script:** `validation/experiments/E3-shared-home.sh`
- **Raw evidence:** `raw/E3/results.md` and both controllers' logs

This is the naive "HA" setup: two JVMs mount the same volume. It is the baseline the coordination tier has to fix.

| Check | Observed |
|---|---|
| Job created on A, visible on B? | **no** (`jobs: []`) until a full stop-the-world `/reload` on B |
| Build `freestyle` triggered on A and B at the same moment | A ran `#2`, B ran `#1`. Neither controller lists the other's build. A logged `LazyBuildMixIn#newBuild: JENKINS-23152: …/builds/1 already existed; will not overwrite … will create a fresh build #2` |
| `nextBuildNumber` on disk | `3`. The default `Job.DefaultBuildNumberAssigner` is `synchronized(job)` in one JVM only (`core/src/main/java/hudson/model/Job.java:378`). The directory-exists check prevented a collision this time, but it is check-then-act and racy |
| Lost update on global config | A sets `systemMessage` and saves. B then changes `numExecutors` and saves its whole in-memory `Jenkins`, so `config.xml` reverts A's message. The two controllers now disagree in memory |
| Cron `* * * * *` over about 3 minutes | **8 builds**: A ran #3, #4, #6, #8 and B ran #1, #2, #5, #7. Each controller lists only its own 4 |
| Collision detection | `DoubleLaunchChecker` reported nothing. It runs every 60–90 minutes (`hudson/util/DoubleLaunchChecker.java:146`), so no warning appears within a test window |

## Implications
- With a shared filesystem, every in-memory model (job list, `RunMap`, `Jenkins` config, triggers) is a per-JVM cache with **no invalidation**. Fully externalised storage needs change notifications that map to targeted reloads (T1 for global config, T2 for items, T3 for runs).
- **Cron, SCM polling and every `PeriodicWork` run once per controller.** They need leader election or sharding (T5).
- **Build numbers must come from one atomic source** (T3, via `Job.BuildNumberAssigner`, which is already a Beta extension point). The JENKINS-23152 guard is no substitute: it depends on a shared filesystem and has a race window.

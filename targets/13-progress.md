# Target 13: progress and branches

Design: [13-cluster-plugins.md](13-cluster-plugins.md). **Done means:** `validation/experiments/ET13-cluster-plugins.sh` passes.

## Branch stack
| # | Repo | Branch | Based on | Scope | Status |
|---|---|---|---|---|---|
| fix | core | `cnj/t12-m1-remainder` | | a controller restarted with its home reclaims the builds it was running before resuming them (a Target 4 fix, 72e176180b) | ✅ `RunningBuildsTest`, `BuildLeasesTest`, `BuildRecordsTest` 19/19; reproduced and verified with a persistent-home compose override |
| M1 | cloud-native-jenkins | `cnj/t13-m1-cluster-plugins` | `cnj/t12-m2-remainder` | `ClusterPlugins` (overlay, reconcile, rolling restarts, monitor), `ClusterUpdateCenterConfiguration`, `ClusterPluginSet` (modes cluster/fixed/local), members' plugin generation in the cluster view, entrypoint `.cluster` markers, ET13 | ✅ ET13 22/22 ×2; `ClusterPluginsTest` 5/5, `ClusterPluginSetTest` 3/3 |

## Log
- 2026-10-11 (manual testing on kind): the cluster page shows plugin status in words ("Up to date", "Applying 2 changes", "Restart pending") and lists the plugin changes; `validation/scenarios/seed-kind.sh` seeds the hand-testing release with mock and Kubernetes agents and Declarative jobs with stages. Pipeline Graph View was installed on it through a cluster install (live on both controllers).
- 2026-10-11: Target 13 planned and implemented at the user's request (install + upgrade + uninstall; rolling safe restarts). **ET13 passes 22/22** in two consecutive runs ([findings](../findings/ET13-cluster-plugins.md)). ET13 found a Target 4 bug in restarts that keep the home (two owners of one Pipeline), fixed in core. A wait-for-idle restart was tried and dropped: busy controllers are never idle, and Pipelines checkpoint and resume.

## Follow-ups
- REST: on the controller that is not running a build, `lastBuild` (and the job's `builds`) omit it until it completes; it is only in `actions[builds]` (running elsewhere). The UI was fixed (Builds card, icon); the API is not yet.
- CLI `install-plugin` from a file or URL stays local (it bypasses the update center).
- The Plugins page does not yet say that changes apply to the whole cluster, nor show other controllers' progress (the cluster view does).

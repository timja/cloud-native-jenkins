# ET19: Target 15 M1 (the plugin manager in a cluster)

**Result: PASS**, every check. Raw evidence: `raw/ET19/`
- `results.md`
- the headers and bodies of the progress polls
- the CLI output
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET19-plugin-manager.sh`
- **Setup:**
  - two tmpfs controllers, Postgres, Azurite, token auth
  - `rr`, a load balancer with no stickiness
  - access to updates.jenkins.io

| Scenario | Result |
|---|---|
| 1 The cluster notice | ✅ the installed, available, advanced and download-progress pages of both controllers say that changes apply to every controller. The status fragment lists both, "Up to date" |
| 2 An install through `rr`, then its progress page | ✅ the install was served by one controller, which set the routing cookie. All 12 progress polls through `rr` were answered by that controller and showed the install. The plugin became active on both controllers. The progress page's "restart when done" was refused (409) |
| 3 `install-plugin` from a file, with the CLI, on B | ✅ active on B, and on A too, loaded live. Installing the store plugin itself with the CLI is refused |
| 4 Executors and the queue | ✅ B's cluster page shows A's busy executor, out of A's 4, and the shared queue |

## What was wrong before
- **The progress page alternated between controllers.** Update-center jobs are in memory on the controller running them, so through a load balancer without stickiness, every other refresh showed nothing.
- **Nothing on the plugin manager's pages said a change affects the whole cluster.** Only the cluster view showed other controllers' progress.
- **CLI installs from a file or URL stayed on one controller.** They bypass the update center, and `fixed` mode and the store-plugin guard did not apply to them.
- **The progress page's "restart when done" restarted only one controller**, outside the cluster's one-at-a-time restarts.
- **The cluster view showed neither executors nor the queue.**

## How it works now
- **Routing.** The progress page and its polling (`pluginManager/updates/…`, `updateCenter/{body,installStatus,incompleteInstallStatus,api}`) go to the controller running the install. That is the one named by a cookie set on install, otherwise the member reporting installs in progress. Members report them with each heartbeat (`jenkins_members.info`).
- **The notice.** A core hook, `PluginManager/notices.jelly`, includes a `PageDecorator`'s `pluginManager.jelly` on the plugin manager's pages. The store plugin's notice says changes apply to every controller and shows each controller's progress, for example "Applying rebuild" or the plugins it could not apply, refreshed every 5 s.
- **The CLI.** A core extension point, `PluginArchiveListener` (Beta), is told about archives `install-plugin` puts in place. The store plugin refuses those the plugin manager would refuse, and records the others in the cluster's plugin set.
- **Restarts.** The single-controller restart is refused in cluster mode and hidden: the cluster restarts its controllers by itself.

## Bugs found while running ET19, now fixed
- **A forwarded answer carried two `X-CNJ-Controller` headers**, the forwarder's and the owner's. The owner's headers now replace those set locally.
- **The status fragment was not served**, because views are not dispatched by URL unless allowlisted. It is now served by a web method that checks the permission.
- **Harness:** `set -e` with the expected failure of the CLI refusal; the progress row shows the plugin's title, not its id.

# JEP-256: Cluster-wide plugin management

| | |
|---|---|
| **JEP** | 256 |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-250, JEP-255 |

## Abstract
A plugin installed, upgraded, uninstalled, disabled or enabled on one controller is applied to every controller of the cluster, and to any controller that later starts with an empty home.
- New plugins load live where they support it.
- Other changes are applied by rolling safe restarts, one controller at a time. Pipelines pause at a save point and resume.

The controller image defines the base plugin set. Runtime changes are an overlay on it.

## Specification

### The overlay
- Store key `cluster/plugins.json` holds a generation and, per plugin, one of three states:
  - `installed`, with version, sha256 and blob
  - `uninstalled`
  - `disabled`
- Archives are the exact bytes the installing controller verified, stored under `cluster-plugins/<name>/<sha256>.jpi`, so no other controller needs the update center.
- An installed entry applies only where the local version is missing or older. A newer image wins, and nothing is downgraded.
- The cluster plugins themselves (`cluster` and its backends) cannot be changed at runtime (409).

### Capture
- **Installs and upgrades.** Every install and upgrade through the update center (from an update site or an upload, dependencies included) passes through `UpdateCenter.UpdateCenterConfiguration.install()`. A wrapper records the archive in the overlay.
- **Uninstall, disable and enable** are recorded by a filter once the plugin manager has made the change.
- **Refused:** downgrades (they restore a backup only one controller has) and upgrades of Jenkins itself (it comes with the image).
- **The CLI.** `install-plugin` from a file or URL bypasses the update center. A core extension point, `PluginArchiveListener`, is told before and after the CLI puts an archive in place: the `cluster` plugin refuses what the plugin manager would refuse (changes to the cluster plugins, any change in `fixed` mode), and records the others in the overlay.

### Through a load balancer
- **The progress page.** Update-center jobs are in the memory of the controller running them, so requests for the install progress (`pluginManager/updates/…`, `updateCenter/{body,installStatus,incompleteInstallStatus,api}`) are forwarded to that controller (JEP-254): the one named by a cookie set when the install was requested, otherwise the member reporting installs in progress with its heartbeat (JEP-255).
- **A notice on the plugin manager's pages.** A core hook, `PluginManager/notices.jelly`, includes each `PageDecorator`'s `pluginManager.jelly` on the plugin manager's pages (installed, available, updates, advanced, download progress). The `cluster` plugin's says that changes apply to every controller, with each controller's status, refreshed every 5 s.

### Apply
- **When.** Every controller reconciles with the overlay when it changes, every minute, and once it is up.
- **How:**
  - Missing plugins are fetched from blob storage, verified, and loaded live (`PluginManager.dynamicLoad`).
  - Upgrades, uninstalls, disables and plugins that cannot load live are written to `plugins/` and applied by a restart.
  - Files the cluster wrote are marked, so the image's entrypoint does not overwrite them on a container restart.
- **Rolling restarts.**
  - One controller at a time holds the restart lock in the store, safe-restarts, and releases the lock once it is back and up to date.
  - The lock is broken only if its holder has not come back within 10 minutes.
  - A controller never restarts twice for the same generation. It shows an administrative monitor instead.
- **Restarts are the cluster's.** The single-controller restart (the progress page's "restart when done", `safeRestart` from the plugin manager) is refused and hidden in `cluster` mode, since it would restart one controller outside the one-at-a-time order.
- **Status.** The cluster view shows each controller's state ("Up to date", "Applying 2 changes", "Restart pending", "Needs attention") and the list of changes: plugin, version, controller and time.
- **Update-site metadata is shared.**
  - Each controller downloads update-site and tool-installer metadata into `updates/` at startup and then daily, as on one controller.
  - A controller whose copy is newer than the cluster's publishes it: `cluster/updates.json` holds each file's time and checksum, and blob storage holds the file.
  - The others take it with the same modification time and reload it. So "Check now" on one controller is what every controller shows: plugin manager requests take newer data first, and a check shares its result before it responds.
  - A fresh home takes the cluster's copy before periodic work starts, so the cluster fetches about once a day rather than once per controller.
- **Modes.** `-Dio.jenkins.plugins.cluster.ClusterPluginSet.mode`:
  - `cluster`: the default
  - `fixed`: all changes refused; plugins only through the image
  - `local`: changes stay on the controller that made them

## Motivation
Found by hand on a cluster: a plugin installed from the Plugins page went to one controller only. Forms of its types then broke whenever their requests reached the other controller (fragments 404). Setting `UpdateCenter.never` does not stop installs.

## Reasoning
- **The update center's `install()` hook** sees every install, with the verified archive, without changing the update center.
- **An overlay with "newer wins"** lets image updates and runtime changes coexist.
- **Safe restarts**, which pause Pipelines, rather than waiting until idle. Busy controllers are never idle, and Pipelines are made to checkpoint and resume.
  - This needed a fix in JEP-252: a controller restarted with its home reclaims its builds' leases before resuming them. Otherwise another controller adopted them too.

## Backwards compatibility
- Without a store, nothing changes.
- With one, the Plugins page works as on one controller, apart from the refusals above.

## Security
Plugin changes still need `Overall/Administer`. Archives are verified by checksum before being installed elsewhere.

## Testing
- **Cluster plugins:** `ClusterPluginsTest` (version rule, actions, restart decisions, status wording), `ClusterPluginSetTest`, `ClusterUpdateSitesTest`, `UpdateCenterRoutingTest`.
- **Acceptance:** `validation/experiments/ET13-cluster-plugins.sh` in the cloud-native-jenkins repository runs the Helm chart with two controllers on a local kind cluster and checks:
  1. a live install with a dependency
  2. an upgrade by rolling restart, with a controller answering throughout and a Pipeline paused and resumed with its whole console
  3. a fresh pod catching up
  4. an uninstall
  5. the guards: downgrade, a cluster plugin, corrupt archive

  `validation/experiments/ET19-plugin-manager.sh` checks the notice on every plugin manager page, an install's progress page through a load balancer without stickiness, a CLI install from a file, and the refused single-controller restart. `ET21-browser.sh` checks the notice in a real browser.

## Prototype implementation
- **Cluster plugins** (`plugins/` in the cloud-native-jenkins repository) on prototype branch `cnj/t13-m1-cluster-plugins`: `ClusterPlugins`, `ClusterUpdateCenterConfiguration`, `ClusterPluginSet`; `ClusterUpdateSites` on `cnj/t14-m1-resilience`.
- **Cluster plugins**, later: `UpdateCenterRouting`, `ClusterPluginManagerNotice`, `ClusterCliPlugins`, `MemberStatus` on `cnj/t15-remaining`.
- **Core:** the restart reclaim on prototype branch `cnj/t12-m1-remainder` of Jenkins core; the notices hook and `PluginArchiveListener` on `cnj/t15-remaining`.
- **Findings:** `findings/ET13-cluster-plugins.md` and `ET19-plugin-manager.md`, a document in the cloud-native-jenkins repository on its prototype branches.

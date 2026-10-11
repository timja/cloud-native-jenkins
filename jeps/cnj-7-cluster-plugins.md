# JEP-CNJ-7: Cluster-wide plugin management

| | |
|---|---|
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-CNJ-1, JEP-CNJ-6 |

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
- The store plugin itself cannot be changed at runtime (409).

### Capture
- **Installs and upgrades.** Every install and upgrade through the update center (from an update site or an upload, dependencies included) passes through `UpdateCenter.UpdateCenterConfiguration.install()`. A wrapper records the archive in the overlay.
- **Uninstall, disable and enable** are recorded by a filter once the plugin manager has made the change.
- **Refused:** downgrades (they restore a backup only one controller has) and upgrades of Jenkins itself (it comes with the image).
- **Not covered:** CLI `install-plugin` from a file or URL bypasses the update center and stays local.

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
- **Status.** The cluster view shows each controller's state ("Up to date", "Applying 2 changes", "Restart pending", "Needs attention") and the list of changes: plugin, version, controller and time.
- **Modes.** `-Dio.jenkins.plugins.cnj.store.ClusterPluginSet.mode`:
  - `cluster`: the default
  - `fixed`: all changes refused; plugins only through the image
  - `local`: changes stay on the controller that made them

## Motivation
Found by hand on a cluster: a plugin installed from the Plugins page went to one controller only. Forms of its types then broke whenever their requests reached the other controller (fragments 404). Setting `UpdateCenter.never` does not stop installs.

## Reasoning
- **The update center's `install()` hook** sees every install, with the verified archive, without changing the update center.
- **An overlay with "newer wins"** lets image updates and runtime changes coexist.
- **Safe restarts**, which pause Pipelines, rather than waiting until idle. Busy controllers are never idle, and Pipelines are made to checkpoint and resume.
  - This needed a fix in JEP-CNJ-3: a controller restarted with its home reclaims its builds' leases before resuming them. Otherwise another controller adopted them too.

## Backwards compatibility
- Without a store, nothing changes.
- With one, the Plugins page works as on one controller, apart from the refusals above.

## Security
Plugin changes still need `Overall/Administer`. Archives are verified by checksum before being installed elsewhere.

## Testing
- **Store plugin:** `ClusterPluginsTest` (version rule, actions, restart decisions, status wording), `ClusterPluginSetTest`.
- **Acceptance:** ET13, on kind:
  1. a live install with a dependency
  2. an upgrade by rolling restart, with a controller answering throughout and a Pipeline paused and resumed with its whole console
  3. a fresh pod catching up
  4. an uninstall
  5. the guards: downgrade, store plugin, corrupt archive

## Prototype implementation
- **Store plugin** on `cnj/t13-m1-cluster-plugins`: `ClusterPlugins`, `ClusterUpdateCenterConfiguration`, `ClusterPluginSet`.
- **Core:** the restart reclaim on `cnj/t12-m1-remainder`.
- **Findings:** `findings/ET13-cluster-plugins.md`.

# Target 13: Cluster-wide plugin installs

| | |
|---|---|
| **Roadmap tier** | S0 / Ops (plugin set) |
| **Status** | **Done** (2026-10-11): ET13 passes 22/22, see [13-progress](13-progress.md) |
| **Depends on** | T3 (blob storage), T4 (Pipelines across restarts), T9 (chart) |

## Goal
A plugin installed, upgraded, uninstalled, disabled or enabled on one controller is applied on every controller, and again on any controller that starts with an empty home. When a change needs a restart, the controllers restart one at a time (safe restarts: Pipelines pause and resume), so the cluster keeps serving.

**Why:** by hand on kind, a plugin installed from the Plugins page landed on one controller only, and its forms broke whenever their requests reached the other. The Target 12 stopgap refused all plugin changes.

## Design
- **The cluster's plugin set:**
  - An overlay on the image's plugins, in the store (`cluster/plugins.json`), with the archives in blob storage.
  - An install applies only where the local version is missing or older: a newer image wins.
- **Capture:**
  - installs and upgrades: the update center's `UpdateCenterConfiguration.install` (every update-site, upload and dependency install)
  - uninstall, disable and enable: a filter on the plugin manager's actions
  - refused: downgrades, core upgrades and changes to the store plugin
- **Apply:** every controller reconciles with the overlay: plugins are loaded live where possible, otherwise written to `plugins/` and applied by a restart.
- **Restarts:** one controller at a time (a lock in the store) performs a safe restart, and releases the lock once it is back and up to date. The same generation is never restarted for twice.
- **Pipelines across those restarts** (a core fix): a controller restarting with its home takes over the leases of the builds it was running before resuming them, so no other controller adopts them too.

## E-T13 acceptance experiment
1. Upload a plugin with a dependency on one controller: both are active on both controllers, live, with no restart.
2. Upgrade it from the update center on the other controller while a Pipeline runs: rolling restart, a controller answering throughout, the Pipeline paused and resumed with its whole console.
3. Replace a controller with a fresh pod: it has the same plugins.
4. Uninstall: gone from both, by a rolling restart.
5. Guards: downgrades and store plugin changes refused; a corrupt archive is not installed and causes no restart.

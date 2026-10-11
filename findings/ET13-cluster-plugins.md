# ET13: Target 13 acceptance (cluster-wide plugin installs, upgrades and uninstalls)

**Result: PASS**, 22/22 checks, in two consecutive runs on the final build. Raw evidence from the last run: `raw/ET13/`
- `results.md`
- the cluster's plugin set (`cluster-plugins.json`)
- the availability probes during each rolling restart
- the running Pipeline's console on both controllers
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET13-cluster-plugins.sh`
- **Setup:** the chart on kind, two controllers behind the non-sticky service, emptyDir homes. Plugins come from the update center (an upgrade) and an upload (an older version fetched from updates.jenkins.io).
- **Build:**
  - core `cnj/t12-m1-remainder` (including the reclaim fix below)
  - plugin on `cnj/t13-m1-cluster-plugins`
  - kubernetes `cnj/t9-m2-cluster-cap`

| Criterion | Result |
|---|---|
| 1 Live install | ✅ `build-timeout` 1.40 was uploaded on plug-0. It and its dependency `token-macro` (resolved from the update center) became active on both controllers, loaded live: neither restarted. The plugin's descriptor exists on plug-1, so its forms and validation work whichever controller serves them (the bug found by hand on kind) |
| 2 Upgrade with a rolling restart | ✅ `build-timeout` was upgraded to 1.41 from the update center on plug-1, while a 2-minute Pipeline ran on plug-0. Both controllers restarted, one at a time: an availability probe every 2 s always had a controller answering, and the second went down only after the first was back. The Pipeline paused for the restart, resumed, and completed with every tick once and in order. The restart lock was released, and both members report generation 3 |
| 3 Fresh controller | ✅ plug-1 was replaced by a fresh pod (empty home). It came up with `build-timeout` 1.41, with exactly the same plugins and versions as plug-0 |
| 4 Uninstall | ✅ `build-timeout` uninstalled on plug-0 is gone from both, again by a rolling restart with a controller answering throughout |
| 5 Guards | ✅ A downgrade and any change to the store plugin are refused (409). A deliberately corrupt entry (wrong checksum) is not installed, causes no restart, and is logged |

## How it works
- **The cluster's plugin set (`ClusterPlugins`, store plugin):**
  - Store key `cluster/plugins.json` holds an overlay on the image's plugins: per plugin, `installed` (version, sha256, archive in blob storage), `uninstalled` or `disabled`, with a generation.
  - Archives are the exact bytes the installing controller verified, under `cluster-plugins/<name>/<sha256>.jpi`, so no other controller needs the update center.
  - An installed entry applies only where the local version is missing or older, so a newer image wins and nothing is downgraded.
- **Capture:**
  - Every install and upgrade through the update center (from an update site or an upload, dependencies included) passes through `UpdateCenterConfiguration.install`, which `ClusterUpdateCenterConfiguration` wraps.
  - Uninstall, disable and enable are recorded by the `ClusterPluginSet` filter once the plugin manager has done them.
- **Apply:** every controller reconciles with the overlay:
  - when it changes (store events), every minute, and once up (catching up after a fresh start)
  - new plugins are loaded live
  - anything else (upgrades, removals, plugins that cannot load live) is written to `plugins/` and applied by a restart
  - files the cluster wrote are marked `.cluster`, so the image's entrypoint leaves them alone on a container restart
- **Rolling restarts:**
  - One controller at a time holds `cluster/plugins.restart` and performs a safe restart: builds that cannot survive a restart are waited for, while Pipelines pause and resume.
  - The lock is released once the restarted controller has reconciled, so the next can go.
  - A controller never restarts twice for the same generation; it shows an administrative monitor instead.
  - The cluster view shows each member's applied generation and whether a restart is pending.
- **Modes:** `-Dio.jenkins.plugins.cnj.store.ClusterPluginSet.mode` = `cluster` (default), `fixed` (all changes refused, the stopgap of Target 12), or `local`.
- **Not covered:**
  - CLI `install-plugin` from a file or URL bypasses the update center and stays local.
  - Upgrades of Jenkins itself are refused: core comes with the image.

## Bugs found by ET13, now fixed
- **Two controllers ran the same Pipeline after a restart that kept the home** (a Target 4 bug, core).
  - **Cause:** the restarted controller resumed the build from its home under its new origin without the lease. The placeholder still named its previous incarnation, which is no longer a member, so another controller adopted it too. The logs were rewritten by both, each recorded its own result, and on kind the published console was truncated.
  - **Why only now:** every earlier failover started the replacement with an empty home.
  - **Fix:** before such a build is loaded, the controller takes its lease over from its previous incarnation (same name, another origin), as an adopter would. If another controller holds it, the local copy is not loaded.
  - **Reproduced and verified** with a compose override giving controller A a persistent home: before the fix A finished SUCCESS and B finished FAILURE; after it, a single execution, SUCCESS with every tick on both controllers.
- **An initializer after `COMPLETED` kept Jenkins from completing its startup** (a `SEVERE` at boot). The first reconcile is now scheduled instead.
- **Rejected approach:** waiting until no build runs before restarting. On a busy controller that never happens, and Pipelines are meant to checkpoint and resume (the user's call).
- **Harness:**
  - the upload form needs its URL field even when empty
  - bare waits and the probe ran under `set -e`

## Run history
| Run | Result | What it found |
|---|---|---|
| 1 | stopped at 2 | the `COMPLETED` initializer; the upload form; `set -e` |
| 2-3 | 18/19 | the Pipeline across the rolling restart: a truncated console (two owners after a restart with the home kept) |
| 4 | (stopped) | wait-for-idle restarts, replaced by safe restarts plus the reclaim fix |
| 5-6 | 22/22 ×2 | final build |

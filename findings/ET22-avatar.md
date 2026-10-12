# ET22: Target 15 item 27 (plugin files in user directories: the avatar plugin)

**Result: PASS**, every check. Raw evidence: `raw/ET22/`
- `results.md`
- the images each controller served
- controller logs

- **Script:** `validation/experiments/ET22-avatar.sh`
- **Setup:**
  - two tmpfs controllers, Postgres, Azurite, token auth
  - the avatar plugin from the update center, in the harness image
  - opt-in sharing of plugin files (`-Djenkins.persistence.ItemFiles.enabled=true`; the chart's `store.sharePluginFiles`), every 5 s for the test

| Scenario | Result |
|---|---|
| 1 An avatar set for admin on A, as the plugin's upload does: the image written next to the user's `config.xml` with file APIs, the property saved with the user | ✅ A serves it; B serves the same image within 60 s, and its API names an avatar |
| 2 B replaced by a fresh controller (empty home) | ✅ the fresh B serves it |
| 3 The image changed on B | ✅ A serves the new image |

## What was wrong before
The avatar plugin keeps the user's image at `users/<id>/avatar.<ext>`, written with `FileOutputStream`. The property saying that a user has an avatar, and its type, is in the user's `config.xml`, which the store shares. The image was not shared. So another controller, or a fresh home, had the property but no image (from the plugin's source: it then logs that the image does not exist, and serves nothing).

## How it works now
The opt-in sharing of plugin files in item directories (`ItemFiles`, item 6) covers user directories too. Every minute, each user's directory is compared with what was last shared. Changed and removed files (up to 1 MiB) are shared through a manifest per directory (`users/<id>/item-files.json`) and blob storage. Other controllers write them into their own copy of the directory, and a fresh home takes them at startup. Files core declares per controller are left out (`users/users.xml`, each user's `apiTokenStats.xml`). No change to the plugin is needed.

**It is opt-in** because it scans every item and user directory, and core does not know what the files mean. The last writer of a file wins.

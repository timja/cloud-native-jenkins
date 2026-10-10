# Target 11: Cross-cutting completion (concurrent edits, inventory check, ecosystem report)

| | |
|---|---|
| **Roadmap tier** | R1 (concurrent writers), Track S (concurrent edits, ET6 10), X (inventory regression check, ecosystem migration) |
| **Status** | Planned |
| **Depends on** | T1, T6 |

## Goal
- **Concurrent edits:** two controllers saving different parts of one file at the same moment both keep their change, and only edits of the same part are last-writer-wins (and counted).
- **Inventory:** a CI check fails if a change starts writing durable state to a controller's home.
- **Ecosystem:** administrators see which installed plugins still keep state the cluster does not share.

## Design
### 1. Three-way XML merge on conflict (core, `XmlFile` routing)
- **When it applies:** a write whose compare-and-set fails (the entry changed since this controller read it) is merged instead of overwriting.
- **The inputs:**
  - **base:** the bytes this controller last read, kept with the version
  - **mine:** the new bytes
  - **theirs:** the stored bytes
- **The merge works on the root element's children, by element name and position** (lists of identical names are compared as wholes):
  - a child only I changed takes mine
  - a child only they changed takes theirs
  - children both changed alike are kept
  - children both changed differently take mine, and count a conflict
- **Afterwards:** the merged bytes are written with compare-and-set (and retried), and the object in memory is reloaded from them, so it also holds their change.
- **Scope:**
  - It covers `config.xml`, descriptors, users, nodes and job configuration: everything routed through `XmlFile`.
  - Files that are not XML documents of that shape (or fail to parse) keep last-writer-wins.

### 2. Inventory regression check
- **The check:** `validation/experiments/inventory-check.sh` runs the E1 + E9 scenario against a two-controller cluster with inotify on (`CNJ_INOTIFY=1`). It classifies every path written under the home with `analyze_inventory.py`, and fails if any written path is in a category of durable state that the store should hold.
- **Allowed:** caches, workspaces and the local copies of shared state (materialised builds, plugin unpacking, logs being shipped).
- **CI:** a GitHub Actions workflow runs it (to be enabled when pushed; local only for now).

### 3. Ecosystem report (store plugin)
- **The monitor:** an administrative monitor and `/manage/cluster/compatibility` list what this controller writes under its home that the store does not own:
  - files written after startup, outside the allowed categories, grouped by path pattern
  - the plugin owning each, from the path or the stack of the writing call (`XmlFile` writers outside the routed keys)
  - with E7's hit list as the reference
- **What it gives administrators:** a list of the plugins to check before relying on HA, and the evidence to file issues upstream.

## Milestones
| # | Repo | Branch | Based on | Scope |
|---|---|---|---|---|
| M1 | core | `cnj/t11-m1-xml-merge` | `cnj/t9-m1-provisioning-demand` | three-way merge on conflict; tests |
| M2 | cloud-native-jenkins | `cnj/t11-m2-cross-cutting` | `cnj/t10-m2-et10` | inventory check, compatibility report, `ET11` |

## E-T11 acceptance experiment
1. **Concurrent edits (ET6 10, strictly):** in 10 rounds, A and B save different global settings at the same moment. Both survive every time, on both controllers.
2. **The same setting:** A and B set the system message at the same moment. One value wins on both, and the conflict is counted.
3. **Job configuration:** A changes a job's description while B changes its triggers. Both survive.
4. **Inventory check:** it passes on the final build. A deliberately bad change (a plugin writing `foo.xml` under the home at runtime, outside the store) makes it fail, naming the path.
5. **Compatibility report:** with that plugin installed, the report lists the path and the plugin.

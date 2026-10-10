# E7: How many plugins assume file-based, single-JVM storage?

- **Tool:** [`jenkins-infra/usage-in-plugins`](https://github.com/jenkins-infra/usage-in-plugins) (ASM bytecode scan of the latest release of every plugin in the update centre), run 2026-10-10.
- **Patterns:** `raw/E7/methods.txt` and `raw/E7/classes.txt`. They were first validated on git, credentials, junit and workflow-cps (`raw/E7/validate/`).
- **Raw reports:** `raw/E7/usage-by-api.json`, `usage-by-plugin.json`, `summary.txt`, `by-tier.txt`.

**Headline: 513 of 2,117 plugins (24%) match at least one pattern, including 99 of the 358 plugins in the plugin BOM.**

| Roadmap tier | API family | Plugins | In BOM | Examples (BOM) |
|---|---|---|---|---|
| S / R1 | `Jenkins#getRootDir` / `Hudson#getRootDir` (files directly under `JENKINS_HOME`) | 166 | | |
| R1 risk | `XmlFile#getFile` (bypasses any store) | 42 | 10 | cloud-stats, config-file-provider, extra-columns, github, job-dsl, jobConfigHistory, junit, promoted-builds, saml, thinBackup |
| R1 | constructs or uses `hudson.XmlFile` directly | 171 | | |
| R1 | `TextFile` / `AtomicFileWriter` | 15 | | branch-api, junit |
| R1 | item dirs: `Job`/`AbstractItem`/`AbstractProject#getRootDir`, `getConfigFile`, `Items.getConfigFile` | 88 | | |
| R5 | build dirs: `Run`/`AbstractBuild`/`WorkflowRun#getRootDir`, `getArtifactsDir`, `getBuildDir` | 212 | 32 | junit, workflow-api, workflow-job, warnings-ng, coverage, copyartifact, timestamper, pipeline-graph-view, htmlpublisher, … |
| R5 | `Run`/`AbstractBuild#getLogFile` | 24 | 3 | google-storage-plugin, maven-plugin, naginator |
| R2 | `PeriodicWork`, `AsyncPeriodicWork`, `AperiodicWork`, `AsyncAperiodicWork`, `jenkins.util.Timer` | 131 | 37 | kubernetes, ec2, docker-plugin, branch-api, github, credentials, lockable-resources, workflow-cps, durable-task, metrics, prometheus, … |

Most-used single APIs:
- `hudson.XmlFile`: 171
- `Jenkins#getRootDir`: 150
- `Run#getRootDir`: 132
- `AbstractBuild#getRootDir`: 68
- `jenkins.util.Timer`: 59
- `AsyncPeriodicWork`: 58
- `PeriodicWork`: 49
- `XmlFile#getFile`: 42

The plugins using the widest range of these APIs are disk-usage (11 distinct), propelo-job-reporter (8), support-core (7) and thinBackup (7).

## Interpretation
- **The persistence SPI has to be transparent to `XmlFile` users.** 171 plugins build their own `XmlFile`, usually `new XmlFile(new File(Jenkins.get().getRootDir(), …))`. Routing inside `XmlFile` (Target 1 §3) covers them without plugin changes, **as long as** routing is path-based on the root dir rather than opt-in.
- **`getFile()` is the main leak.** 42 plugins (10 in BOM) reach the raw file. Target 1's read-only materialised cache keeps reads working; writes through it are lost, so these 10 BOM plugins are the first PR targets once the store owns their keys.
- **Build directories are the biggest ecosystem dependency.** 212 plugins write or read files in the build dir, including the core Pipeline plugins. R5 needs an attachment/blob API for "per-build plugin data" (think `VirtualFile` for writes) before build records can leave disk. A full SPI cutover is not realistic; the plan has to be **hybrid with migration**.
- **Periodic work is everywhere, and much of it is correctly per-JVM.** Examples are durable-task's channel watchdogs and per-agent monitors. R2 needs a way to *declare* cluster-singleton work (an annotation or `ExtensionFilter`) rather than electing a leader for everything. Each of the 37 BOM plugins needs classifying.

## Blind spots
- Bytecode only. Jelly, Groovy, reflection and string class names are not seen, so zero hits means "no bytecode reference", not "no usage".
- Calls through subclasses not listed in `methods.txt` are missed: e.g. `FreeStyleBuild#getRootDir` is listed, but third-party `Run` subclasses are not.
- One update-centre snapshot. This is the latest release of each plugin, not every version.

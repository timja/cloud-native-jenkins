## Phases

- +   0.00s start
- +   5.71s init: Started initialization
- +   7.52s init: Listed all plugins
- +   8.74s init: Prepared all plugins
- +   8.75s init: Started all plugins
- +   8.76s init: Augmented all extensions
- +   9.23s init: System config loaded
- +   9.73s init: Loaded all jobs
- +   9.73s init: System config adapted
- +   9.74s init: Configuration for all jobs updated
- +  10.26s init: Completed initialization
- +  11.07s booted
- +  11.45s token-created
- +  11.52s token-used
- +  11.80s global-config-saved
- +  15.02s agent-online
- +  15.88s jobs-created
- +  34.23s builds-done
- +  43.16s input-paused
- +  56.32s cron-fired
- +  56.62s init: Started termination
- +  56.66s init: Completed termination
- +  57.35s stopped
- +  60.83s init: Started initialization
- +  60.88s init: Listed all plugins
- +  62.18s init: Prepared all plugins
- +  62.19s init: Started all plugins
- +  62.19s init: Augmented all extensions
- +  62.53s init: System config loaded
- +  62.96s init: System config adapted
- +  62.99s init: Loaded all jobs
- +  62.99s init: Configuration for all jobs updated
- +  63.44s init: Completed initialization
- +  64.92s restarted
- +  72.67s input-done

## State categories touched

| Category | Tier | Paths written | Paths opened | First touched (after marker) | Example paths written |
|---|---|---|---|---|---|
| Extracted webapp (war/) | immutable | 358 | 353 | start |  |
| Plugin binaries (plugins/) | immutable | 1474 | 1646 | start |  |
| JVM/OS caches | ephemeral | 22 | 11 | start | `.cache/fontconfig/3830d5c3ddfd5cd38a049b759396e72e-le64.cache-9`, `.cache/fontconfig/3830d5c3ddfd5cd38a049b759396e72e-le64.cache-9.LCK`, `.cache/fontconfig/3830d5c3ddfd5cd38a049b759396e72e-le64.cache-9.NEW`, `.cache/fontconfig/3830d5c3ddfd5cd38a049b759396e72e-le64.cache-9.TMP-YFOcX6` |
| Install/boot markers | ephemeral | 6 | 6 | start | `.lastStarted`, `copy_reference_file.log`, `failed-boot-attempts.txt`, `jenkins.install.InstallUtil.lastExecVersion` |
| Secrets & instance identity | T1 | 12 | 12 | start | `identity.key.enc`, `secret.key`, `secret.key.not-so-secret`, `secrets/hudson.console.ConsoleNote.MAC` |
| Global config: Jenkins config.xml | T1 | 1 | 1 | init: System config loaded | `config.xml` |
| System credentials store | T2 | 1 | 1 | init: System config loaded | `credentials.xml` |
| Queue persistence | T5 | 2 | 1 | init: Completed termination | `queue.xml`, `queue.xml.bak` |
| Running Pipeline index | T4 | 1 | 1 | jobs-created | `org.jenkinsci.plugins.workflow.flow.FlowExecutionList.xml` |
| Update center cache | ephemeral | 2 | 2 | init: Augmented all extensions | `hudson.model.UpdateCenter.xml`, `updates/hudson.tasks.Maven.MavenInstaller` |
| Global config: descriptor/plugin XML | T1 | 8 | 8 | init: Augmented all extensions | `hudson.plugins.copyartifact.CopyArtifactConfiguration.xml`, `hudson.plugins.git.GitTool.xml`, `io.jenkins.plugins.junit.storage.JunitTestResultStorageConfiguration.xml`, `jenkins.model.ArtifactManagerConfiguration.xml` |
| Pipeline runtime state (flow nodes, program.dat) | T4 | 32 | 32 | jobs-created | `jobs/*/builds/N/program.dat`, `jobs/*/builds/N/workflow-completed/flowNodeStore.xml`, `jobs/*/builds/N/workflow/10.xml`, `jobs/*/builds/N/workflow/11.xml` |
| Build logs | T3 | 2 | 2 | jobs-created | `jobs/*/builds/N/log`, `jobs/*/builds/N/log-index` |
| Build records | T3 | 2 | 2 | jobs-created | `jobs/*/builds/N/build.xml`, `jobs/*/builds/N/changelog.xml` |
| Build index (permalinks) | T3 | 1 | 1 | jobs-created | `jobs/*/builds/permalinks` |
| Build numbering | T3 | 1 | 1 | jobs-created | `jobs/*/nextBuildNumber` |
| Item config (jobs/folders) | T2 | 1 | 1 | agent-online | `jobs/*/config.xml` |
| Fingerprints | T3 | 1 | 1 | jobs-created | `fingerprints/**` |
| Node config | T2 | 1 | 1 | init: System config loaded | `nodes/agent1/config.xml` |
| Users (config, API tokens, id mapping) | T2 | 2 | 2 | init: System config loaded | `users/*/apiTokenStats.xml`, `users/*/config.xml` |
| Task logs (agents, tasks) | T3 | 2 | 1 | global-config-saved | `logs/slaves/agent1/slave.log`, `logs/slaves/agent1/slave.log.1` |
| Built-in node workspaces | T6 | 2 | 2 | jobs-created | `workspace/freestyle/out/app.txt`, `workspace/freestyle/reports/TEST-fs.xml` |
| userContent / caches | T6 | 1 | 1 | init: Configuration for all jobs updated | `userContent/readme.txt` |

## Durable state read before `Started all plugins`

Core-only reads happen before any plugin class is loaded, so a backend serving them cannot be a plugin.

**core-only (before Listed all plugins):**

- `secret.key`
- `secret.key.not-so-secret`
- `secrets/master.key`

**plugin load/start (Listed → Started all plugins):**

- `credentials.xml`

## Durable state rewritten during a plain restart

Every replica booting against shared state would rewrite these (CasC re-apply, migrations, monitors).

- `config.xml`
- `credentials.xml`
- `io.jenkins.plugins.junit.storage.JunitTestResultStorageConfiguration.xml`
- `jenkins.model.ArtifactManagerConfiguration.xml`
- `jenkins.model.JenkinsLocationConfiguration.xml`
- `jobs/input/builds/1/build.xml`
- `jobs/input/builds/1/log`
- `jobs/input/builds/1/program.dat`
- `nodeMonitors.xml`
- `nodes/agent1/config.xml`
- `org.jenkinsci.plugins.database.GlobalDatabaseConfiguration.xml`
- `queue.xml.bak`
- `users/*/config.xml`

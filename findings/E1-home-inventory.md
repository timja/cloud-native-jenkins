# E1: What does Jenkins keep in JENKINS_HOME?

- **Script:** `validation/experiments/E1-home-inventory.sh`
- **Analyser:** `validation/experiments/analyze_inventory.py`
- **Raw evidence:** `raw/E1/`
  - `inotify.log`: 31k filesystem events, ms timestamps
  - `analysis.md`
  - `home-files.txt`
  - `jenkins.log`
- **Run:** 2026-10-10 on `jenkins/jenkins:2.580.1-jdk21` plus `validation/jenkins/plugins.txt`. The plugins included junit-sql-storage on Postgres and azure-artifact-manager on Azurite.

## Method
1. Start `inotifywait -m -r` on `JENKINS_HOME` before Jenkins starts, recording every create, modify, move, delete and open with a millisecond timestamp.
2. Run the scenario. Each step drops a timestamped marker:
   1. boot
   2. API token create and use
   3. global config save
   4. inbound WebSocket agent connects
   5. create a freestyle job, a pipeline job, an `input` pipeline and a cron pipeline
   6. build them, with archive + fingerprint, stash/unstash, junit and `withCredentials`
   7. wait for a cron tick
   8. restart while the input build is paused, then approve the input
3. Classify each path by state category. Line events up with the `InitMilestone` log lines to see what is read before plugins load.

## Results
After the scenario, everything except `war/` and `plugins/` came to 584 KB in 79 files. The external stores did their job:
- Artifacts and stashes left nothing under `builds/N/archive`; they went to Azurite.
- No `junitResult.xml` was written; results went to Postgres.

Everything else is still local:

| Category | Files written in scenario | Notes |
|---|---|---|
| Secrets & identity | `secret.key`, `secret.key.not-so-secret`, 9 × `secrets/*`, `identity.key.enc` | `master.key` and `secret.key` are created or read **before plugins are even listed** |
| Global config | `config.xml` plus 8 descriptor XMLs | first read at `System config loaded`, which is after plugins start |
| System credentials | `credentials.xml` | read while plugins load (credentials plugin) |
| Item config | `jobs/*/config.xml` | |
| Build numbering | `jobs/*/nextBuildNumber` | |
| Build records | `builds/N/build.xml`, `changelog.xml`, `builds/permalinks` | |
| Build logs | `builds/N/log`, `log-index` | used for Pipeline too: no `LogStorageFactory` installed |
| Pipeline runtime | `program.dat`, `workflow/*.xml` → `workflow-completed/flowNodeStore.xml` | |
| Running-pipeline index | `org.jenkinsci.plugins.workflow.flow.FlowExecutionList.xml` | |
| Queue | `queue.xml` / `queue.xml.bak` | written at termination |
| Fingerprints | `fingerprints/**` | written even though artifacts are external |
| Nodes | `nodes/agent1/config.xml` | |
| Users | `users/<id>_<hash>/config.xml`, `apiTokenStats.xml` | |
| Task logs | `logs/slaves/agent1/slave.log*` | |
| Workspaces | `workspace/freestyle/**` | only because the freestyle job ran on the built-in node |

Ephemeral or immutable paths that need no externalising: `war/`, `plugins/`, `.cache/`, `.java/`, `updates/`, `hudson.model.UpdateCenter.xml`, `copy_reference_file.log`, `jenkins.install.*`, `.lastStarted` and `failed-boot-attempts.txt`.

## Findings that shape the design
1. **Only key material is read before plugins exist.** `secret.key`, `secret.key.not-so-secret` and `secrets/master.key` are touched before `Listed all plugins`. `credentials.xml` is read while plugins load. Jenkins `config.xml` and every descriptor XML are first read at or after `System config loaded`. So the storage backend has to be in place before plugins load, at least for secrets, but global config could in principle be served by something that starts early in the plugin phase. See `raw/E1/analysis.md`, section "Durable state read before".
2. **A plain restart rewrites durable state.** With no user action, the restart rewrote:
   - `config.xml`, `credentials.xml`
   - five descriptor XMLs (re-applied by JCasC)
   - `nodes/agent1/config.xml`, `users/*/config.xml`
   - `nodeMonitors.xml`
   - the paused build's `build.xml` and `program.dat`

   With N replicas sharing one store, every replica boot is a write burst against shared keys. The store needs cheap idempotent writes (skip if the content is unchanged), or CasC needs to stop re-saving unchanged config.
3. **Freestyle and Pipeline both leave four kinds of per-build file locally:** `build.xml`, `log`, `nextBuildNumber` and `permalinks`. Externalising builds is a core change, not a plugin one.
4. **Instance identity is stored twice:** `identity.key.enc` (legacy) and `secrets/…InstanceIdentity.KEY`. Both must move with the secrets.
5. **A fresh home logs a WARNING** `User$AllUsers#scanAll: entry for admin … duplicates one seen earlier`. JCasC creating a user can produce two user directories. Worth re-checking once users move into the store.

# Cloud-native Jenkins

This repo revives the [pluggable storage](https://www.jenkins.io/doc/book/using/pluggable-storage/) effort, with a concrete goal: **several Jenkins controllers running active/active with ephemeral `JENKINS_HOME`s, sharing state through external services.**

- [ROADMAP.md](ROADMAP.md): everything that has to change, in tiers, each item tied to evidence
- [targets/01-shared-item-config.md](targets/01-shared-item-config.md): the first target (item config created and changed at runtime), planned in detail
- [targets/90-static-state-store.md](targets/90-static-state-store.md): deferred, static state in the store; the S0 workaround covers it for now
- [findings/](findings/): what running real Jenkins instances showed (E1–E9); raw evidence in `findings/raw/`
- [UPSTREAMING.md](UPSTREAMING.md): every pull request, and the order for taking them upstream

**Moved out (2026-10-14):**
- **The code:** the Cluster plugins, the Helm chart and the harness (`validation/`, the experiment scripts that produce the findings) are in [timja/cluster-plugin](https://github.com/timja/cluster-plugin), with their history.
- **The JEPs** (JEP-250 to JEP-257) are AsciiDoc drafts in [timja/jep#4](https://github.com/timja/jep/pull/4), for jenkinsci/jep.
- **What stays here:** the roadmap, the targets and the findings.
- **Record evidence here:** run the experiments in cluster-plugin with `FINDINGS_DIR` pointing at this repository's `findings/`.

## Decision record
**Fully externalised storage, no shared filesystem.** Decided 2026-10-10. Alternatives considered:

| Option | Why not (for now) |
|---|---|
| Shared `JENKINS_HOME` (RWX volume) + coordination layer, like CloudBees CI HA | Smallest code change, and core already has Beta hooks for it. But E3 shows every in-memory model goes stale with no invalidation, and it ties HA to a shared POSIX filesystem. We may still reuse those hooks (`BuildNumberAssigner`, `QueueIdStrategy`, `RunListener.allowLoad`, `recognizeNumber`, `ComputerSource`) |
| Active/passive only | No horizontal scaling. E4 shows cold failover already works on a shared disk today |

**Refocus on runtime state.** Decided 2026-10-10. Static, admin-changed state (global config, credentials, key material) is handled short-term by identical JCasC plus a seeded key set (S0, validated in E8). The roadmap focuses on state that builds and events change, and that can't be synced that way (E9):
- job config, including multibranch-generated jobs
- build scheduling
- running builds
- Pipeline data
- build history

Lessons from the first attempt (Cloud Native SIG, 2018–2020; JEP-202/207/210/212/213/226):
- Narrow seams shipped: artifacts, Pipeline logs, fingerprints, test results.
- Cross-cutting abstractions stalled: configuration (JEP-213) and core build logs (JEP-207).
- The work died when staffing ended.

This plan therefore lands in small, upstreamable Beta PRs, and widens one seam (the persistence SPI) step by step rather than all at once.

## Running the harness
The harness is in [timja/cluster-plugin](https://github.com/timja/cluster-plugin) (`validation/`); see its README. The early experiments E1–E9 (stock Jenkins, `validation/compose.yaml`) are there too. To record their evidence in this repository:

```sh
cd ../cluster-plugin/validation
FINDINGS_DIR=../../cloud-native-jenkins/findings bash experiments/E1-home-inventory.sh
```

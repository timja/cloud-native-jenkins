# Cloud-native Jenkins

This repo revives the [pluggable storage](https://www.jenkins.io/doc/book/using/pluggable-storage/) effort, with a concrete goal: **several Jenkins controllers running active/active with ephemeral `JENKINS_HOME`s, sharing state through external services.**

- [ROADMAP.md](ROADMAP.md): everything that has to change, in tiers, each item tied to evidence
- [targets/01-persistence-spi-global-config.md](targets/01-persistence-spi-global-config.md): the first target, planned in detail
- [findings/](findings/): what running real Jenkins instances showed (E1–E7); raw evidence in `findings/raw/`
- [validation/](validation/): the harness and experiment scripts that produce the findings

## Decision record
**Fully externalised storage, no shared filesystem.** Decided 2026-10-10. Alternatives considered:

| Option | Why not (for now) |
|---|---|
| Shared `JENKINS_HOME` (RWX volume) + coordination layer, like CloudBees CI HA | Smallest code change, and core already has Beta hooks for it. But E3 shows every in-memory model goes stale with no invalidation, and it ties HA to a shared POSIX filesystem. We may still reuse those hooks (`BuildNumberAssigner`, `QueueIdStrategy`, `RunListener.allowLoad`, `recognizeNumber`, `ComputerSource`) |
| Active/passive only | No horizontal scaling. E4 shows cold failover already works on a shared disk today |

Lessons from the first attempt (Cloud Native SIG, 2018–2020; JEP-202/207/210/212/213/226):
- Narrow seams shipped: artifacts, Pipeline logs, fingerprints, test results.
- Cross-cutting abstractions stalled: configuration (JEP-213) and core build logs (JEP-207).
- The work died when staffing ended.

This plan therefore lands in small, upstreamable Beta PRs, and widens one seam (the persistence SPI) step by step rather than all at once.

## Running the harness
Requirements: Docker with about 6 GB free for images; bash; python3.

```sh
cd validation
docker compose build jenkins-a
bash experiments/E1-home-inventory.sh          # ~2 min; writes findings/raw/E1
python3 -I experiments/analyze_inventory.py ../findings/raw/E1
bash experiments/E2-disposable-controller.sh   # run right after E1
bash experiments/E3-shared-home.sh             # ~5 min
bash experiments/E4-pipeline-failover.sh       # ~4 min
bash experiments/E5-stateless-http.sh
bash experiments/E6-bootstrap-seam.sh
```

Endpoints:
- `http://localhost:8081` (jenkins-a), `:8082` (jenkins-b), `:8080` (non-sticky LB)
- login `admin` / `admin`

Knobs (see `compose.yaml`):
- `HOME_A` / `HOME_B`: which volume is `JENKINS_HOME`; set both to `home-shared` for shared-home experiments
- `JENKINS_WAR_DIR`: run a locally built `jenkins.war`
- `CNJ_INOTIFY=1`: record filesystem events

The Azure artifact manager runs against Azurite, using the well-known development key; JUnit results go to Postgres via junit-sql-storage. MinIO was the original plan but its images are no longer pullable.

# Draft JEPs: multi-controller, cloud-native Jenkins

Drafts of Jenkins Enhancement Proposals for the work in this repository. There is one per challenge in running N controllers active/active with ephemeral homes and all state in external services. They follow the [JEP template](https://github.com/jenkinsci/jep/blob/master/jep/0000/README.adoc) loosely, in Markdown.

Nothing here has been submitted. Numbers are placeholders (`JEP-CNJ-n`) until they are. Every claim about behaviour is backed by an acceptance experiment in `validation/experiments/ET*.sh`; its findings are on the branch named in each JEP's *Prototype* section.

| Draft | Title | Covers (targets) |
|---|---|---|
| [JEP-CNJ-1](cnj-1-persistence-store.md) | Pluggable persistence store for `JENKINS_HOME` state | T1, T6, T11 |
| [JEP-CNJ-2](cnj-2-shared-build-history.md) | Shared build records, logs and history | T3, T8, T12, T14 (logs) |
| [JEP-CNJ-3](cnj-3-running-build-ownership.md) | Ownership of running builds: leases, fencing, adoption | T4, T12, T13, T14 |
| [JEP-CNJ-4](cnj-4-cluster-scheduling.md) | Cluster scheduling: build numbers, triggers, a shared queue | T2, T5 |
| [JEP-CNJ-5](cnj-5-request-routing.md) | Request routing and stateless sessions | T6, T7, T14 (REST) |
| [JEP-CNJ-6](cnj-6-membership-and-operations.md) | Cluster membership, store resilience and operations | T9, T10, T14 |
| [JEP-CNJ-7](cnj-7-cluster-plugins.md) | Cluster-wide plugin management | T13 |
| [JEP-CNJ-8](cnj-8-cloud-provisioning.md) | Cloud provisioning across controllers | T9 |

Dependencies: CNJ-1 is the foundation of all the others. CNJ-2 and CNJ-4 need CNJ-1. CNJ-3 needs CNJ-1 and CNJ-2. CNJ-5 needs CNJ-1 and CNJ-3. CNJ-6 needs CNJ-1 and CNJ-3. CNJ-7 and CNJ-8 need CNJ-6.

**Upstreaming** is out of scope for now. The core changes are `@Restricted(Beta.class)` or `NoExternalUse` hooks on a local stack of branches, `cnj/t1-*` to `cnj/t14-*` in the core worktree. There are small hooks in workflow-cps, cloudbees-folder, lockable-resources, credentials, stapler and kubernetes. The store, routing and clustering live in one plugin, `impl/persistence-store-postgresql`.

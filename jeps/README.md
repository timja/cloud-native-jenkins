# Draft JEPs: multi-controller, cloud-native Jenkins

Drafts of Jenkins Enhancement Proposals for the work in this repository. There is one per challenge in running N controllers active/active with ephemeral homes and all state in external services. They follow the [JEP template](https://github.com/jenkinsci/jep/blob/master/jep/0000/README.adoc) loosely, in Markdown.

Nothing here has been submitted. They are numbered from 250, leaving a gap after the JEPs in [jenkinsci/jep](https://github.com/jenkinsci/jep/tree/master/jep), and laid out as there (`<number>/README`); the numbers are placeholders until submitted. Every claim about behaviour is backed by an acceptance experiment in `validation/experiments/ET*.sh`; its findings are on the branch named in each JEP's *Prototype* section.

| Draft | Title | Covers (targets) |
|---|---|---|
| [JEP-250](250/README.md) | Pluggable persistence store for `JENKINS_HOME` state | T1, T6, T11 |
| [JEP-251](251/README.md) | Shared build records, logs and history | T3, T8, T12, T14 (logs) |
| [JEP-252](252/README.md) | Ownership of running builds: leases, fencing, adoption | T4, T12, T13, T14 |
| [JEP-253](253/README.md) | Cluster scheduling: build numbers, triggers, a shared queue | T2, T5 |
| [JEP-254](254/README.md) | Request routing and stateless sessions | T6, T7, T14 (REST) |
| [JEP-255](255/README.md) | Cluster membership, store resilience and operations | T9, T10, T14 |
| [JEP-256](256/README.md) | Cluster-wide plugin management | T13 |
| [JEP-257](257/README.md) | Cloud provisioning across controllers | T9 |

Dependencies: JEP-250 is the foundation of all the others. JEP-251 and JEP-253 need JEP-250. JEP-252 needs JEP-250 and JEP-251. JEP-254 needs JEP-250 and JEP-252. JEP-255 needs JEP-250 and JEP-252. JEP-256 and JEP-257 need JEP-255.

**Upstreaming** is out of scope for now. The core changes are `@Restricted(Beta.class)` or `NoExternalUse` hooks on a local stack of branches, `cnj/t1-*` to `cnj/t14-*` in the core worktree. There are small hooks in workflow-cps, cloudbees-folder, lockable-resources, credentials, stapler and kubernetes. The store, routing and clustering live in one plugin, `impl/persistence-store-postgresql`.

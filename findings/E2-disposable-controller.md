# E2: Disposable controller against existing external stores

- **Script:** `validation/experiments/E2-disposable-controller.sh`, run straight after E1
- **Raw evidence:** `raw/E2/`

## Method
1. Run E1, which leaves state in Postgres (junit-sql) and Azurite (azure-artifact-manager).
2. Delete the controller container **and** its `JENKINS_HOME` volume.
3. Start a new controller with an empty home against the same Postgres, Azurite and JCasC.
4. Recreate jobs with the same names and build again.

## What survives
| State | Survived? | Why |
|---|---|---|
| System message, security realm, admin user, node definition, credential values | yes | re-created from JCasC each boot, not by any store |
| Secret-encrypted values **entered in the UI** | no | new `master.key` and `secret.key`; only plaintext from JCasC can be re-encrypted |
| Jobs | no | `jobs/` was on the deleted volume |
| Build history (records, logs, flow nodes) | no | |
| API token created in E1 | **no** (`401`) | stored in `users/*/config.xml` |
| JUnit rows in Postgres | yes, orphaned | no job or build left to show them |
| Artifact and stash blobs in Azurite | yes, orphaned | |

## Identity collision: external data is keyed by `(job full name, build number)`
After recreating `freestyle` and `pipeline` and building `#1` again:
- **JUnit (junit-sql-storage):** `freestyle #1` now reports **4** passing tests (2 expected) and `pipeline #1` reports **2** (1 expected). `caseresults` holds the old and new rows under the same `(job, build)`, and the plugin merges them. *The new build silently shows the old build's test results.*
- **Artifacts (azure-artifact-manager):** the blob key is `<prefix><job full name>/<build number>/…`. It comes from `BUILD_PREFIX_FORMAT` with `getParent().getFullName()` and `getNumber()` (`azure-artifact-manager-plugin/.../AzureArtifactManager.java:113`), and the console log confirms it: `blob artifacts/pipeline/1/stashes/stuff.tgz`. New `#1` therefore writes to the same keys as the old `#1`, overwriting the old build's artifacts, and deleting either build would delete the other's blobs (`deleteWithPrefix`, `:388`). *This is inferred from the key scheme. The harness did not list blob versions, because Azurite listing needs a signed request.*

**Implication:** any externalised store needs a build identity that is unique across controller lifetimes and deployments. Possible keys:
- a per-build UUID, or
- `(controller-cluster id, job id, build number)` with build numbers that never reset, i.e. external sequences (T3).

Job names are not stable either: a rename, or a delete-and-recreate, reuses them.

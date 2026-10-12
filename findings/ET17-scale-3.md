# ET17: Target 15 item 17 (scale: three controllers, 200,000 build records, an indexing storm)

**Result: PASS**, every check. Raw evidence: `raw/ET17/`
- `results.md`
- `top-statements.txt` (`pg_stat_statements`)
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET17-scale-3.sh`
- **Setup:**
  - three tmpfs controllers (`compose.et17.yaml`; `jenkins-c` is in `compose.et1.yaml` under the profile `et17`), with heaps capped at 768 MB: three controllers share one 8 GB Docker VM, and use about 130–440 MB
  - Postgres with `pg_stat_statements`, Azurite, token auth
  - 10 jobs × 20,000 builds (synthetic records, with the permalinks the controllers completing them would have stored, and snapshots of the newest 100)
  - 30 multibranch projects × 10 branches in local git repositories

| Scenario | Result |
|---|---|
| 1 200,000 build records inserted | 4 s |
| 2 The three controllers start together on them | ✅ all three up 18 s after starting. A job page with 20,000 builds: 0.64–0.78 s cold, 0.02–0.03 s warm, on each controller. The newest 50 builds by REST 0.24–0.52 s; `lastSuccessfulBuild` under 0.02 s; the newest build's console under 0.01 s |
| 2 A job with 20,000 builds deleted | ✅ 1 s; the others drop it within 60 s, with its records (its build numbers are kept, so a recreated job never reuses one) |
| 3 Every project scanned from all three controllers at once (90 requests); C killed with `SIGKILL` 22 s in | ✅ all 300 branch jobs on A and B, the storm settled 35 s after the scans, and every branch has a published build: 186 successful, and 114 closed as aborted (they were running on C when it was killed, before their first save point, and their consoles say so). No branch built twice. C, restarted, has every branch job |

**Busiest statements during the run:** the children listing (5,345 calls, 3.5 s in total), the event poll (1,840 calls, 2.0 s), reads by key (12,112 calls, 0.3 s).

## Bugs found while running ET17, fixed
- **A fresh controller resolved permalinks by going through every build** (core). With no `permalinks` file in a fresh home, a job page's "last failed build" and similar went back through all 20,000 builds, loading each from the store and blob storage. A cold job page took 90 s, and over 180 s on a busy controller. Permalinks are now cached in the store per job (`StoredPermalinks`), kept up to date by the controller completing or deleting a build: 0.6–0.8 s. A controller that has not seen a newer build yet never moves a target back.
- **Concurrent scans of a project built new branches twice** (store plugin). A folder computation is not a job, so it does not go through the shared queue: each controller asked to scan a project scanned it, and scans running at once each found a new branch before any had recorded its revision. A lease per folder (`ClusterComputations`) now lets one controller compute it at a time; the others wait, then find nothing new. The lease lapses with its holder.
- **A user changed in memory and never saved never got changes made elsewhere again** (core). JCasC re-hashes a declared local password on each controller (a new salt each time), so the controllers wrote different hashes of the same password, and one was left holding a hash it never saved. Its reloads were put off forever. A user's change is now put off a few times, then applied, as for items.
- **The fresh controller's load progress printed percentages like 27,600 %** (cloudbees-folder): the total counts job directories on disk, which a home whose items are in the store does not have. The percentage is left out then.
- **Harness:**
  - The third controller inherited the first one's port through compose's `extends`, and never started.
  - The script replaced the git repositories' directory while the controllers had it bind-mounted, so every scan failed ("not a git repository").
  - The multibranch sources were made in code without branch discovery (`GitSCMSource` has no traits by default), so they found no branches.
  - The checks counted `#1` per branch. A number C took just before it was killed may never become a build (as after a crash on one controller); the check now counts builds per branch.

## Observations
- **One slow restart.** In one run, C restarted while A and B were still busy after the storm, and took 9 minutes to load its 340 jobs (2–3 s per branch job), after its first connections to the store timed out. It did not recur: restarted with the cluster idle, C loaded them in 1.7 s, and in the final run within the check's time. Faster job loading on a fresh controller is on the ROADMAP.
- **Builds closed rather than resumed.** C started more than 100 builds in its 22 s and none had reached a save point when it was killed. What nothing can resume is closed as aborted, not lost (JEP-252). A Pipeline with no save point could be restarted from its beginning instead. That is not done, because steps are not known to be idempotent.

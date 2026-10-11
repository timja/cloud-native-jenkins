# ET16: Target 14 M6 (scale on a laptop)

**Result: PASS**, every check. Raw evidence: `raw/ET16/` (`results.md`, page timings, query plan, controller logs, `BUILD`).

- **Script:** `validation/experiments/ET16-scale.sh`
- **Setup:** two controllers on Docker (8 CPUs, 8 GB): 2,000 jobs (a quarter in 10 folders), 2,000 builds of 200 of them, then 50 Pipelines running at once.

| Measure | Result | Bar |
|---|---|---|
| Creating 2,000 jobs on A (one script) | 8 s | |
| B lists all 2,000 | 10 s after A started creating (event lag about 2 s) | ✅ within 120 s |
| 2,000 builds run and published | 46 s | |
| Warm page loads on both controllers: `/` (2,000 jobs), `/api/json?tree=jobs[name,color]`, a job page, a job's API at depth 1, the cluster view | `/` 0.4–0.7 s, the others under 0.03 s; cold `/` 3.1 s | ✅ under 5 s warm |
| Heap in use | about 125 MB per controller | |
| Store | 4,226 keys (4.4 MB), 20,531 events | |
| Listing a job's builds (`children`) | 0.12 ms, using the new key index | ✅ uses the index |
| Restarting B with all of that | up in 14 s; cold `/` 6.3 s, warm 0.4 s | ✅ within 180 s |
| `/` on B with 50 builds running on A | 0.7 s | |
| A killed with 50 Pipelines running | all adopted by B within 24 s of the kill, and all completed | ✅ within 90 s |

## A fresh controller's start (Target 15, scenario 4)
B is replaced by a fresh container with an empty home, three times, with every plugin in the image and the 2,000 jobs and 2,000 builds in the stores; then once more with a plugin installed at runtime, which the fresh controller fetches from blob storage and loads.

| Run | API served after (from container start) | Container and JVM start | Jenkins initialization | First `/` (2,000 jobs) |
|---|---|---|---|---|
| fresh 1 | 20.4 s | 4.9 s | 9.8 s | 19.0 s |
| fresh 2 | 16.4 s | 4.0 s | 10.9 s | 5.4 s |
| fresh 3 | 12.8 s | 2.6 s | 7.1 s | 6.8 s |
| with a runtime plugin | 12.9 s | 3.0 s | 7.2 s | 6.9 s |

- **Median 16.4 s** to serving the API, and every fresh controller listed all 2,000 jobs. A warm `/` afterwards took 0.6 s.
- **Builds are not fetched at start,** only when first looked at, so the history's size barely matters. The rest of the time to serving is the HTTP listener and loading items from the store.
- **A plugin installed at runtime adds nothing measurable.** The image is still the faster place for plugins: it avoids a restart when a plugin cannot load live.
- **Persistent volumes would not help:** a fresh home costs these seconds, while a volume attached to a dead node can keep a replacement pod waiting for minutes (JEP-255).

## Changes for scale (Target 14 M6)
- **Prefix listings** (`children`, the start of moves) used `starts_with(key, …)`, which no index serves, so every call scanned the whole table. That is about 2 calls per job at startup, and one per running-build cache miss. They now use an escaped `LIKE 'prefix%'` with a `text_pattern_ops` index. A test checks that `_` and `%` in names are literal.
- **Pruning the event log** by age uses an index.
- **The connection pool** (Target 14 M1) means requests no longer queue behind one synchronized connection.

## Not measured (follow-ups)
- More than two controllers, and much larger histories (20k+ builds). The synthetic-history approach the research suggested (copying records in SQL) was not needed for these bars.
- Branch-indexing storms (many multibranch projects indexing at once).

# Target 14 M3: controllers occasionally not coming up

**Result: not reproduced.** One cause was found by reading the code, and it is fixed. The harness now records enough to diagnose the next occurrence.

## Symptom
In earlier regression runs, a controller now and then did not answer within the harness's wait. A rerun passed, and no thread dump was taken at the time, so the cause was never seen directly.

## What was done
- **A race found by reading the code** (core `91f302e55e`).
  - Controllers booting together all rewrite the shared admin user (CasC, and the token initializer).
  - `User.changedElsewhere` reloaded the user without its lock, so a reload could drop a change made locally in the meantime, such as the API token the harness then authenticates with. The controller would then be up but refuse the harness's token, which from outside looks like a boot that never finishes.
  - The user is now reloaded under its lock. A reload is put off while a save of the user is pending; it throws, and the applier retries with backoff.
- **Startup waits for the stores** (Target 14 M1). A controller started while Postgres or Azurite is not ready yet waits up to 10 minutes and logs that it is waiting. It used to fail its boot or hang on a connection with no timeout.
- **Diagnostics in `validation/lib.sh`.**
  - `wait_up` reports `/login` and the authenticated `/api/json` status while waiting, which tells "not up" apart from "up but the token is wrong".
  - On a timeout, `boot_diagnostics` saves a thread dump of every controller, plus the store's `pg_stat_activity` and `pg_locks`.
  - `wait_postgres` waits for the database before a controller is started.
- **The token initializer** retries for 5 minutes (60 × 5 s) instead of giving up when the store is slow at boot.

## Since then
It has not happened in the final sweep, in which ET1–ET16 all passed (ET15 twice). If it happens again, the dumps from `boot_diagnostics` are under the experiment's `raw/` directory.

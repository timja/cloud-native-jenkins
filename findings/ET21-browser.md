# ET21: Target 15 item 20 (a real browser through the load balancer)

**Result: PASS**, all 4 Playwright tests. Raw evidence: `raw/ET21/`
- `results.md`
- Playwright's output and JSON report
- controller logs, `BUILD`

- **Script:** `validation/experiments/ET21-browser.sh`; tests in `validation/ui-tests/cluster.spec.js` (`@playwright/test` 1.63.0, Chromium)
- **Setup:**
  - two tmpfs controllers, Postgres, Azurite, token auth
  - `rr`, a round-robin load balancer with no stickiness: consecutive requests from the browser reach different controllers

| Test | Result |
|---|---|
| A live console follows a build running on one controller, whichever controller each request reaches | ✅ the page's progressive-log requests that reach the other controller are forwarded to the owner, and the console fills in to the end |
| A job configured in the browser is saved for every controller | ✅ the configure form (loaded from one controller, its fragments from either) saves; the change is on both controllers |
| The cluster page shows both controllers with their executors | ✅ both controllers, and busy/total executors |
| The plugin manager says changes apply to every controller, and shows their status | ✅ the notice and each controller's status |

## Notes
- **Authentication** is an API token sent preemptively (an `Authorization` header on every request): Playwright's own basic authentication waits for a challenge, which Jenkins answers with a redirect to the login page.
- **Install:** `cd validation/ui-tests && npm ci && npx playwright install chromium` (the browser build matching 1.63.0).

## Bugs found while running ET21, now fixed (harness)
- Playwright's basic authentication was not preemptive (see above).
- The cluster page's controller names also appear in the internal URLs' cells, so the test matches cells exactly.

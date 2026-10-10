# E5: Two controllers behind a non-sticky load balancer

- **Script:** `validation/experiments/E5-stateless-http.sh`
- **Raw evidence:** `raw/E5/results.md`

**Setup:** A and B share a home, so they have the same users, `master.key`, crumb salt and agent secrets. They sit behind nginx round-robin, which adds an `X-Upstream` header.

| Check | Observed |
|---|---|
| Form login | The login form now carries a **session-bound crumb**. When `/login` and the POST hit the **same** node, login works. Across nodes it goes to `/loginError` (5 of 5) |
| Session after login | Authenticated only on requests that reach the node holding the session. Sessions are in-memory Jetty sessions, and the cookie name is per instance (`JSESSIONID.f888d7ad` vs `JSESSIONID.1909d929`) |
| Crumb via LB, POST via LB (password auth) | `403 No valid crumb was included in the request` every time the POST landed on the other node. All 6 attempts crossed nodes, so 6/6 failed. The crumb is a hash of the session ID (`DefaultCrumbIssuer`) |
| API token created on A after both booted | A: `200`, **B: `401`**, via LB: `200 401 200 401 …`. B keeps `User` in memory and never sees A's `users/*/config.xml` write |
| POST with API token via LB | `200 401 200 401`. Token auth needs no crumb, but hits the same stale-user problem |
| Inbound WebSocket agent through the LB | Connected to exactly **one** controller: `agent1` online on B, offline on A. Both have `agent1` defined, and builds on A wait forever for it |

## Implications (S0 short-term, then R3 and X)
- **Sessions and crumbs need shared or stateless handling:** shared session storage, or a signed session or JWT cookie, plus crumbs bound to the user rather than the session. Sticky sessions are a stopgap that breaks on failover.
- **User and API-token changes need cross-controller invalidation.** This is the same SPI change-event mechanism as R1, applied to users in Track S.
- **Agents connect to one controller.** Two models are possible:
  - Build ownership follows agent ownership, and requests are routed to the owner.
  - Channels are brokered so any controller can use any agent.

  Either way, "node is defined" and "node is connected here" must become different concepts. `ComputerSet.ComputerSource` / `Label.LabelComputerSource` (Beta) already point at this.

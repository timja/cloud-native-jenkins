# JEP-254: Request routing and stateless sessions

| | |
|---|---|
| **JEP** | 254 |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-10-11 |
| **Requires** | JEP-250, JEP-252 |

## Abstract
Users and tools reach any controller through a load balancer with no stickiness. For that to work:
- a browser stays signed in, and its forms keep working, on every controller
- requests that only the owner of something can answer are forwarded to it: a running build's live console, `input`, stop and replay; an agent's pages, through the controller holding its channel; the UI fragments a page bound to objects in one controller's memory
- builds running on another controller appear in every controller's UI and REST API

## Specification

### Sessions and crumbs
- **Sign-in.** A browser's sign-in is a signed cookie (Jenkins' remember-me token, with the user's seed), valid on every controller, instead of a server-side HTTP session. Signing out, or changing the password, invalidates it everywhere.
- **Crumbs.** A cluster crumb issuer (`crumbIssuer: cluster`) binds crumbs to the sign-in cookie, or, before sign-in, to a nonce cookie. Every controller validates them with the same salt.
- **API tokens** are stored with the user (JEP-250), so they work everywhere.

### Bound objects
- Stapler bound objects (`renderOnDemand`, progressive rendering, JavaScript proxies) live in one controller's memory. Their ids are prefixed with that controller's name; this is a Stapler change.
- A request for another controller's bound object is forwarded to it, cookies included.

### Forwarding to the owner
- **`OwnerRoutingFilter`** handles requests addressed to things another controller holds:
  - **build URLs** (`/job/…/<N>/…`) for a build running on another controller, located by its placeholder's origin (not its name, which a replacement controller may share)
  - **agent URLs** (`/computer/<name>/…`) for an agent connected to another controller, located through the channel table
- **How forwarding works.** It uses the target's internal URL from the membership table, waits at most 20 s for the response headers, and streams the body (live consoles). If the owner cannot be reached, the request is served locally.
- **`input`, stop and replay** of a running build therefore work from any controller. After an adoption (JEP-252), they reach the new owner.
- **Permalinks.** `/job/x/lastBuild/…` redirects to the newest build when that build runs elsewhere, and the build URL is then forwarded.

### Agent channels
- Each controller publishes which agents it holds a channel to.
- Inbound agents reconnect to a failover URL (the load balancer) when their controller dies, and resume within about 40 s, as measured in the acceptance tests.

### Builds running elsewhere in the UI and REST
See JEP-251: the job page, the Builds card, the job's icon, and REST `builds`, `allBuilds` and `lastBuild` all include them.

## Motivation
- **Sessions.** In an experiment with two controllers behind a load balancer without stickiness, sessions were lost and crumbs failed across controllers, and an API token made on one controller gave `401` on another.
- **Ownership.** The same experiment, and one that failed a controller over to another, showed that a running build and an agent belong to one JVM. Their pages must be answered by that JVM.

## Reasoning
- **Stateless sign-in** reuses Jenkins' own remember-me mechanism, so the session model needs no new security primitive.
- **Forwarding** keeps every controller a full Jenkins, with no request-time knowledge in the load balancer. A sticky load balancer would only help the browser; agents, webhooks and API clients would still land anywhere.
- **Placeholders are addressed by origin,** so a restarted pod with the same name never answers for its predecessor's builds.

## Backwards compatibility
- Without a store, nothing changes.
- **Known limitations:**
  - WebSocket requests that reach a non-owner are not forwarded; the HTTP fallbacks are.
  - CLI commands addressing a build running on another controller (`console`, `get-build`) do not see it.
  - The REST stand-in for a build running elsewhere carries a `Run`'s properties, but its `_class` is the stand-in's. Deeper detail is at its own URL, which the owner answers.
- **Password changes** sign out every browser of that user, including the one that made the change, as on one controller.

## Security
- Forwarded requests carry the user's credentials (cookies, `Authorization`), and the owner authorizes them as if received directly.
- Controllers reach each other on internal URLs, which must not be reachable from outside the cluster network.
- The trust model for requests between controllers is under security review, through a separate process.

## Testing
- **Core:** `StatelessSessionsTest`, `RunningBuildsTest` (UI and REST).
- **Store plugin:** `OwnerRoutingTest`, `ForwardingTest`, `ClusterViewTest`.
- **Acceptance** (scripts in `validation/experiments/` of the cloud-native-jenkins repository; each runs two controllers against one store):
  - `ET6-static-state.sh`: a scripted browser signs in and works through a round-robin load balancer, including when a controller is lost
  - `ET7-owner-routing.sh`: from the controller that does not run a build, its console can be followed, and `input`, stop and replay work; agent pages answer; an agent reconnects after its controller dies; webhooks; `input` after a failover
  - `ET15-logs-api-failover.sh`: REST and `lastBuild` for builds running elsewhere

## Prototype implementation
- **Core:** prototype branches of Jenkins core, `cnj/t6-m6-stateless-sessions`, `cnj/t14-m1-resilience` (REST stand-in).
- **Stapler:** prototype branch `cnj/t6-m5-bound-id-prefix`.
- **Store plugin** (`impl/persistence-store-postgresql` in the cloud-native-jenkins repository): `OwnerRoutingFilter`, `Forwarding`, `BoundRequestForwarder`, `ClusterCrumbIssuer`, `ChannelPublisher`.
- **Findings:** documents in the cloud-native-jenkins repository, on its prototype branches: `findings/ET6-static-state.md`, `ET7-owner-routing.md`.

## References
- `findings/E5-stateless-http.md` in the cloud-native-jenkins repository: two controllers behind a load balancer without stickiness

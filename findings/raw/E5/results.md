## E5 results
### 1. Browser session: form login through the LB (login form carries a session-bound crumb)
| attempt | /login served by | POST served by | login result | later page loads authenticated |
|---|---|---|---|---|
| 1 | 192.168.16.4 | 192.168.16.4 | redirect to `/` | 5/6 |
| 2 | 192.168.16.5 | 192.168.16.4 | redirect to `/loginError` | 0/6 |
| 3 | 192.168.16.5 | 192.168.16.4 | redirect to `/loginError` | 0/6 |
| 4 | 192.168.16.4 | 192.168.16.5 | redirect to `/loginError` | 0/6 |
| 5 | 192.168.16.4 | 192.168.16.5 | redirect to `/loginError` | 0/6 |
| 6 | 192.168.16.5 | 192.168.16.4 | redirect to `/loginError` | 0/6 |
### 2. Crumb obtained via LB, then POST via LB (password auth + session cookie + crumb)
| attempt | crumb issued by | POST served by | HTTP |
|---|---|---|---|
| 1 | 192.168.16.5 | 192.168.16.4 | 403 |
| 2 | 192.168.16.4 | 192.168.16.5 | 403 |
| 3 | 192.168.16.4 | 192.168.16.5 | 403 |
| 4 | 192.168.16.5 | 192.168.16.4 | 403 |
| 5 | 192.168.16.5 | 192.168.16.4 | 403 |
| 6 | 192.168.16.4 | 192.168.16.5 | 403 |
### 3. API token created on A after both controllers booted
- token on A: 200
- token on B: 401
- token via LB x6: 200 401 200 401 200 401 
- POST via LB with API token (no crumb needed): 200 401 200 401 
### 4. Inbound WebSocket agent connecting through the LB
- agent secret identical on A and B (HMAC of shared master.key): yes
- agent1 online on A: false, on B: true
```
agent-1  | INFO: Connected
```

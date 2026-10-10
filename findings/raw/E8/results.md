## Variant M: shared master.key only (separate homes, identical JCasC)
| Check | Result |
|---|---|
| `Secret` encrypted on A decrypts on B | NO (null) |
| agent1 inbound secret identical | NO |
| instance identity public key identical | NO |
| legacy `secret.key` identical | NO |
| API token created on A accepted by B | 401 |
| `secrets/*` files differing between A and B | hudson.model.User.DIRNAMES hudson.util.Secret jenkins.model.Jenkins.crumbSalt jenkins.security.csp.ReportingContext.key jenkins.slaves.JnlpSlaveAgentProtocol.secret org.jenkinsci.main.modules.instance_identity.InstanceIdentity.KEY  |

Seed key set: `hudson.console.ConsoleNote.MAC hudson.model.Job.serverCookie hudson.model.User.DIRNAMES hudson.util.Secret jenkins.model.Jenkins.crumbSalt jenkins.security.csp.ReportingContext.key jenkins.slaves.JnlpSlaveAgentProtocol.secret master.key org.jenkinsci.main.modules.instance_identity.InstanceIdentity.KEY `

## Variant S: full key set seeded into both homes (separate homes, identical JCasC)
| Check | Result |
|---|---|
| `Secret` encrypted on A decrypts on B | e8-value |
| agent1 inbound secret identical | yes |
| instance identity public key identical | yes |
| legacy `secret.key` identical | yes |
| API token created on A accepted by B | 401 |
| `secrets/*` files differing between A and B |  |
| new keys B generated after a build |  |
| console note (`ConsoleNote.MAC`) from B's log renders on A? | n/a: logs are not shared yet (MAC is synced, so it would) |

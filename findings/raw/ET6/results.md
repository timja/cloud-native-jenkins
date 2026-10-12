| ℹ️ | seeded keys: hudson.model.User.DIRNAMES hudson.util.Secret jenkins.model.Jenkins.crumbSalt master.key org.jenkinsci.main.modules.instance_identity.InstanceIdentity.KEY  |
## ET6 results (2026-10-12T15:22Z)

| | Check |
|---|---|
| ✅ | both controllers use the cluster crumb issuer |
| ✅ | 1 the token works on B |
| ✅ | 1 the token works through the round-robin LB (10/10) |
| ✅ | 1 revoked on A, it is refused on B |
| ✅ | 2 the user logs in on B |
| ✅ | 2 the old password fails on B |
| ✅ | 2 the new password works on B |
| ✅ | 3 quiet period visible on B without a restart |
| ✅ | 3 admin address (a descriptor) visible on B |
| ✅ | 3 B's next save does not revert A's change |
| ✅ | 4 listed on B |
| ✅ | 4 a build on B reads its value |
| ✅ | 5 the node appears on B |
| ✅ | 5 its agent secret is the same on A and B |
| ✅ | 5 the agent connects through B |
| ✅ | 5 deleted on A, it is gone on B |
| ✅ | 5 and B dropped its connection |
| ✅ | 6 the key has the same value on B |
| ✅ | 6 data encrypted with a runtime key on A decrypts on B |
| ℹ️ | 9 versions before: config.xml@26 jenkins.model.JenkinsLocationConfiguration.xml@25 nodes/n7/config.xml@56 users/u9_fb6cec85fb5157cb0d2c658f78cde0389396204e9e1c86694ff0c6bb96ba493d/config.xml@55 ; after: config.xml@26 jenkins.model.JenkinsLocationConfiguration.xml@25 nodes/n7/config.xml@56 users/u9_fb6cec85fb5157cb0d2c658f78cde0389396204e9e1c86694ff0c6bb96ba493d/config.xml@55  |
| ✅ | 9 B's boot (JCasC) bumped no versions of unchanged config |
| ✅ | 9 a setting JCasC does not declare keeps its runtime value |
| ✅ | 9 a user created at runtime survives B's boot |
| ✅ | 9 JCasC's declared setting is still JCasC's |
| ✅ | 9 the node added in the UI survives B's boot |
| ℹ️ | 10 result 9/4, conflicting writes detected: 3 |
| ✅ | 10 both changes survive, or the overwrite was detected |
| ✅ | 10 A and B agree on the result |
| ✅ | 8 logged in through the load balancer |
| ✅ | 8 signed in on every request (10/10) |
| ✅ | 8 requests reached both controllers [jenkins-b, jenkins-a] |
| ℹ️ | 8 bound objects on the configure page: 82 e.g. jenkins-b-3236db18-31ff-4f41-a9c5-917062e64ab3 |
| ✅ | 8 bound objects answer through either controller (12/12 [200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200]) |
| ✅ | 8 configure form saved (200) |
| ✅ | 8 the saved description is there (2 read(s), from [jenkins-a, jenkins-b]) |
| ✅ | 8 progressive renderer's proxy script served (200) |
| ✅ | 8 build time trend rendered progressively (start 200 , news 200 {"data":[{"iconName":"blue","iconColorOrdinal":4,"iconColorDescription":"Success","number":2,"displayName":"#2","dur) |
| ℹ️ | 8 after changing the password, the browser which changed it is signed in: false |
| ℹ️ | 8 login answered 401 |
| ✅ | 8 the old password no longer works |
| ✅ | 8 logged in with the new password |
| ℹ️ | 8 upstreams seen: [jenkins-b, jenkins-a] |
| ℹ️ | 8 bound-object requests forwarded between controllers: 9 |
| ✅ | 8 some bound-object requests were forwarded |
| ✅ | 8 after losing A the browser is still signed in |
| ✅ | 7 users and passwords survive (u2 new password on B) |
| ✅ | 7 global settings survive (A) |
| ✅ | 7 the descriptor setting survives (B) |
| ✅ | 7 the credential survives (B) |
| ✅ | 7 the UI-added node survives, the deleted one stays deleted |
| ✅ | 7 the runtime key survives |

Failures: 0

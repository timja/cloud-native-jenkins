| ℹ️ | seeded keys: hudson.model.User.DIRNAMES hudson.util.Secret jenkins.model.Jenkins.crumbSalt master.key org.jenkinsci.main.modules.instance_identity.InstanceIdentity.KEY  |
## ET6 results (2026-10-14T11:45Z)

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
| ℹ️ | 9 versions before: config.xml@32 jenkins.model.JenkinsLocationConfiguration.xml@31 nodes/n7/config.xml@62 users/u9_527ec4087cf8acbc822288f657a187f663072363c2323837b7a9f07e1c914570/config.xml@61 ; after: config.xml@32 jenkins.model.JenkinsLocationConfiguration.xml@31 nodes/n7/config.xml@62 users/u9_527ec4087cf8acbc822288f657a187f663072363c2323837b7a9f07e1c914570/config.xml@61  |
| ✅ | 9 B's boot (JCasC) bumped no versions of unchanged config |
| ✅ | 9 a setting JCasC does not declare keeps its runtime value |
| ✅ | 9 a user created at runtime survives B's boot |
| ✅ | 9 JCasC's declared setting is still JCasC's |
| ✅ | 9 the node added in the UI survives B's boot |
| ℹ️ | 10 result 9/4, conflicting writes detected: 4 |
| ✅ | 10 both changes survive, or the overwrite was detected |
| ✅ | 10 A and B agree on the result |
| ✅ | 8 logged in through the load balancer |
| ✅ | 8 signed in on every request (10/10) |
| ✅ | 8 requests reached both controllers [jenkins-a, jenkins-b] |
| ℹ️ | 8 bound objects on the configure page: 83 e.g. jenkins-a-79bbe80f-f1a1-432f-9065-9579cabcaf3e |
| ✅ | 8 bound objects answer through either controller (12/12 [200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200]) |
| ✅ | 8 configure form saved (200) |
| ✅ | 8 the saved description is there (1 read(s), from [jenkins-a]) |
| ✅ | 8 progressive renderer's proxy script served (200) |
| ✅ | 8 build time trend rendered progressively (start 200 , news 200 {"data":[{"iconName":"blue","iconColorOrdinal":4,"iconColorDescription":"Success","number":2,"displayName":"#2","dur) |
| ℹ️ | 8 after changing the password, the browser which changed it is signed in: false |
| ℹ️ | 8 login answered 401 |
| ✅ | 8 the old password no longer works |
| ✅ | 8 logged in with the new password |
| ℹ️ | 8 upstreams seen: [jenkins-a, jenkins-b] |
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

## ET13 results (2026-10-11T11:58Z)

| | Check |
|---|---|
| ✅ | neither controller has build-timeout or token-macro to begin with |
| ✅ | 1 build-timeout 1.40 is active on plug-0 |
| ✅ | 1 ... and on plug-1, live |
| ✅ | 1 its dependency token-macro is active on both |
| ✅ | 1 neither controller restarted for it |
| ✅ | 1 its build wrapper's descriptor is there on plug-1 |
| ℹ️ | 1 cluster plugin set: generation 2 token-macro installed 477.vd4f0dc3cb_cf1, build-timeout installed 1.40 |
| ✅ | 2 both controllers run the upgraded build-timeout (1.41) |
| ✅ | 2 both restarted for it |
| ✅ | 2 one at a time: a controller answered throughout |
| ✅ | 2 one at a time: the second went down only after the first was back |
| ❌ | 2 the Pipeline running meanwhile completed |
| ✅ | 2 the restart lock was released |
| ℹ️ | 2 members: plug-0 11:58:59 generation 3;plug-1 11:59:29 generation 3; |
| ✅ | 3 the fresh plug-1 has build-timeout 1.41 |
| ✅ | 3 and the same plugins as plug-0 |
| ✅ | 4 build-timeout is gone from both controllers |
| ✅ | 4 one at a time again: a controller answered throughout |
| ✅ | 5 a downgrade is refused |
| ✅ | 5 changing the store plugin is refused |
| ✅ | 5 a corrupt archive is not installed |
| ✅ | 5 and causes no restart |
| ✅ | 5 and is logged |

Failures: 1

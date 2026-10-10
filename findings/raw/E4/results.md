## E4 results
- sleep #1 at: tick 20; input #1 paused
- program.dat present: /var/jenkins_home/jobs/input/builds/1/program.dat /var/jenkins_home/jobs/sleep/builds/1/program.dat 
- B up 8s after A was killed
- agent online on B (same secret, derived from shared master.key): true
- B sees running builds: {"_class":"hudson.model.ComputerSet","busyExecutors":1}
- input approved on B
- input #1 result: SUCCESS
- sleep #1 result: SUCCESS
- B ran sleep #1 to completion: last tick tick 120
### sleep #1 console (resume section)
```
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] node
Running on agent1 in /home/jenkins/agent/workspace/sleep
[Pipeline] {
[Pipeline] sh
+ seq 1 120
Resuming build at Sat Oct 10 09:06:21 UTC 2026 after Jenkins restart
Waiting for reconnection of agent1 before proceeding with build
Ready to run at Sat Oct 10 09:06:26 UTC 2026
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```
### input #1 console (tail)
```
Ready to run at Sat Oct 10 09:06:21 UTC 2026
Approved by admin
[Pipeline] node
Running on agent1 in /home/jenkins/agent/workspace/input
[Pipeline] {
[Pipeline] sh
+ echo on agent after input
on agent after input
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

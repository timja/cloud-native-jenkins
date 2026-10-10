## E3 results
### 1. Job created on A, visible on B?
- A jobs: {"_class":"hudson.model.Hudson","jobs":[{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowJob","name":"cron"},{"_class":"hudson.model.FreeStyleProject","name":"freestyle"}]}
- B jobs (no reload): {"_class":"hudson.model.Hudson","jobs":[]}
- B jobs (after full /reload): {"_class":"hudson.model.Hudson","jobs":[{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowJob","name":"cron"},{"_class":"hudson.model.FreeStyleProject","name":"freestyle"}]}
### 2. Build number collision: trigger freestyle on A and B at the same time
- A builds: {"_class":"hudson.model.FreeStyleProject","builds":[{"_class":"hudson.model.FreeStyleBuild","number":2,"url":"http://localhost:8081/job/freestyle/2/"}]}
- B builds: {"_class":"hudson.model.FreeStyleProject","builds":[{"_class":"hudson.model.FreeStyleBuild","number":1,"url":"http://localhost:8082/job/freestyle/1/"}]}
- build dirs on disk: 1 2 permalinks 
- nextBuildNumber on disk: 3
- hostnames recorded in log of #1: 2 'hello' lines
```
Started by user [8mha:////4MeFTsKjB9ZOKdIzPnJ6E22AX0F0EwQtgs8PTIHrVdgaAAAAlx+LCAAAAAAAAP9b85aBtbiIQTGjNKU4P08vOT+vOD8nVc83PyU1x6OyILUoJzMv2y+/JJUBAhiZGBgqihhk0NSjKDWzXb3RdlLBUSYGJk8GtpzUvPSSDB8G5tKinBIGIZ+sxLJE/ZzEvHT94JKizLx0a6BxUmjGOUNodHsLgAzWEgZu/dLi1CL9xJTczDwAj6GcLcAAAAA=[0madmin
+ echo hello from freestyle 1
hello from freestyle 1
Finished: SUCCESS
Started by user [8mha:////4MeFTsKjB9ZOKdIzPnJ6E22AX0F0EwQtgs8PTIHrVdgaAAAAlx+LCAAAAAAAAP9b85aBtbiIQTGjNKU4P08vOT+vOD8nVc83PyU1x6OyILUoJzMv2y+/JJUBAhiZGBgqihhk0NSjKDWzXb3RdlLBUSYGJk8GtpzUvPSSDB8G5tKinBIGIZ+sxLJE/ZzEvHT94JKizLx0a6BxUmjGOUNodHsLgAzWEgZu/dLi1CL9xJTczDwAj6GcLcAAAAA=[0madmin
+ echo hello from freestyle 2
hello from freestyle 2
Finished: SUCCESS
```
### 3. Lost update on global config
- config.xml systemMessage after A saves: <systemMessage>written by A
- config.xml systemMessage after B changes numExecutors: <systemMessage>cloud-native-jenkins harness (jenkins-b)
- A in-memory systemMessage: written by A; B: cloud-native-jenkins harness (jenkins-b)
### 4. Cron (`* * * * *`) over ~3 minutes, both controllers have the job loaded
- cron builds seen by A: {"_class":"org.jenkinsci.plugins.workflow.job.WorkflowJob","builds":[{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":8},{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":6},{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":4},{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":3}]}
- cron builds seen by B: {"_class":"org.jenkinsci.plugins.workflow.job.WorkflowJob","builds":[{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":7},{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":5},{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":2},{"_class":"org.jenkinsci.plugins.workflow.job.WorkflowRun","number":1}]}
- cron build dirs on disk: 1 2 3 4 5 6 7 8 permalinks 
```
/var/jenkins_home/jobs/cron/builds/1: cron run 1 on http://localhost:8082/
/var/jenkins_home/jobs/cron/builds/2: cron run 2 on http://localhost:8082/
/var/jenkins_home/jobs/cron/builds/3: cron run 3 on http://localhost:8081/
/var/jenkins_home/jobs/cron/builds/4: cron run 4 on http://localhost:8081/
/var/jenkins_home/jobs/cron/builds/5: cron run 5 on http://localhost:8082/
/var/jenkins_home/jobs/cron/builds/6: cron run 6 on http://localhost:8081/
/var/jenkins_home/jobs/cron/builds/7: cron run 7 on http://localhost:8082/
/var/jenkins_home/jobs/cron/builds/8: cron run 8 on http://localhost:8081/
```
### 5. Collision detection
- DoubleLaunchChecker SEVERE in logs: A=0 B=0 (check period is 60–90 min, hudson/util/DoubleLaunchChecker.java:146)
- SEVERE/WARNING lines: A=7 B=5

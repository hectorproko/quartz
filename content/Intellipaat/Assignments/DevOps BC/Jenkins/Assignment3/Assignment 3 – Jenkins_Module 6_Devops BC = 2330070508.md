---
tags:
  - Jenkins
---



> [!info] Module 6: Jenkins Assignment - 3
> **Tasks To Be Performed:** 
> 1. Create a pipeline in Jenkins 
> 2. Once push is made to “develop” a branch in Git, trigger job “test”. This will copy Git files to test node 
> 3. If test job is successful, then prod job should be triggered 
> 4. Prod jobs should copy files to prod node

"Building upon [[Assignment 2 – Jenkins_Module 6_Devops BC = 2330070508|Assignment 2 – Jenkins]], we'll make it so that 'Job2' triggers as a result of a successful execution of 'Job1.'"

We go to 'Job2's' configuration under 'Build Triggers,' uncheck 'GitHub hook trigger for GITScm polling,' and select 'Build after other projects are built' instead, ensuring it triggers only if the build is successful/stable.
<br>![[Pasted image 20231108140711.png|300]]


I will install the 'Build Pipeline' plugin, which visualizes the connections between upstream and downstream jobs as a graph.
`Manage Jenkins > Plugins`
<br>![[Pasted image 20231108141806.png]]


I'll create a new view from the Dashboard.
<br>![[Pasted image 20231108141940.png|450]]

Give it a name
<br>![[Pasted image 20231108142040.png|300]]
After clicking 'Create,' I am prompted for additional configuration, which I leave at the default settings. Then, I click the 'OK' button to finish.

%%[[jenkins_view.png]]%%

Now, we can see the new view.
<br>![[Pasted image 20231108144804.png|450]]

When we click "Pipeline View" we can see the pipeline. We then click "Run" to start it
<br>![[Pasted image 20231108142627.png]]

If I check the 'Console Output' for each job, I can see that 'Job2' was triggered by the successful completion of 'Job1.

**Job1**
<br>![[Pasted image 20231108144402.png|460]]
**Job2**
> [!success]
> <br>![[Pasted image 20231108144518.png|460]]

After the run, we observe that 'Job1' build number 5 has successfully triggered 'Job2' build number 3.
<br>![[Pasted image 20231108144941.png]]

Now, I'll intentionally cause 'Job1' to fail to observe the resulting behavior. As expected, when 'Job1' fails, 'Job2' does not execute.
<br>![[Pasted image 20231108145327.png]]




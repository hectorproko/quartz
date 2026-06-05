---
tags:
  - Jenkins
---



> [!info] Module 6: Jenkins Assignment - 1
> **Tasks To Be Performed:** 
> 1. Trigger a pipeline using Git when push on develop branch 
> 2. Pipeline should pull Git content to a folder


> [!attention] Prerequisite
> <br>![[Installing Jenkins On AWS]] 

I create a repository named `JenkinsAssignment` on GitHub and then create a `develop` branch within it.
<br>![[Pasted image 20231108102220.png|300]]

> [!NOTE]
> My terminal is already set up to be able to clone using SSH.

I'll create Freestyle job called `Job1`  
`Dashboard > New Item`  
<br>![[Pasted image 20231108102959.png|400]]

<br>![[Pasted image 20231108103235.png|400]]
<br>![[Pasted image 20231108105216.png|400]]


We'll be working with branch `develop`
<br>![[branch specifier develop.png]]
We want the job to be triggered by a GitHub webhook. 
<br>![[GitHub hook trigger for GITScm polling.png|280]]
We want this job to execute in `Slave1`
<br>![[Pasted image 20231108114842.png|280]]
Save


In GitHub, we set up the webhook. We navigate to 'Webhooks' in the repository settings.  
Click on "Add webhook"  
<br>![[Pasted image 20231108105740.png]]

Enter the public IP address of the EC2 instance hosting Jenkins, followed by the Jenkins port, which is 8080.
<br>![[webhook payload URL.png|400]]
Click button "Add webhook"


> [!attention]
> I need to be careful when restarting my EC2 instance (Jenkins Master), as this will change the public IP address.

> [!success]
> GitHub sends a ping to test it out.
> <br>![[Pasted image 20231108110428.png]]

We navigate back to our Jenkins dashboard, select 'Job1,' and execute it by clicking 'Build Now.'
<br>![[build now button.png|280]]

If I click on the build and check its 'Console Output,' I can see that it was started by me and that it was successful.
<br>![[Pasted image 20231108111550.png|280]]

To test the webhook, I'll make a push to the develop branch. I'll clone the repository, switch to the develop branch, create a file `somefile`, and then push it.
<br>![[Pasted image 20231108112325.png]]

> [!success]
> We get another build this time triggered by the POST request that GitHub sends instead of "[[build now button.png|Build Now]]" button.
> <br>![[Pasted image 20231108112502.png|280]]

> [!success]
> Now, if we check inside the 'Slave1' node, we can see 'Job1's' workspace, which contains the contents from the repository.
> <br>![[Pasted image 20231108115633.png]]


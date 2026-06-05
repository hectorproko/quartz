---
tags:
  - Jenkins
---



> [!info] Module 6: Jenkins Assignment - 2
> **Training Tasks To Be Performed:** 
> 1. Add 2 nodes to Jenkins master 
> 2. Create 2 jobs with the following jobs: 
>    a. Push to test 
>    b. Push to prod 
> 3. Once a push is made to test branch, copy Git files to test server 
> 4. Once a push is made to master branch, copy Git files to prod server

To build upon [[Assignment 1 – Jenkins_Module 6_Devops BC = 2330070508|Assignment 1 – Jenkins]], the following updates will be carried out:

1. Switch from using the 'test' branch to the 'develop' branch for all relevant operations.
2. Rename the node 'Slave1' to 'Test' within both AWS EC2 and Jenkins nodes to reflect the updated branch strategy.
3. Update the configuration of 'Job1' to ensure it [[Pasted image 20231108120638.png|points to the 'Test']] node instead of 'Slave1'.
   
I will create an additional node named 'Prod' by setting up a new EC2 instance and then adding it as a node to the Jenkins master, following the same procedure as in [[Assignment 1 – Jenkins_Module 6_Devops BC = 2330070508|Assignment 1 – Jenkins]].
%%technically like we did in [[Installing Jenkins On AWS#Setting up slaves|Installing Jenkins On AWS]] %%

<br>![[Pasted image 20231108121842.png]]
<br>![[Pasted image 20231108121801.png]]

- The 'develop' branch is configured to run on the node named 'Test,' which has the private IP address `10.0.1.65`.
- The 'master' branch is set to run on the node named 'Prod,' with the private IP address `10.0.1.233`.  

I will create 'Job2' to be similar to 'Job1' with the exception that 'Job2' will be restricted to run only on the 'Prod' node. Additionally, 'Job2' will be configured to build from the 'master' branch exclusively.
<br>![[prod node restriction.png|270]]
<br>![[branch specifier master.png]]


> [!attention]
> I need to be careful when restarting my EC2 instance (Jenkins Master), as this will change the public IP address.


To verify that 'Job2' is executed by the webhook, we will push a change to the 'master' branch.
<br>![[Pasted image 20231108124621.png]]

The job gets <mark style="background: #BBFABBA6;">triggered by the webhook</mark>, but the build fails.

> [!failure]
> <br>![[Pasted image 20231108124725.png]]

> [!done] Solution
> The job execution failed because the 'master' branch has been renamed to 'main.' We have updated 'Job2's configuration to point to the 'main' branch accordingly.
> <br>![[branch specifier main.png]]


I executed the job manually to confirm that this was indeed the issue.
> [!success]
> <br>![[Pasted image 20231108135244.png]]


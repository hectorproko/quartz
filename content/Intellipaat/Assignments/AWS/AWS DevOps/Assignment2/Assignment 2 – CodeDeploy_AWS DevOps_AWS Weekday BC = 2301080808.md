---
tags:
  - AWS
  - CodeDeploy
  - AWSDevOps
title: Implementing Automated Deployments with CodeDeploy and EC2
---
<!--
**Mini-Project: Streamlining Deployment Processes with AWS CodeDeploy!** I've successfully completed a project on AWS CodeDeploy, aiming to automate and simplify deployment tasks for XYZ Corporation. The project involved creating a CodeDeploy application for an EC2 instance and setting up a deployment group. This practical exercise deepened my understanding of AWS deployment services and their crucial role in efficient DevOps practices. It was a valuable opportunity to apply theoretical knowledge in a real-world scenario, enhancing my skills in automated deployment strategies.
-->
 
> [!info] AWS DevOps Assignment - 2
> **Training Problem Statement:**
> You work for XYZ Corporation. Your corporation wants the files to be stored on a private repository on the AWS Cloud. Once done, you are required to automate a few tasks for ease of the development team. 
> 
> **Tasks To Be Performed:**
> 1. Create a CodeDeploy Application with the compute platform as EC2. 
> 2. Create a deployment group with the selected EC2 instance. 

### 1. Setting up CodeDeploy Application for EC2:

**In the AWS Management Console:** 
a. I sign in. 
b. I go to AWS CodeDeploy under Developer Tools. 
c. I click "Getting started", then choose "Create Application". 
d. I type "XYZ_Corporation_App" as the Application Name. 
e. I select "EC2/On-premises" for the "Compute platform". 
f. I click "Create application".
<br>![[Pasted image 20231019092334.png]]
    

### 2. Setting up Deployment Group for the EC2 Instance:

*EC2 instance need AWS CodeDeploy Agent installed and running. This agent facilitates the deployment tasks and communicates with the CodeDeploy service.*

**In the AWS Management Console:**

a. I navigate to the CodeDeploy console and click on the application name I previously created.
b. Under the "Deployment groups" tab, I click "Create deployment group".
c. I name the Deployment Group as "XYZ_Deployment_Group".
d. For the Service role, I choose the previously created role "AWSCodeDeployRole" which has the "AWSCodeDeployRole" policy attached.
e. I opt for "In-place deployment" in the "Deployment type" section.
f. In "Environment configuration", I select "Amazon EC2 instances".
g. I decide to identify my EC2 instance using tags. I ensure the EC2 instance I want to deploy to has the tag `Name` set to `XYZ_Instance`. Alternatively, I could manually select the instance.
h. In "Deployment settings", I select the "CodeDeployDefault.OneAtATime" configuration for simplicity.

i. I finalize by clicking "Create deployment group".
<br>![[Pasted image 20231019092629.png]]

You've now successfully created an AWS CodeDeploy application and a deployment group targeting an EC2 instance using the AWS Management Console.



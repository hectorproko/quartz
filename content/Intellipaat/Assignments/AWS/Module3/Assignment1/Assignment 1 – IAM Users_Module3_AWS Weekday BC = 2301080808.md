---
tags:
  - AWS
title: Implementing IAM User and Group Management for Enhanced Security in AWS
---
<!--
**Mastering AWS Identity and Access Management!** I've recently completed an assignment focused on enhancing security and organization in AWS through IAM. My task involved creating and managing users and groups, specifically creating four IAM users named Dev1, Dev2, Test1, and Test2, and organizing them into DevTeam and OpsTeam groups. This exercise strengthened my ability to structure access and roles effectively, ensuring a secure and efficient AWS environment.
-->  
> [!info] Module 3: IAM Users Assignment
> **Problem Statement:** 
> You work for XYZ Corporation. To maintain the security of the AWS account and the resources you have been asked to implement a solution that can help easily recognize and monitor the different users. 
> 
> **Tasks To Be Performed:** 
> 1. Create 4 IAM users named `Dev1`, `Dev2`, `Test1`, and `Test2`. 
> 2. Create 2 groups named `DevTeam` and`OpsTeam`. 
> 3. Add Dev1 and Dev2 to the DevTeam. 
> 4. Add Dev1, Test1 and Test2 to the OpsTeam.

### Using AWS Management Console:

1. **Creating 4 IAM Users:**
 - First, I logged into the AWS Management Console and navigated to`Services > IAM > Users`.
 - Then, I clicked on the`Create user` button to create four new users. I named them `Dev1`, `Dev2`, `Test1`, and `Test2` respectively.  
   <br>![[create user.png]]
 - During this process permissions are configured  for each user as needed.  
   <br>![[IAM users.png]]

2. **Creating 2 IAM Groups:**
 - Next, I went to the IAM Dashboard and clicked on`Services > IAM > User groups`.
 - I clicked on`Create Group` and made two new groups named `DevTeam` and `OpsTeam.`
   <br>![[Pasted image 20230920154327.png]]
   *During this step we can also add users*
 - I also attached the necessary policies to these groups.
   
   <br>![[user groups.png]]
   
3. **Adding Dev1 and Dev2 to the DevTeam:**
 - I clicked on the `DevTeam` group and then selected `Add users` to Group."
 - I chose `Dev1` and `Dev2` from the list and added them to this group.
   <br>![[devteam add users.png]]

4. **Adding Dev1, Test1, and Test2 to the OpsTeam:**
 - I repeated the same process for the `OpsTeam` group.
 - I clicked on `Add users` and added `Dev1`, `Test1`, and `Test2` to this group.
 <br>![[opsteams add user.png]]

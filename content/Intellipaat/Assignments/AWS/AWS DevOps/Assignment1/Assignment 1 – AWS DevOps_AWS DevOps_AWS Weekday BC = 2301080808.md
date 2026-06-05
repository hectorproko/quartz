---
tags:
  - AWS
  - AWSDevOps
title: Setting Up a CodeCommit Repository and Migrating Data from GitHub
---
<!--
**Mini-Project: Streamlining DevOps Workflows with AWS CodeCommit!** In this project, I embraced the role of a DevOps engineer at XYZ Corporation, tackling the challenge of migrating repository content from GitHub to AWS CodeCommit. This involved creating an IAM user, uploading SSH keys, setting up a CodeCommit repository, and executing a seamless repository transfer. This task not only demonstrated my skills in repository management using AWS services but also highlighted the practical aspects of DevOps workflows in a cloud environment.
-->
 
> [!info] AWS DevOps Assignment - 1
> **Problem Statement:**
> You work for XYZ Corporation. Your corporation wants the files to be stored on a private repository on the AWS Cloud. 
> 
> **Tasks To Be Performed:** 
> 1. Create a CodeCommit repository and import a GitHub repository content into it.



### 1. Creating an IAM User:
1. I log into AWS.
2. I navigate to IAM.
3. I create a user named "GitUser."
4. I attach the `AWSCodeCommitFullAccess` policy.

### 2. Uploading Public SSH Key to the IAM User:
1. I click on the user.
2. I select the "Security credentials" tab.
3. I press the "Upload SSH public keys" button.
4. A window appears.
5. I paste my public key into the provided input box.
6. I click the "Upload SSH public key" button.
   <br>![[Pasted image 20231018201404.png]]
7. Once done, AWS gives me an SSH Key ID.
8. I note down this ID for future use.
   <br>![[Pasted image 20231018201734.png]]

### 3. Setting Up the SSH Client:
1. I open the `~/.ssh/config` file, or create it if it doesn't exist.
2. I add the following configuration:
	```css
	Host git-codecommit.*.amazonaws.com
	User APKAG4NQRHE3SUWI647
	IdentityFile ~/.ssh/id_rsa
	```
	<br>![[Pasted image 20231018202828.png]]


### 4. **Set Up CodeCommit Repository**
1. I go to AWS CodeCommit in the AWS Management Console.
2. I click the "Create repository" button.
3. I name the repository "Packer", matching the GitHub repo name.
4. I press "Create".
<br>![[Pasted image 20231018203812.png]]

### 5. Cloning GitHub Repository
I navigate to my GitHub repository and copy the SSH URL: `git@github.com:hectorproko/Packer.git`.
<br>![[Pasted image 20231018203924.png]]

On my local machine, I run:
```bash
git clone git@github.com:hectorproko/Packer.git
cd Packer
```

### 6. **Adding CodeCommit Repository as a New Remote:**

I go to the AWS CodeCommit console and navigate to the repository I just set up. I click on the Clone URL dropdown and choose Clone SSH, then I copy this URL.
<br>![[Pasted image 20231018204407.png]]

Back in my local repository directory on my machine, I enter:
```bash
git remote add codecommit ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/Packer
```

To verify the remote was successfully added, I type:
```
git remote -v
```
<br>![[Pasted image 20231018205114.png]]


### 7. **Pushing Repository Content to CodeCommit:**

Ensuring my SSH setup is correct, I can now push my local repository content to CodeCommit. I use the command:
```
git push codecommit main
```

<br>![[Pasted image 20231018210300.png]]


### **8. Verifying the Repository Content:**
I open the CodeCommit repository in the AWS Management Console. By checking the listed files and their commit history, I ensure they match what's in my original GitHub repository.
<br>![[Pasted image 20231018210356.png]]

> [!success]
> The files in CodeCommit are the same as those in my GitHub repository.
---
title: Continuous Integration Pipeline for Tooling Website
tags:
  - Linux
  - LoadBalancer
  - Jenkins
  - webhooks
  - CD/CD
  - draft
linkedin: "False"
quartz: "True"
refactored: "True"
darey.io: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
aliases:
  - "Project 9: Continuous Integration Pipeline for Tooling Website"
---
## Overview

This project builds on the load-balanced architecture from [[Project 8 APACHE LOAD-BALANCER|Project 8: Load Balancer Solution with Apache]] by introducing Jenkins as a CI automation layer. The goal is to eliminate manual deployments: any code change pushed to the GitHub repository automatically triggers a build that delivers the updated source to the NFS server, keeping the tooling website in sync without human intervention.

![Architecture diagram](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/architecture.png)

---

## Step 1: Provision the Jenkins Server on AWS

A dedicated EC2 instance is created to host Jenkins, separate from the web servers. This separation keeps the CI tooling isolated from the application layer.

1. Launch an **Ubuntu Server 20.04** EC2 instance. [Creating an Ubuntu instance in AWS (EC2)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/AWS_Ubuntu_Instnace.md)
    
2. Open **TCP port 8080** in the instance's Security Group Inbound Rules - Jenkins listens on this port by default. [Opening Ports in AWS (EC2)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/OpenPortAWS.md)
    

---

## Step 2: Install Jenkins

Jenkins is installed via its official Debian package repository to ensure we get a stable, supported release.

```bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'

sudo apt update
sudo apt-get install jenkins
```

Once installed, Jenkins is accessible through the browser at `http://<instance-public-ip>:8080`. On first login, Jenkins requires an initial admin password to confirm the setup is intentional and not accidental.

![Jenkins unlock screen](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/unlock.png)

The password is retrieved from the server:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Example output:**

```
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

8cba9d4d4dcc4658931ba22fe9435521

This may also be found at: /root/.jenkins/secrets/initialAdminPassword

*************************************************************
```

When prompted, choose **Install suggested plugins**. These cover the most common use cases including Git integration, which is needed for this project.

![Suggested plugins screen](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/suggested.png)

After plugins install, create an admin user and the setup is complete.

---

## Step 3: Configure Jenkins to Pull from GitHub via Webhooks

The job is configured so that Jenkins polls for source code changes and more importantly, GitHub notifies Jenkins directly via a webhook whenever a push happens, so builds trigger in real time rather than on a schedule.

### 3.1 Create the Jenkins Freestyle Job

In the Jenkins web console, click **New Item**, name the project, and select **Freestyle project**. Click **OK**.

![New job creation](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/job.png)

In the job configuration, go to the **Source Code Management** tab, select **Git**, and enter the tooling repository URL:

```
https://github.com/hectorproko/tooling.git
```

Click **Save**.

![Source code management configuration](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/sourcecode.png)

#### Troubleshooting: Branch Name Mismatch

When specifying the repository branch, Jenkins defaults to `master`. If the local repository uses `main` as the default branch, changing the branch specifier to `main` produces the following error:

```
❌ERROR: Couldn't find any revision to build. Verify the repository and branch configuration for this job.
Finished: FAILURE
```

The fix is to leave the branch specifier as `master`, or update it to match the exact branch name that exists on the remote, and verify the branch name directly on GitHub before saving.

---

### 3.2 Configure Build Triggers and Artifacts

After saving, go back into the job configuration:

- Under **Build Triggers**, enable **GitHub hook trigger for GITScm polling**. This tells Jenkins to listen for incoming webhook payloads from GitHub instead of polling on a timer.
- Under **Post-build Actions**, add **Archive the artifacts** and set the file pattern to `**` to capture everything the build produces.

![Build triggers and post-build actions](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/buildtriggers.png)

---

### 3.3 Configure the GitHub Webhook

The webhook is what connects GitHub to Jenkins. When code is pushed to the repository, GitHub sends an HTTP POST to the Jenkins webhook endpoint, which starts the build automatically.

In the GitHub repository, go to **Settings** > **Webhooks** > **Add webhook**.

![Webhook list](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/webhooks1.png)

Fill in the webhook settings:

- **Payload URL:** `http://<jenkins-public-ip>:8080/github-webhook/`
- **Content type:** `application/json`

![Webhook configuration](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/webhooks2.png)

---

## Step 4: Verify the Pipeline End-to-End

To confirm the full pipeline works, a test file is committed and pushed directly to the repository.

![Test commit in GitHub](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/commit.png)

The push immediately triggers build **#5** in Jenkins - no manual action needed.

![Build #5 triggered](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/build5.png)

Reviewing the console output for build #5 confirms the job was started by a GitHub push event, not a manual trigger.

![Build #5 console log](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/log.png)

At this point the pipeline is fully operational: every code push to the GitHub repository automatically kicks off a Jenkins build that retrieves the latest source, archives the artifacts, and makes them available for delivery to the NFS server.

%%
# Original

PROJECT 9

> [!info]
> In this project, we'll extend the architecture developed in [[Project 8 APACHE LOAD-BALANCER]] by integrating a Jenkins server. Our focus will be on automating routine tasks using Jenkins, a free and open-source automation tool. We'll set up a Jenkins job designed to automatically deploy any changes made to the source code in our GitHub repository, [tooling](https://github.com/hectorproko/tooling), directly to the Tooling Website via an NFS server.
> 
> ![[darey.io/Project9/images/architecture.png]]
> 

## Provision AWS Server

 1. Create an Ubuntu Server 20.04  
[Creating Ubuntu instance in AWS (**EC2**)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/AWS_Ubuntu_Instnace.md)

2. Open TCP port **8080** creating an **Inbound Rule** in Security Group.  
[Open Ports in AWS (**EC2**)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/OpenPortAWS.md)

## Install Jenkins
``` bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'

sudo apt update
sudo apt-get install jenkins
```
We access jenkins through the browser using the instance IP.
You will be prompted to provide a default **admin password**

![Markdown Logo](media/Markdown_Logo-27.png)
  
We retrieve the password from the specified path
``` bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

``` bash
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

8cba9d4d4dcc4658931ba22fe9435521

This may also be found at: /root/.jenkins/secrets/initialAdminPassword

*************************************************************
```

Then you will be asked which plugings to install – choose **suggested plugins**.  

![Markdown Logo](media/Markdown_Logo-25.png)  

Once plugins installation is done you are prompted to create an admin user and you will get your Jenkins server address and the installation is complete.



## Configure Jenkins to retrieve source codes from GitHub using Webhooks
In this part we'll configure a simple Jenkins job/project that will be triggered by GitHub webhooks and will execute a ‘build’ task to retrieve codes from GitHub and store it locally on Jenkins server.

### 1. Create Jenkins Job
Go to Jenkins web console, click **New Item** and create a **Freestyle project**  
Press **OK** at the bottom

![Markdown Logo](media/Markdown_Logo-23.png)  

We get prompted to the job configuration. In the **Source Code management** tab we select **Git** and put the **URL** of **tooling** repo
https://github.com/hectorproko/tooling.git  
Click **Save**

![Markdown Logo](media/Markdown_Logo-28.png)  


*Keep In Mind:  
When specifying repo **branch**, Jenkins uses **master** even though my local repo shows **main**, if I change branch specifier to **main** I get error  
ERROR: Couldn't find any revision to build. Verify the repository and branch configuration for this job.
Finished: FAILURE*

Once configuration is saved you are taken to the **Jobs Dashboard** where we can do test build by clicking on **Build Now**

![Markdown Logo](media/Markdown_Logo-26.png)  

  
Now we configure the job to trigger whenever there is a change in the **sourcecode** in Github's Repo

In **Build Triggers** Tab check box **GitHub hook trigger for GITScm polling**  
In **Post-build Actions** we archive all the artifacts with **\***

![Markdown Logo](media/Markdown_Logo-26.png)  

### Configuring webhook in github repo
To add a webhook we go to  **Settings** > **Webhooks** > **Add webhook**

![Markdown Logo](media/Markdown_Logo-27.png)  

On **Payload URL** we put the **public IP** (of Jenkins instance) and port **8080** followed by **/github-webhhook/**  
Make sure **Content type** is set to **application/json**

![Markdown Logo](media/Markdown_Logo-28.png)


Here I'm creating a **test** file in the repo to test the webhook

![Markdown Logo](media/Markdown_Logo-27.png)  

This triggers build **#5**

![Markdown Logo](media/Markdown_Logo-24.png)  

If I look at the **conosole output** or log of **build 5** I confirm that the job was triggered by **Github push**

![Markdown Logo](media/Markdown_Logo-24.png)  


%%
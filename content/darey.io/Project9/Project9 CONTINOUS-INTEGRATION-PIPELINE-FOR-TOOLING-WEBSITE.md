---
title: Continuous Integration Pipeline for Tooling Website
tags:
  - Linux
  - LoadBalancer
  - Jenkins
  - webhooks
  - CI/CD
---

 ==*~~(old [Project 9](https://github.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE))~~*==
 
> [!info]
> In this project, we'll extend the architecture developed in [[Project_8_APACHE_LOAD-BALANCER]] by integrating a Jenkins server. Our focus will be on automating routine tasks using Jenkins, a free and open-source automation tool. We'll set up a Jenkins job designed to automatically deploy any changes made to the source code in our GitHub repository, [tooling](https://github.com/hectorproko/tooling), directly to the Tooling Website via an NFS server.
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

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/unlock.png)
  
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

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/suggested.png)  

Once plugins installation is done you are prompted to create an admin user and you will get your Jenkins server address and the installation is complete.



## Configure Jenkins to retrieve source codes from GitHub using Webhooks
In this part we'll configure a simple Jenkins job/project that will be triggered by GitHub webhooks and will execute a ‘build’ task to retrieve codes from GitHub and store it locally on Jenkins server.

### 1. Create Jenkins Job
Go to Jenkins web console, click **New Item** and create a **Freestyle project**  
Press **OK** at the bottom

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/job.png)  

We get prompted to the job configuration. In the **Source Code management** tab we select **Git** and put the **URL** of **tooling** repo
https://github.com/hectorproko/tooling.git  
Click **Save**

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/sourcecode.png)  


*Keep In Mind:  
When specifying repo **branch**, Jenkins uses **master** even though my local repo shows **main**, if I change branch specifier to **main** I get error  
ERROR: Couldn't find any revision to build. Verify the repository and branch configuration for this job.
Finished: FAILURE*

Once configuration is saved you are taken to the **Jobs Dashboard** where we can do test build by clicking on **Build Now**

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/buildnow.png)  

  
Now we configure the job to trigger whenever there is a change in the **sourcecode** in Github's Repo

In **Build Triggers** Tab check box **GitHub hook trigger for GITScm polling**  
In **Post-build Actions** we archive all the artifacts with **\***

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/buildtriggers.png)  

### Configuring webhook in github repo
To add a webhook we go to  **Settings** > **Webhooks** > **Add webhook**

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/webhooks1.png)  

On **Payload URL** we put the **public IP** (of Jenkins instance) and port **8080** followed by **/github-webhhook/**  
Make sure **Content type** is set to **application/json**

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/webhooks2.png)


Here I'm creating a **test** file in the repo to test the webhook

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/commit.png)  

This triggers build **#5**

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/build5.png)  

If I look at the **conosole output** or log of **build 5** I confirm that the job was triggered by **Github push**

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/main/images/log.png)  



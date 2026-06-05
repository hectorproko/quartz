---
tags:
  - Jenkins
---



> [!info] Module 6: Case Study - Integration of DevOps Tools with Jenkins
> **Problem Statement:** 
> You have been hired as a DevOps Engineer in XYZ software company. They want to implement CI/CD pipeline in their company. You have been asked to implement this lifecycle as fast as possible. As this is a product-based company, their product is available on this GitHub link. https://github.com/hshar/website.git 
> 
> Following are the specifications of the continuous integration: 
> 1. Git workflow has to be implemented 
> 2. CodeBuild should automatically be triggered once a commit is made to master branch or develop branch. If a commit is made to master branch, build and publish a website on port 82. If a commit is made to develop a branch, just build the product, do not publish. 
> 3. Create a pipeline for the above tasks 
> 4. Create a container with Ubuntu and Apache installed in it and use that container to build the code and the code should be on ‘/var/www/html’.

> [!attention] Prerequisite
> I encountered issues while attempting to fork the '[https://github.com/hshar/website.git](https://github.com/hshar/website.git)' repository. Consequently, I created my own repository named [HelloWorld_website](https://github.com/hectorproko/HelloWorld_website)
> 


I start by creating a `Dockerfile` and then push it to the **master** branch with the following content:
```dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install apache2 -y
ADD . /var/www/html
ENTRYPOINT apachectl -D  FOREGROUND
```

I create a  **develop** branch
```bash
git branch develop
```

In our Prod node we need to install docker
```bash
sudo apt update -y 
sudo apt install docker.io -y
```
<br>![[Pasted image 20231108153326.png]]

We create a Freestyle project named 'Develop Job.' This job is configured only to build (clone repo) and its not meant to publish the output.

%%[[Develop Job config.png]]%%
<br>![[Pasted image 20231108154808.png|330]]
<br>![[prod node restriction.png|330]]
<br>![[branch specifier develop.png|330]]
<br>![[GitHub hook trigger for GITScm polling.png|330]]

In our new repository, we need to configure Webhooks as we did for [[Assignment 1 – Jenkins_Module 6_Devops BC = 2330070508|Assignment 1 – Jenkins]].
<br>![[webhook payload URL.png|450]]
%%This is from Asig1, by the time we got to this step i had restarted server and had a new public ip%%


Then, we create a 'Master Job' identical to the 'Develop Job' but with a different Branch Specifier.
<br>![[branch specifier main.png]]


We execute the job to verify that the files are correctly appearing in the Prod node.
<br>![[Pasted image 20231108173633.png]]

I add 'Build Steps' configured to execute shell commands.
<br>![[Pasted image 20231108174132.png]]
Then, I insert the commands into the designated section.
<br>![[Pasted image 20231108185028.png]]

> [!NOTE] Commands explained: 
> Removes a specific container (if it exists):
> ```bash
> sudo docker rm -f websitecontainer || true
> ```
> *The `|| true` part ensures that the command will not fail if th container does not exist (which would otherwise stop the Jenkins job if set to fail on any error).*  
> 
> > [!attention]
> > If we do not include a cleanup step, running the job multiple times can lead to conflicts due to the presence of a previously running container with the same name. <br>![[Pasted image 20231107214943.png]]
> 
> Builds the new Docker image:
> ```bash
> sudo docker build . -t masterapp
> ```
> 
> Runs the new container:
> ```bash
> sudo docker run -d --name websitecontainer -p 82:80 masterapp
> ```
> *Maps port 82 on the host machine to port 80 on the container*
> 
> %%Tutorial had -itd which as an interactive terminal, which doesnt make sense for our use case%%

^77e322

I "Build Now" the job and it success
<br>![[Pasted image 20231108180221.png]]

We verify the hosted page by accessing the Prod node's public IP address through port 82. 

> [!NOTE]
> An inbound rule was added to the Security Group to allow traffic on port 82.

> [!success]
> <br>![[Pasted image 20231108183535.png]]


Now, let's test the webhook functionality. To do this, I will edit the 'index.html'. The changes will be made directly on GitHub, followed by committing the update to the repository.

Job successfully triggered and built
<br>![[Pasted image 20231108184259.png|500]]

> [!success]
> <br>![[Pasted image 20231108184333.png|500]]

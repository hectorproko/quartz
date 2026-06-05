---
tags:
  - Ansible
  - docker
  - Jenkins
---

> [!info]- Project : Capstone I
> You have been hired as a Sr. DevOps Engineer in Abode Software. They want to implement DevOps Lifecycle in their company. You have been asked to implement this lifecycle as fast as possible. Abode Software is a product-based company and their product is available on this GitHub link. 
> https://github.com/hshar/website.git 
> 
> Following are the specifications of the lifecycle:
> 1. Install the necessary software on the machines using a configuration management tool 
> 2. Git workflow has to be implemented 
> 3. CodeBuild should automatically be triggered once a commit is made to master branch or develop branch. 
>    `a.` If a commit is made to master branch, test and push to prod 
>    `b.` If a commit is made to develop branch, just test the product, do not push to prod 
> 4. The code should be containerized with the help of a Dockerfile. The Dockerfile should be built every time there is a push to GitHub. Use the following pre-built container for your application: `hshar/webapp` The code should reside in `/var/www/html `
> 5. The above tasks should be defined in a Jenkins Pipeline with the following jobs: 
>    `a.` Job1 : build 
>    `b.` Job2 : test 
>    `c.` Job3 : prod


### Provisioning Instances

I create three Ubuntu 22.04 EC2 instances, each located in a public subnet and configured to have a public IP enabled. This setup ensures that the instances are accessible and can handle network traffic effectively.

%%'All Traffic' security group%%

> [!NOTE] I name them:
> 
> Master `10.0.1.76`
> Test   `10.0.1.51`
> Prod `10.0.1.218`
> 

### Master
Installing Ansible<br>![[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508#^aa5175]]
[[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508]]

<br>![[Pasted image 20231205121617.png]]
*This is a more updated version than [[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508|Assignment 1 – Ansible]]*


---

%%Now we creates ssh key to be able to ssh into our salves [[ssh (Secure Shell)#Install ssh keys on a remote machine|following]] could not follow those steps cuse EC2 ssh doesnt use password%%



I generate the key by accepting the default options, simply pressing 'Enter' at each prompt.
<br>![[Pasted image 20231205123901.png]]

I copy the contents of the public key
```bash
cat ~/.ssh/id_rsa.pub
```

I'll paste the contents of `id_rsa.pub` in the `authorize_keys` file belonging to the node we want ansible to connect to "Test" and "Prod"

```bash
sudo vi ~/.ssh/authorized_keys
```

---
I add 'Test' and 'Prod' environments to my Ansible inventory.

%%forgot how to (the format) and looked at [[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508|Assignment 1 – Ansible]]%%

```bash
[webservers]
Test ansible_host=10.0.1.51 ansible_user=ubuntu
Prod ansible_host=10.0.1.218 ansible_user=ubuntu
```

> [!help] `ansible -m ping` fail
> > [!fail]
> > <br>![[ansible -m ping all fail.png]]
> 
> > [!done] Solution: `authorized_keys` can NOT have space
> > <br>![[authorize_keys can not have space.png]]
> 

^061158

I verify the node connection once again.
<br>![[Pasted image 20231205131501.png]]

---

I create playbook `play.yaml`

```yaml
---
- name: Install Jenkins on Master
  hosts: localhost
  become: true
  tasks:
    - name: Install OpenJDK 17
      apt:
        name: openjdk-17-jdk
        state: present
        update_cache: true

    - name: Add Jenkins repository key
      apt_key:
        url: https://pkg.jenkins.io/debian/jenkins.io-2023.key
        state: present

    - name: Add Jenkins repository
      apt_repository:
        repo: deb https://pkg.jenkins.io/debian binary/
        state: present
        update_cache: true

    - name: Install Jenkins
      apt:
        name: jenkins
        state: present
        update_cache: true

- name: Install Docker.io, Java on Slave Nodes
  hosts: all
  become: true
  tasks:
    - name: Install OpenJDK 17
      apt:
        name: openjdk-17-jdk
        state: present
        update_cache: true
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install docker.io
      apt:
        name: docker.io
        state: present

    - name: Ensure Docker service is running
      systemd:
        name: docker
        state: started
        enabled: yes
```

Before I check syntax
<br>![[Pasted image 20231205143354.png]]


We successfully ran the command `ansible-playbook play.yaml` with 0 failures.
<br>![[Pasted image 20231205144134.png]]


**Verifying installations**

master:
<br>![[Pasted image 20231205144439.png]]
slaves:
<br>![[Pasted image 20231205144336.png]]
<br>![[Pasted image 20231205144358.png]]


---
%%the steps are really in [[Installing Jenkins On AWS]] %%

In addition to getting Jenkins ready, I add SSH credentials and configure agents, following the same steps from [[Assignment 1 – Jenkins_Module 6_Devops BC = 2330070508|Assignment 1 – Jenkins]].

%%[[Pasted image 20231104211939.png|Creating admin user]]
[[Installing Jenkins On AWS#Create credentials|Creating credentials]]%%
<br>![[Pasted image 20231205184831.png]]

%%[[Installing Jenkins On AWS#Setting up slaves|adding agents]]%%
<br>![[Pasted image 20231205190333.png]]

---
I'm using the [HelloWorld_website](https://github.com/hectorproko/HelloWorld_website) repository, the same one used in [[Case Study 1 – Jenkins_Module 6_Devops BC = 2330070508|Case Study 1 – Jenkins]]. This repository includes a Dockerfile and the website's components, such as 'index.html' and image files.

`Dockerfile`
```bash
FROM ubuntu
RUN apt-get update
RUN apt-get install apache2 -y
ADD . /var/www/html
ENTRYPOINT apachectl -D  FOREGROUND
```
Install apache and hosts the page in Github
This Dockerfile creates a Docker image based on Ubuntu that installs and runs the Apache web server, serving the contents of [HelloWorld_website](https://github.com/hectorproko/HelloWorld_website) from the `/var/www/html` directory within the container.

---
<br>![[Pasted image 20231130202539.png]]

### Job1
I have configured Job1 as a Freestyle Project. This job is designed to run in the "Test" environment using the "Develop" branch of the [HelloWorld_website](https://github.com/hectorproko/HelloWorld_website) repository.
<br>![[Pasted image 20231205192552.png|330]]
<br>![[Pasted image 20231205192807.png|450]]
<br>![[GitHub hook trigger for GITScm polling.png|330]]

I insert the commands into "Execute shell" as I did in [[Case Study 1 – Jenkins_Module 6_Devops BC = 2330070508#^77e322|Case Study 1 – Jenkins]]
<br>![[Pasted image 20231205194113.png|500]]

In the "Test" environment, I observe that within the workspace, **Job1** is present, and additionally, there's a container actively running.
<br>![[Pasted image 20231205194659.png]]

I test this container by using the Public IP of the 'Test' server and directing traffic to port 83, which is mapped to the container.

> [!success]
> <br>![[Pasted image 20231205195704.png]]

---
### Job2
Job2 is configured as a Freestyle Project in the Jenkins setup. It is designed to run in the "Test" environment using the "main" branch.

<br>![[Pasted image 20231205203034.png]]
<br>![[Pasted image 20231205203336.png]]

Once again I check the "Test" environment, I observe that within the workspace, **Job2** is present, and now there is a second container .
<br>![[Pasted image 20231205210203.png]]

I test this container by using the Public IP of the 'Test' server and directing traffic this time to port 82, which is mapped to the container.

> [!success]
> <br>![[Pasted image 20231205210514.png]]


---
### Job 3
Job3 is configured as a Freestyle Project in the Jenkins setup. It is designed to run in the "Prod" environment using the "main" branch.

<br>![[Pasted image 20231205203034.png]]

<br>![[Pasted image 20231205202852.png]]

<br>![[Pasted image 20231205211700.png]]

In the "Prod" environment, I observe that within the workspace, **Job3** is present, and additionally, there's a container actively running.
<br>![[Pasted image 20231205211852.png]]

I test this container by using the Public IP of the "Prod" server and directing traffic this time to port 80 (default), which is mapped to the container.

> [!success]
> <br>![[Pasted image 20231205212855.png]]





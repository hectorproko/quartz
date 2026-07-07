---
title: "Project 11: Ansible-Automate"
tags:
  - Ansible
linkedin: "False"
quartz: "False"
refactored: "False"
darey.io: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
aliases:
  - "Project 11: Ansible-Automate"
---
%%~~*(Old [Project 11](https://github.com/hectorproko/ANSIBLE-AUTOMATE))*~~%%
The goal of this project is to begin automating tasks using **Ansible Configuration Management**, reducing manual effort and making infrastructure changes repeatable and consistent.

---
## Environment

| Role           | IP              | OS      |
| -------------- | --------------- | ------- |
| Control Node   | `172.31.94.159` | Ubuntu  |
| Managed Node 1 | `3.220.20.204`  | Ubuntu  |
| Managed Node 2 | `54.209.253.1`  | Red Hat |
## Install and Configure Ansible on EC2 Instance

An existing **EC2** instance with **Jenkins** already installed is used as the control node for Ansible.

A new repository named **ansible-config-mgt** is created in **GitHub** to store all configuration and playbook files.

Ansible is installed and the version is confirmed:

```bash
sudo apt update
sudo apt install ansible
ansible --version
```

---

## Configure Jenkins to Trigger on Repository Changes

A new **Freestyle** project named **ansible** is created in **Jenkins** and pointed to the **[ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt)** repository. This job will pull the latest repository contents each time a change is pushed.

![Jenkins - New Freestyle Project](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/enterItem.png)

![Jenkins - Repository URL](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/repositoryURL.png)

A **Webhook** is configured to automatically trigger the **ansible** build job on each push.

![Jenkins - Build Triggers](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/buildTriggers.png)

The **Webhook** is also registered on the **GitHub** repository side.

![GitHub - Add Webhook](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/addWebhook.png)

A **Post-build** step is configured to archive all files (`**`). This is similar to the setup done in [[Project9 CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE|Project 9: Continuous Integration Pipeline for Tooling Website]]. 

The setup is tested by making a change to the **README.md** file on the `main` branch. The build starts automatically and **Jenkins** saves the artifacts at:

```
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

---

## Set Up Directory Structure in a Feature Branch

In the [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt/tree/main) repository, a new branch named [NewFeature](https://github.com/hectorproko/ansible-config-mgt/tree/NewFeature) is created for development work.

The branch is checked out locally to build the directory structure:

- **playbooks/** - stores all playbook files
- **inventory/** - keeps host definitions organized by environment

The first playbook, **common.yml**, is created inside the `playbooks/` folder.

Inventory files are created for each environment inside `inventory/`:

|File|Environment|
|---|---|
|`dev`|Development|
|`staging`|Staging|
|`uat`|User Acceptance Testing|
|`prod`|Production|

![Inventory Files Created](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/inventory.png)

> Note: These inventory files are placeholders at this stage and do not yet have `.yml` or `.ini` extensions. This project uses a [Dynamic Inventory](https://github.com/hectorproko/Ansible/blob/main/Ansible_setup.md) with the **aws_ec2** plugin instead.

---

## Configure SSH Agent for Key-Based Access
%%[[ssh (Secure Shell)#ssh agent, eval]]%%
The SSH key is imported into `ssh-agent` so Ansible can connect to managed nodes without specifying a `.pem` file on each command:

```bash
ubuntu@ip-172-31-94-159:~$ eval `ssh-agent -s`
Agent pid 35692
ubuntu@ip-172-31-94-159:~$ ssh-add /home/ubuntu/daro.io.pem
Identity added: /home/ubuntu/daro.io.pem (/home/ubuntu/daro.io.pem)
ubuntu@ip-172-31-94-159:~$ ssh-add -l
2048 SHA256:wq3aSIyEU9dbylJv50YOhgXXPfcxZT7J5EmRIjLXZeA /home/ubuntu/daro.io.pem (RSA)
```

SSH agent forwarding is enabled when connecting to the Jenkins/Ansible instance:

```bash
ssh -A ubuntu@public-ip
```

---

## Merge NewFeature Branch into Main

Before merging, the differences in directory contents between branches are visible:

**Main branch:**

```bash
hector@hector-Laptop:~/ansible-config-mgt$ git branch
* main
hector@hector-Laptop:~/ansible-config-mgt$ ls
2plays.yml  README.md
```
%%The branch may have existed on GitHub already, but `git branch` only lists **local** branches. Until you run `git checkout NewFeature` (or `git checkout -b NewFeature` to create it), it won't show up in that list.%%
**NewFeature branch:**

```bash
hector@hector-Laptop:~/ansible-config-mgt$ git branch
* NewFeature
  main
hector@hector-Laptop:~/ansible-config-mgt$ ls
inventory  playbooks  README.md
```

A pull request is opened to merge **NewFeature** into **main**.

![GitHub - Comparing Changes](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/comparingChanges.png)

![GitHub - Merge Pull Request](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/mergePull.png)

![GitHub - Successfully Merged](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/successfullyMerged.png)

Switching back to `main` locally and pulling the merged changes:

```bash
hector@hector-Laptop:~/ansible-config-mgt$ git checkout main
Switched to branch 'main'
Your branch is behind 'origin/main' by 3 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)
hector@hector-Laptop:~/ansible-config-mgt$ git pull
Updating 4fae522..84424ae
Fast-forward
 inventory/dev        | 0
 inventory/prod       | 0
 inventory/staging    | 0
 inventory/uat        | 0
 playbooks/common.yml | 0
 5 files changed, 0 insertions(+), 0 deletions(-)
hector@hector-Laptop:~/ansible-config-mgt$ ls
2plays.yml  inventory  playbooks  README.md
```

The merge triggers a new Jenkins build via the webhook.

![Jenkins - Console Output](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/consoleOutput.png)

---

## Install Ansible Plugin in Jenkins

The Ansible plugin is installed in Jenkins for future integration. For this project, playbooks are run manually via the command line to validate the setup first.

![Jenkins - Ansible Plugin](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/plugin.png)

*Used in [[Project 14 Continues Integration with Jenkins, Ansible, Artifactory, SonarQube & PHP#Running Ansible Playbooks from Jenkins|Project 14: Continues Integration with Jenkins, Ansible, Artifactory, SonarQube & PHP]]*

---

## Test Connectivity with a Ping Playbook

Before running any real tasks, connectivity to all managed nodes is validated using a ping playbook (**2plays.yml**).

Two **EC2** instances are used as targets, tagged `Jenkins_Ansible` and `NFS`. The dynamic inventory plugin maps these tags to host groups `_Jenkins_Ansible` and `_NFS`.

```yaml
---
- name: update web servers
  hosts: _Jenkins_Ansible
  remote_user: ubuntu

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: Pinging
      ping:

- name: update db servers
  hosts: _NFS
  remote_user: ec2-user

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: Pinging
      ping:
```

The playbook is run without specifying an inventory file since the dynamic inventory is set as the default:

```bash
ansible-playbook 2plays.yml
```

Both hosts respond successfully with 0 failures:

```
PLAY RECAP ************************************************************
ec2-3-220-20-204.compute-1.amazonaws.com  : ok=2  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
ec2-54-209-253-1.compute-1.amazonaws.com  : ok=2  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

---

## Deploy Wireshark with common.yml

With connectivity confirmed, **common.yml** is used to install Wireshark on both servers. The playbook handles the package manager difference between Ubuntu (`apt`) and Red Hat (`yum`):

```yaml
---
- name: update web servers
  hosts: _Jenkins_Ansible
  remote_user: ubuntu

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest

- name: update db servers
  hosts: _NFS
  remote_user: ec2-user

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest
```

Both hosts report a successful install (`changed=1`):

```
PLAY RECAP ************************************************************
ec2-3-220-20-204.compute-1.amazonaws.com  : ok=3  changed=1  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
ec2-54-209-253-1.compute-1.amazonaws.com  : ok=3  changed=1  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

Installation is verified directly on the NFS server:

```bash
[ec2-user@ip-172-31-81-201 ~]$ sudo yum list installed | grep wire
wireshark.x86_64
```

Wireshark is then removed by changing the state in the playbook:

```yaml
state: absent
```

---

## Additional Ansible Playbooks

- [Project 7: DevOps Tooling Website Solution](https://github.com/hectorproko/Ansible/tree/main/Project7)
- [Project 8: Load Balancer Solution with Apache](https://github.com/hectorproko/Ansible/tree/main/Project8)
- [Project 9: Continuous Integration Pipeline for Tooling Website](https://github.com/hectorproko/Ansible/tree/main/Project9)

<!---
# ANSIBLE AUTOMATION
The aim of this project is to start automating tasks with **Ansible Configuration Management**.

## INSTALL AND CONFIGURE ANSIBLE ON EC2 INSTANCE
I will use an already existing **EC2** instance with **Jenkins** Installed
 

In my **GitHub** account I will create a new repository and name it **ansible-config-mgt**

Check your Ansible version by running ansible 

``` bash
sudo apt update
sudo apt install ansible
ansible --version #confirm intallation
```


I'll create a **Jenkins** job *(project)* that will update the contents of the repository locally each time there is a change in the repo

Creating a new **Freestyle** project named **ansible** in **Jenkins** and pointing it to your **ansible-config-mgt** repository.

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/enterItem.png)     

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/repositoryURL.png)  


Setting **Webhook** to trigger **ansible** job build.  

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/buildTriggers.png)  

Configuring **Webhook** in **GitHub**  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/addWebhook.png)  


Configured a **Post-build** step to save all **\(\**)** files. Similar to [Project 9](https://github.com/hectorproko/CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE/blob/main/Project9.md#configure-jenkins-to-retrieve-source-codes-from-github-using-webhooks)

We can test the setup by making a change in the **README.md** file in *main branch*, build starts automatically and **Jenkins** saves the files (build artifacts) in the following path

`ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/`


In the **[ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt/tree/main)** **GitHub** repository, I create a new branch [NewFeature](https://github.com/hectorproko/ansible-config-mgt/tree/NewFeature) that will be used for development of a new feature.  


I'll checkout the newly created branch in my local machine to build code and directory structure  

Created a directory **playbooks** – used to store all my playbook files.  

Created a directory **inventory** – used to keep your hosts organized.

Within the playbooks folder I will create my first playbook, and name it **common.yml**

Within the inventory folder, created inventory files for each environment (Development, Staging Testing and Production) **dev**, **staging**, **uat**, and **prod** respectively.

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/inventory.png)

Above image of created files (inventory).  
_This manual inventory is a placeholder for now (doesn't even have **.yml** or **.ini** extentions )_

In this project I will be using a [Dynamic Inventory](https://github.com/hectorproko/Ansible/blob/main/Ansible_setup.md) with plugin **aws_ec2.yml**   

Now I import your key into ssh-agent:


	
``` bash
ubuntu@ip-172-31-94-159:~$ eval `ssh-agent -s`
Agent pid 35692 #agent started
ubuntu@ip-172-31-94-159:~$ ssh-add /home/ubuntu/daro.io.pem
Identity added: /home/ubuntu/daro.io.pem (/home/ubuntu/daro.io.pem)
ubuntu@ip-172-31-94-159:~$ ssh-add -l #confirm key was added
2048 SHA256:wq3aSIyEU9dbylJv50YOhgXXPfcxZT7J5EmRIjLXZeA /home/ubuntu/daro.io.pem (RSA)
```
Now I can **ssh** into my Jenkins/Ansible instance without having to specify a **.pem** key

``` bash 
ssh -A ubuntu@public-ip
```


Current files in **Main**

``` bash
hector@hector-Laptop:~/ansible-config-mgt$ git branch
* main
hector@hector-Laptop:~/ansible-config-mgt$ ls
2plays.yml  README.md
hector@hector-Laptop:~/ansible-config-mgt$
```

Current files in **NewFeature**

``` bash
hector@hector-Laptop:~/ansible-config-mgt$ git branch
* NewFeature
  main
hector@hector-Laptop:~/ansible-config-mgt$ ls
inventory  playbooks  README.md
hector@hector-Laptop:~/ansible-config-mgt$
```

We do a pull request to **merge** NewFeature to Main
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/comparingChanges.png)  

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/mergePull.png)  


Confirmation  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/successfullyMerged.png)  


``` bash
hector@hector-Laptop:~/ansible-config-mgt$ git checkout main
Switched to branch 'main'
Your branch is behind 'origin/main' by 3 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)
hector@hector-Laptop:~/ansible-config-mgt$ git branch
  NewFeature
* main
hector@hector-Laptop:~/ansible-config-mgt$ ls
2plays.yml  README.md
hector@hector-Laptop:~/ansible-config-mgt$ git pull
Warning: Permanently added the ECDSA host key for IP address '140.82.112.3' to the list of known hosts.
Updating 4fae522..84424ae
Fast-forward
 inventory/dev        | 0
 inventory/prod       | 0
 inventory/staging    | 0
 inventory/uat        | 0
 playbooks/common.yml | 0
 5 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 inventory/dev
 create mode 100644 inventory/prod
 create mode 100644 inventory/staging
 create mode 100644 inventory/uat
 create mode 100644 playbooks/common.yml
hector@hector-Laptop:~/ansible-config-mgt$ ls
2plays.yml  inventory  playbooks  README.md #all files together
hector@hector-Laptop:~/ansible-config-mgt$
```


This triggers the build from the webhook  

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/consoleOutput.png)


**Installing Ansible plugin**  
In this project I will not use the plugin just yet and will test the playbooks with manual commands  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-AUTOMATE/main/images/plugin.png)    
    

Here I'm using **2plays.yml** with **ping** modules just for me to test the inventory/connection of target machines.  
I have two **EC2** Instances with tag Name: **Jenkins_Ansible** and **NFS** which the dynamic inventory interprets as **_Jenkins_Ansible** and **_NFS**

``` perl
ubuntu@ip-172-31-94-159:~$ cat 2plays.yml
```
``` perl
---
- name: update web servers
  hosts: _Jenkins_Ansible
  remote_user: ubuntu #Ubuntu

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: Pinging
      ping:

- name: update db servers
  hosts: _NFS
  remote_user: ec2-user #Red Hat

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: Pinging
      ping:
ubuntu@ip-172-31-94-159:~$
```
I run the playbook  
No need to specify an inventory because dynamic inventory is set to default.
``` bash
ansible-playbook 2plays.yml
```
Successful output with 0 fails
``` bash   
PLAY RECAP *********************************************************************************************************************
ec2-3-220-20-204.compute-1.amazonaws.com : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
ec2-54-209-253-1.compute-1.amazonaws.com : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
``` 

Now that everything seems good I will install wireshark with **common.yml**

``` bash
---
- name: update web servers
  hosts: _Jenkins_Ansible
  remote_user: ubuntu

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest

- name: update db servers
  hosts: _NFS
  remote_user: ec2-user

  vars_files:
    - /etc/ansible/vars/info.yml

  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest
```

``` bash
PLAY RECAP *********************************************************************************************************************  
ec2-3-220-20-204.compute-1.amazonaws.com : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
ec2-54-209-253-1.compute-1.amazonaws.com : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```


Confirming wireshark is installed in NFS server
``` bash
[ec2-user@ip-172-31-81-201 ~]$ sudo yum list installed | grep wire
wireshark.x86_64
```

Then I changed the state to absent to remove them
``` bash
state: absent 
```

### Additional Ansible Playbooks
[Project 7: Devops Tooling Website Solution](https://github.com/hectorproko/Ansible/tree/main/Project7)  
[Project 8: LOAD BALANCER SOLUTION WITH APACHE](https://github.com/hectorproko/Ansible/tree/main/Project8)  
[Project 9: CONTINOUS INTEGRATION PIPELINE FOR TOOLING WEBSITE](https://github.com/hectorproko/Ansible/tree/main/Project9)  

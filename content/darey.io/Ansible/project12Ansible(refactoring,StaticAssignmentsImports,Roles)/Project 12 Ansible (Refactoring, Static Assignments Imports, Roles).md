---
tags:
  - Ansible
title: "Project 12: Ansible Refactoring, Static Assignments, Imports, and Roles"
hardlinked: "True"
linkedin: "False"
quartz: "True"
refactored: "False"
darey.io: "True"
hands-on: "True"
completed: "True"
aliases:
  - "Project 12: Ansible Refactoring, Static Assignments, Imports, and Roles"
---

%%~~*(Old [Project 12](https://github.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/blob/main/Project12_Steps.md))*~~%%

In this project I continue working with the [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt.git) repository and make improvements to the code. The goal is to refactor the Ansible code, create assignments, and learn how to use the **imports** functionality to effectively re-use previously created playbooks, it allows me to organize tasks and reuse them as needed.

Up to this point, in [[Project9 CONTINOUS-INTEGRATION-PIPELINE-FOR-TOOLING-WEBSITE|Project 9]] and [[Project 11 Ansible-Automate|11]], every new change in the code created a separate directory, which is not convenient when I want to run commands from one place. To solve this I use the Jenkins plugin [Copy Artifact](https://plugins.jenkins.io/copyartifact/).

---

## Step 1 - Setting Up the Copy Artifact Plugin in Jenkins

### Installing the Plugin

I install **Copy Artifact** via **Manage Jenkins > Manage Plugins**.

![copyArtifact.png](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/copyArtifact.png)

### Creating the Artifact Directory

I create a directory on the Jenkins instance to store all artifacts after each build.

```bash
sudo mkdir /home/ubuntu/ansible-config-artifact
```

I then open up the permissions so Jenkins can write to it.

```bash
sudo chmod -R 0777 /home/ubuntu/ansible-config-artifact
```

Running `ls -l` confirms the directory was created with the correct permissions (`drwxrwxrwx`).

```bash
ubuntu@ip-172-31-94-159:~$ ls -l
total 24
-rw-r--r-- 1 root   root    333 Apr 15 23:09 2plays.yml
drwxrwxrwx 2 root   root   4096 Apr 17 15:53 ansible-config-artifact
-rw-r--r-- 1 ubuntu ubuntu  572 Apr 17 15:16 common.yml
-rw------- 1 ubuntu ubuntu 1799 Apr  2 14:45 daro.io.pem
-rw-rw-r-- 1 ubuntu ubuntu    4 Apr  2 15:41 hello
drwxrwxr-x 3 ubuntu ubuntu 4096 Apr  2 14:44 key
```
%%Extra files from [[Project 11 Ansible-Automate]] %%
### Creating the `save_artifacts` Jenkins Job

I create a new Jenkins job named **save_artifacts**.

![itemName.png](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/itemName.png)

This job is configured to trigger automatically on completion of the existing **ansible** job.

![copy_artifact_trigger.png](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/copy_artifact_trigger.png)

💡*Job created in [[Project 11 Ansible-Automate#Configure Jenkins to Trigger on Repository Changes|Project 11: Ansible-Automate]]*

I then configure it to copy artifacts into the directory I just created at `/home/ubuntu/ansible-config-artifact`.

![build1.png](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/build1.png)

![build2.png](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/build2.png)

### Testing the Pipeline

To test, I push a change to the [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt/tree/main) repo (a README update) to trigger the webhook.

The **ansible** job console log confirms it was triggered by the push and then kicked off `save_artifacts`:

> [!NOTE]- log
> ```log
> Job name  ansible
> 
> Started by GitHub push by hectorproko ⬅️
> 
> Running as SYSTEM
> Building in workspace /var/lib/jenkins/workspace/ansible
> The recommended git tool is: NONE
> using credential 4339e2c2-cc4c-4596-b5a3-b06d893f6339
>  > git rev-parse --resolve-git-dir /var/lib/jenkins/workspace/ansible/.git # timeout=10
> Fetching changes from the remote Git repository
>  > git config remote.origin.url https://github.com/hectorproko/ansible-config-mgt.git # timeout=10
> Fetching upstream changes from https://github.com/hectorproko/ansible-config-mgt.git
>  > git --version # timeout=10
>  > git --version # 'git version 2.25.1'
> using GIT_ASKPASS to set credentials 
>  > git fetch --tags --force --progress -- https://github.com/hectorproko/ansible-config-mgt.git +refs/heads/*:refs/remotes/origin/* # timeout=10
>  > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
> Checking out Revision 5db5aa77bd6c8b9cb3563c81f359026ccee275ce (refs/remotes/origin/main)
>  > git config core.sparsecheckout # timeout=10
>  > git checkout -f 5db5aa77bd6c8b9cb3563c81f359026ccee275ce # timeout=10
> 
> Commit message: "Update README.md" ⬅️
>  
>  > git rev-list --no-walk 4a520c83d57729a09264bec6bf5831d470a7e728 # timeout=10
> Archiving artifacts
> 
> Triggering a new build of save_artifacts ⬅️
> 
> Finished: SUCCESS
> ```

The `save_artifacts` console output confirms it was triggered by the upstream **ansible** job:

> [!NOTE]- log
> ![consoleOutput.png](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/consoleOutput.png)
> 
> ```log
> Started by upstream project "ansible" build number 10  ⬅️
> originally caused by:
>   Started by GitHub push by hectorproko
> Running as SYSTEM
> Building in workspace /var/lib/jenkins/workspace/save_artifacts
> Copied 7 artifacts from "ansible" build number 10
> Finished: SUCCESS
> ```
> 

The directory is now populated with the artifacts.

> [!summary] The flow is:
> 1. You push a change to the **GitHub repo** [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt/tree/main)
> 2. The webhook triggers the **ansible** Jenkins job, which pulls the latest files from the repo and archives them inside Jenkins
> 3. Completion of the **ansible** job triggers **save_artifacts**, which uses the Copy Artifact plugin to copy those archived files out of Jenkins' workspace and into `/home/ubuntu/ansible-config-artifact` on the server's home directory

**Output:**

```bash
ubuntu@ip-172-31-94-159:~/ansible-config-artifact$ sudo tree
.
├── 2plays.yml
├── README.md
├── inventory [error opening dir]
└── playbooks [error opening dir]

2 directories, 2 files
```

To further confirm the pipeline is working end to end, I add a **TestFile** to the repo and verify it appears in `ansible-config-artifact` after the build.

```bash
ubuntu@ip-172-31-94-159:~/ansible-config-artifact$ sudo tree
.
├── 2plays.yml
├── README.md
├── TestFile  ⬅️
├── inventory [error opening dir]
└── playbooks [error opening dir]

2 directories, 3 files
```

Any files I add, move, or delete in the repository are reflected in `ansible-config-artifact` after each build.

---

## Step 2 - Refactoring Ansible Code by Importing Playbooks into `site.yml`

Rather than keeping all tasks in a single playbook like `common.yml` from [[Project 11 Ansible-Automate#Set Up Directory Structure in a Feature Branch|Project 11: Ansible-Automate]], I break things up into separate files. This makes complex configurations easier to manage and promotes reuse.

I make the following changes to the repository structure:

1. Inside the `playbooks` folder, I create `site.yml`. This becomes the entry point for the entire infrastructure configuration. All other playbooks are imported into it, making it the parent playbook.
    
2. At the root of the repository I create a `static-assignments` folder to store all child playbooks.
    
3. I move `common.yml` into `static-assignments`.
    
4. Inside `site.yml` I add the import:
    

```yaml
import_playbook: ../static-assignments/common.yml
```

I also create `common-del.yml`, a copy of `common.yml` with **state** set to **absent**, to undo changes (uninstall Wireshark).

**Resulting directory structure:**

```bash
ubuntu@ip-172-31-94-159:~/ansible-config-artifact$ sudo tree
.
├── 2plays.yml
├── README.md
├── inventory               # Will start using static inventories
│   ├── dev.yml
│   ├── prod.yml
│   ├── staging
│   └── uat
├── playbooks
│   └── site.yml
└── static-assignments
    ├── common-del.yml
    └── common.yml
```

---

## Step 3 - Configuring UAT Webservers with the `webserver` Role

### Provisioning UAT Instances

I create 2 new **Red Hat** EC2 instances:

- **Web1-UAT**
- **Web2-UAT**

### Creating the Role

Ansible roles provide a standard directory structure to organize tasks, handlers, variables, templates, and more. I create a `roles` directory inside the [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt/tree/main) repo and use **ansible-galaxy** to scaffold a role named **webserver**.

```bash
mkdir roles
cd roles
ansible-galaxy init webserver
```

**Resulting structure:**

```bash
hector@hector-Laptop:~/ansible-config-mgt/roles$ tree
.
└── webserver
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml
```

_Note: The default scaffold may include directories that won't be used and can be removed._

I then register the roles path in the Ansible configuration file `/etc/ansible/ansible.cfg`:

```ini
roles_path = /home/ubuntu/ansible-config-mgt/roles
```

---

## Step 4 - Referencing the Webserver Role

Inside `static-assignments` I create a new playbook `uat-webservers.yml` that references the **webserver** role and targets the UAT hosts.

```yaml
---
- hosts: uat-webservers   # From static inventory uat.ini
  remote_user: ec2-user
  roles:
    - webservers
```

The UAT inventory file `ansible-config-mgt/inventory/uat.ini` contains:

```ini
[uat-webservers]
172.31.81.182 ansible_ssh_user='ec2-user'
172.31.89.227 ansible_ssh_user='ec2-user'
```

I verify the inventory resolves correctly:

```bash
ansible-inventory -i ansible-config-artifact/inventory/uat.ini --graph
```

**Output:**

```
@all:
  |--@uat-webservers:
  |  |--172.31.81.182
  |  |--172.31.89.227
  |--@ungrouped:
```

I then import this playbook into the parent `site.yml`:

```yaml
- import_playbook: ../static-assignments/uat-webservers.yml
```

Finally, I run the playbook against the UAT inventory:

```bash
sudo ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/uat.yml \
  /home/ubuntu/ansible-config-mgt/playbooks/site.yaml
```

<!--#### ANSIBLE REFACTORING AND STATIC ASSIGNMENTS (IMPORTS AND ROLES)

In this project I will continue working with [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt.git) repository and make some improvements to the code. Now I need to refactor the **Ansible** code, create assignments, and learn how to use the **imports** functionality to effectively re-use previously created playbooks in a new playbook – it allows us to organize tasks and reuse them as needed.

So far in Projects 9 & 11 every **new** change in the code creates a separate directory which is not very convenient when we want to run some commands from one place so we'll use plug in [Copy Artifact](https://plugins.jenkins.io/copyartifact/)   


Installing **Copy Artifact**  
**Manage Jenkins** >  **Manage Plugins**  

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/copyArtifact.png)


Creating directory in **Jenkins** instance to store all artifacts after each build
``` bash
sudo mkdir /home/ubuntu/ansible-config-artifact
```

Changing permission of the directory `ansible-config-artifact`
``` bash
ubuntu@ip-172-31-94-159:~$ sudo chmod -R 0777 /home/ubuntu/ansible-config-artifact	
ubuntu@ip-172-31-94-159:~$ ls -l
total 24
-rw-r--r-- 1 root   root    333 Apr 15 23:09 2plays.yml
drwxrwxrwx 2 root   root   4096 Apr 17 15:53 ansible-config-artifact
-rw-r--r-- 1 ubuntu ubuntu  572 Apr 17 15:16 common.yml
-rw------- 1 ubuntu ubuntu 1799 Apr  2 14:45 daro.io.pem
-rw-rw-r-- 1 ubuntu ubuntu    4 Apr  2 15:41 hello
drwxrwxr-x 3 ubuntu ubuntu 4096 Apr  2 14:44 key
ubuntu@ip-172-31-94-159:~$
```


Creating **Jenkins** Job **save_artifacts**

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/itemName.png)


This project/job will be triggered by completion of our existing **ansible** project.   
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/copy_artifact_trigger.png)


Now we configure the job to actually save the artifact to the directory we just created  `/home/ubuntu/ansible-config-artifact`  

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/build1.png)

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/build2.png)

I will test the job by making a change in repo `ansible-config-mgt` specifically the readme file to trigger the project through the webhook

**Console Log** of **ansible** job.  
Shows it was triggered by a **push** *(a commit with message "Update README.md")* and it triggered job **save_artifacts**
``` bash
Job name  ansible

Started by GitHub push by hectorproko #<<<<

Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/ansible
The recommended git tool is: NONE
using credential 4339e2c2-cc4c-4596-b5a3-b06d893f6339
 > git rev-parse --resolve-git-dir /var/lib/jenkins/workspace/ansible/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/hectorproko/ansible-config-mgt.git # timeout=10
Fetching upstream changes from https://github.com/hectorproko/ansible-config-mgt.git
 > git --version # timeout=10
 > git --version # 'git version 2.25.1'
using GIT_ASKPASS to set credentials 
 > git fetch --tags --force --progress -- https://github.com/hectorproko/ansible-config-mgt.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision 5db5aa77bd6c8b9cb3563c81f359026ccee275ce (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 5db5aa77bd6c8b9cb3563c81f359026ccee275ce # timeout=10

Commit message: "Update README.md" #<<<<
 
 > git rev-list --no-walk 4a520c83d57729a09264bec6bf5831d470a7e728 # timeout=10
Archiving artifacts

Triggering a new build of save_artifacts #<<<<

Finished: SUCCESS
```

Looking at `save_artifacts`'s **Console Output** we see it was triggered by upstream project `ansible`

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/main/images/consoleOutput.png)

``` bash
Started by upstream project "ansible" build number 10	
originally caused by:
 Started by GitHub push by hectorproko
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/save_artifacts
Copied 7 artifacts from "ansible" build number 10
Finished: SUCCESS
```

Now the directory we created is populated 
``` bash
ubuntu@ip-172-31-94-159:~/ansible-config-artifact$ sudo tree
.
├── 2plays.yml
├── README.md
├── inventory [error opening dir] #tree didnt work
└── playbooks [error opening dir]
2 directories, 2 files
```

Just to see my pipeline worked I created **TestFile** in the repo `ansible-config-mng` to trigger the `ansible` job automatically and I can see the file in the directory `ansible-config-artifact`

``` bash
ubuntu@ip-172-31-94-159:~/ansible-config-artifact$ sudo tree
.
├── 2plays.yml
├── README.md
├── TestFile #<<<
├── inventory [error opening dir]
└── playbooks [error opening dir]

2 directories, 3 files
```

As I add/move/delete files in the repo and change the structure, I can see it copied to `ansible-config-artifact`

#### REFACTOR ANSIBLE CODE BY IMPORTING OTHER PLAYBOOKS INTO SITE.YML  

I will now move away from having all my tasks in a single playbook like I did in [Project 11](https://github.com/hectorproko/ANSIBLE-AUTOMATE/blob/main/Steps_Project11.md) with `common.yml`. Breaking tasks up into different files is an excellent way to organize complex sets of tasks and reuse them.

1. Within playbooks folder I will create `site.yml` – This file will now be considered as an entry point into the entire infrastructure configuration. By including other playbooks in it, `site.yml` becomes the parent to all other playbooks that will be developed.

2. At the root of the repository I create a folder `static-assignments` where I will store all other children playbooks
   
3. I'll move `common.yml` file into the newly created `static-assignments` folder.
 
4. Inside `site.yml` file I'll import `common.yml` playbook.

``` bash
import_playbook: ../static-assignments/common.yml
```
In addition, I created another playbook `common-del.yml` which is just a copy of `common.yml` with **state** set to **absent** to undo changes *(uninstall wireshark)*  

 So far my `ansible-config-artifact` directory structure looks like this  
 
``` bash
ubuntu@ip-172-31-94-159:~/ansible-config-artifact$ sudo tree
.
├── 2plays.yml
├── README.md
├── inventory #Will start using static inventories
│   ├── dev.yml
│   ├── prod.yml
│   ├── staging
│   └── uat
├── playbooks
│   └── site.yml
└── static-assignments
    ├── common-del.yml
    └── common.yml
```


#### CONFIGURE UAT WEBSERVERS WITH A ROLE ‘WEBSERVER’  

Created 2 new instances with **RedHat**  
**Web1-UAT**  
**Web2-UAT**  

To create a role, I'll create a directory called `roles`, relative to the playbook.

Inside repo `ansible-config-mgt` I create a directory `roles`
and used utility **ansible-galaxy** to create the a structure which I named **webservers** *(I commit and push to Hithub)*

``` bash
mkdir roles
cd roles
ansible-galaxy init webserver
```
``` bash
hector@hector-Laptop:~/ansible-config-mgt/roles$ tree
.
└── webserver
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml
```
*This default structure might have extra directories we won't use and can delete*  

Need to make an entry on **ansible** configuration file `/etc/ansible/ansible.cfg` specifying the location of the roles `/home/ubuntu/ansible-config-mgt/roles`

``` bash
roles_path = /home/ubuntu/ansible-config-mgt/roles
```

#### REFERENCE WEBSERVER ROLE  

Within the `static-assignments` folder I create a new assignment/**playbook** for **uat-webservers** `uat-webservers.yml` which contains a reference to the role we just created **webservers**

``` bash
---
- hosts: uat-webservers #From static inventory uat.ini
  remote_user: ec2-user
  roles:
    - webservers
```
I will be using inventory file `ansible-config-mgt/inventory/uat.ini`  
``` bash
[uat-webservers]
172.31.81.182 ansible_ssh_user='ec2-user'
172.31.89.227 ansible_ssh_user='ec2-user'
```
I test it  
`ansible-inventory -i ansible-config-artifact/inventory/uat.ini --graph`

``` bash
@all:
  |--@uat-webservers:
  |  |--172.31.81.182
  |  |--172.31.89.227
  |--@ungrouped:
```

Now I have to import this **playbook** in our **parent playbook** `site.yml`
``` bash
- import_playbook: ../static-assignments/uat-webservers.yml
```

Finally I can run the **playbook** against my uat inventory and see what happens:  
``` bash
sudo ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/uat.yml /home/ubuntu/ansible-config-mgt/playbooks/site.yaml
```


---
tags:
  - Ansible
title: "Project 13: Ansible (Dynamic Assignments, Include and Community Roles)"
hardlinked: "True"
linkedin: "False"
quartz: "True"
refactored: "False"
darey.io: "True"
hands-on: "True"
completed: "True"
aliases:
  - "Project 13: Ansible (Dynamic Assignments, Include and Community Roles)"
---

%%~~*(Old [Project 13](https://github.com/hectorproko/ANSIBLE-DYNAMIC-ASSIGNMENTS-INCLUDE-AND-COMMUNITY-ROLES/blob/main/Project13_Steps.md))%%*~~

## Overview

Building on the static assignment approach from [[Project 12 Ansible (Refactoring, Static Assignments Imports, Roles)|Project 12: Ansible Refactoring, Static Assignments, Imports, and Roles]], this project introduces **dynamic assignments** using Ansible's `include` module. The goal is to make the playbook structure flexible enough to load environment-specific variables at runtime, and to integrate community roles for load balancer configuration.

---

## Static vs. Dynamic Assignments

Ansible handles reuse in two ways depending on which module is used:

|Module|Type|Behavior|
|---|---|---|
|`import`|Static|Processed at parse time; changes after parsing are ignored|
|`include`|Dynamic|Processed at execution time; picks up any changes encountered during the run|

The key difference is **when** Ansible reads the content. With `import`, everything is resolved before the playbook runs. With `include`, content is resolved as the playbook executes, which allows for conditional logic and environment-specific flexibility.

---

## Introducing Dynamic Assignments Into the Structure

In the [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt) repository, a new branch called `dynamic-assignments` is created. Inside it, a `dynamic-assignments/` folder is added with a file named `env-vars.yml`.

The purpose of this file is to load the correct set of environment variables at runtime by looping through a list of known environment files and using the first one found.

```yaml
---
- name: collate variables from env specific file, if it exists
  hosts: all
  tasks:
    - name: looping through list of available files
      include_vars: "{{ item }}"
      with_first_found:
        - files:
            - uat.yml
            - stage.yml
            - prod.yml
            - dev.yml
          paths:
            - "{{ playbook_dir }}/../env-vars"
      tags:
        - always
```

`with_first_found` loops through the file list and loads the first match. This ensures a default is always available even if an environment-specific file does not exist. The `{{ playbook_dir }}` variable dynamically resolves the path relative to wherever the playbook is running from.

The resulting folder structure looks like this:

```
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
│   └── dev.yml
│   └── stage.yml
│   └── uat.yml
│   └── prod.yml
```

Each `.yml` file under `env-vars/` holds the variable values specific to that environment, such as server names and IP addresses.

---

## Updating site.yml with Dynamic Assignments

`site.yml` is updated to import the `env-vars.yml` playbook, which loads the correct variables before the rest of the playbook runs. At this stage the full flow cannot be tested yet since the load balancer roles are not installed, but the structure is being put in place.

---

## Load Balancer Roles

To handle load balancing, two community roles are installed from [Ansible Galaxy](https://galaxy.ansible.com/):

- [nginxinc.nginx](https://galaxy.ansible.com/nginxinc/nginx)
- [geerlingguy.apache](https://galaxy.ansible.com/geerlingguy/apache)

```bash
ansible-galaxy install nginxinc.nginx && ansible-galaxy install geerlingguy.apache
```

By default, `ansible-galaxy` installs roles to `~/.ansible/roles/`. To keep everything version-controlled, the roles are moved into the local repository and renamed in the process:

```bash
hector@hector-Laptop:~/ansible-config-mgt/roles$ mv /home/hector/.ansible/roles/nginxinc.nginx ./nginx
hector@hector-Laptop:~/ansible-config-mgt/roles$ mv /home/hector/.ansible/roles/geerlingguy.apache ./apache
```

After the move, the `roles/` directory contains:

```bash
hector@hector-Laptop:~/ansible-config-mgt/roles$ ls
apache  common  nginx  webservers
```

---

## Wiring It All Together in site.yml

`ansible-config-mgt/playbooks/site.yml` is updated to first load the environment variables, then conditionally run the load balancer assignment:

```yaml
- name: include the env-vars playbook
  import_playbook: ../dynamic-assignments/env-vars.yml

- name: Loadbalancers assignment
  import_playbook: ../static-assignments/loadbalancers.yml
  when: load_balancer_is_required
```

Calling `env-vars.yml` loads the variables into the playbook. For this test, the values come from `ansible-config-mgt/env-vars/uat.yml`:

```yaml
enable_nginx_lb: true
enable_apache_lb: false
load_balancer_is_required: true
```

`site.yml` then imports `ansible-config-mgt/static-assignments/loadbalancers.yml`, which decides which role to apply based on those variable values:

```yaml
- hosts: lb
  roles:
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
```

With `enable_nginx_lb: true` and `enable_apache_lb: false`, only the Nginx role runs, executing the tasks defined in `ansible-config-mgt/roles/nginx/tasks/main.yml`. Switching to Apache is as simple as flipping those two values in the environment file, with no changes needed to the playbook itself.


%%
## ANSIBLE-DYNAMIC-ASSIGNMENTS-INCLUDE-AND-COMMUNITY-ROLES
PROJECT 13

In this project we will introduce [dynamic assignments](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse.html#includes-dynamic-re-use) by using `include` module.  

In [Project 12](https://github.com/hectorproko/ANSIBLE-REFACTORING-ASSIGNMENTS-IMPORTS/blob/main/Project12_Steps.md), we usedd **static** assignments using `import` Ansible module. The module that enables dynamic assignments is include.  

`import = Static`  
`include = Dynamic`  

When the `import` module is used, all statements are pre-processed at the time playbooks are **[parsed](https://en.wikipedia.org/wiki/Parsing)**. Meaning, when you execute `site.yml` playbook, Ansible will process all the playbooks referenced during the time it is parsing the statements. This also means that, during actual execution, if any statement changes, such statements will not be considered. Hence, it is static.  

On the other hand, when `include` module is used, all statements are processed only during execution of the playbook. Meaning, after the statements are **[parsed](https://en.wikipedia.org/wiki/Parsing)**, any changes to the statements encountered during execution will be used.  

#### ANSIBLE DYNAMIC ASSIGNMENTS (INCLUDE) AND COMMUNITY ROLES



#### INTRODUCING DYNAMIC ASSIGNMENT INTO OUR STRUCTURE
In repo [ansible-config-mgt](https://github.com/hectorproko/ansible-config-mgt) I create a new **branch** ``dynamic-assignments``  

In this new **branch** I create a folder ``dynamic-assignments`` and inside it a file ``env-vars.yml``  

``` bash 
---
- name: collate variables from env specific file, if it exists
  hosts: all
  tasks:
    - name: looping through list of available files
      include_vars: "{{ item }}" 
      with_first_found: #implies that the first one found is used
        - files:
            - uat.yml
            - stage.yml
            - prod.yml
            - dev.yml
          paths:
            - "{{ playbook_dir }}/../env-vars" #playbook_dir determines the location of the running playbook
      tags:
        - always
```  
We are including the variables using a loop. `with_first_found` implies that, looping through the list of files, the first one found is used. This is good so that we can always set default values in case an environment specific env file does not exist.  

Since we will be using the same Ansible to configure multiple environments, and each of these environments will have certain unique attributes, such as servername, ip-address etc., we will need a way to set values to variables per specific environment.

``` bash
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
    └── dev.yml
    └── stage.yml
    └── uat.yml
    └── prod.yml
```
#### UPDATE SITE.YML WITH DYNAMIC ASSIGNMENTS

**Updating site.yml with dynamic assignments**  

Updating `site.yml` file to make use of the **dynamic** assignment.  
*(At this point, we cannot test it yet. We are just setting the stage for what is yet to come)*  


#### LOAD BALANCER ROLES

Installing 2 new **roles** [Nginx](https://galaxy.ansible.com/nginxinc/nginx) and [Apache](https://galaxy.ansible.com/geerlingguy/apache)  

``` bash
ansible-galaxy install nginxinc.nginx && ansible-galaxy install geerlingguy.apache
```
Moving the **roles** from default location to local repo (renaming it in the process) to be **pushed**

``` bash
hector@hector-Laptop:~/ansible-config-mgt/roles$ mv /home/hector/.ansible/roles/nginxinc.nginx ./nginx
hector@hector-Laptop:~/ansible-config-mgt/roles$ mv /home/hector/.ansible/roles/geerlingguy.apache ./apache
hector@hector-Laptop:~/ansible-config-mgt/roles$ ls
apache  common nginx  webservers
hector@hector-Laptop:~/ansible-config-mgt/roles$
```
For this particular test I'm going to edit **site.yml** `ansible-config-mgt/playbooks/site.yml`  

``` bash
- name: include the env-vars playbook
  import_playbook: ../dynamic-assignments/env-vars.yml 

- name: Loadbalancers assignment
  import_playbook: ../static-assignments/loadbalancers.yml
  when: load_balancer_is_required
```
Calling `env-vars.yml` will load variables into the playbook `site.yml`, in this case variable values from `ansible-config-mgt/env-vars/uat.yml` which for this example contains the following

``` bash
enable_nginx_lb: true
enable_apache_lb: false
load_balancer_is_required: true
```
Next `site.yml` imports `ansible-config-mgt/static-assignments/loadbalancers.yml`  

``` bash
- hosts: lb
  roles: 
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
```

So based on the values of the variables in `uat.yml`, `loadbalancers.yml` calls either:  
`ansible-config-mgt/roles/apache/tasks/main.yml` or `ansible-config-mgt/roles/nginx/tasks/main.yml`%%
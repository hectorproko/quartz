---

tags:
  - hardlink
  - Ansible
  - darey
  - inquartz
title: "Project 13: Ansible (Dynamic Assignments, Include and Community Roles)"
hardlinked: "True"
---

~~*(Old [Project 13](https://github.com/hectorproko/ANSIBLE-DYNAMIC-ASSIGNMENTS-INCLUDE-AND-COMMUNITY-ROLES/blob/main/Project13_Steps.md))*~~
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
`ansible-config-mgt/roles/apache/tasks/main.yml` or `ansible-config-mgt/roles/nginx/tasks/main.yml`
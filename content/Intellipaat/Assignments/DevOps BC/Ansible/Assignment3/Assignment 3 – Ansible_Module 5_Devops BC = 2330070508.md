---
tags:
  - Ansible
title: Installing Apache2 and NGINX on Different Hosts Using Ansible Roles
---
<!--
🚀 **Expanding DevOps Capabilities with Ansible: Role-Based Automation!** I've recently completed an enriching assignment in my DevOps training, focusing on advanced Ansible role creation and management. The task required developing two distinct Ansible roles for installing Apache2 and NGINX on separate nodes. This assignment honed my skills in modularizing configurations and automating complex deployments, showcasing the flexibility and power of Ansible in managing diverse server environments efficiently.

#Ansible #DevOps #Automation #ConfigurationManagement #ProfessionalDevelopment
-->


> [!info] Module 5: Ansible Assignment - 3
> **Tasks To Be Performed:** 
> 1. Create 2 Ansible roles 
> 2. Install Apache2 on slave1 using one role and NGINX on slave2 using the other role 
> 3. Above should be implemented using different Ansible roles

Using the same setup from [[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508#^a2c939|Assignment 1 – Ansible]]

> [!NOTE] EC2 instances
> Ansible Control Machine `10.0.1.223`
> Slave1 `10.0.1.65`
> Slave2 `10.0.1.233`

To perform this assignment, I will create two separate Ansible roles: one for installing Apache2 and another for installing NGINX. 

### Step 1: Create the Ansible Roles

Firstly, I will create the directory structure for the two roles. I can use the `ansible-galaxy` command to do this:
```bash
ansible-galaxy init apache_role
ansible-galaxy init nginx_role
```
This will create two directories with subdirectories and files needed for each role.
<br>![[Pasted image 20231106140400.png]]

### Step 2: Configure the Apache2 Role

Inside the `apache_role` directory, I will modify the `tasks/main.yml` file to include the following:
```yaml
---
# tasks file for apache_role
- name: Install Apache2
  apt:
    name: apache2
    state: present
    update_cache: yes
```
This ensures that Apache2 is installed when this role is applied.

### Step 3: Configure the NGINX Role

Similarly, inside the `nginx_role` directory, I will edit the `tasks/main.yml` file to contain:
```yaml
---
# tasks file for nginx_role
- name: Install NGINX
  apt:
    name: nginx
    state: present
    update_cache: yes
```
This will install NGINX when the role is executed.


### Step 4: Create the Playbook

I will create a playbook called `install_web_servers.yml`. In this playbook, I will specify which host should use which role:
```yaml
---
- hosts: slave1
  become: yes
  roles:
    - apache_role

- hosts: slave2
  become: yes
  roles:
    - nginx_role
```



This playbook runs the `apache_role` on `slave1` and the `nginx_role` on `slave2`.


### Step 5: Run the Playbook

I will execute my playbook with the following command:

```bash
ansible-playbook -i inventory.ini install_web_servers.yml 
```
<br>![[Pasted image 20231106142220.png]]

### Step 6: Verify the Installations

After running the playbook, I will verify that Apache and NGINX have been installed and are running on their respective servers using the `ansible` command with the `shell` module. Here's how I will do it:
```bash
ansible slave1 -m shell -a "systemctl status apache2" -i inventory.ini --become
ansible slave2 -m shell -a "systemctl status nginx" -i inventory.ini --become
```


> [!success]
> <br>![[Pasted image 20231106143527.png]]


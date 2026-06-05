---
tags:
  - Ansible
title: Streamlining Web Server Management with Ansible
---
<!--
🚀 **Case Study Mastery: Streamlining Web Server Management with Ansible!** I've accomplished a comprehensive case study in my DevOps training, which involved using Ansible to manage Apache and NGINX server groups. The project required creating custom HTML files, setting up two server groups, and configuring the servers with Ansible roles. This assignment provided an in-depth experience in using Ansible for complex configuration management tasks, enhancing my proficiency in automating server setup and maintenance.

#Ansible #DevOps #ServerManagement #Automation #ProfessionalDevelopment
-->

> [!info] Module 5: Case Study - Ansible
> **Problem Statement:** 
> You are a DevOps Engineer and the organization you are working on needs to set up two configuration management server groups. One for Apache, another for NGINX. Being a DevOps Engineer it is your task to deal with this configuration management issue. 
> 
> Let us see the tasks that you need to perform using Ansible: 
> 1. Create two server groups. One for Apache and another for NGINX 
> 2. Push two html files with their server information 
> 
> Make sure that you don’t forget to start the services once the installation is done. Also send post installation messages for both the server groups. 
> 
> Using Ansible roles accomplish the above tasks. Also, once the Apache server configuration is done you need to install Java on that server group using Ansible role in a Playbook.


I will reuse the Ansible roles from [[Assignment 4 – Ansible_Module 5_Devops BC = 2330070508|Assignment 4 – Ansible]] and the EC2 instances that were created in [[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508#^a2c939|Assignment 1 – Ansible]].
%%[[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508#^be0450|SSH agent]]%%
> [!NOTE] EC2 instances
> Ansible Control Machine `10.0.1.223`
> Slave1 `10.0.1.65`
> Slave2 `10.0.1.233`

**Create HTML Files**:

- I prepare two `index.html` files, one for Apache and the other for Nginx, with the respective server information.

```bash
ubuntu@ip-10-0-1-223:~$ tree nginx_role/
nginx_role/
├── README.md
├── defaults
│   └── main.yml
├── files
│   └── index.html
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml

7 directories, 9 files
```

```bash
ubuntu@ip-10-0-1-223:~$ cat nginx_role/files/index.html 
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ansible Deployment</title>
</head>
<body>
    <div class="header">
        <h1>Deployed from Ansible to Nginx</h1>
    </div>
</body>
</html>
```

```bash
ubuntu@ip-10-0-1-223:~$ tree apache_role/
apache_role/
├── README.md
├── defaults
│   └── main.yml
├── files
│   └── index.html
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml

7 directories, 9 files
```

```bash
ubuntu@ip-10-0-1-223:~$ cat apache_role/files/index.html 
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ansible Deployment</title>
</head>
<body>
    <div class="header">
        <h1>Deployed from Ansible to Apache</h1>
    </div>
</body>
</html>
```


**Create Ansible Inventory Groups**:

- I create an inventory file (`hosts.ini`) to define two groups, `[apache]` and `[nginx]`.

```yaml
[all:vars]
ansible_user=ubuntu

[apache]
slave1 ansible_host=10.0.1.65 
slave2 ansible_host=10.0.1.233

[nginx]
slave3 ansible_host=10.0.1.210
slave4 ansible_host=10.0.1.85
```


`apache_role/tasks/main.yml`
```yaml
---
- name: Install Apache
  apt:
    name: apache2
    state: present
  notify: start apache
  
- name: Gather the Apache service facts
  service_facts:
  
- name: Push custom HTML file for Apache
  copy:
    src: index.html
    dest: /var/www/html/index.html
    group: root
    mode: '0644'
  when: ansible_facts.services["apache2.service"] is defined and ansible_facts.services["apache2.service"].state == "running"
  notify: reload apache
  
- name: Display post-installation message for Apache
  debug:
    msg: "Apache installation and configuration complete!"
```

%%
> [!attention]- `service_facts:` required by `when:`
> 
> ```
> name: Gather the Apache service facts
>   service_facts:
> ```
> [[Assignment 4 – Ansible_Module 5_Devops BC = 2330070508#^3e4b13|Previous Issue]]

%%

`apache_role/handlers/main.yml`
```yaml
---
- name: start apache
  service:
    name: apache2
    state: started

- name: reload apache
  ansible.builtin.service:
    name: apache2
    state: reloaded

```

%%
> [!NOTE]- notify: start apache
> In Ansible, the `notify` directive is used to trigger a handler based on the result of a task. If a task results in a change on the managed host—for instance, if a package like Apache is installed or updated—the task will notify the handler to take an action. Handlers are typically used to restart services or perform any action that is required after a change.

%%
`nginx_role/tasks/main.yml`
```yaml
---
# tasks file for nginx_role
- name: Install NGINX
  apt:
    name: nginx
    state: present
    update_cache: yes
  notify: start nginx

- name: Gather the NGINX service facts
  service_facts:
  
- name: Push custom HTML file for NGINX
  ansible.builtin.copy:
    src: index.html
    dest: /var/www/html/index.nginx-debian.html
    owner: root
    group: root
    mode: '0644'
  when: ansible_facts.services["nginx.service"] is defined and ansible_facts.services["nginx.service"].state == "running"
  notify: reload nginx

- name: Display post-installation message for NGINX
  debug:
    msg: "NGINX installation and configuration complete!"
```

`nginx_role/handlers/main.yml`
```yaml
---
# handlers file for nginx_setup
- name: start nginx
  service:
    name: nginx
    state: started

- name: reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

**site.yml**:
```bash
---
- hosts: apache
  become: yes
  roles:
    - apache_role
    - java_role

- hosts: nginx
  become: yes
  roles:
    - nginx_role
```
To run these playbooks, I would use the `ansible-playbook` command, specifying the inventory file and the main site playbook:
```bash
ansible-playbook -i hosts.ini site.yml
```

%%
> [!attention]- Issue
> > [!fail]
> > Playbook output
> > <br>![[Pasted image 20231107121243.png]]
> > Output when trying to start Apache manually
> > <br>![[Pasted image 20231107121149.png]]
> > 
> 
> > [!done] Solution
> > Nginx was installed and running on this EC2 which was using the same port 80, a conflict. So we stop it `sudo systemctl stop nginx`
> 

I re-run the playbook after stopping the nginx
%%



I've executed the playbook multiple times already *(troubleshooting an issue)*, resulting in all tasks being marked green. This indicates that the current state matches the desired state, and no changes were required. In addition, the appearance of our custom message confirms the successful execution of the playbook.

> [!success]
> <br>![[Pasted image 20231107122739.png]]
> <br>![[Pasted image 20231107122817.png]]
> <br>![[Pasted image 20231107122856.png]]



---
tags:
  - Ansible
title: Executing a Custom Script to Add Text to Files on Multiple Hosts
---
<!--
🚀 **Expanding Automation Skills with Ansible: Script Execution Across Nodes!** I've completed an assignment in my DevOps training focusing on using Ansible to run custom scripts across multiple nodes. The task was to create a script that adds specific text to a file and then use Ansible to execute this script on all host machines. This exercise was crucial in understanding Ansible’s power in automating tasks over various systems, reinforcing my proficiency in widespread script deployment and execution in a networked environment.

#Ansible #DevOps #Automation #Scripting #ProfessionalDevelopment
-->


> [!info] Module 5: Ansible Assignment - 2
> **Tasks To Be Performed:** 
> 1. Create a script which can add text “This text has been added by custom script” to /tmp.1.txt 
> 2. Run this script using Ansible on all the hosts 


Using the same setup from [[Assignment 1 – Ansible_Module 5_Devops BC = 2330070508#^a2c939|Assignment 1 – Ansible]]

> [!NOTE] EC2 instances
> Ansible Control Machine `10.0.1.223`
> Slave1 `10.0.1.65`
> Slave2 `10.0.1.233`

### Step 1: Write My Custom Script

I create a script on my Ansible control machine that will append text to a file. I name this script `add_text.sh` and write the following lines into it:

```bash
#!/bin/bash 
echo "This text has been added by custom script" >> /tmp/1.txt
```

I then make the script executable by running:
```bash
chmod +x add_text.sh
```

### Step 2: Craft My Ansible Playbook

I write an Ansible playbook that will distribute and execute this script on all my managed nodes (hosts). I name the playbook `run_script.yml` *(location, same working directory as `add_text.sh`)* and ensure it contains the following tasks:

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: Copy my script to the remote hosts
      copy:
        src: /add_text.sh
        dest: /tmp/add_text.sh
        mode: '0755'

    - name: Execute my script
      command: /tmp/add_text.sh
```


### Step 3: Run My Playbook

I execute my playbook using the following command:
```bash
ansible-playbook run_script.yml -i inventory.ini
```

This command tells Ansible to run my playbook on all the hosts listed in my inventory, copying my script to each one and then executing it, thereby appending the text to `/tmp/1.txt` on each host.
<br>![[Pasted image 20231106133516.png]]

### Step 4: Verify the Changes

I verify that my script ran successfully on all hosts by checking the content of `/tmp/1.txt`. If needed, I can do this with another Ansible command:

```bash
ansible all -m shell -a 'cat /tmp/1.txt' -i inventory.ini
```

This command connects to all the hosts in my inventory and outputs the contents of `/tmp/1.txt`, allowing me to ensure that "This text has been added by custom script" has been appended successfully.

> [!success]
> <br>![[Pasted image 20231106133654.png]]
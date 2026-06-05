```yaml
---
- name: Installations on Master
  hosts: localhost
  become: true
  tasks:
    - name: Executing script on master
      script: Jenkins_terraform_ansible.sh

- name: Installations on Kmaster
  hosts: Kmaster
  become: true
  tasks:
    - name: Executing script on Kmaster
      script: kmaster.sh
```
This playbook executes `.sh` scripts in the target nodes

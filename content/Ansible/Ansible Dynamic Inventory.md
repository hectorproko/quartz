---
hardlinked: "True"
---
%%https://github.com/hectorproko/Ansible/blob/main/Ansible_setup.md Dynamic Inventory%%

I have 4 files: [[#ansible.cfg]], [[#aws_ec2.yml]], `daro.io.pem`, and [[#info.yml]]

```bash
tree /etc/ansible/
```

```bash
├── ansible.cfg #Configuration file
├── aws_ec2.yml
└── vars
    ├── daro.io.pem
    └── info.yml

2 directories, 8 files
```

### ansible.cfg
In the configuration file `ansible.cfg`, line 2 specifies the inventory to be `aws_ec2.yml`, a plugin to get inventory dynamically from AWS.

```bash
cat /etc/ansible/ansible.cfg
```

```ini
[defaults]
inventory = aws_ec2.yml #Dynamic inventory
host_key_checking = False
   
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
  
[inventory]
enable_plugins = aws_ec2
```

%%
```bash
hector@hector-Laptop:/etc/ansible$ bat ansible.cfg
───────┬───────────────────────────────────────────────────────────────────────────
       │ File: ansible.cfg
───────┼───────────────────────────────────────────────────────────────────────────
   1   │ [defaults]
   2   │ inventory = aws_ec2.yml #Dynamic inventory
   3   │ host_key_checking = False
   4   │
   5   │ [privilege_escalation]
   6   │ become=True
   7   │ become_method=sudo
   8   │ become_user=root
   9   │ become_ask_pass=False
  10   │
  11   │ [inventory]
  12   │ enable_plugins = aws_ec2
───────┴───────────────────────────────────────────────────────────────────────────
```
%%
### aws_ec2.yml
Inside `aws_ec2.yml`, line 11 specifies grouping the inventory based on the tag `Name`.

```bash
cat /etc/ansible/aws_ec2.yml
```

```yml
plugin: aws_ec2

remote_user: Ansible  # IAM user with AmazonEC2FullAccess

vars_files:
  - /etc/ansible/vars/info.yml  # Credentials

regions:
  - "us-east-1"

keyed_groups:  # Groups based on tag ex: LB, NFS, web7
  - key: tags.Name

hostnames:
  - dns-name
```

%%
```bash
hector@hector-Laptop:/etc/ansible$ bat aws_ec2.yml
───────┬───────────────────────────────────────────────────────────────────────────
       │ File: aws_ec2.yml
───────┼───────────────────────────────────────────────────────────────────────────
   1   │ plugin: aws_ec2
   2   │
   3   │ remote_user: Ansible #IAM user with AmazonEC2FullAccess
   4   │   vars_files:
   5   │    - /etc/ansible/vars/info.yml #Credentials
   6   │
   7   │  regions:
   8   │   - "us-east-1"
   9   │
  10   │ keyed_groups: #Groups based on tag ex: LB, NFS, web7
  11   │     - key: tags.Name
  12   │
  13   │ hostnames: dns-name
───────┴───────────────────────────────────────────────────────────────────────────
```
%%
#### Example of dynamic inventory grouped by tag `Name`:
%%hector@hector-Laptop:/etc/ansible$ sudo ansible-inventory --graph%%

```bash
sudo ansible-inventory --graph
```

```
@all:
  |--@_Jenkins9:
  |  |--ec2-3-220-20-204.compute-1.amazonaws.com
  |--@_LB:
  |  |--ec2-100-24-22-230.compute-1.amazonaws.com
  |--@_NFS:
  |  |--ec2-52-91-225-30.compute-1.amazonaws.com
  |--@_web7:
  |  |--ec2-52-207-235-80.compute-1.amazonaws.com
  |  |--ec2-54-89-100-125.compute-1.amazonaws.com
  |--@aws_ec2:
  |  |--ec2-100-24-22-230.compute-1.amazonaws.com
  |  |--ec2-3-220-20-204.compute-1.amazonaws.com
  |  |--ec2-52-207-235-80.compute-1.amazonaws.com
  |  |--ec2-52-91-225-30.compute-1.amazonaws.com
  |  |--ec2-54-89-100-125.compute-1.amazonaws.com
  |--@ungrouped:
```

### info.yml
`info.yml` holds the credentials, the key name, and the path to the `.pem` file.

```bash
cat /etc/ansible/vars/info.yml
```

```
aws_id: XXXXXXXXXXXXXXXXXXXX
aws_key: XXXXXXXXXXXXXXXXXXXXXXXXXX
aws_region: us-east-1
ssh_keyname: daro.io #needed for provisioning module ec2
ansible_ssh_private_key_file: /etc/ansible/vars/daro.io.pem #used by the configuring part
```

%%
```bash
hector@hector-Laptop:/etc/ansible$ bat vars/info.yml
───────┬───────────────────────────────────────────────────────────────────────────
       │ File: vars/info.yml
───────┼───────────────────────────────────────────────────────────────────────────
   1   │ aws_id: XXXXXXXXXXXXXXXXXXXX
   2   │ aws_key: XXXXXXXXXXXXXXXXXXXXXXXXXX
   3   │ aws_region: us-east-1
   4   │ ssh_keyname: daro.io #needed for provisioning module ec2
   5   │ ansible_ssh_private_key_file: /etc/ansible/vars/daro.io.pem #used by the configuring part
───────┴───────────────────────────────────────────────────────────────────────────
```
%%
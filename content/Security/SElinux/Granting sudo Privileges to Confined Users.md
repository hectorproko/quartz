---
tags:
  - draft
  - selinux
linkedin: "False"
quartz: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
## Introduction

When SELinux is enforcing policies on a system, Linux users mapped to confined SELinux users can lose the ability to run `sudo`. This happens because confined users operate under strict role and type restrictions that block privilege escalation by default. Simply adding a user to the `sudoers` file is not enough. The sudoers entry must also specify the SELinux `TYPE` and `ROLE` that the user needs to transition into, and the user's home directory context must match their assigned SELinux identity.

In this lab, I walk through the full process of granting `sudo` access to two confined users, `pbeesly` and `jhalpert`, on a CentOS 7 system.

**Environment:** CentOS Linux 7 (Core)

---

## Map Users to the Appropriate SELinux User

The first step is to associate each Linux user with an SELinux user identity. By default, new users are mapped to `unconfined_u`, which has no restrictions. Mapping them to `staff_u` places them under confinement while still allowing controlled privilege escalation through `sudo` when properly configured.

I use `semanage login` to create the mappings. The `-a` flag adds a new record, and `-s` specifies the SELinux user to assign.

Map `pbeesly` to `staff_u`:

```bash
semanage login -a -s "staff_u" pbeesly
```
%%[[semanage (SELinux Management)|semanage]]%%
Map `jhalpert` to `staff_u`:

```bash
semanage login -a -s "staff_u" jhalpert
```

After each mapping, I run `semanage login -l` to verify the changes took effect.

```bash
semanage login -l
```

**Output after mapping both users:**

```
Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
jhalpert             staff_u              s0-s0:c0.c1023       *
pbeesly              staff_u              s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
system_u             system_u             s0-s0:c0.c1023       *
```

^5b29cf

Both users now appear in the login table under `staff_u`, confirming the mappings are active.

---

## Add Users to the `sudoers` File

With a standard `sudoers` entry, a confined user would attempt to run a command as root but remain in their current SELinux context (`staff_u:staff_r`), which does not have permission to do administrative tasks. The solution is to include the `TYPE` and `ROLE` fields in the sudoers rule, which tells SELinux to transition the user into the `administrator_t` type and `administrator_r` role when they invoke `sudo`.%% [[Pre-Defined SELinux Identities]]%%

> [!NOTE]
> The role and type used in the sudoers rule depend on what your system's SELinux policy has available. Before writing the rule, check which roles are present:
> 
> ```bash
> seinfo -r | grep -E "administrator_r|sysadm_r"
> ```
> 
> If your system has `administrator_r`, use `TYPE=administrator_t ROLE=administrator_r`. If it has `sysadm_r`, use `TYPE=sysadm_t ROLE=sysadm_r`. Some systems may have both.

Open the sudoers file using `visudo`, which validates syntax before saving:

```bash
visudo
```

Search for the line `root ALL=(ALL) ALL` by typing `/root ALL` in the editor, then enter insert mode with `i` and add the following two lines directly beneath it:

```
pbeesly  ALL=(ALL)  TYPE=administrator_t  ROLE=administrator_r  /bin/sh
jhalpert  ALL=(ALL)  TYPE=administrator_t  ROLE=administrator_r  /bin/sh
```

Save and exit with `:wq!`.

The `TYPE=administrator_t` and `ROLE=administrator_r` fields are the critical difference here. Without them, the confined user's session would not transition to a context with the permissions needed to act as root.

---

## Update the SELinux Security Context of Each User's Home Directory

Even after mapping the SELinux user and updating sudoers, the users' home directories still carry the old `unconfined_u` context label. SELinux uses file contexts to determine access, so the home directory context must be updated to match the new `staff_u` identity, otherwise the user's login environment would be inconsistent with their assigned SELinux user.

First, I check the current context labels on all home directories:

```bash
ls -lZ /home/
```

**Output before the reset:**

```
drwx------. cloud_user cloud_user unconfined_u:object_r:user_home_dir_t:s0 cloud_user
drwx------. jhalpert   jhalpert   unconfined_u:object_r:user_home_dir_t:s0 jhalpert
drwx------. pbeesly    pbeesly    unconfined_u:object_r:user_home_dir_t:s0 pbeesly
drwx------. ssm-user   ssm-user   unconfined_u:object_r:user_home_dir_t:s0 ssm-user
```

Both `pbeesly` and `jhalpert` still show `unconfined_u`. I use `restorecon` to relabel their directories. The `-F` flag forces the relabeling to use the policy-defined context (overriding any existing labels), and `-R` applies the change recursively to all files inside the directory. The `-v` flag prints each change so I can confirm what was updated.%%[[restorecon (Restore Context)#Options]]%%

```bash
restorecon -FR -v /home/pbeesly
```

```bash
restorecon -FR -v /home/jhalpert
```

**Output after running `restorecon` on both users:**

```
restorecon reset /home/pbeesly context unconfined_u:object_r:user_home_dir_t:s0->staff_u:object_r:user_home_dir_t:s0
restorecon reset /home/pbeesly/.bash_logout context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/pbeesly/.bash_profile context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/pbeesly/.bashrc context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0

restorecon reset /home/jhalpert context unconfined_u:object_r:user_home_dir_t:s0->staff_u:object_r:user_home_dir_t:s0
restorecon reset /home/jhalpert/.bash_logout context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/jhalpert/.bash_profile context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/jhalpert/.bashrc context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
```

Finally, I verify the updated contexts:

```bash
ls -lZ /home/
```

**Output after the reset:**

```
drwx------. cloud_user cloud_user unconfined_u:object_r:user_home_dir_t:s0 cloud_user
drwx------. jhalpert   jhalpert   staff_u:object_r:user_home_dir_t:s0 jhalpert
drwx------. pbeesly    pbeesly    staff_u:object_r:user_home_dir_t:s0 pbeesly
drwx------. ssm-user   ssm-user   unconfined_u:object_r:user_home_dir_t:s0 ssm-user
```

Both `jhalpert` and `pbeesly` now show `staff_u` as their SELinux user in the context label, consistent with their login mappings.

---

## Summary

Granting `sudo` access to SELinux confined users requires three coordinated steps, not just one. Editing sudoers alone is insufficient because SELinux enforces the user's context at every step. The full process is:

1. Map the Linux user to an SELinux user identity with `semanage login`.
2. Add a sudoers rule that includes the target `TYPE` and `ROLE` for the SELinux transition.
3. Relabel the user's home directory with `restorecon` to match the new SELinux identity.

Skipping any one of these steps would leave the configuration incomplete and `sudo` would still fail for the confined user.
<!--
https://app.pluralsight.com/hands-on/labs/9321ef6b-d0f4-4de6-acac-a31ff38a10c6?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F

Cloud Server public instance

Username

cloud_user

Password

#*el66gG

Private ip address of public instance

10.0.0.165

Public ip address of public instance

13.220.62.195

```
[root@ip-10-0-0-165 cloud_user]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

[root@ip-10-0-0-165 cloud_user]#

```
# Granting `sudo` Privileges to Confined Users

_This course is not approved or sponsored by Red Hat._

## Introduction

In this lab, we'll review the process of granting `sudo` privileges to SELinux confined users. When working with SELinux confined users, you may run into problems with Linux users not being able to use `sudo` anymore. We'll discuss why this happens and how to resolve the issue.

## Connecting to the Lab

1. Open your terminal application, and run the following command (remember to replace `<PUBLIC_IP>` with the public IP you were provided on the lab instructions page):

```bash
ssh cloud_user@<PUBLIC_IP>
```

1. Enter `yes` at the prompt.
2. Enter your `cloud_user` password at the prompt.
3. Become `root`.

```
sudo su 
```

## Map `pbeesly` and `jhalpert` to the Appropriate SELinux User

1. Map the user `pbeesly` to the `staff_u` user.

```
semanage login -a -s "staff_u" pbeesly
```

1. View the current logins to verify that this was successful.

```
semanage login -l
```

1. Map the user `jhalpert` to the `staff_u` user.

```
semanage login -a -s "staff_u" jhalpert
```

1. View the current logins to verify that this was successful.

```
semanage login -l
```

```
[cloud_user@ip-10-0-0-165 ~]$ sudo su
[root@ip-10-0-0-165 cloud_user]# semanage login -a -s "staff_u" pbeesly
[root@ip-10-0-0-165 cloud_user]# semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
pbeesly              staff_u              s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
system_u             system_u             s0-s0:c0.c1023       *
[root@ip-10-0-0-165 cloud_user]# semanage login -a -s "staff_u" jhalpert
[root@ip-10-0-0-165 cloud_user]# semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
jhalpert             staff_u              s0-s0:c0.c1023       *
pbeesly              staff_u              s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
system_u             system_u             s0-s0:c0.c1023       *
[root@ip-10-0-0-165 cloud_user]#
```
## Add `pbeesly` and `jhalpert` to the `sudoers` File

1. Open the `sudoers` file.

```
visudo
```

1. Type `/` and search for the line `root ALL=(ALL) ALL`.
2. Type `i` to enter insert mode.
3. Under the line `root ALL=(ALL) ALL`, add the following two lines:

```
pbeesly  ALL=(ALL)  TYPE=administrator_t  ROLE=administrator_r  /bin/sh
jhalpert  ALL=(ALL)  TYPE=administrator_t  ROLE=administrator_r  /bin/sh
```

1. Type `:wq!` to save and exit the file.

## Update the SELinux Security Context of Each User's Home Directory

1. View the current SELinux security context for both users.

```
ls -lZ /home/
```

2. Reset the SELinux security context for the user `pbeasly`.

```
restorecon -FR -v /home/pbeesly
```

3. Reset the SELinux security context for the user `jhalpert`.

```
restorecon -FR -v /home/jhalpert
```

1. Verify that the SELinux security context for both users has been updated.

```
ls -lZ /home/
```

```
[root@ip-10-0-0-165 cloud_user]# ls -lZ /home/
drwx------. cloud_user cloud_user unconfined_u:object_r:user_home_dir_t:s0 cloud_user
drwx------. jhalpert   jhalpert   unconfined_u:object_r:user_home_dir_t:s0 jhalpert
drwx------. pbeesly    pbeesly    unconfined_u:object_r:user_home_dir_t:s0 pbeesly
drwx------. ssm-user   ssm-user   unconfined_u:object_r:user_home_dir_t:s0 ssm-user
[root@ip-10-0-0-165 cloud_user]# restorecon -FR -v /home/pbeesly
restorecon reset /home/pbeesly context unconfined_u:object_r:user_home_dir_t:s0->staff_u:object_r:user_home_dir_t:s0
restorecon reset /home/pbeesly/.bash_logout context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/pbeesly/.bash_profile context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/pbeesly/.bashrc context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
[root@ip-10-0-0-165 cloud_user]# restorecon -FR -v /home/jhalpert
restorecon reset /home/jhalpert context unconfined_u:object_r:user_home_dir_t:s0->staff_u:object_r:user_home_dir_t:s0
restorecon reset /home/jhalpert/.bash_logout context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/jhalpert/.bash_profile context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
restorecon reset /home/jhalpert/.bashrc context unconfined_u:object_r:user_home_t:s0->staff_u:object_r:user_home_t:s0
[root@ip-10-0-0-165 cloud_user]# ls -lZ /home/
drwx------. cloud_user cloud_user unconfined_u:object_r:user_home_dir_t:s0 cloud_user
drwx------. jhalpert   jhalpert   staff_u:object_r:user_home_dir_t:s0 jhalpert
drwx------. pbeesly    pbeesly    staff_u:object_r:user_home_dir_t:s0 pbeesly
drwx------. ssm-user   ssm-user   unconfined_u:object_r:user_home_dir_t:s0 ssm-user
[root@ip-10-0-0-165 cloud_user]#
```
## Conclusion

Congratulations, you've successfully completed this hands-on lab!
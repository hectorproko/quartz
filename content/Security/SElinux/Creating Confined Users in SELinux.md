---
tags:
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

In this lab, we'll create an SELinux confined user by mapping an SELinux user %%(the user portion of the security context)%%to a Linux user. Confined users help us to impart restrictions on users to help protect our systems.

**Environment:** CentOS Linux 7 (Core)

### Map Linux users `jhalpert` and `pbeesley` to SELinux users

1. Map Linux user `jhalpert` to SELinux user `user_u`:

```bash
semanage login -a -s user_u jhalpert
```
%%[[semanage (SELinux Management)#Options]]%%
2. Map Linux user `pbeesley` to SELinux user `staff_u`:

```bash
semanage login -a -s staff_u pbeesley
```

3. Check the user mappings:

```bash
semanage login -l
```

We can see our Linux users successfully mapped to the assigned SELinux users.

```
Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
jhalpert             user_u               s0                   *
pbeesley             staff_u              s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
system_u             system_u             s0-s0:c0.c1023       *
```

### Ensure the SELinux user `xguest` cannot mount media

1. Check SELinux booleans for `xguest`:

```bash
getsebool -a | grep xguest
```
%%[[Pre-Defined SELinux Identities]]%%
We see `xguest_mount_media` is an option and it is enabled, so we need to disable it.

```
xguest_connect_network --> on
xguest_exec_content --> on
xguest_mount_media --> on
xguest_use_bluetooth --> on
```

2. Disable the `xguest_mount_media` boolean:

```bash
setsebool -P xguest_mount_media off
```

3. Verify the change was successful:

```bash
getsebool -a | grep xguest
```

We can see the change was successful.

```
xguest_connect_network --> on
xguest_exec_content --> on
xguest_mount_media --> off
xguest_use_bluetooth --> on
```

### Put SELinux into enforcing mode and ensure that setting is persistent

1. Check the current SELinux state:

```bash
getenforce
```

It is in permissive mode, so we need to change it to enforcing mode.

```
Permissive
```

2. Set SELinux to enforcing mode:

```bash
setenforce 1
```

3. Verify SELinux is now in enforcing mode:

```bash
getenforce
```

We can see the change worked and SELinux is now in enforcing mode.

```
Enforcing
```

4. Make the enforcing mode setting persistent across reboots by editing the config file:

```bash
vi /etc/selinux/config
```

Set the following value:

```
SELINUX=enforcing
```

> [!NOTE]- config
> ```
> [root@ip-10-0-0-220 cloud_user]# cat /etc/selinux/config
> 
> # This file controls the state of SELinux on the system.
> # SELINUX= can take one of these three values:
> #     enforcing - SELinux security policy is enforced.
> #     permissive - SELinux prints warnings instead of enforcing.
> #     disabled - No SELinux policy is loaded.
> SELINUX=enforcing
> # SELINUXTYPE= can take one of three two values:
> #     targeted - Targeted processes are protected,
> #     minimum - Modification of targeted policy. Only selected processes are protected.
> #     mls - Multi Level Security protection.
> SELINUXTYPE=targeted
> ```

## Conclusion

In this lab we mapped Linux users to SELinux identities using `semanage login`, disabled the `xguest_mount_media` boolean using `setsebool -P`, and switched SELinux from permissive to enforcing mode using `setenforce 1`, making it persistent by editing `/etc/selinux/config`.


%%
[Creating Confined Users in SELinux](https://app.pluralsight.com/hands-on/labs/d9a09e9a-bd01-41d2-bd16-44e8ae931c19?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

```
[cloud_user@ip-10-0-0-171 ~]$ cat /etc/os-release
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

[cloud_user@ip-10-0-0-171 ~]$
```
# Creating Confined Users in SELinux


## Introduction

In this lab, we'll create an SELinux confined user by mapping an SELinux user to a Linux user. Confined users help us to impart restrictions on users to help protect our systems.

## Solution

1. Begin by logging into the lab server using the credentials provided on the hands-on lab page:

```
ssh cloud_user@PUBLIC_IP_ADDRESS
```

2. Become the `root` user:

```
sudo su
```

### Map Linux users `jhalpert` and `pbeesley` to SELinux users

1. Map Linux user `jhalpert` to SELinux user `user_u`:

```
semanage login -a -s user_u jhalpert
```

2. Map Linux user `pbeesley` to SELinux user `staff_u`:

```
semanage login -a -s staff_u pbeesley
```

3. Check the user mappings:

```
semanage login -l
```

- We can see our Linux users successfully mapped to the assigned SELinux users.

```
[cloud_user@ip-10-0-0-220 ~]$ sudo su
[sudo] password for cloud_user:
[root@ip-10-0-0-220 cloud_user]# semanage login -a -s user_u jhalpert
[root@ip-10-0-0-220 cloud_user]# semanage login -a -s staff_u pbeesley
[root@ip-10-0-0-220 cloud_user]# semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
jhalpert             user_u               s0                   *
pbeesley             staff_u              s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
system_u             system_u             s0-s0:c0.c1023       *
[root@ip-10-0-0-220 cloud_user]#
```
### Ensure the SELinux user `xguest` can not mount media

1. Check SELinux booleans for "xguest":

```
`getsebool -a | grep xguest`
```

- We see "xguest_mount_media" is an option and it is enabled, so lets disable it.

2. Disable SELinux boolean "xguest_mount_media":

```
setsebool -P xguest_mount_media off
```

3. Check to make sure our changes were successful:

```
getsebool -a | grep xguest
```

- We can see our change was successful.

```
[root@ip-10-0-0-220 cloud_user]# getsebool -a | grep xguest
xguest_connect_network --> on
xguest_exec_content --> on
xguest_mount_media --> on
xguest_use_bluetooth --> on
[root@ip-10-0-0-220 cloud_user]# setsebool -P xguest_mount_media off
[root@ip-10-0-0-220 cloud_user]# getsebool -a | grep xguest
xguest_connect_network --> on
xguest_exec_content --> on
xguest_mount_media --> off
xguest_use_bluetooth --> on
[root@ip-10-0-0-220 cloud_user]#
```

### Put SELinux into enforcing mode and ensure that setting is persistent

1. Check SELinux state:

```
getenforce
```

- It is in permissive mode, so we need to change it to enforcing mode.

2. Put SELinux into enforcing mode:

```
setenforce 1  
```

3. Check to make sure SELinux is now in enforcing mode:

```
getenforce
```

- We can see our change worked and SELinux is now in enforcing mode.

4. Ensure SELinux boots into enforcing mode:

```
vi /etc/selinux/config
```

```
SELINUX=enforcing
```

```
[root@ip-10-0-0-220 cloud_user]# getenforce
Permissive
[root@ip-10-0-0-220 cloud_user]# setenforce 1
[root@ip-10-0-0-220 cloud_user]# getenforce
Enforcing
[root@ip-10-0-0-220 cloud_user]# vi /etc/selinux/config
[root@ip-10-0-0-220 cloud_user]# cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted


[root@ip-10-0-0-220 cloud_user]#
```

## Conclusion

Congratulations — you've completed this hands-on lab!%%
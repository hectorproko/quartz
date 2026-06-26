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

SELinux (Security-Enhanced Linux) can run in one of three modes, and the goal of this lab was to practice moving between all three on a CentOS 7 instance:

- **Enforcing** - SELinux actively enforces its security policy. Any action that violates a policy rule is blocked and logged.
- **Permissive** - SELinux does not block anything, but it still logs what _would_ have been blocked if enforcing mode were active. This is useful for testing or troubleshooting policy without breaking functionality.
- **Disabled** - SELinux is completely turned off. No policy is loaded, and nothing is logged.

Mode changes can be made in two ways:

- **Session-only (temporary)** - changes the running state until the next reboot, using the `setenforce` command.
- **Persistent** - changes the `/etc/selinux/config` file so the mode survives a reboot.

For this lab, the task was to go enforcing → permissive → disabled → enforcing, applying both the session and persistent change at each step.

## Environment

- AWS EC2 instance running CentOS, accessed via SSH
- Cloud Server public instance:
  - Username: `cloud_user`
  - Private IP: `10.0.1.29`
  - Public IP: `32.192.49.112`

## Solution

Since all three transitions follow the same pattern (edit the config file for the persistent change, then run a command for the session change), the steps below are general. Just substitute the target mode (`permissive`, `disabled`, or `enforcing`) wherever shown.

### General Steps to Change SELinux Mode

1. Open the SELinux configuration file:

```
   sudo vim /etc/selinux/config
```

2. Find the `SELINUX=` line and change it to the target mode, for example:

```
   SELINUX=permissive
```

3. Apply the change to the current session:
    - To switch to **enforcing**, run:

```
     sudo setenforce 1
```

- To switch to **permissive**, run:

```
     sudo setenforce 0
```

- To switch to **disabled**, `setenforce` cannot be used since SELinux must be unloaded entirely. Instead, reboot the system after saving the config file:

```
     sudo reboot
```

```
 A reboot is also required if you want to move from disabled back into enforcing or permissive, since SELinux needs to reload its policy from a fully disabled state.
```

### Example: Enforcing to Permissive

```
[cloud_user@ip-10-0-1-29 ~]$ sudo vim /etc/selinux/config
[sudo] password for cloud_user:
[cloud_user@ip-10-0-1-29 ~]$ cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

[cloud_user@ip-10-0-1-29 ~]$ sudo setenforce 0
[cloud_user@ip-10-0-1-29 ~]$
```
%%`SELINUXTYPE=targeted` is a separate setting in that same config file — it controls _which policy_ SELinux applies, not the mode (enforcing/permissive/disabled).%%
## Conclusion

This exercise reinforced the difference between SELinux's three modes (enforcing, permissive, and disabled), and the distinction between a temporary session change (`setenforce`) versus a persistent change made in `/etc/selinux/config`. It also highlighted a key limitation: **disabling** or **re-enabling** SELinux always requires a reboot, since `setenforce` only toggles between enforcing and permissive while SELinux is already loaded.

<!--very  simple
Cloud Server public instance

**Username**
cloud_user
**Password**

**Private ip address of public instance**
10.0.1.29
**Public ip address of public instance**
32.192.49.112

(https://app.pluralsight.com/hands-on/labs/1ade0280-072f-4472-afdf-77f65aae6e9e?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)



**This lab has been retired.**
This lab is accessible via a direct link but is no longer discoverable in our library. For our most up-to-date content, please visit the [Hands-on page](https://app.pluralsight.com/hands-on/?tab=hands-on-labs&trial=true).


# Rotate Between_3 SELinux Modes

## Introduction

The goal of this lab is to change between the three SELinux modes: enforcing, permissive, and disabled. You will begin with the enforcing mode being active first, and then you will need to switch to permissive mode. After that, you will switch to disabled mode and then back to enforcing mode again. When rotating between modes, you can do a rotation of modes for the current session or a permanent mode change. An example of a permanent change would be when you go into a configuration file and you write permissive instead of enforcing or instead of permissive you write disabled. The goal is to set SELinux to permissive mode both for persistent reboot and for the session without a system reboot. After that, set the mode to disabled for both the persistent reboot and for the session with a system reboot. Lastly, change back to enforcing mode for both the reboot and the session.

## Solution

im using alamax linux in virtualbox sshing into it using [[port forwarding#VirtualBox]]

```
[cloud_user@ip-10-0-1-29 ~]$ cat /etc/os-release
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

[cloud_user@ip-10-0-1-29 ~]$
```


### Change SELinux Status from Enforcing to Permissive

1. Open the configuration file for SELinux:
    
    ```
    sudo vim /etc/selinux/config
    ```
    
2. In the file, delete `SELINUX=enforcing` and write `SELINUX=permissive`.
3. Save and exit the file by pressing **Escape** followed by **:wq!**.
4. Press **Enter**.
5. Reload the configuration file in permissive mode:
    
    ```
    sudo setenforce 0
    ```
    
```
[cloud_user@ip-10-0-1-29 ~]$ sudo vim /etc/selinux/config
[sudo] password for cloud_user:
[cloud_user@ip-10-0-1-29 ~]$ cat /etc/selinux/
cat: /etc/selinux/: Is a directory
[cloud_user@ip-10-0-1-29 ~]$ cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted


[cloud_user@ip-10-0-1-29 ~]$ sudo setenforce 0
[cloud_user@ip-10-0-1-29 ~]$

```
### Change SELinux Status from Permissive to Disabled

1. Open the configuration file for SELinux:
    
    ```
    sudo vim /etc/selinux/config
    ```
    
2. In the file, delete `SELINUX=permissive` and write `SELINUX=disabled`.
3. Save and exit the file by pressing **Escape** followed by **:wq!**.
4. Press **Enter**.
5. Reboot SELinux:

```
[cloud_user@ip-10-0-1-29 ~]$ sudo su
[root@ip-10-0-1-29 cloud_user]# vim /etc/selinux/config
[root@ip-10-0-1-29 cloud_user]# cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimium - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted


[root@ip-10-0-1-29 cloud_user]#
```


```
sudo reboot
```
    

### Change SELinux Status from Disabled to Enforcing

1. Open the configuration file for SELinux:

```
sudo vim /etc/selinux/config
```

1. In the file, delete `SELINUX=disabled` and write `SELINUX=enforcing`.
2. Save and exit the file by pressing **Escape** followed by **:wq!**.
3. Press **Enter**.
4. Reboot SELinux:
    
    ```
    sudo reboot
    ```
    
just repeating the same with differente keywords
## Conclusion

Congratulations — you've completed this hands-on lab!
---
tags:
  - selinux
linkedin: "False"
quartz: "False"
refactored: "False"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---

%%**Source:** [Pluralsight Hands-On Lab](https://app.pluralsight.com/hands-on/labs/2ed3e317-3f75-4304-8a61-9fbb118abae3)

![[users_diagram.excalidraw]]
We probably using RHEL7
%%
## Overview

SELinux is well known for confining processes, files, and daemons, but it can also confine access on a per-user basis. By mapping Linux users to SELinux users, you can restrict what actions a given account is allowed to perform based on the role it belongs to. In this lab, I changed default SELinux user settings and added users to both privileged and unprivileged roles to see how those assignments affect what each account can do.

### Environment

| Component    | Details                        |
| ------------ | ------------------------------ |
| OS           | Amazon Linux 2 (RHEL 7-based)  |
| Private IP   | 10.0.1.101                     |
| Public IP    | 54.237.219.67                  |
| Default User | `cloud_user` (sudo privileges) |

---

## Add Staff Users

Staff users are privileged users that belong to the `wheel` group and are mapped to the `staff_u` SELinux user. This allows them to use `sudo` while still being confined by SELinux policy.

1. Add `bobb` as a staff user:

```bash
sudo useradd -G wheel -Z staff_u bobb
```
*We use `-Z` to map user bobb to [[Pre-Defined SELinux Identities|Pre-Defined SELinux Identity]] staff_u*

2. Add `lindab` as a staff user:

```bash
sudo useradd -G wheel -Z staff_u lindab
```

3. Assign `lindab` a password:

```bash
sudo passwd lindab
```

```
Changing password for user lindab.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

4. Exit the current session:

```bash
exit
```

```
logout
Connection to 54.237.219.67 closed.
```

5. Log in as `lindab` to verify the assignment:

```bash
ssh lindab@<PUBLIC_IP>
id -Z
```

The `id -Z` command prints the SELinux security context for the current user. This confirms which SELinux user, role, and type the account is running under.

**Output:**

```bash
[lindab@logging ~]$ id -Z
staff_u:staff_r:staff_t:s0-s0:c0.c1023
```
%%[[SELinux#SELinux Contexts]] look for 3 different types%%
The context confirms `lindab` is mapped to `staff_u`, running in `staff_r` role, as expected.

6. Log out and return to `cloud_user`:

```bash
exit
ssh cloud_user@<PUBLIC_IP>
```

---

## Add Non-Privileged Users

Guest users are unprivileged accounts mapped to `guest_u`. They have no `sudo` access and are confined to a minimal set of permissions, making them suitable for temporary or restricted access scenarios.%%[[Pre-Defined SELinux Identities]]%%

Add `teddy` as a guest user:

```bash
sudo useradd -Z guest_u teddy
```

**Output:**

```bash
[cloud_user@logging ~]$ sudo useradd -Z guest_u teddy
[sudo] password for cloud_user:
[cloud_user@logging ~]$
```

---

## Change the Default SELinux Role

By default, new Linux users are mapped to the `unconfined_u` SELinux user. Changing the default mapping to `user_u` means any new account created without an explicit `-Z` flag will automatically be confined to the `user_r` role, which is more restrictive than the default. %%[[Pre-Defined SELinux Identities]]%%

1. Change the default SELinux mapping to `user_u`:

```bash
sudo semanage login -m -s user_u -r s0 __default__
```
%%[[semanage (SELinux Management)#Options]]
The `__default__` entry in SELinux is a catch-all mapping. When a Linux user logs in and there is no explicit SELinux user mapping defined for their account, SELinux falls back to whatever `__default__` is set to.
%%
The `__default__` entry controls what SELinux user is assigned to any Linux user not explicitly mapped. Setting it to `user_u` applies the `user` role to all future accounts by default.

2. Add `tinab` without specifying a role, relying on the new default:

```bash
sudo useradd tinab
sudo passwd tinab
```

```
Changing password for user tinab.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

3. Log out and log in as `tinab`:

```bash
exit
ssh tinab@<PUBLIC_IP>
```

4. Verify the SELinux context:

```bash
id -Z
```

```bash
[tinab@logging ~]$ id -Z
user_u:user_r:user_t:s0
```

The context shows `tinab` was automatically assigned `user_u:user_r:user_t:s0`, confirming the default mapping change took effect without needing an explicit `-Z` flag.

---

## Conclusion

This lab demonstrated how SELinux user mappings add an additional layer of access control beyond standard Linux permissions. By assigning users to roles like `staff_u`, `guest_u`, and `user_u`, and by modifying the `__default__` mapping, you can enforce the principle of least privilege at the user level across the entire system.

<!--

[Working with SELinux Users and Roles](https://app.pluralsight.com/hands-on/labs/2ed3e317-3f75-4304-8a61-9fbb118abae3?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

![[users_diagram.excalidraw]]

---
Cloud Server MariaDB Server

Username
cloud_user

Password
4Sj]%n[w

MariaDB Server Private IP
10.0.1.101

MariaDB Server Public IP
54.237.219.67
## Lab Overview

SELinux is known for confining our processes, files, and daemons — but did you know it can confine on a user-specific basis, too? By assigning SELinux users to our regular Linux users, we can limit access based on need. In this lab, you'll change your default SELinux user settings as well as add users to privileged and unprivileged roles.

# Working with SELinux Users and Roles

## Introduction

SELinux is known for confining our processes, files, and daemons — but did you know it can confine on a user-specific basis, too? By assigning SELinux users to our regular Linux users, we can limit access based on need. In this lab, you'll change your default SELinux user settings as well as add users to privileged and unprivileged roles.

## Solution

Log in to the server using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP>
```

sudo su
### Add Staff Users

1. Add user `bobb` as a `staff` user. When prompted for a password, use the same one you used to log in:
    
    ```
    sudo useradd -G wheel -Z staff_u bobb
    ```
    
2. Add user `lindab` as a `staff` user:
    
    ```
    sudo useradd -G wheel -Z staff_u lindab
    ```
    
3. Assign `lindab` any password:
    
    ```
    sudo passwd lindab
    ```
    
4. Exit your current `cloud_user` session:
    
    ```
    exit
    ```


```
[cloud_user@logging ~]$     sudo useradd -G wheel -Z staff_u bobb
[sudo] password for cloud_user:

[cloud_user@logging ~]$
[cloud_user@logging ~]$     sudo useradd -G wheel -Z staff_u lindab
[cloud_user@logging ~]$     sudo passwd lindab
Changing password for user lindab.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
[cloud_user@logging ~]$ exit
logout
Connection to 54.237.219.67 closed.

```


1. Log in as `lindab` to ensure everything is working. Use the same public IP address you used to log in as `cloud_user`. Use the password you just created for `lindab`:
    
    ```
    ssh lindab@<PUBLIC_IP>
    ```
    
2. Verify `lindab` is assigned to the `staff` user context:
    
    ```
    id -Z
    ```
    
3. Once verified, log out of the session as `lindab` and log in again as `cloud_user`:
    
    ```
    exit
    ssh cloud_user@<PUBLIC_IP>
    ```
    
```
  20/05/2026   12:02.06   /home/mobaxterm  ssh lindab@54.237.219.67
lindab@54.237.219.67's password:
X11 forwarding request failed on channel 0

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
47 package(s) needed for security, out of 80 available
Run "sudo yum update" to apply all updates.
[lindab@logging ~]$ id -Z
staff_u:staff_r:staff_t:s0-s0:c0.c1023
[lindab@logging ~]$ exit
logout
Connection to 54.237.219.67 closed.
```

### Add Non-privileged Users

Add `teddy` as a `guest` user:

```
sudo useradd -Z guest_u teddy
```

### Change Default Role

1. Change the default role to `user`:
    
    ```
    sudo semanage login -m -s user_u -r s0 __default__
    ```
    
2. Add `tinab` as a `user` user:
    
    ```
    sudo useradd tinab
    ```
    
3. Assign `tinab` a password:
    
    ```
    `sudo passwd tinab`
    ```
    
4. Log out of `cloud_user` and log in as `tinab`:
    
    ```
    exit
    ssh tinab@<PUBLIC_IP>
    ```
    
5. Verify `tinab` is assigned as the SELinux user:
    
    ```
    id -Z
    ```
    

```
[cloud_user@logging ~]$ sudo useradd -Z guest_u teddy
[sudo] password for cloud_user:
[cloud_user@logging ~]$ sudo semanage login -m -s user_u -r s0 __default__
[cloud_user@logging ~]$ sudo useradd tinab
[cloud_user@logging ~]$ sudo passwd tinab
Changing password for user tinab.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
[cloud_user@logging ~]$ exit
logout
Connection to 54.237.219.67 closed.
                                                                                                                                      ✓

  20/05/2026   12:05.56   /home/mobaxterm  ssh tinab@54.237.219.67
tinab@54.237.219.67's password:
X11 forwarding request failed on channel 0

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
47 package(s) needed for security, out of 80 available
Run "sudo yum update" to apply all updates.
[tinab@logging ~]$ id -Z
user_u:user_r:user_t:s0
[tinab@logging ~]$
```
## Conclusion

Congratulations — you've completed this hands-on lab!
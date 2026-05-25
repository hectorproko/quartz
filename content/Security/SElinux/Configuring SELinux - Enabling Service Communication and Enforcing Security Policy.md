---
title: "Configuring SELinux: Enabling Service Communication and Enforcing Security Policy"
tags:
  - selinux
  - RHEL
  - hands-on
quartz: "True"
linkedin: "False"
hardlinked: "True"
refactored: "True"
pluralsight: "True"
---
## Overview

Security-Enhanced Linux (SELinux) is a mandatory access control (MAC) framework built into the Linux kernel. Unlike traditional discretionary access controls (file permissions), SELinux enforces security policies at the kernel level, meaning even a process running as `root` can be denied access if the policy says so.

In this lab, I worked through a real-world scenario: a network team needed SELinux configured to allow their **Zabbix** network monitoring application to communicate through `httpd`. My tasks were to:

1. Use an **SELinux boolean** to permit `httpd` → Zabbix communication.
2. Switch SELinux from **permissive** to **enforcing** mode.
3. Make the enforcing mode setting **persistent** across reboots.

---

## Environment

| Detail             | Value                                                      |
| ------------------ | ---------------------------------------------------------- |
| OS                 | Red Hat-based Linux (RHEL/CentOS)                          |
| Access             | SSH as `cloud_user`, then escalated to `root`              |
| SELinux tools used | `getsebool`, `setsebool`, `getenforce`, `setenforce`, `vi` |

---

## Part 1 - Permitting `httpd` to Communicate with Zabbix

### Why This Matters

SELinux ships with a large set of **booleans**, on/off switches that toggle specific pre-defined behaviors without requiring you to write custom policies. This is the safest and most maintainable way to adjust SELinux for known applications like Zabbix.

<!--
**Zabbix** is a network monitoring platform. It has two main pieces:
- **Zabbix Server** — the brain, collects and processes monitoring data
- **Zabbix Frontend** — the web dashboard (apache httpd) you open in a browser to _see_ all your graphs and alerts

That web dashboard is a **PHP web application** — and it runs _through_ **Apache (`httpd`)**. So when you open Zabbix in your browser, you're hitting Apache, and Apache needs to talk to the Zabbix server backend to pull the data.

So the flow looks like this:

```
Your Browser
     │
     ▼
Apache (httpd)          ← this is what SELinux was blocking
     │
     ▼
Zabbix Server/Backend
     │
     ▼
Monitored hosts (agents)
```

SELinux by default says: _"Hey Apache, you're a web server — you have no business making outbound connections to some other service."_ That's the whole point of SELinux's least-privilege model. It doesn't know or care that Zabbix is legitimate — it blocks it anyway.

The boolean `httpd_can_connect_zabbix` is essentially telling SELinux: _"I know what I'm doing — this Apache instance is the **Zabbix frontend,** so yes, allow it to reach the Zabbix backend."_
-->
### Step 1: Discover the Relevant Boolean

Before changing anything, I queried all SELinux booleans and filtered for Zabbix-related ones:

```bash
getsebool -a | grep zabbix
```

**Output:**

```
httpd_can_connect_zabbix --> off
zabbix_can_network --> off
zabbix_run_sudo --> off
```

> Three booleans exist for Zabbix. The one I need is `httpd_can_connect_zabbix`, which controls whether the Apache `httpd` service is allowed to initiate connections to Zabbix, currently **off**.

---

### Step 2: Enable the Boolean

The `-P` flag makes the change **persistent**, it survives a reboot by writing to the SELinux policy store.

```bash
setsebool -P httpd_can_connect_zabbix on
```

> Without `-P`, the boolean change would only last until the next reboot. Always use `-P` in production unless you intentionally want a temporary change.

---

### Step 3: Verify the Change

```bash
getsebool -a | grep zabbix
```

**Output:**

```
httpd_can_connect_zabbix --> on
zabbix_can_network --> off
zabbix_run_sudo --> off
```

✅ `httpd_can_connect_zabbix` is now **on**. The other two booleans remain off, I only modified what was needed, following the principle of least privilege.

---

## Part 2 - Switching SELinux to Enforcing Mode

### Understanding SELinux Modes

| Mode         | Behavior                                                   |
| ------------ | ---------------------------------------------------------- |
| `Enforcing`  | Actively blocks policy violations and logs them            |
| `Permissive` | Logs violations but **does not block**, useful for testing |
| `Disabled`   | SELinux is completely off, not recommended                 |

The system was in **permissive** mode, which means violations were being logged but not stopped. The network team needed it in **enforcing** mode to properly test security behavior.

---

### Step 1: Check the Current Mode

```bash
getenforce
```

**Output:**

```
Permissive
```

---

### Step 2: Switch to Enforcing Mode

```bash
setenforce 1
```

> `setenforce` accepts either `1` (Enforcing) or `0` (Permissive). This change is **immediate but not persistent** on its own, it lives only in the running kernel.

---

### Step 3: Confirm the Change

```bash
getenforce
```

**Output:**

```
Enforcing
```

✅ SELinux is now actively enforcing policy.

---

## Part 3 - Making Enforcing Mode Persistent

`setenforce` only affects the running system. If the server reboots, SELinux reads its startup mode from `/etc/selinux/config`. I needed to update that file.

### Editing the SELinux Config File

```bash
vi /etc/selinux/config
```

I located the `SELINUX=` directive and changed it to:

```
SELINUX=enforcing
```

### Verifying the Config File

```bash
cat /etc/selinux/config
```

**Output:**

```
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
```

✅ The config confirms `SELINUX=enforcing`. On the next boot, the system will start SELinux in enforcing mode automatically.

> Notice `SELINUXTYPE=targeted` - this means SELinux only confines specific high-risk processes (like `httpd`, `sshd`, etc.) rather than every single process on the system. This is the standard and practical default for most RHEL-based systems.

---


## Key Takeaways

- **SELinux booleans** are the cleanest way to enable known application behaviors without writing custom policies. Always search with `getsebool -a | grep <app>` before doing anything more complex.
- **`setsebool -P`** is critical - always include `-P` unless you deliberately want a temporary change.
- **`setenforce`** changes the running mode instantly but is not persistent. You must **also** update `/etc/selinux/config` to survive reboots.
- **Never disable SELinux** just because it blocks something. Find the right boolean or policy adjustment instead.
- The `targeted` policy type is the practical default, it protects what matters most without locking down every process.

---

## Commands Reference

| Command                          | Purpose                                 |
| -------------------------------- | --------------------------------------- |
| `getsebool -a \| grep <name>`    | List and filter SELinux booleans        |
| `setsebool -P <boolean> on\|off` | Set a boolean persistently              |
| `getenforce`                     | Show current SELinux mode               |
| `setenforce 1\|0`                | Switch mode at runtime (non-persistent) |
| `vi /etc/selinux/config`         | Edit persistent SELinux config          |
| `cat /etc/selinux/config`        | Verify config file contents             |


<!--

**"Enabling Service Communication"** → refers to **Part 1**, where you used `setsebool` to allow `httpd` to talk to Zabbix.

By default, SELinux blocks services from communicating with each other unless explicitly permitted. You _enabled_ that communication channel by flipping the boolean on. Without this step, Zabbix simply wouldn't work through `httpd` — SELinux would silently block it.

---

**"Enforcing Security Policy"** → refers to **Part 2 & 3**, where you switched SELinux from `Permissive` to `Enforcing` mode and made it persistent.

In permissive mode, SELinux watches but doesn't act — it's essentially not protecting anything. Switching to enforcing mode means the security policy is now _actively applied_. That's the whole point of SELinux existing on the system.


**Cloud Server public instance**

Username
cloud_user

Password
7|Cxw(g^

Private ip address of public instance
10.0.0.31

Public ip address of public instance
54.91.128.208
## Lab Overview

In this lab we will edit SELinux settings, using booleans to allow communications between services. Then we will place SELinux into _enforcing_ mode and ensure that setting is persistent.

_This course is not approved or sponsored by Red Hat._

# Configuring SELinux

_This course is not approved or sponsored by Red Hat._

## Introduction

The network team needs help setting up SELinux to function with their new Zabbix network monitoring application. We need configure SELinux to permit `httpd` to communicate with Zabbix. Then we have to put SELinux into enforcing mode so that the network team can test our settings. We need to be sure enforcing mode is persistent.

## Setting Up the Environment

1. Open your terminal application, and log in to the environment using the credentials provided on the lab instructions page. (Remember to replace `<PUBLIC_IP_ADDRESS>` with the actual public IP address.)

```
ssh cloud_user@&lt;PUBLIC_IP_ADDRESS&gt;
```

2. Type `yes` at the prompt.
    
3. Enter your password at the prompt.
    
4. Become `root` (by executing `su -`).
    

## Permit `httpd` to Communicate with Zabbix

1. Find the necessary boolean to permit `httpd` to communicate with Zabbix

```
[root@host]# getsebool -a | grep zabbix
```

We'll see the boolean is _off_

2. Set the boolean to "on"

```
[root@host]# setsebool -P httpd_can_connect_zabbix on
```

3. Verify that change took effect

```
[root@host]# getsebool -a | grep zabbix
```

Now we'll see that the boolean is _on_.

my output
```
[cloud_user@ip-10-0-0-31 ~]$ sudo su
[sudo] password for cloud_user:
[root@ip-10-0-0-31 cloud_user]# getsebool -a | grep zabbix
httpd_can_connect_zabbix   off
zabbix_can_network   off
zabbix_run_sudo   off
[root@ip-10-0-0-31 cloud_user]# setsebool -P httpd_can_connect_zabbix on
[root@ip-10-0-0-31 cloud_user]# getsebool -a | grep zabbix
httpd_can_connect_zabbix  on
zabbix_can_network   off
zabbix_run_sudo   off
[root@ip-10-0-0-31 cloud_user]#
```
## Put SELinux into _enforcing_ Mode and Ensure That the Setting Is Persistent

1. Check the SELinux state

```
[root@host]# getenforce
```

This will show that it is in _permissive_ mode, so we need to change it to _enforcing_ mode.

2. Put SELinux into enforcing mode

```
[root@host]# setenforce 1
```

3. Check to make sure SELinux is now in enforcing mode

```
[root@host]# getenforce
```

We can see our change worked and SELinux is now in enforcing mode.

4. Ensure SELinux boots into enforcing mode
    
    Edit the SELinux configuration file:
    
    ```
    [root@host]# vi /etc/selinux/config
    ```
    
    Type `i` to enter _Insert_ mode, arrow down to the `SELINUX` line, and set it to `enforcing`:
    
    ```
    SELINUX=enforcing
    ```
    
    Type `Esc`, then `:wq` to exit. When the server boots again, SELinux will remain in _enforcing_ mode.
    

```
[root@ip-10-0-0-31 cloud_user]# getenforce
Permissive
[root@ip-10-0-0-31 cloud_user]# setenforce 1
[root@ip-10-0-0-31 cloud_user]# getenforce
Enforcing
[root@ip-10-0-0-31 cloud_user]# vi /etc/selinux/config
[root@ip-10-0-0-31 cloud_user]# cat /etc/selinux/config

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


[root@ip-10-0-0-31 cloud_user]#
```
## Conclusion

Congratulations, you've successfully completed this hands-on lab!

Refactored ✅ [[Configuring SELinux - Enabling Service Communication and Enforcing Security Policy]]



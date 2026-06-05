---

tags:
  - security
  - hardening
  - compliance
linkedin: "False"
quartz: "False"
refactored: "True"
pluralsight: "True"
hands-on: "True"
hardlinked: "True"
---
## Overview

In enterprise and government environments, systems must meet strict security baselines to be considered trustworthy. One of the most widely adopted standards is the **[[STIG (Security Technical Implementation Guide)]]**, a set of configuration guidelines published by the [[DISA (Defense Information Systems Agency)|Defense Information Systems Agency (DISA)]] to harden operating systems against known vulnerabilities.

In this lab, I configured the Linux **[[auditd]]** service on a Red Hat host to load the precompiled STIG rule set. The goal is to give the system visibility into whether it meets STIG compliance requirements by actively tracking the security-relevant events those rules define.
<!--
this one configures _what_ auditd monitors (STIG rules), and the OpenSCAP lab verifies _whether the system is compliant_.

**The log itself is not "violations" exactly** — it's more like a receipt. auditd records every time a watched event happens, whether it was authorized or not. It's up to a human or a tool to decide if it's suspicious.

we used in auditbeat
-->
---

## Why This Matters

The Linux audit daemon (**[[auditd]]**) is the kernel-level system responsible for recording security events, things like file access, privilege escalation, system calls, and user logins. Out of the box, it ships with a minimal rule set. However, Red Hat provides **prebuilt rule files** for compliance frameworks like STIG, PCI-DSS, and NISPOM, located at:

```
/usr/share/doc/audit-*/rules/
```

By appending these rules to the active audit configuration, we tell the kernel exactly what events to monitor according to STIG requirements. This is a foundational step before running any compliance scan or audit report.
<!--
when scanning with OpenSCAP you also need to have auditd

A building can have locks on every door, but if there's no camera footage, no visitor log, and no access record — a security auditor will still flag it. The locks are the hardening. The camera footage is auditd. **Both are required to pass.**
-->
---

## Environment

| Detail | Value |
|---|---|
| **OS** | Red Hat Enterprise Linux (EC2 instance) |
| **Audit version** | `audit-2.8.5` |
| **Connection** | SSH via MobaXterm |
| **Privilege level** | `cloud_user` → escalated to `root` |

---

## Step 1 - Connect to the Server

I connected to the cloud instance over SSH using MobaXterm:

```bash
ssh cloud_user@52.87.246.170
```

**Output:**
```
Warning: Permanently added '52.87.246.170' (ED25519) to the list of known hosts.
cloud_user@52.87.246.170's password:
X11 forwarding request failed on channel 0
[cloud_user@ip-10-0-1-77 ~]$
```

## Step 2 - Back Up the Existing Audit Rules

Before making any changes to a system configuration file, it's critical to create a backup. This lets us restore the original state if something goes wrong.

```bash
sudo cp /etc/audit/rules.d/audit.rules /etc/audit/rules.d/audit.rules_backup
```

**What I learned here:** My first attempt ran the command *without* `sudo` and hit a permission error, the audit rules directory is root-protected. This is expected and is itself a good sign that the system is properly restricting access to sensitive config files.

```
cp: failed to access '/etc/audit/rules.d/audit.rules_backup': Permission denied
```

Running with `sudo` resolved it immediately.

---

## Step 3 - Locate the STIG Rule Files

The installed version on this server was **`audit-2.8.5`**. I discovered this by using tab-completion to explore the directory:

```bash
cd /usr/share/doc/au<TAB>
```

```
audit-2.8.5/      authconfig-6.2.8/
```

I navigated to the correct path:

```bash
cd /usr/share/doc/audit-2.8.5/rules/
```

---

## Step 4 - Apply the STIG Rules

This is the core action of the lab. The `30-stig.rules` file contains the STIG-specific audit rules, and `99-finalize.rules` adds a locking directive that prevents further modification of the ruleset at runtime (a security measure in itself).
<!--
What `99-finalize.rules` contains:
-e 2, Enable auditing + **lock the ruleset**
`-e` sets the **enforcement mode** of the audit system:
-->
I needed full root access, even `sudo` on a redirect (`>>`) is blocked because the shell opens the file *before* sudo can elevate. The fix is to escalate to a proper root shell first:

```bash
sudo -i
```

Then from the root shell:

```bash
cd /usr/share/doc/audit-2.8.5/rules/
cat 30-stig.rules 99-finalize.rules >> /etc/audit/rules.d/audit.rules
```

**Why `cat` with `>>`?** We're *appending* the STIG rules to the existing `audit.rules` file rather than replacing it. This preserves any baseline rules already configured while adding the STIG compliance layer on top.

---

## Step 5 - Restart and Verify `auditd`

Configuration files are read at service startup, so we need to restart `auditd` to load the new rules into the kernel:
<!--
When auditd loads `-e 2` it tells the kernel:
The audit ruleset is now frozen.
No changes allowed until next reboot.

What exactly gets locked:
Not just `30-stig.rules` — the **entire active ruleset in memory**. Remember `augenrules` compiles all the `.rules` files into one running ruleset in the kernel. `-e 2` locks that compiled result.
-->
```bash
service auditd restart
```

```
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
```

Then I verified the service came back up and is actively running:

```bash
service auditd status
```

**Output:**
```
● auditd.service - Security Auditing Service
   Loaded: loaded (/usr/lib/systemd/system/auditd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2026-05-13 21:08:19 UTC; 8s ago
     Docs: man:auditd(8)
           https://github.com/linux-audit/audit-documentation
  Process: 1341 ExecStartPost=/sbin/augenrules --load (code=exited, status=0/SUCCESS)
  Process: 1336 ExecStart=/sbin/auditd (code=exited, status=0/SUCCESS)
 Main PID: 1337 (auditd)
   CGroup: /system.slice/auditd.service
           └─1337 /sbin/auditd

May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: enabled 1
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: backlog_limit 8192
May 13 21:08:19 ip-10-0-1-77.ec2.internal systemd[1]: Started Security Auditing Service.
```

Key things to confirm in this output:

| Field | Expected Value | Why It Matters |
|---|---|---|
| `Active` | `active (running)` | Service is live |
| `ExecStartPost` | `status=0/SUCCESS` | `augenrules --load` compiled and loaded rules without errors |
| `enabled` | `1` | Audit enforcement is on |
| `failure` | `1` | Audit failure mode is set (logs and continues on error) |
| `backlog_limit` | `8192` | Buffer size for event queue, sufficient for normal workloads |

The line `augenrules --load` with `status=0/SUCCESS` is especially important, it confirms that `augenrules` parsed and loaded our newly appended STIG rules into the running kernel successfully.

---

## Troubleshooting Notes

| Issue | Cause | Fix |
|---|---|---|
| `Permission denied` on `cp` | Missing `sudo` | Prefix with `sudo` |
| `No such file or directory` on `cd` | Lab doc referenced older package version (`2.8.1`) | Use tab-completion to find installed version (`2.8.5`) |
| `Permission denied` on `>>` redirect even with `sudo` | Shell redirects run as current user, not sudo | Escalate to root with `sudo -i` first |

---

## Key Concepts Reinforced

- **[[auditd]]** is the Linux Audit Daemon, it hooks into the kernel to record security-relevant system calls and file access events.
- **STIG rules** define *what* to monitor; `auditd` is the engine that does the monitoring.
- **`augenrules`** compiles all `.rules` files in `/etc/audit/rules.d/` into a single active ruleset and loads it into the kernel, this happens automatically on service restart.
- **`99-finalize.rules`** sets the audit configuration to immutable mode, meaning the rules can't be changed without a reboot. This is a deliberate STIG hardening requirement.
- **File redirect permissions**, even with `sudo`, `>>` redirects are handled by the shell as the invoking user. Always escalate to a root shell for operations involving privileged file writes.

---

## Takeaway

This lab is a practical introduction to compliance-driven security configuration. Rather than manually writing audit rules from scratch, Red Hat provides precompiled, standards-aligned rule sets that can be applied in minutes. The real skill is understanding *why* each step exists, the backup, the escalation strategy, the service restart, and being able to adapt when the environment doesn't match the documentation exactly.

---

<!--
###### DRAFT#######
[Configuring Audit Settings for STIG Compliance on Red Hat](https://app.pluralsight.com/hands-on/labs/d6011d55-ed1b-4328-bdcd-0ec6c4ce1f90?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

so we are just adding rules to the audit service that red hat to track compliance with STIG?

Cloud Server public instance

Username
cloud_user

Password
```
JsE|(ZU7
```

Private ip address of public instan
10.0.1.77

Public ip address of public instanc
52.87.246.170
## Lab Overview

The Red Hat Linux audit service comes with precompiled rule sets for various compliance requirements. In this lab, we will configure a Red Hat host's audit rules to include the STIG (Security Technical Implementation Guide) compliance rule set. This will allow us to identify any points at which we are not compliant with STIG requirements.

_This course is not approved or sponsored by Red Hat._

# Configuring Audit Settings for STIG Compliance on Red Hat

_This course is not approved or sponsored by Red Hat._

## Introduction

The Red Hat Linux audit service comes with precompiled rule sets for various compliance requirements. In this lab, we will configure a Red Hat host's audit rules to include the STIG (Security Technical Implementation Guide) compliance rule set. This will allow us to identify any points at which we are not compliant with STIG requirements.

## Solution

1. Begin by logging in to the lab server using the credentials provided on the hands-on lab page:

```
ssh cloud_user@PUBLIC_IP_ADDRESS
```

2. Become the `root` user:

```
sudo su
```

### Implement the Red Hat included STIG audit rules

1. Make a backup of the current audit rules using the following command:

```
cp /etc/audit/rules.d/audit.rules /etc/audit/rules.d/audit.rules_backup
```

2. Copy the STIG audit rules into the `audit.rules` file with the following command:

```
cd /usr/share/doc/audit-2.8.1/rules
```

```
cat 30-stig.rules 99-finalize.rules >> /etc/audit/rules.d/audit.rules
```

### Restart the `auditd` service

1. To restart the `auditd` service, use the following command:

```
service auditd restart  
```

2. Run the following command to verify the status is active (running):

```
service auditd status
```

## Conclusion

Congratulations — you've completed this hands-on lab!


my output
```
  13/05/2026   17:03.50   /home/mobaxterm  ssh cloud_user@52.87.246.170
Warning: Permanently added '52.87.246.170' (ED25519) to the list of known hosts.
cloud_user@52.87.246.170's password:
cloud_user@52.87.246.170's password:
X11 forwarding request failed on channel 0
[cloud_user@ip-10-0-1-77 ~]$ clear
[cloud_user@ip-10-0-1-77 ~]$ cp /etc/audit/rules.d/audit.rules /etc/audit/rules.d/audit.rules_backup
cp: failed to access ‘/etc/audit/rules.d/audit.rules_backup’: Permission denied
[cloud_user@ip-10-0-1-77 ~]$ sudo cp /etc/audit/rules.d/audit.rules /etc/audit/rules.d/audit.rules_backup
[sudo] password for cloud_user:
[cloud_user@ip-10-0-1-77 ~]$ cd /usr/share/doc/audit-2.8.1/rules
-bash: cd: /usr/share/doc/audit-2.8.1/rules: No such file or directory
[cloud_user@ip-10-0-1-77 ~]$ cd /usr/share/doc/au
audit-2.8.5/      authconfig-6.2.8/
[cloud_user@ip-10-0-1-77 ~]$ cd /usr/share/doc/au
audit-2.8.5/      authconfig-6.2.8/
[cloud_user@ip-10-0-1-77 ~]$ cd /usr/share/doc/au
audit-2.8.5/      authconfig-6.2.8/
[cloud_user@ip-10-0-1-77 ~]$ cd /usr/share/doc/au
audit-2.8.5/      authconfig-6.2.8/
[cloud_user@ip-10-0-1-77 ~]$ cd /usr/share/doc/audit-2.8.5/rules/
[cloud_user@ip-10-0-1-77 rules]$ cat 30-stig.rules 99-finalize.rules >> /etc/audit/rules.d/audit.rules
-bash: /etc/audit/rules.d/audit.rules: Permission denied
[cloud_user@ip-10-0-1-77 rules]$ sudo cat 30-stig.rules 99-finalize.rules >> /etc/audit/rules.d/audit.rules
-bash: /etc/audit/rules.d/audit.rules: Permission denied
[cloud_user@ip-10-0-1-77 rules]$ sudo -i
[root@ip-10-0-1-77 ~]# pwd
/root
[root@ip-10-0-1-77 ~]# cd /usr/share/doc/audit-2.8.5/rules/
[root@ip-10-0-1-77 rules]# cat 30-stig.rules 99-finalize.rules >> /etc/audit/rules.d/audit.rules
[root@ip-10-0-1-77 rules]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
[root@ip-10-0-1-77 rules]# service auditd status
Redirecting to /bin/systemctl status auditd.service
● auditd.service - Security Auditing Service
   Loaded: loaded (/usr/lib/systemd/system/auditd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2026-05-13 21:08:19 UTC; 8s ago
     Docs: man:auditd(8)
           https://github.com/linux-audit/audit-documentation
  Process: 1341 ExecStartPost=/sbin/augenrules --load (code=exited, status=0/SUCCESS)
  Process: 1336 ExecStart=/sbin/auditd (code=exited, status=0/SUCCESS)
 Main PID: 1337 (auditd)
   CGroup: /system.slice/auditd.service
           └─1337 /sbin/auditd

May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: lost 0
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: backlog 0
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: enabled 1
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: failure 1
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: pid 1337
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: rate_limit 0
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: backlog_limit 8192
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: lost 0
May 13 21:08:19 ip-10-0-1-77.ec2.internal augenrules[1341]: backlog 1
May 13 21:08:19 ip-10-0-1-77.ec2.internal systemd[1]: Started Security Auditing Service.
[root@ip-10-0-1-77 rules]#
```
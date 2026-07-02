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
title: Fixing an SELinux Port Denial for Apache
---
## Introduction

Security Enhanced Linux (SELinux) is a security architecture that adds expanded access controls to applications, processes, and files on a Linux system. While it greatly improves security, it is also a common source of issues - especially when services run on non-standard ports or deviate from default configurations. In this lab, I work through diagnosing and resolving an SELinux-related failure using RHEL 8 tools such as `ausearch`, `sealert`, and `semanage`.

**Environment:** RHEL 8

---

## Reproducing the Issue

The first step is to confirm the problem exists and understand what is failing. I check the status of `httpd`, attempt to start it, and observe the error.

```bash
systemctl status httpd
systemctl start httpd
systemctl status httpd.service
```

**Example output:**

```
🔴 httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2026-05-20 15:21:48 UTC; 5min ago

May 20 15:21:48 app httpd[4166]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:3333
May 20 15:21:48 app httpd[4166]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:3333
May 20 15:21:48 app httpd[4166]: no listening sockets available, shutting down
May 20 15:21:48 app systemd[1]: Failed to start The Apache HTTP Server.
```

The `Permission denied` errors on port 3333 are the key clue. `httpd` is configured to listen on a non-standard port and something is blocking it.

---

## Discovering the Issue

### Ruling Out the Firewall

Before blaming SELinux, I check whether the firewall is the culprit.

```bash
systemctl status firewalld
```

**Example output:**

```
Unit firewalld.service could not be found.
```

The firewall is not running, so it is not the cause. This points to SELinux as the likely source of the problem.

### Confirming SELinux is the Cause

To confirm, I temporarily put SELinux into permissive mode (disabled enforcement) and try starting `httpd` again.

```bash
setenforce 0
systemctl start httpd
systemctl status httpd.service
```

**Example output:**

```
🟢 httpd.service - The Apache HTTP Server
   Active: active (running) since Wed 2026-05-20 15:29:04 UTC; 8s ago
   Status: "Started, listening on: port 3333"
```

`httpd` starts successfully with SELinux in permissive mode, confirming SELinux enforcement is what was blocking it. I stop the service and re-enable enforcement before proceeding to a proper fix.

```bash
systemctl stop httpd
setenforce 1
```

### Reading the SELinux Audit Logs

With the cause confirmed, I use `ausearch` to query recent SELinux denials from the audit log.

```bash
ausearch -m avc -ts recent
```
%%[[ausearch (Audit Search)|ausearch]]%%
**Example output:**

```
type=AVC msg=audit(1779290508.001:246): avc:  denied  { name_bind } for  pid=4166 comm="httpd" src=3333
scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

The log shows SELinux denied `httpd` the ability to bind to port 3333 because that port is classified as `unreserved_port_t`, a type that `httpd` is not allowed to use.

### Reviewing Full Alert Details

For a more detailed, human-readable analysis, I use `sealert` to parse the full audit log.

```bash
sealert -a /var/log/audit/audit.log
```
%%[[sealert]]%%
**Example output (relevant alert):**

```
SELinux is preventing /usr/sbin/httpd from name_bind access on the tcp_socket port 3333.

*****  Plugin bind_ports (92.2 confidence) suggests  ************************

If you want to allow /usr/sbin/httpd to bind to network port 3333
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 3333
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, ...
```

`sealert` identifies the exact problem and suggests the correct fix: use `semanage` to assign port 3333 the `http_port_t` type, which `httpd` is allowed to bind to.

I also check the currently allowed HTTP ports to understand what is already in the policy.

```bash
semanage port -l | grep http
```

**Example output:**

```
http_cache_port_t    tcp    8080, 8118, 8123, 10001-10010
http_port_t          tcp    80, 81, 443, 488, 8008, 8009, 8443, 9000
```

Port 3333 is not listed, which is why SELinux blocked the bind attempt.

---

## Resolving the Issue

The fix is to add port 3333 to the `http_port_t` type using `semanage`, which updates the SELinux policy to allow `httpd` to bind to it.

```bash
semanage port -a -t http_port_t -p tcp 3333
```

I verify the change took effect.

```bash
semanage port -l | grep http
```

**Example output:**

```
http_port_t    tcp    3333, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

Port 3333 now appears under `http_port_t`. I start `httpd` and confirm it is running.

```bash
systemctl start httpd
systemctl status httpd
```

**Example output:**

```
🟢 httpd.service - The Apache HTTP Server
   Active: active (running) since Wed 2026-05-20 15:34:37 UTC; 4s ago
   Status: "Started, listening on: port 3333"

May 20 15:34:37 app systemd[1]: Started The Apache HTTP Server.
May 20 15:34:37 app httpd[4771]: Server configured, listening on: port 3333
```

`httpd` is running successfully on port 3333 with SELinux fully enforcing.

---

## Conclusion

This lab demonstrated a common real-world SELinux scenario: a service fails to start because it is configured to use a port that SELinux does not recognize as valid for that service type. The key steps were:

- Using `systemctl` to identify the failure
- Ruling out the firewall with `systemctl status firewalld`
- Temporarily disabling SELinux enforcement with `setenforce 0` to confirm it was the cause
- Using `ausearch` and `sealert` to read and interpret the audit logs
- Applying a permanent policy fix with `semanage port -a -t http_port_t -p tcp 3333`

Rather than disabling SELinux or switching to permissive mode permanently - which would remove the security protections it provides,  the correct approach is to extend the policy to accommodate the legitimate configuration change.
<!--
[Resolving SELinux Issues](https://app.pluralsight.com/hands-on/labs/40c90b41-bcd0-47b2-a141-6e508cfbb3a2?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)


Cloud Server - RHEL 8 MySQL Client
 
Username
cloud_user

Password
Q3yVCY *-

Private IP - RHEL 8 MySQL Client
10.0.1.101

Public IP - RHEL 8 MySQL Client
32.195.62.205


# Resolving SELinux Issues

## Introduction

Security Enhanced Linux is a security architecture that adds expanded security controls to applications, processes, and files. It is also the cause of many software issues, especially when using uncommon configurations or deviating too much from the default. Luckily, there are a number of tools for viewing, changing, and otherwise managing our SELinux profile. This hands-on lab tests on these tools. _This course is not approved or sponsored by Red Hat._

## Solution

1. Connect to both the `app` and `db` server using the credentials provided:
    
    ```
    ssh cloud_user@<PUBLIC_IP_ADDRESS>
    ```
    
2. Become the root user, entering the provided password when prompted:
    
    ```
    sudo -i
    ```
    

### Reproduce the Issue

1. Check the status of `httpd`:
    
    ```
    systemctl status httpd
    ```
    
2. Try to start `httpd`:
    
    ```
    systemctl start httpd
    ```
    
3. Run the status again:
    
    ```
    systemctl status httpd.service
    ```
    
    > **Note:** Use `CTRL + C` to exit.

```
[cloud_user@app ~]$ sudo -i
[sudo] password for cloud_user:
[root@app ~]# clear
[root@app ~]#     systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2026-05-20 15:21:48 UTC; 5min ago
     Docs: man:httpd.service(8)
  Process: 4166 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 4166 (code=exited, status=1/FAILURE)
   Status: "Reading configuration..."

May 20 15:21:47 app systemd[1]: Starting The Apache HTTP Server...
May 20 15:21:48 app httpd[4166]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using fe80::55>
May 20 15:21:48 app httpd[4166]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:3333
May 20 15:21:48 app httpd[4166]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:3333
May 20 15:21:48 app httpd[4166]: no listening sockets available, shutting down
May 20 15:21:48 app httpd[4166]: AH00015: Unable to open logs
May 20 15:21:48 app systemd[1]: httpd.service: Main process exited, code=exited, status=1/FAILURE
May 20 15:21:48 app systemd[1]: httpd.service: Failed with result 'exit-code'.
May 20 15:21:48 app systemd[1]: Failed to start The Apache HTTP Server.

[root@app ~]#     systemctl start httpd
Job for httpd.service failed because the control process exited with error code.
See "systemctl status httpd.service" and "journalctl -xe" for details.
[root@app ~]#     systemctl status httpd.service
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2026-05-20 15:27:31 UTC; 6s ago
     Docs: man:httpd.service(8)
  Process: 4292 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 4292 (code=exited, status=1/FAILURE)
   Status: "Reading configuration..."

May 20 15:27:31 app systemd[1]: Starting The Apache HTTP Server...
May 20 15:27:31 app httpd[4292]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using fe80::55>
May 20 15:27:31 app httpd[4292]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:3333
May 20 15:27:31 app httpd[4292]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:3333
May 20 15:27:31 app httpd[4292]: no listening sockets available, shutting down
May 20 15:27:31 app httpd[4292]: AH00015: Unable to open logs
May 20 15:27:31 app systemd[1]: httpd.service: Main process exited, code=exited, status=1/FAILURE
May 20 15:27:31 app systemd[1]: httpd.service: Failed with result 'exit-code'.
May 20 15:27:31 app systemd[1]: Failed to start The Apache HTTP Server.
...skipping...
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2026-05-20 15:27:31 UTC; 6s ago
     Docs: man:httpd.service(8)
  Process: 4292 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 4292 (code=exited, status=1/FAILURE)
   Status: "Reading configuration..."

May 20 15:27:31 app systemd[1]: Starting The Apache HTTP Server...
May 20 15:27:31 app httpd[4292]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using fe80::55>
May 20 15:27:31 app httpd[4292]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:3333
May 20 15:27:31 app httpd[4292]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:3333
May 20 15:27:31 app httpd[4292]: no listening sockets available, shutting down
May 20 15:27:31 app httpd[4292]: AH00015: Unable to open logs
May 20 15:27:31 app systemd[1]: httpd.service: Main process exited, code=exited, status=1/FAILURE
May 20 15:27:31 app systemd[1]: httpd.service: Failed with result 'exit-code'.
May 20 15:27:31 app systemd[1]: Failed to start The Apache HTTP Server.
```

1. Check if the issue is with the firewall:
    
    ```
    systemctl status firewalld
    ```
    
2. Check if the issue is with SELinux by disabling it:
    
    ```
    setenforce 0
    ```
    
3. Restart `httpd`:
    
    ```
    systemctl start httpd
    ```
    
4. Check the status of `httpd`:
    
    ```
    systemctl status httpd.service
    ```
    
    > **Note:** Use `CTRL + C` to exit.
    
5. Stop `httpd`:
    
    ```
    systemctl stop httpd
    ```
    
6. Enable SELinux:
    
    ```
    setenforce 1
    ```
    
```
[root@app ~]#     systemctl status firewalld
Unit firewalld.service could not be found.
[root@app ~]#     setenforce 0
[root@app ~]#     systemctl start httpd
[root@app ~]#     systemctl status httpd.service
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2026-05-20 15:29:04 UTC; 8s ago
     Docs: man:httpd.service(8)
 Main PID: 4344 (httpd)
   Status: "Started, listening on: port 3333"
    Tasks: 213 (limit: 10564)
   Memory: 27.6M
   CGroup: /system.slice/httpd.service
           ├─4344 /usr/sbin/httpd -DFOREGROUND
           ├─4346 /usr/sbin/httpd -DFOREGROUND
           ├─4347 /usr/sbin/httpd -DFOREGROUND
           ├─4348 /usr/sbin/httpd -DFOREGROUND
           └─4349 /usr/sbin/httpd -DFOREGROUND

May 20 15:29:04 app systemd[1]: Starting The Apache HTTP Server...
May 20 15:29:04 app httpd[4344]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using fe80::55>
May 20 15:29:04 app systemd[1]: Started The Apache HTTP Server.
May 20 15:29:04 app httpd[4344]: Server configured, listening on: port 3333

[root@app ~]#     systemctl stop httpd
[root@app ~]#     setenforce 1
[root@app ~]#

```
### Discover Issue

1. Review SELinux-related logs:
    
    ```
    ausearch -m avc -ts recent
    ```
    
2. Review the audit logs and scroll up to the `httpd` section once it is loaded:
    
    ```
    sealert -a /var/log/audit/audit.log
    ```

```
[root@app ~]#     ausearch -m avc -ts recent
----
time->Wed May 20 15:21:48 2026
type=PROCTITLE msg=audit(1779290508.001:246): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SYSCALL msg=audit(1779290508.001:246): arch=c000003e syscall=49 success=no exit=-13 a0=4 a1=559d8b79e980 a2=1c a3=7ffe0d913b0c items=0 ppid=1 pid=4166 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1779290508.001:246): avc:  denied  { name_bind } for  pid=4166 comm="httpd" src=3333 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
----
time->Wed May 20 15:21:48 2026
type=PROCTITLE msg=audit(1779290508.001:247): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SYSCALL msg=audit(1779290508.001:247): arch=c000003e syscall=49 success=no exit=-13 a0=3 a1=559d8b79e8c0 a2=10 a3=7ffe0d913afc items=0 ppid=1 pid=4166 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1779290508.001:247): avc:  denied  { name_bind } for  pid=4166 comm="httpd" src=3333 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
----
time->Wed May 20 15:27:31 2026
type=PROCTITLE msg=audit(1779290851.300:260): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SYSCALL msg=audit(1779290851.300:260): arch=c000003e syscall=49 success=no exit=-13 a0=4 a1=56153fc01980 a2=1c a3=7ffcc71a644c items=0 ppid=1 pid=4292 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1779290851.300:260): avc:  denied  { name_bind } for  pid=4292 comm="httpd" src=3333 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
----
time->Wed May 20 15:27:31 2026
type=PROCTITLE msg=audit(1779290851.300:261): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SYSCALL msg=audit(1779290851.300:261): arch=c000003e syscall=49 success=no exit=-13 a0=3 a1=56153fc018c0 a2=10 a3=7ffcc71a643c items=0 ppid=1 pid=4292 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1779290851.300:261): avc:  denied  { name_bind } for  pid=4292 comm="httpd" src=3333 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
----
time->Wed May 20 15:29:04 2026
type=PROCTITLE msg=audit(1779290944.277:267): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SYSCALL msg=audit(1779290944.277:267): arch=c000003e syscall=49 success=yes exit=0 a0=4 a1=55a61b58e980 a2=1c a3=7ffd6a4a0e4c items=0 ppid=1 pid=4344 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1779290944.277:267): avc:  denied  { name_bind } for  pid=4344 comm="httpd" src=3333 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=1
[root@app ~]#     sealert -a /var/log/audit/audit.log
  9% donetype=AVC msg=audit(1557344322.484:186): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

type=AVC msg=audit(1557344322.484:187): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

type=AVC msg=audit(1557344322.484:188): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

 10% donetype=AVC msg=audit(1557344472.375:219): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

type=AVC msg=audit(1557344472.375:220): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

type=AVC msg=audit(1557344472.375:221): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

 10% donetype=AVC msg=audit(1557344613.147:240): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

type=AVC msg=audit(1557344613.147:241): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

type=AVC msg=audit(1557344613.147:242): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0

**** Recorded AVC is allowed in current policy ****

 10% donetype=AVC msg=audit(1557345001.342:287): avc:  denied  { read } for  pid=1 comm="systemd" name="ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=1

**** Recorded AVC is allowed in current policy ****

type=AVC msg=audit(1557345001.342:288): avc:  denied  { open } for  pid=1 comm="systemd" path="/home/cloud_user/.vnc/ip-172-31-27-101.us-east-2.compute.internal:1.pid" dev="nvme0n1p2" ino=10155875 scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=1

**** Recorded AVC is allowed in current policy ****

 40% donetype=AVC msg=audit(1657213956.904:822): avc:  denied  { getattr } for  pid=8144 comm="groupadd" name="/" dev="proc" ino=1 scontext=unconfined_u:unconfined_r:groupadd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:proc_t:s0 tclass=filesystem permissive=0

**** Recorded AVC is allowed in current policy ****

100% done
found 4 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/libexec/geoclue from search access on the directory 6028.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that geoclue should be allowed search access on the 6028 directory by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'geoclue' --raw | audit2allow -M my-geoclue
# semodule -X 300 -i my-geoclue.pp


Additional Information:
Source Context                system_u:system_r:geoclue_t:s0
Target Context                system_u:system_r:unconfined_service_t:s0
Target Objects                6028 [ dir ]
Source                        geoclue
Source Path                   /usr/libexec/geoclue
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           geoclue2-2.5.5-2.el8.x86_64
Target RPM Packages
SELinux Policy RPM            selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Local Policy RPM              selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     app
Platform                      Linux app 4.18.0-553.120.1.el8_10.x86_64 #1 SMP
                              Wed Apr 8 22:26:17 EDT 2026 x86_64 x86_64
Alert Count                   10
First Seen                    2019-05-08 19:38:46 UTC
Last Seen                     2019-05-09 22:07:11 UTC
Local ID                      dc480fc7-cddd-4f5d-a608-7c03691592b1

Raw Audit Messages
type=AVC msg=audit(1557439631.18:74): avc:  denied  { search } for  pid=6281 comm="geoclue" name="6028" dev="proc" ino=38340 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=dir permissive=0


type=SYSCALL msg=audit(1557439631.18:74): arch=x86_64 syscall=openat success=no exit=EACCES a0=ffffff9c a1=55e07595fad0 a2=0 a3=0 items=0 ppid=1 pid=6281 auid=4294967295 uid=991 gid=985 euid=991 suid=991 fsuid=991 egid=985 sgid=985 fsgid=985 tty=(none) ses=4294967295 comm=geoclue exe=/usr/libexec/geoclue subj=system_u:system_r:geoclue_t:s0 key=(null)ARCH=x86_64 SYSCALL=openat AUID=unset UID=geoclue GID=geoclue EUID=geoclue SUID=geoclue FSUID=geoclue EGID=geoclue SGID=geoclue FSGID=geoclue

Hash: geoclue,geoclue_t,unconfined_service_t,dir,search

--------------------------------------------------------------------------------

SELinux is preventing /usr/libexec/geoclue from getattr access on the file /proc/<pid>/cgroup.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that geoclue should be allowed getattr access on the cgroup file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'geoclue' --raw | audit2allow -M my-geoclue
# semodule -X 300 -i my-geoclue.pp


Additional Information:
Source Context                system_u:system_r:geoclue_t:s0
Target Context                system_u:system_r:unconfined_service_t:s0
Target Objects                /proc/<pid>/cgroup [ file ]
Source                        geoclue
Source Path                   /usr/libexec/geoclue
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           geoclue2-2.5.5-2.el8.x86_64
Target RPM Packages
SELinux Policy RPM            selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Local Policy RPM              selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     app
Platform                      Linux app 4.18.0-553.120.1.el8_10.x86_64 #1 SMP
                              Wed Apr 8 22:26:17 EDT 2026 x86_64 x86_64
Alert Count                   1
First Seen                    2019-05-08 19:50:06 UTC
Last Seen                     2019-05-08 19:50:06 UTC
Local ID                      4b8057e5-e5cc-4041-aa1a-1cf310b9d9be

Raw Audit Messages
type=AVC msg=audit(1557345006.213:298): avc:  denied  { getattr } for  pid=9638 comm="geoclue" path="/proc/9560/cgroup" dev="proc" ino=66760 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=file permissive=1


type=SYSCALL msg=audit(1557345006.213:298): arch=x86_64 syscall=fstat success=yes exit=0 a0=9 a1=7fff7aa8a780 a2=7fff7aa8a780 a3=0 items=0 ppid=1 pid=9638 auid=4294967295 uid=991 gid=985 euid=991 suid=991 fsuid=991 egid=985 sgid=985 fsgid=985 tty=(none) ses=4294967295 comm=geoclue exe=/usr/libexec/geoclue subj=system_u:system_r:geoclue_t:s0 key=(null)ARCH=x86_64 SYSCALL=fstat AUID=unset UID=geoclue GID=geoclue EUID=geoclue SUID=geoclue FSUID=geoclue EGID=geoclue SGID=geoclue FSGID=geoclue

Hash: geoclue,geoclue_t,unconfined_service_t,file,getattr

--------------------------------------------------------------------------------

SELinux is preventing /usr/bin/login from using the transition access on a process.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that login should be allowed transition access on processes labeled unconfined_t by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'login' --raw | audit2allow -M my-login
# semodule -X 300 -i my-login.pp


Additional Information:
Source Context                system_u:system_r:unconfined_service_t:s0
Target Context                unconfined_u:unconfined_r:unconfined_t:s0
Target Objects                /usr/bin/bash [ process ]
Source                        login
Source Path                   /usr/bin/login
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           util-linux-2.32.1-48.el8_10.x86_64
Target RPM Packages           bash-4.4.20-6.el8_10.x86_64
SELinux Policy RPM            selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Local Policy RPM              selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     app
Platform                      Linux app 4.18.0-553.120.1.el8_10.x86_64 #1 SMP
                              Wed Apr 8 22:26:17 EDT 2026 x86_64 x86_64
Alert Count                   5
First Seen                    2019-05-08 20:18:18 UTC
Last Seen                     2019-05-08 20:24:35 UTC
Local ID                      61772535-4b37-4ee9-b2d9-8d8380805b47

Raw Audit Messages
type=AVC msg=audit(1557347075.257:311): avc:  denied  { transition } for  pid=8205 comm="login" path="/usr/bin/bash" dev="xvda2" ino=4195894 scontext=system_u:system_r:unconfined_service_t:s0 tcontext=unconfined_u:unconfined_r:unconfined_t:s0 tclass=process permissive=0


type=SYSCALL msg=audit(1557347075.257:311): arch=x86_64 syscall=execve success=no exit=EACCES a0=55aa11c0f0ef a1=7ffeda716be8 a2=55aa11c1de90 a3=d items=0 ppid=8200 pid=8205 auid=1001 uid=1001 gid=1001 euid=1001 suid=1001 fsuid=1001 egid=1001 sgid=1001 fsgid=1001 tty=pts1 ses=10 comm=login exe=/usr/bin/login subj=system_u:system_r:unconfined_service_t:s0 key=(null)ARCH=x86_64 SYSCALL=execve AUID=cloud_user UID=cloud_user GID=cloud_user EUID=cloud_user SUID=cloud_user FSUID=cloud_user EGID=cloud_user SGID=cloud_user FSGID=cloud_user

Hash: login,unconfined_service_t,unconfined_t,process,transition

--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/httpd from name_bind access on the tcp_socket port 3333.

*****  Plugin bind_ports (92.2 confidence) suggests   ************************

If you want to allow /usr/sbin/httpd to bind to network port 3333
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 3333
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Plugin catchall_boolean (7.83 confidence) suggests   ******************

If you want to allow system to run with NIS
Then you must tell SELinux about this by enabling the 'nis_enabled' boolean.

Do
setsebool -P nis_enabled 1

*****  Plugin catchall (1.41 confidence) suggests   **************************

If you believe that httpd should be allowed name_bind access on the port 3333 tcp_socket by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'httpd' --raw | audit2allow -M my-httpd
# semodule -X 300 -i my-httpd.pp


Additional Information:
Source Context                system_u:system_r:httpd_t:s0
Target Context                system_u:object_r:unreserved_port_t:s0
Target Objects                port 3333 [ tcp_socket ]
Source                        httpd
Source Path                   /usr/sbin/httpd
Port                          3333
Host                          <Unknown>
Source RPM Packages           httpd-2.4.37-65.module+el8.10.0+23815+1b5e1c66.7.x
                              86_64
Target RPM Packages
SELinux Policy RPM            selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Local Policy RPM              selinux-policy-targeted-3.14.3-139.el8_10.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     app
Platform                      Linux app 4.18.0-553.120.1.el8_10.x86_64 #1 SMP
                              Wed Apr 8 22:26:17 EDT 2026 x86_64 x86_64
Alert Count                   7
First Seen                    2026-05-20 15:20:14 UTC
Last Seen                     2026-05-20 15:29:04 UTC
Local ID                      e1529664-3468-44e8-bc54-f1355de9714f

Raw Audit Messages
type=AVC msg=audit(1779290944.277:267): avc:  denied  { name_bind } for  pid=4344 comm="httpd" src=3333 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=1


type=SYSCALL msg=audit(1779290944.277:267): arch=x86_64 syscall=bind success=yes exit=0 a0=4 a1=55a61b58e980 a2=1c a3=7ffd6a4a0e4c items=0 ppid=1 pid=4344 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=httpd exe=/usr/sbin/httpd subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID=unset UID=root GID=root EUID=root SUID=root FSUID=root EGID=root SGID=root FSGID=root

Hash: httpd,httpd_t,unreserved_port_t,tcp_socket,name_bind

[root@app ~]#
```


1. Verify `http` ports:
    
    ```
    semanage port -l | grep http
    ```
    
```
[root@app ~]#     semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@app ~]#

```
### Resolve Issue

1. Use `semange` to find more information about adding ports:
    
    ```
    semanage port --help
    ```
    
2. Add port:
    
    ```
    semanage port -a -t http_port_t -p tcp 3333
    ```
    
3. Verify if port 3333 is listed:
    
    ```
    semanage port -l | grep http
    ```
    
4. Start `httpd`:
    
    ```
    systemctl start httpd
    ```
    
5. Check the status:
    
    ```
    systemctl status httpd
    ```
    
```
[root@app ~]#     semanage port -a -t http_port_t -p tcp 3333
[root@app ~]#     semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      3333, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@app ~]#     systemctl start httpd
[root@app ~]#     systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2026-05-20 15:34:37 UTC; 4s ago
     Docs: man:httpd.service(8)
 Main PID: 4771 (httpd)
   Status: "Started, listening on: port 3333"
    Tasks: 213 (limit: 10564)
   Memory: 25.6M
   CGroup: /system.slice/httpd.service
           ├─4771 /usr/sbin/httpd -DFOREGROUND
           ├─4772 /usr/sbin/httpd -DFOREGROUND
           ├─4773 /usr/sbin/httpd -DFOREGROUND
           ├─4774 /usr/sbin/httpd -DFOREGROUND
           └─4775 /usr/sbin/httpd -DFOREGROUND

May 20 15:34:37 app systemd[1]: Starting The Apache HTTP Server...
May 20 15:34:37 app httpd[4771]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using fe80::55>
May 20 15:34:37 app systemd[1]: Started The Apache HTTP Server.
May 20 15:34:37 app httpd[4771]: Server configured, listening on: port 3333
lines 1-19/19 (END)

```
## Conclusion

Congratulations — you've completed this hands-on lab!
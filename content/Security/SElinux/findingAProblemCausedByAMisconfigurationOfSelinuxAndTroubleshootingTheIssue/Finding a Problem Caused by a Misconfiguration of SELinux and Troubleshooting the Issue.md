---
tags:
  - selinux
linkedin: "False"
quartz: "True"
refactored: "False"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
title: Fixing SELinux Port and File Context Violations on Apache
---
## Introduction

In this hands-on lab, I worked through a scenario where SELinux was misconfigured and actively blocking a web server from functioning correctly. The goal was to diagnose the problem and apply fixes that would persist across reboots, without ever falling back on disabling SELinux entirely (no setting it to permissive mode as a "fix").

There were two issues to solve:

1. Apache needed to run on a non-standard port (9100) that SELinux wasn't allowing.
2. Once the server was reachable, it still wasn't serving content properly to users due to an incorrect file context.

The point of the exercise was to practice real SELinux troubleshooting: reading audit logs, identifying the exact policy or context causing a denial, and applying the correct, permanent fix rather than a workaround.

### Installing Troubleshooting Tools

Before diagnosing anything, I installed the SELinux troubleshooting utilities, which provide tools like `audit2why` and `audit2allow` that translate raw audit log entries into human-readable explanations.

```bash
sudo yum install -y setroubleshoot setools
```

I then checked the `auditd` service configuration to understand how audit logging was running on this system:

```bash
sudo systemctl cat auditd
```
%%[[systemctl#Cat config]]%%

> [!NOTE]- auditd config
> ```bash
> [root@ip-10-0-1-138 ~]# sudo systemctl cat auditd
> # /usr/lib/systemd/system/auditd.service
> [Unit]
> Description=Security Auditing Service
> DefaultDependencies=no
> ## If auditd is sending or recieving remote logging, copy this file to
> ## /etc/systemd/system/auditd.service and comment out the first After and
> ## uncomment the second so that network-online.target is part of After.
> ## then comment the first Before and uncomment the second Before to remove
> ## sysinit.target from "Before".
> After=local-fs.target systemd-tmpfiles-setup.service
> ##After=network-online.target local-fs.target systemd-tmpfiles-setup.service
> Before=sysinit.target shutdown.target
> ##Before=shutdown.target
> Conflicts=shutdown.target
> RefuseManualStop=yes
> ConditionKernelCommandLine=!audit=0
> Documentation=man:auditd(8) https://github.com/linux-audit/audit-documentation
> 
> [Service]
> Type=forking
> PIDFile=/run/auditd.pid
> ExecStart=/sbin/auditd
> ## To not use augenrules, copy this file to /etc/systemd/system/auditd.service
> ## and comment/delete the next line and uncomment the auditctl line.
> ## NOTE: augenrules expect any rules to be added to /etc/audit/rules.d/
> ExecStartPost=-/sbin/augenrules --load
> #ExecStartPost=-/sbin/auditctl -R /etc/audit/audit.rules
> # By default we don't clear the rules on exit. To enable this, uncomment
> # the next line after copying the file to /etc/systemd/system/auditd.service
> #ExecStopPost=/sbin/auditctl -R /etc/audit/audit-stop.rules
> 
> [Install]
> WantedBy=multi-user.target
> 
> [root@ip-10-0-1-138 ~]#
> ```

> [!attention] `auditd` Restart Needed
> When we installed `setroubleshoot`, it also installs an **auditd plugin** (`sedispatch`) that hooks into the audit system and translates AVC denials into human-readable messages. For auditd to pick that plugin up, it needs to be restarted so it re-reads its plugin directory and loads the new dispatcher.

#### Troubleshooting: `auditd` Won't Restart Cleanly

By default, `auditd.service` has `RefuseManualStop=yes` set, which prevents it from being manually restarted, even by root. Since I needed to restart the service after making changes, I temporarily disabled this protection:

```bash
sudo vim /usr/lib/systemd/system/auditd.service
```

Set to:

```
RefuseManualStop=no
```

Then reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart auditd
```

Once the restart succeeded, I reverted the setting back to `yes` to restore the original protection, since leaving it disabled isn't good practice outside of active troubleshooting.

### Attempting to Start Apache on Port 9100

The first task was to get Apache listening on port 9100 instead of the default port 80.

```bash
sudo vim /etc/httpd/conf/httpd.conf
```

I commented out the default listener and added the new port:

```
#Listen 80
Listen 9100
```

Then I tried starting the service:

```bash
sudo systemctl start httpd
```

```
Job for httpd.service failed because the control process exited with error code. See "systemctl status httpd.service" and "journalctl -xe" for details.
```

This failed immediately. Checking the service status confirmed a permissions-related failure:

```bash
sudo systemctl status httpd -l
```

^932d3c

```
Job for httpd.service failed because the control process exited with error code. See "systemctl status httpd.service" and "journalctl -xe" for details.
[root@ip-10-0-1-138 ~]# sudo systemctl status httpd -l
🔴 httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2026-06-25 16:15:11 EDT; 11s ago
     Docs: man:httpd(8)
           man:apachectl(8)
  Process: 27442 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 27442 (code=exited, status=1/FAILURE)

Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: Starting The Apache HTTP Server...
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:9100
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:9100
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: no listening sockets available, shutting down
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: AH00015: Unable to open logs
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: httpd.service: main process exited, code=exited, status=1/FAILURE
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: Failed to start The Apache HTTP Server.
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: Unit httpd.service entered failed state.
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: httpd.service failed.
```

^systemctlStatusHttpd-l

%%
Look at the lines right before it in your log:

```
(13)Permission denied: AH00072: make_sock: could not bind to address [::]:9100
(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:9100
no listening sockets available, shutting down
```

Apache tries to create a listening socket for every address/port combo in its config — here that's the IPv6 wildcard (`[::]:9100`) and the IPv4 wildcard (`0.0.0.0:9100`). Both attempts got `Permission denied`, because SELinux's `httpd_t` domain wasn't allowed to bind to port 9100 (it was labeled `hplip_port_t` at that point, not `http_port_t`).

So: Apache tried to open 2 sockets → both got blocked by SELinux → it ended up with **zero working sockets** → and a server with no sockets to listen on has nothing to do, so it logs "no listening sockets available, shutting down" and exits.

The real cause is the `Permission denied` lines above it (which trace back to the SELinux `name_bind` denial you found in the audit log). "No listening sockets available" is just Apache's way of saying "I had nothing left to try, so I'm exiting" — it's the symptom, not the disease.
%%

%%
[[Linux File Types#Socket]]
t o bind to address does this create 2 files of type sockets
_________
Yes, exactly. When Apache sees `Listen 9100` without a specific address, it tries to bind to **two sockets**:

- `[::]:9100` — an IPv6 socket (listens on all IPv6 interfaces)
- `0.0.0.0:9100` — an IPv4 socket (listens on all IPv4 interfaces)

That's why you saw two separate "Permission denied" lines in the error output — SELinux denied the bind attempt twice, once per socket.

When you changed it to `Listen 0.0.0.0:9100`, you told Apache to only create the IPv4 socket, which is why it worked — one bind attempt, one approval from SELinux.

These aren't files on disk the way [[Linux File Types#Regular File|regular files]] are. They're **network sockets**, which in Linux are kernel objects. They do show up in the filesystem under `/proc` and you can see them with tools like `ss` or `netstat`, but they're not created in a directory you'd browse to. The "two files" mental model is close though — each socket is a distinct kernel resource with its own context that SELinux evaluates independently.
%%
#### Troubleshooting: Identifying SELinux as the Cause

To confirm SELinux was actually responsible (rather than a regular Linux permissions issue), I searched the audit log for relevant entries:

```bash
sudo grep httpd /var/log/audit/audit.log
```

This turned up an AVC (Access Vector Cache) denial:

```
type=AVC msg=audit(...): avc: denied { name_bind } for pid=27442 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0
```

This line confirmed the issue: SELinux's `httpd_t` domain was being denied permission to bind to port 9100, because port 9100 was labeled `hplip_port_t` (a port type meant for HP printer services), not a port type Apache is allowed to use.

To get a plain-English explanation of the denial, I piped the audit log through `audit2why`:

```bash
sudo grep httpd /var/log/audit/audit.log | audit2why
```

```bash
[root@ip-10-0-1-138 ~]#     sudo grep httpd /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1782418511.727:404): avc:  denied  { name_bind } for  pid=27442 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
type=AVC msg=audit(1782418511.727:405): avc:  denied  { name_bind } for  pid=27442 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
type=AVC msg=audit(1782419024.520:448): avc:  denied  { name_bind } for  pid=27547 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
type=AVC msg=audit(1782419024.521:449): avc:  denied  { name_bind } for  pid=27547 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
[root@ip-10-0-1-138 ~]#
```

> [!NOTE]
> `audit2why` doesn't always point us to the _best_ fix, it points you to _a_ fix, based on whatever SELinux booleans or rules exist that would allow the denied action. In this case, it found a broad boolean that happens to permit `httpd` to bind to non-standard ports, and suggested that. The problem is that enabling `httpd_enable_ftp_server` is a blunt instrument, it grants wider permissions than needed.
> 
> The real diagnostic information was already in the raw AVC line itself:
> ```
> avc: denied { name_bind } for pid=27442 comm="httpd" src=9100 tcontext=...hplip_port_t
> ```
> 
> - `name_bind` — Apache is trying to bind to a port
> - `src=9100` — the port in question
> - `tcontext=...hplip_port_t` — the port is labeled as an HP printer port, not an HTTP port
> 
> That tells you the real fix: relabel the port to `http_port_t` using `semanage`. The boolean suggestion from `audit2why` was noted and ignored in favor of the more targeted, correct approach.

%%
### **Why "blunt"?**

When I said blunt I meant it's an _all-or-nothing_ permission. Setting `httpd_enable_ftp_server 1` tells SELinux "let Apache bind to a wide range of non-standard ports" — it doesn't say "let Apache bind specifically to port 9100."
### Why is it called `httpd_enable_ftp_server`?

This one is genuinely confusing naming. The reason is historical — Apache (`httpd`) has a module called `mod_proxy_ftp` that lets it act as an FTP proxy, meaning Apache can sit in front of an FTP server and forward FTP traffic. FTP uses non-standard ports by nature, so this boolean was originally created to let Apache bind to those FTP-related ports.

Over time, SELinux's logic of "this boolean allows non-standard port binding for httpd" ended up making `audit2why` suggest it as a fix for _any_ non-standard port denial, even ones that have nothing to do with FTP. The name reflects the original use case, not what it actually ends up being suggested for.

So `audit2why` wasn't wrong exactly — it would have "worked" — but it would have also quietly granted Apache a lot more port access than you intended, which is exactly the kind of thing SELinux is supposed to prevent in the first place.%%

%%[[SELinux#Rules / Policy]] what is meant by rule
Lab used in [[Writing a Custom SELinux Policy]]%%

To go one step further and watch a denial happen in real time, I used a two-terminal setup. In the first terminal, I started following the audit log live:

```bash
sudo tail -f /var/log/audit/audit.log
```

With that running, I opened a second terminal, SSH'd into the same server, and from there attempted to start `httpd` again:

```bash
sudo systemctl start httpd
```

The moment `httpd` tried to bind to port 9100 and SELinux blocked it, the AVC denial appeared live in the first terminal. This made it clear exactly which action triggered the denial and when, without having to stop and grep the log after the fact.

This technique is especially useful when you're not sure _which_ action is causing a denial,running `tail -f` while you manually test things lets you catch the exact moment a denial fires, rather than hunting through a log after the fact.

*Truncated Output*
```
type=CRYPTO_SESSION msg=audit(...:425): ... op=start direction=from-server cipher=chacha20-poly1305@openssh.com ... addr=73.49.202.63 lport=22 exe="/usr/sbin/sshd" ...
type=CRYPTO_SESSION msg=audit(...:426): ... op=start direction=from-client cipher=chacha20-poly1305@openssh.com ... addr=73.49.202.63 lport=22 exe="/usr/sbin/sshd" ...
type=USER_AUTH msg=audit(...:427): ... op=pubkey acct="cloud_user" exe="/usr/sbin/sshd" ... res=failed
type=USER_AUTH msg=audit(...:428): ... op=PAM:authentication grantors=pam_unix acct="cloud_user" exe="/usr/sbin/sshd" ... res=success
type=AVC msg=audit(...:448): avc: denied { name_bind } for pid=27547 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0
type=SERVICE_START msg=audit(...:450): ... unit=httpd comm="systemd" ... res=failed
```


Notice the sequence: the `CRYPTO_SESSION` and `USER_AUTH` lines are the second terminal's SSH connection appearing in the log first, followed immediately by the AVC denial the moment `httpd` tried to bind to port 9100. This made it clear exactly which action triggered the denial and when, without having to stop and grep the log after the fact.
### Finding and Assigning the Correct Port Label

With the root cause identified, the next step was to correctly label port 9100 so SELinux would allow Apache to bind to it.

First, I listed the **existing port labels** related to HTTP to understand what was already in use:

```bash
sudo semanage port -l | grep -i http
```

```
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000 ⬅️
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989  
```

This showed that `http_port_t` already covered ports like 80, 81, 443, 8008, 8009, 8443, and 9000, but **not 9100**. 

I tried adding 9100 directly: %%adding a new mapping%%

```bash
sudo semanage port -a -t http_port_t -p tcp 9100
```
%%[[semanage (SELinux Management)#Options]]%%
*We're mapping a **port number** to an **SELinux type***

```
❌ValueError: Port tcp/9100 already defined
```
#### Troubleshooting: Port Already Defined Under a Different Label

This returned an error, since port 9100 was already assigned to `hplip_port_t`. SELinux won't let the same port belong to two labels at once, so instead of **adding a new mapping**, I had to modify the existing one to reassign it:

```bash
sudo semanage port -m -t http_port_t -p tcp 9100
```
%%[[semanage (SELinux Management)#Options|semanage]]%%
Verifying the change confirmed port 9100 now appeared under `http_port_t`:

```bash
sudo semanage port -l | grep -i http
```

```
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      9100, 80, 81, 443, 488, 8008, 8009, 8443, 9000  ⬅️
pegasus_http_port_t            tcp      5988
pegasus_https_port_t  
```
### Restarting Apache and Hitting the Next Roadblock

With the port correctly labeled, I tried loading the site in the browser at `<PUBLIC_IP_ADDRESS>:9100`. It didn't load.

![[Pasted image 20260625163344.png|600]]
#### Troubleshooting: Apache Still Not Running

Checking the service status showed Apache wasn't actually running:

```bash
sudo systemctl status httpd
```

```
🔴 httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2026-06-25 16:23:44 EDT; 10min ago
  Process: 27547 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 27547 (code=exited, status=1/FAILURE)

Jun 25 16:23:44 ip-10-0-1-138.ec2.internal httpd[27547]: (13)Permission denied: AH00072: make_sock: could not bind to address ...:9100
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal httpd[27547]: (13)Permission denied: AH00072: make_sock: could not bind to address ...:9100
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal httpd[27547]: no listening sockets available, shutting down
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal systemd[1]: Failed to start The Apache HTTP Server.
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal systemd[1]: httpd.service failed.
```

The error message looked identical to the [[#^systemctlStatusHttpd-l|earlier SELinux denial]]. To rule SELinux out, I checked the audit log again, no new AVC entries appeared this time, which confirmed SELinux was no longer blocking the bind. The issue was instead how the `Listen` directive was written...

Looking back at the config, `Listen 9100` tells Apache to bind to port 9100 on **all interfaces**, both IPv4 (`0.0.0.0`) and IPv6 (`[::]`). 

You can actually see this in the [[#^systemctlStatusHttpd-l|first failure output]]:
```
Permission denied: make_sock: could not bind to address [::]:9100
Permission denied: make_sock: could not bind to address 0.0.0.0:9100
```
%%
The status right above us which did not use `-l` truncates output, coud be missleading because i wont not have notice IPv4 an Ipv6 
...:9100
...:9100
%%
I updated it to explicitly bind to all interfaces:
```bash
sudo vim /etc/httpd/conf/httpd.conf
```

Change `Listen 9100` to `Listen 0.0.0.0:9100`

After saving and restarting, Apache started successfully:

```bash
sudo systemctl start httpd
sudo systemctl status httpd
```

Trying the browser again at `<PUBLIC_IP_ADDRESS>:9100` got further this time, but returned a "Forbidden" error instead of the expected page.

![[Pasted image 20260625163616.png]]
### Fixing the SELinux Context on `index.html`

A "Forbidden" response with the server otherwise running pointed to a file context issue rather than a port issue. I checked the security context of the web root:

```bash
sudo su
cd /var/www/html/
ls -lZ
```

```
-rw-r--r--. root root system_u:object_r:user_home_t:s0 index.html
```

This revealed that `index.html` had the context `user_home_t`, which is not a context Apache is permitted to read from. I created a test file in the same directory to confirm what the correct context should be. Since SELinux file contexts are path-based, the policy automatically labeled the new file with `httpd_sys_content_t` for anything under `/var/www/html/`, making it clear that `index.html` should have that same type.

```bash
touch test
ls -lZ
```

```
unconfined_u:object_r:httpd_sys_content_t:s0 test
```
#### Troubleshooting: Fixing the Mismatched Context Permanently

Simply changing the context wouldn't survive a relabel or restore operation, so I updated the persistent file context rule first, then applied it:

```bash
semanage fcontext -a -t httpd_sys_content_t /var/www/html/index.html
```
%%[[semanage (SELinux Management)#Options|semanage]],[[SELinux#^6143c4]]%%
> If `-a` fails because a rule for that path is already defined in the policy, run `-m` instead to modify the existing entry:
> 
> ```
> Sample
> ❌ValueError: Port tcp/9100 already defined
> ```
> 
> ```bash
> semanage fcontext -m -t httpd_sys_content_t /var/www/html/index.html
> ```

Then I applied the corrected context to the actual file:

```bash
restorecon -v /var/www/html/index.html
```
%%[[restorecon (Restore Context)#Options|restorecon]]%%
Output confirmed the relabel:

```
restorecon reset /var/www/html/index.html context system_u:object_r:user_home_t:s0->system_u:object_r:httpd_sys_content_t:s0
```
### Confirming the Fix

With both the port label and file context corrected, I loaded `<PUBLIC_IP_ADDRESS>:9100` one more time. This time, the page loaded successfully.


![[Pasted image 20260625163852.png|600]]
## Conclusion

This lab was a solid exercise in SELinux troubleshooting methodology: rather than disabling SELinux to "make the error go away," I worked through identifying the actual policy violations using `audit.log`, `audit2why`, and `semanage`, then applied targeted, persistent fixes for each one. The two issues, an incorrectly labeled port and an incorrect file context, are common, real-world SELinux problems, and resolving both reinforced how SELinux enforces security at a much more granular level than standard Linux permissions.

**Key takeaways:** 

- AVC denials in `/var/log/audit/audit.log` are the starting point for any SELinux investigation.
- `audit2why` translates cryptic denial messages into actionable causes.
- `semanage port` manages which ports a given SELinux type is allowed to use, port labels can't overlap, so reassigning (`-m`) is often necessary instead of adding (`-a`).
- `semanage fcontext` + `restorecon` is the correct, permanent way to fix file context issues, as opposed to using `chcon`, which doesn't persist through a relabel.

<!--

(https://app.pluralsight.com/hands-on/labs/22973e4f-fcc8-4eaa-867f-2b5b1d5b597c?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)


# Finding a Problem Caused by a Misconfiguration of SELinux and Troubleshooting the Issue

## Introduction

In this lab, you will be presented with inadequate SELinux configurations that are causing problems. Your job is to perform the correct reconfigurations so everything works properly for the given cases. The first problem involves a web server running that needs to be accessed through an atypical port not usually used by the web servers. SELinux, however, is not allowing you to do this. You need to figure out why, how it is doing this, and effect changes that will persist after reboots. There's also another problem: The web server is not able to serve the proper files to the end user due to improper configuration. The idea is to be able to grant or revoke access with SELinux depending on the needs and problems you encounter. In order to troubleshoot problems with SELinux, you will need to access and analyze the log files, locate the problems, and then implement an adequate solution. You should not use the global SELinux permissive state for verification.

## Solution

Log in to the server using the credentials provided:

```bash
ssh cloud_user@<PUBLIC_IP_ADDRESS>
```

```
$ ssh cloud_user@34.231.255.116
The authenticity of host '34.231.255.116 (34.231.255.116)' can't be established.
ECDSA key fingerprint is SHA256:eE+bkc51Q1L1Tr06jzcFLzhQB9wv1Ai2PqnhoZstU84.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:88: 54.89.225.30
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '34.231.255.116' (ECDSA) to the list of known hosts.
(cloud_user@34.231.255.116) Password:
[cloud_user@ip-10-0-1-138 ~]$ sudo -i
[sudo] password for cloud_user:
[root@ip-10-0-1-138 ~]#
```

### Install Troubleshooting Tools

1. Install the troubleshooting tools:
    
    ```
    sudo yum install -y setroubleshoot setools
    ```
    
2. View the contents of `auditd`:
    
    ```
    sudo systemctl cat auditd
    ```

```
[root@ip-10-0-1-138 ~]# sudo systemctl cat auditd
# /usr/lib/systemd/system/auditd.service
[Unit]
Description=Security Auditing Service
DefaultDependencies=no
## If auditd is sending or recieving remote logging, copy this file to
## /etc/systemd/system/auditd.service and comment out the first After and
## uncomment the second so that network-online.target is part of After.
## then comment the first Before and uncomment the second Before to remove
## sysinit.target from "Before".
After=local-fs.target systemd-tmpfiles-setup.service
##After=network-online.target local-fs.target systemd-tmpfiles-setup.service
Before=sysinit.target shutdown.target
##Before=shutdown.target
Conflicts=shutdown.target
RefuseManualStop=yes
ConditionKernelCommandLine=!audit=0
Documentation=man:auditd(8) https://github.com/linux-audit/audit-documentation

[Service]
Type=forking
PIDFile=/run/auditd.pid
ExecStart=/sbin/auditd
## To not use augenrules, copy this file to /etc/systemd/system/auditd.service
## and comment/delete the next line and uncomment the auditctl line.
## NOTE: augenrules expect any rules to be added to /etc/audit/rules.d/
ExecStartPost=-/sbin/augenrules --load
#ExecStartPost=-/sbin/auditctl -R /etc/audit/audit.rules
# By default we don't clear the rules on exit. To enable this, uncomment
# the next line after copying the file to /etc/systemd/system/auditd.service
#ExecStopPost=/sbin/auditctl -R /etc/audit/audit-stop.rules

[Install]
WantedBy=multi-user.target

[root@ip-10-0-1-138 ~]#

```


not sure why q, cat is just an output

1. Press `q` to get back to the prompt.
    
2. Open the `auditd.service` file:
    
    ```
    sudo vim /usr/lib/systemd/system/auditd.service
    ```
    
3. Change `RefuseManualStop=yes` to:
    
    ```
    RefuseManualStop=no
    ```
    
4. Save and quit the file by pressing **Escape** followed by `:wq`.
    
5. Run the following:
    
    ```
    sudo systemctl daemon-reload
    ```
    
6. Restart `auditd`:
    
    ```
    sudo systemctl restart auditd
    ```

```
[root@ip-10-0-1-138 ~]# vi /usr/lib/systemd/system/auditd.service
[root@ip-10-0-1-138 ~]# cat /usr/lib/systemd/system/auditd.service
[Unit]
Description=Security Auditing Service
DefaultDependencies=no
## If auditd is sending or recieving remote logging, copy this file to
## /etc/systemd/system/auditd.service and comment out the first After and
## uncomment the second so that network-online.target is part of After.
## then comment the first Before and uncomment the second Before to remove
## sysinit.target from "Before".
After=local-fs.target systemd-tmpfiles-setup.service
##After=network-online.target local-fs.target systemd-tmpfiles-setup.service
Before=sysinit.target shutdown.target
##Before=shutdown.target
Conflicts=shutdown.target
RefuseManualStop=no
ConditionKernelCommandLine=!audit=0
Documentation=man:auditd(8) https://github.com/linux-audit/audit-documentation

[Service]
Type=forking
PIDFile=/run/auditd.pid
ExecStart=/sbin/auditd
## To not use augenrules, copy this file to /etc/systemd/system/auditd.service
## and comment/delete the next line and uncomment the auditctl line.
## NOTE: augenrules expect any rules to be added to /etc/audit/rules.d/
ExecStartPost=-/sbin/augenrules --load
#ExecStartPost=-/sbin/auditctl -R /etc/audit/audit.rules
# By default we don't clear the rules on exit. To enable this, uncomment
# the next line after copying the file to /etc/systemd/system/auditd.service
#ExecStopPost=/sbin/auditctl -R /etc/audit/audit-stop.rules

[Install]
WantedBy=multi-user.target

[root@ip-10-0-1-138 ~]# sudo systemctl daemon-reload
[root@ip-10-0-1-138 ~]# sudo systemctl restart auditd
[root@ip-10-0-1-138 ~]#

```

1. Open the `auditd.service` file:
    
    ```
    sudo vim /usr/lib/systemd/system/auditd.service
    ```
    
2. Change `RefuseManualStop=no` to:
    
    ```
    RefuseManualStop=yes
    ```
    
3. Save and quit the file by pressing **Escape** followed by `:wq`.
    
```
[root@ip-10-0-1-138 ~]# vi /usr/lib/systemd/system/auditd.service
[root@ip-10-0-1-138 ~]# cat /usr/lib/systemd/system/auditd.service | grep RefuseManualStop
RefuseManualStop=yes
[root@ip-10-0-1-138 ~]#
```
### Attempt to Start Apache Web Server on Port 9100. When It Fails, Find the Line in the Log Files Confirming SELinux Is the Core Issue.

1. Configure Apache to run on port 9100:
    
    ```
    sudo vim /etc/httpd/conf/httpd.conf
    ```
    
2. Comment out the line `Listen 80` by adding a `#` before it.
    
3. Underneath `#Listen 80`, add the following:
    
    ```
    Listen 9100
    ```
    
4. Save and quit the file by pressing **Escape** followed by `:wq`.
    
5. Start `httpd`:
    
    ```
    sudo systemctl start httpd
    ```
    
    We'll get a failure.
    
6. View its status:
    
    ```
    sudo systemctl status httpd -l
    ```
    
    We'll see some permissions errors. Press `q` to get back to the prompt.

```
[root@ip-10-0-1-138 ~]# vi /etc/httpd/conf/httpd.conf
[root@ip-10-0-1-138 ~]# cat /etc/httpd/conf/httpd.conf | grep Listen
# Listen: Allows you to bind Apache to specific IP addresses and/or
# Change this to Listen on specific IP addresses as shown below to
#Listen 12.34.56.78:80
Listen 9100

[root@ip-10-0-1-138 ~]# systemctl start httpd
Job for httpd.service failed because the control process exited with error code. See "systemctl status httpd.service" and "journalctl -xe" for details.
[root@ip-10-0-1-138 ~]# sudo systemctl status httpd -l
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2026-06-25 16:15:11 EDT; 11s ago
     Docs: man:httpd(8)
           man:apachectl(8)
  Process: 27442 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 27442 (code=exited, status=1/FAILURE)

Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: Starting The Apache HTTP Server...
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:9100
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:9100
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: no listening sockets available, shutting down
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal httpd[27442]: AH00015: Unable to open logs
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: httpd.service: main process exited, code=exited, status=1/FAILURE
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: Failed to start The Apache HTTP Server.
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: Unit httpd.service entered failed state.
Jun 25 16:15:11 ip-10-0-1-138.ec2.internal systemd[1]: httpd.service failed.
[root@ip-10-0-1-138 ~]#
```


1. Search the file:
    
    ```
    sudo grep httpd /var/log/audit/audit.log
    ```
    
2. Run the following:
    
    ```
    sudo tail -f /var/log/audit/audit.log
    ```

```
[root@ip-10-0-1-138 ~]# sudo grep httpd /var/log/audit/audit.log
type=SOFTWARE_UPDATE msg=audit(1782417775.337:214): pid=26787 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="httpd-tools-2.4.6-99.el7_9.1.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1782417782.096:229): pid=26787 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="httpd-2.4.6-99.el7_9.1.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1782418511.727:404): avc:  denied  { name_bind } for  pid=27442 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1782418511.727:404): arch=c000003e syscall=49 success=no exit=-13 a0=4 a1=56443cea9e28 a2=1c a3=7ffe7ea1fda0 items=0 ppid=1 pid=27442 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1782418511.727:405): avc:  denied  { name_bind } for  pid=27442 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1782418511.727:405): arch=c000003e syscall=49 success=no exit=-13 a0=3 a1=56443cea9d68 a2=10 a3=7ffe7ea2097c items=0 ppid=1 pid=27442 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1782418511.748:406): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=httpd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'

[root@ip-10-0-1-138 ~]# sudo tail -f /var/log/audit/audit.log
type=USER_ACCT msg=audit(1782418582.166:413): pid=27466 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:accounting grantors=pam_unix,pam_localuser acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'
type=USER_CMD msg=audit(1782418582.166:414): pid=27466 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='cwd="/root" cmd=67726570206874747064202F7661722F6C6F672F61756469742F61756469742E6C6F67 terminal=pts/0 res=success'
type=CRED_REFR msg=audit(1782418582.166:415): pid=27466 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'
type=USER_START msg=audit(1782418582.170:416): pid=27466 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'
type=USER_END msg=audit(1782418582.461:417): pid=27466 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_close grantors=pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'
type=CRED_DISP msg=audit(1782418582.461:418): pid=27466 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'
type=USER_ACCT msg=audit(1782418591.602:419): pid=27469 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:accounting grantors=pam_unix,pam_localuser acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'
type=USER_CMD msg=audit(1782418591.603:420): pid=27469 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='cwd="/root" cmd=7461696C202D66202F7661722F6C6F672F61756469742F61756469742E6C6F67 terminal=pts/0 res=success'
type=CRED_REFR msg=audit(1782418591.603:421): pid=27469 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'
type=USER_START msg=audit(1782418591.605:422): pid=27469 uid=0 auid=1003 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/0 res=success'

```

1. Open a new terminal.
    
2. Log in to the lab server in the new terminal:
    
    ```
    ssh cloud_user@<PUBLIC_IP_ADDRESS>
    ```
    
3. After we're logged in to the second terminal, highlight the line ending `AUID="cloud_user" SUID="cloud_user"` at the bottom of the output in the first terminal. (This way we can see where our new output will start.)

 in the other cli console this appears
```
type=CRYPTO_KEY_USER msg=audit(1782418694.580:423): pid=27475 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:fe:89:fa:c4:f0:bb:16:e5:7f:4b:49:d8:af:3a:a9:41:70:92:d8:cb:cf:eb:ab:c1:c0:1e:d6:c3:1f:d0:4f:91 direction=? spid=27475 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRYPTO_KEY_USER msg=audit(1782418694.580:424): pid=27475 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:78:4f:9b:91:ce:75:43:52:f5:4e:bd:3a:8f:37:05:2f:38:50:07:dc:2f:d4:08:b6:3e:a9:e1:a1:9b:2d:53:ce direction=? spid=27475 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRYPTO_SESSION msg=audit(1782418694.624:425): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=start direction=from-server cipher=chacha20-poly1305@openssh.com ksize=512 mac=<implicit> pfs=curve25519-sha256 spid=27475 suid=74 rport=56739 laddr=10.0.1.138 lport=22  exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=? res=success'
type=CRYPTO_SESSION msg=audit(1782418694.624:426): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=start direction=from-client cipher=chacha20-poly1305@openssh.com ksize=512 mac=<implicit> pfs=curve25519-sha256 spid=27475 suid=74 rport=56739 laddr=10.0.1.138 lport=22  exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=? res=success'
type=USER_AUTH msg=audit(1782418695.061:427): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=pubkey acct="cloud_user" exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=ssh res=failed'
type=USER_AUTH msg=audit(1782418706.900:428): pid=27476 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:authentication grantors=pam_unix acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=USER_ACCT msg=audit(1782418706.905:429): pid=27476 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:accounting grantors=pam_unix,pam_localuser acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=CRYPTO_KEY_USER msg=audit(1782418706.940:430): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=session fp=? direction=both spid=27475 suid=74 rport=56739 laddr=10.0.1.138 lport=22  exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=? res=success'
type=USER_AUTH msg=audit(1782418706.942:431): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=success acct="cloud_user" exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=ssh res=success'
type=CRED_ACQ msg=audit(1782418706.942:432): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=LOGIN msg=audit(1782418706.943:433): pid=27474 uid=0 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 old-auid=4294967295 auid=1003 tty=(none) old-ses=4294967295 ses=3 res=1
type=USER_ROLE_CHANGE msg=audit(1782418707.047:434): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='pam: default-context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 selected-context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=USER_START msg=audit(1782418707.064:435): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_selinux,pam_loginuid,pam_selinux,pam_namespace,pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix,pam_lastlog acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=CRYPTO_KEY_USER msg=audit(1782418707.065:436): pid=27479 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:fe:89:fa:c4:f0:bb:16:e5:7f:4b:49:d8:af:3a:a9:41:70:92:d8:cb:cf:eb:ab:c1:c0:1e:d6:c3:1f:d0:4f:91 direction=? spid=27479 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRYPTO_KEY_USER msg=audit(1782418707.065:437): pid=27479 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:78:4f:9b:91:ce:75:43:52:f5:4e:bd:3a:8f:37:05:2f:38:50:07:dc:2f:d4:08:b6:3e:a9:e1:a1:9b:2d:53:ce direction=? spid=27479 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRED_ACQ msg=audit(1782418707.067:438): pid=27479 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=USER_LOGIN msg=audit(1782418707.186:439): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=login id=1003 exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=/dev/pts/1 res=success'
type=USER_START msg=audit(1782418707.186:440): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=login id=1003 exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=/dev/pts/1 res=success'
```


4. In the second terminal, start `httpd`:
    
    ```
    sudo systemctl start httpd
    ```
    
    We'll get an error saying:
    
    ```
    Job for httpd.service failed because the control process exited with error code.
    See "systemctl status httpd.service" and "journalctl -xe" for details.
    ```
    
    note the update to the error log on the other terminal.
    
5. In the first terminal, highlight the huge block of new output (that starts under the `AUID="cloud_user" SUID="cloud_user"` line we noted before).

in the second console i did get tht message

not sure what to look at in the first console so copying the log here
```
type=CRYPTO_KEY_USER msg=audit(1782418694.580:423): pid=27475 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:fe:89:fa:c4:f0:bb:16:e5:7f:4b:49:d8:af:3a:a9:41:70:92:d8:cb:cf:eb:ab:c1:c0:1e:d6:c3:1f:d0:4f:91 direction=? spid=27475 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRYPTO_KEY_USER msg=audit(1782418694.580:424): pid=27475 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:78:4f:9b:91:ce:75:43:52:f5:4e:bd:3a:8f:37:05:2f:38:50:07:dc:2f:d4:08:b6:3e:a9:e1:a1:9b:2d:53:ce direction=? spid=27475 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRYPTO_SESSION msg=audit(1782418694.624:425): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=start direction=from-server cipher=chacha20-poly1305@openssh.com ksize=512 mac=<implicit> pfs=curve25519-sha256 spid=27475 suid=74 rport=56739 laddr=10.0.1.138 lport=22  exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=? res=success'
type=CRYPTO_SESSION msg=audit(1782418694.624:426): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=start direction=from-client cipher=chacha20-poly1305@openssh.com ksize=512 mac=<implicit> pfs=curve25519-sha256 spid=27475 suid=74 rport=56739 laddr=10.0.1.138 lport=22  exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=? res=success'
type=USER_AUTH msg=audit(1782418695.061:427): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=pubkey acct="cloud_user" exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=ssh res=failed'
type=USER_AUTH msg=audit(1782418706.900:428): pid=27476 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:authentication grantors=pam_unix acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=USER_ACCT msg=audit(1782418706.905:429): pid=27476 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:accounting grantors=pam_unix,pam_localuser acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=CRYPTO_KEY_USER msg=audit(1782418706.940:430): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=session fp=? direction=both spid=27475 suid=74 rport=56739 laddr=10.0.1.138 lport=22  exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=? res=success'
type=USER_AUTH msg=audit(1782418706.942:431): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=success acct="cloud_user" exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=ssh res=success'
type=CRED_ACQ msg=audit(1782418706.942:432): pid=27474 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=LOGIN msg=audit(1782418706.943:433): pid=27474 uid=0 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 old-auid=4294967295 auid=1003 tty=(none) old-ses=4294967295 ses=3 res=1
type=USER_ROLE_CHANGE msg=audit(1782418707.047:434): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='pam: default-context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 selected-context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=USER_START msg=audit(1782418707.064:435): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_selinux,pam_loginuid,pam_selinux,pam_namespace,pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix,pam_lastlog acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=CRYPTO_KEY_USER msg=audit(1782418707.065:436): pid=27479 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:fe:89:fa:c4:f0:bb:16:e5:7f:4b:49:d8:af:3a:a9:41:70:92:d8:cb:cf:eb:ab:c1:c0:1e:d6:c3:1f:d0:4f:91 direction=? spid=27479 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRYPTO_KEY_USER msg=audit(1782418707.065:437): pid=27479 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:78:4f:9b:91:ce:75:43:52:f5:4e:bd:3a:8f:37:05:2f:38:50:07:dc:2f:d4:08:b6:3e:a9:e1:a1:9b:2d:53:ce direction=? spid=27479 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRED_ACQ msg=audit(1782418707.067:438): pid=27479 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=USER_LOGIN msg=audit(1782418707.186:439): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=login id=1003 exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=/dev/pts/1 res=success'
type=USER_START msg=audit(1782418707.186:440): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=login id=1003 exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=/dev/pts/1 res=success'
type=SERVICE_START msg=audit(1782418902.092:441): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=NetworkManager-dispatcher comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1782418912.017:442): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=NetworkManager-dispatcher comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=USER_AUTH msg=audit(1782419024.451:443): pid=27537 uid=1003 auid=1003 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:authentication grantors=pam_unix acct="cloud_user" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/1 res=success'
type=USER_ACCT msg=audit(1782419024.455:444): pid=27537 uid=1003 auid=1003 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:accounting grantors=pam_unix,pam_localuser acct="cloud_user" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/1 res=success'
type=USER_CMD msg=audit(1782419024.456:445): pid=27537 uid=1003 auid=1003 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='cwd="/home/cloud_user" cmd=73797374656D63746C207374617274206874747064 terminal=pts/1 res=success'
type=CRED_REFR msg=audit(1782419024.456:446): pid=27537 uid=0 auid=1003 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/1 res=success'
type=USER_START msg=audit(1782419024.463:447): pid=27537 uid=0 auid=1003 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/1 res=success'
type=AVC msg=audit(1782419024.520:448): avc:  denied  { name_bind } for  pid=27547 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1782419024.520:448): arch=c000003e syscall=49 success=no exit=-13 a0=4 a1=560219e53e28 a2=1c a3=7ffe23a56d60 items=0 ppid=1 pid=27547 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=PROCTITLE msg=audit(1782419024.520:448): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=AVC msg=audit(1782419024.521:449): avc:  denied  { name_bind } for  pid=27547 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1782419024.521:449): arch=c000003e syscall=49 success=no exit=-13 a0=3 a1=560219e53d68 a2=10 a3=7ffe23a5792c items=0 ppid=1 pid=27547 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=PROCTITLE msg=audit(1782419024.521:449): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SERVICE_START msg=audit(1782419024.544:450): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=httpd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=USER_END msg=audit(1782419024.547:451): pid=27537 uid=0 auid=1003 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_close grantors=pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/1 res=success'
type=CRED_DISP msg=audit(1782419024.548:452): pid=27537 uid=0 auid=1003 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_unix acct="root" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts/1 res=success'
^F
```


6. Copy and paste it into a text file to examine it. We should see binding to the port was denied.
    
7. Close out of the second terminal.

when i close the second terminal i see logs generates

```
Ftype=CRYPTO_KEY_USER msg=audit(1782419223.709:453): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=session fp=? direction=both spid=27479 suid=1003 rport=56739 laddr=10.0.1.138 lport=22  exe="/usr/sbin/sshd" hostname=? addr=73.49.202.63 terminal=? res=success'
type=USER_END msg=audit(1782419223.848:454): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:session_close grantors=pam_selinux,pam_loginuid,pam_selinux,pam_namespace,pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix,pam_lastlog acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=CRED_DISP msg=audit(1782419223.848:455): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="cloud_user" exe="/usr/sbin/sshd" hostname=c-73-49-202-63.hsd1.fl.comcast.net addr=73.49.202.63 terminal=ssh res=success'
type=USER_END msg=audit(1782419223.848:456): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=login id=1003 exe="/usr/sbin/sshd" hostname=? addr=? terminal=/dev/pts/1 res=success'
type=USER_LOGOUT msg=audit(1782419223.848:457): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=login id=1003 exe="/usr/sbin/sshd" hostname=? addr=? terminal=/dev/pts/1 res=success'
type=CRYPTO_KEY_USER msg=audit(1782419223.849:458): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:fe:89:fa:c4:f0:bb:16:e5:7f:4b:49:d8:af:3a:a9:41:70:92:d8:cb:cf:eb:ab:c1:c0:1e:d6:c3:1f:d0:4f:91 direction=? spid=27474 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'
type=CRYPTO_KEY_USER msg=audit(1782419223.849:459): pid=27474 uid=0 auid=1003 ses=3 subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 msg='op=destroy kind=server fp=SHA256:78:4f:9b:91:ce:75:43:52:f5:4e:bd:3a:8f:37:05:2f:38:50:07:dc:2f:d4:08:b6:3e:a9:e1:a1:9b:2d:53:ce direction=? spid=27474 suid=0  exe="/usr/sbin/sshd" hostname=? addr=? terminal=? res=success'

```


8. In the first terminal (the only one that should be open now), cancel the monitoring process by pressing **Ctrl**+**C**.   
    
9. Run the following:
    
    ```
    sudo grep httpd /var/log/audit/audit.log | audit2why
    ```
    
    At the beginning of the output, we should see a similar error as before: `avc denied { name_bind } for pid=13982 comm="httpd" src=9100"`.
    
```
[root@ip-10-0-1-138 ~]#     sudo grep httpd /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1782418511.727:404): avc:  denied  { name_bind } for  pid=27442 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
type=AVC msg=audit(1782418511.727:405): avc:  denied  { name_bind } for  pid=27442 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
type=AVC msg=audit(1782419024.520:448): avc:  denied  { name_bind } for  pid=27547 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
type=AVC msg=audit(1782419024.521:449): avc:  denied  { name_bind } for  pid=27547 comm="httpd" src=9100 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hplip_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean httpd_enable_ftp_server was set incorrectly.
        Description:
        Allow httpd to enable ftp server

        Allow access by executing:
        # setsebool -P httpd_enable_ftp_server 1
[root@ip-10-0-1-138 ~]#
```
### Find the Right Port Label for Apache Web Server and Add the Needed Port. Then Restart the Apache Web Server.

1. List all the possible port labels and search the list for `http`:
    
    ```
    sudo semanage port -l | grep -i http
    ```
    
    Note the ports listed for `http_port_t`.
    
2. Add port 9100 to the `http_port_t` label:
    
    ```
    sudo semanage port -a -t http_port_t -p tcp 9100
    ```
    
    We'll get an error saying it's already defined.
    
3. Run the following:
    
    ```
    sudo semanage port -m -t http_port_t -p tcp 9100
    ```
    
4. List all the possible port labels and search the list for `http`:
    
    ```
    sudo semanage port -l | grep -i http
    ```
    
    Note the ports listed for `http_port_t`. We should see 9100 on the list now.


```
[root@ip-10-0-1-138 ~]# sudo semanage port -l | grep -i http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@ip-10-0-1-138 ~]# sudo semanage port -a -t http_port_t -p tcp 9100
ValueError: Port tcp/9100 already defined
[root@ip-10-0-1-138 ~]# sudo semanage port -m -t http_port_t -p tcp 9100
[root@ip-10-0-1-138 ~]# sudo semanage port -l | grep -i http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      9100, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t    
```


1. In a new browser tab, navigate to `<PUBLIC_IP_ADDRESS>:9100`, replacing `<PUBLIC_IP_ADDRESS>` with the IP on the lab page that you used to log in to the terminal. It won't load.

![[Pasted image 20260625163344.png]]


2. In the terminal, check the status of `httpd`:
    
    ```
    sudo systemctl status httpd
    ```
    
    We'll see it says Apache is not running. Press **Ctrl**+**C** to cancel the process.
    
3. Open the `httpd` file:
    
    ```
    sudo vim /etc/httpd/conf/httpd.conf
    ```
    
4. Change `Listen 9100` to:
    
    ```
    Listen 0.0.0.0:9100
    ```
    
5. Save and quit the file by pressing **Escape** followed by `:wq`.
    
6. Start `httpd`:
    
    ```
    sudo systemctl start httpd
    ```
    
    This time, it will work.
    
7. Check its status:
    
    ```
    sudo systemctl status httpd
    ```
    
    Press **Ctrl**+**C** to cancel the process.

```
[root@ip-10-0-1-138 ~]# sudo systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2026-06-25 16:23:44 EDT; 10min ago
     Docs: man:httpd(8)
           man:apachectl(8)
  Process: 27547 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 27547 (code=exited, status=1/FAILURE)

Jun 25 16:23:44 ip-10-0-1-138.ec2.internal systemd[1]: Starting The Apache HTTP Server...
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal httpd[27547]: (13)Permission denied: AH00072: make_sock: could not bind to address ...:9100
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal httpd[27547]: (13)Permission denied: AH00072: make_sock: could not bind to address ...:9100
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal httpd[27547]: no listening sockets available, shutting down
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal httpd[27547]: AH00015: Unable to open logs
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal systemd[1]: httpd.service: main process exited, code=exited, status=1/FAILURE
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal systemd[1]: Failed to start The Apache HTTP Server.
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal systemd[1]: Unit httpd.service entered failed state.
Jun 25 16:23:44 ip-10-0-1-138.ec2.internal systemd[1]: httpd.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
[root@ip-10-0-1-138 ~]# vi /etc/httpd/conf/httpd.conf
[root@ip-10-0-1-138 ~]# systemctl start httpd
[root@ip-10-0-1-138 ~]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2026-06-25 16:35:05 EDT; 13s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 27650 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─27650 /usr/sbin/httpd -DFOREGROUND
           ├─27651 /usr/sbin/httpd -DFOREGROUND
           ├─27652 /usr/sbin/httpd -DFOREGROUND
           ├─27653 /usr/sbin/httpd -DFOREGROUND
           ├─27654 /usr/sbin/httpd -DFOREGROUND
           └─27655 /usr/sbin/httpd -DFOREGROUND

Jun 25 16:35:05 ip-10-0-1-138.ec2.internal systemd[1]: Starting The Apache HTTP Server...
Jun 25 16:35:05 ip-10-0-1-138.ec2.internal systemd[1]: Started The Apache HTTP Server.
[root@ip-10-0-1-138 ~]#
```


2. In the browser, try `<PUBLIC_IP_ADDRESS>:9100` again. We'll get a message saying it's forbidden.
    
![[Pasted image 20260625163616.png]]
### Find the Right SELinux Context for the `index.html` in `/var/www/html/` and Set It Permanently

1. In the terminal, become `root`:
    
    ```
    sudo su
    ```
    
2. Change directory:
    
    ```
    cd /var/www/html/
    ```
    
3. List what's in it:
    
    ```
    ls
    ```
    
4. See the context of what's in it:
    
    ```
    ls -lZ
    ```
    
5. Create a file:
    
    ```
    touch test
    ```
    
6. See the context again:
    
    ```
    ls -lZ
    ```
    
7. Change the context of the file:
    
    ```
    semanage fcontext -a -t httpd_sys_content_t /var/www/html/index.html
    ```
    
    We may get an error.
    
8. Modify it:
    
    ```
    semanage fcontext -m -t httpd_sys_content_t /var/www/html/index.html
    ```
    
9. Reset the security context:
    
    ```
    restorecon -v /var/www/html/index.html
    ```

```
[root@ip-10-0-1-138 ~]# sudo su
[root@ip-10-0-1-138 ~]# cd /var/www/html/
[root@ip-10-0-1-138 html]# ls
index.html
[root@ip-10-0-1-138 html]# ls -lZ
-rw-r--r--. root root system_u:object_r:user_home_t:s0 index.html
[root@ip-10-0-1-138 html]# touch test
[root@ip-10-0-1-138 html]# ls -lZ
-rw-r--r--. root root system_u:object_r:user_home_t:s0 index.html
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 test
[root@ip-10-0-1-138 html]# semanage fcontext -a -t httpd_sys_content_t /var/www/html/index.html
[root@ip-10-0-1-138 html]# semanage fcontext -m -t httpd_sys_content_t /var/www/html/index.html
[root@ip-10-0-1-138 html]# restorecon -v /var/www/html/index.html
restorecon reset /var/www/html/index.html context system_u:object_r:user_home_t:s0->system_u:object_r:httpd_sys_content_t:s0
[root@ip-10-0-1-138 html]#
```


1. In the browser, try `<PUBLIC_IP_ADDRESS>:9100` again. This time, it should work.
    
![[Pasted image 20260625163852.png]]
## Conclusion

Congratulations on successfully completing this hands-on lab!
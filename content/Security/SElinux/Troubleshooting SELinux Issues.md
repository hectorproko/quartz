---

tags:
  - selinux
linkedin: "False"
quartz: "False"
refactored: "True"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
## Overview

In this lab, I worked through a scenario where SELinux was blocking a web service (Apache/httpd) from starting and from serving content correctly. The goal was to identify the SELinux denials causing the failures and resolve them so Apache could run and serve a page successfully.

**Environment:**

- Cloud server running RHEL/CentOS 7
- Access via SSH as `cloud_user`, escalated to root with `sudo su -`

%%
## Why This Matters

SELinux is a mandatory access control system that frequently gets blamed (or ignored) when services misbehave. Knowing how to read AVC denials, trace them back to the affected file/inode, and apply the correct fix, instead of just disabling SELinux, is a core Linux administration skill. This lab walks through that exact diagnostic process from end to end.
%%

## Step 1: Attempt to Start Apache

```bash
systemctl start httpd
systemctl status httpd -l
tail /var/log/audit/audit.log
```

**Output:**

```
[cloud_user@ip-10-0-1-10 ~]$ sudo su -

[root@ip-10-0-1-10 ~]# systemctl start httpd
Failed to start httpd.service: Unit not found.

[root@ip-10-0-1-10 ~]# systemctl status httpd -l
Unit httpd.service could not be found.
```

At this point httpd wasn't even installed/recognized as a unit yet, so I moved on to checking SELinux denials directly with `ausearch`.

## Step 2: Identify SELinux Denials

```bash
ausearch -m avc -ts recent
```
%%
[[ausearch (Audit Search)#Options|ausearch]]
[[AVC (Access Vector Cache)]]
%%
This returned several AVC `denied` entries, including:

```
type=AVC msg=audit(...): avc: denied { read } for pid=1234 comm="sshd" name="lastlog" dev="xvda2" ino=43046 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0
```

The denial includes a filename (`name="lastlog"`), but only the basename, not the full path, and on a busy system there can be multiple files with the same name. The inode number (`ino=43046`) uniquely identifies the exact file on that filesystem, so the next step was to resolve that inode to its full path.

### Troubleshooting: Resolving the inode number


```bash
find / -inum 43046
```

```
/var/log/lastlog
```

That confirmed the denial was tied to `/var/log/lastlog`.

## Step 3: Check and Restore File Contexts

```bash
ls -Z /var/log/httpd
restorecon -Rv /var/log/httpd
```
%%[[restorecon (Restore Context)#Options]]%%
```
restorecon reset /var/log/httpd context system_u:object_r:httpd_config_t:s0 -> system_u:object_r:httpd_log_t:s0
```

The context on `/var/log/httpd` had been mislabeled as `httpd_config_t` instead of `httpd_log_t`. `restorecon` reset it to the correct type.

## Step 4: Start Apache Again

```bash
systemctl start httpd
```

This time the service started without errors.

## Step 5: Resolve SELinux Issues Preventing Web Content From Loading

```bash
curl localhost
```

```
403 Forbidden
You don't have permission to access /index.html on this server.
```

Apache was running, but SELinux was still blocking access to the content itself.

```bash
ausearch -m avc -ts recent
```

```
type=AVC msg=audit(1779312775.449:190): avc: denied { read } for pid=1713 comm="httpd" name="index.html" dev="xvda2" ino=18641 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0
```

This denial showed `index.html` labeled as `user_home_t` - the default context for files in a user's home directory, which `httpd_t` isn't allowed to read by default.

Resolved the inode to confirm the file path:

```bash
find / -inum 18641
ls -Z /home/cloud_user/html/index.html
```

```
/home/cloud_user/html/index.html
-rw-rw-r--. cloud_user cloud_user system_u:object_r:user_home_t:s0 /home/cloud_user/html/index.html
```

## Step 6: Use `sealert` for a Deeper Root-Cause Analysis

To get more detail (and a suggested fix) than the raw audit log provides, I installed and ran `setroubleshoot`:

```bash
yum -y install setroubleshoot setroubleshoot-server
service auditd restart #older SysV-style command
sealert -a /var/log/audit/audit.log
```
%%`service auditd restart` is the older SysV-style command. On modern RHEL/CentOS 7+ (which uses systemd), the equivalent is:%%

`auditd` specifically has historically had quirks with `systemctl restart` on some RHEL 7 versions *(it would sometimes refuse to restart cleanly via systemctl and recommend using `service auditd restart` instead)*

`sealert` confirmed the same finding and suggested a Boolean-based fix rather than a one-off file relabel:
%%
`sealert` it's provided by the **`setroubleshoot`** and **`setroubleshoot-server`** package
%%
```
SELinux is preventing /usr/sbin/httpd from read access on the file index.html.

If you want to allow httpd to read user content
Then you must tell SELinux about this by enabling the 'httpd_read_user_content' boolean.

Do
setsebool -P httpd_read_user_content 1
```

## Step 7: Enable the Correct SELinux Boolean

```bash
getsebool httpd_read_user_content
```

```
httpd_read_user_content --> off
```

```bash
setsebool -P httpd_read_user_content=1
```
%%[[setsebool#Options]]%%
## Step 8: Confirm the Fix

```bash
curl localhost
```

```
Hello world
```

Apache was now serving content successfully, with SELinux fully enforcing and no denials.

## Conclusion

This lab reinforced a methodical approach to SELinux troubleshooting:

1. Reproduce the failure.
2. Check `ausearch -m avc -ts recent` for the relevant denial.
3. Resolve the inode/context involved (`find / -inum`, `ls -Z`).
4. Decide between a context fix (`restorecon`) or a policy/boolean fix (`setsebool`), depending on whether the file is mislabeled or the service genuinely needs broader access.
5. Use `sealert` when the raw audit log isn't enough context to decide on a fix.

The key takeaway: SELinux denials almost always point to either (a) a mislabeled file context that `restorecon` can fix, or (b) a legitimate access pattern that needs an SELinux boolean enabled, not a reason to disable SELinux outright.



<!--
[Troubleshooting SELinux Issues](https://app.pluralsight.com/hands-on/labs/ea03224d-cbb4-45b0-a1bb-ea7b6ff6bba3?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)
Cloud Server server1

Username
cloud_user

Password
tn)1iHTN

Public ip address of server1
52.23.238.175

Private ip address of the server1
10.0.1.10
## Lab Overview

In this exercise, you will troubleshoot SELinux issues preventing a service from starting, as well as functioning correctly.

_This course is not approved or sponsored by Red Hat_


# Troubleshooting SELinux issues

_This course is not approved or sponsored by Red Hat._

## Introduction

In this exercise, you will troubleshoot SELinux issues preventing a service from starting, as well as functioning correctly.

Successfully start Apache and have it serve content from the configured location by resolving SELinux issues.

## Solution

Start by logging in to the lab server using the credentials provided on the hands-on lab page:

```
ssh cloud_user@PUBLIC_IP_ADDRESS
```

Become the root user:

```
sudo su -
```

### Start Apache

1. Attempt to start the service:
    
    ```
     systemctl start httpd
    ```
    
2. View the status log:
    
    ```
     systemctl status httpd -l
    ```
    
3. View the `audit.log` file for the error:
    
    ```
     tail /var/log/audit/audit.log
    ```

```
[cloud_user@ip-10-0-1-10 ~]$ sudo su -
[sudo] password for cloud_user:

[root@ip-10-0-1-10 ~]# systemctl start httpd
Failed to start httpd.service: Unit not found.

[root@ip-10-0-1-10 ~]# systemctl status httpd -l
Unit httpd.service could not be found.

[root@ip-10-0-1-10 ~]# tail /var/log/audit/audit.log
type=USER_START msg=audit(1779311874.157:95): pid=1283 uid=0 auid=1002 ses=1 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_keyinit,pam_keyinit,pam_limits,pam_systemd,pam_unix,pam_xauth acct="root" exe="/usr/bin/su" hostname=ip-10-0-1-10.ec2.internal addr=? terminal=pts/0 res=success'
type=SOFTWARE_UPDATE msg=audit(1779311903.505:96): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="apr-1.4.8-7.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311903.646:97): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="apr-util-1.5.2-6.el7_9.1.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311903.883:98): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="httpd-tools-2.4.6-99.el7_9.1.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311906.819:99): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="ebtables-2.0.10-16.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311906.988:100): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="ipset-libs-7.1-1.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311907.107:101): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="ipset-7.1-1.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311908.488:102): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="redhat-logos-70.7.0-1.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311908.673:103): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="python-slip-0.4.0-4.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1779311908.821:104): pid=1205 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:rpm_t:s0 msg='sw="python-slip-dbus-0.4.0-4.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
[root@ip-10-0-1-10 ~]#
```


1. View recent SELinux errors:
    
    ```
     ausearch -m avc -ts recent
    ```

```
[root@ip-10-0-1-10 ~]# ausearch -m avc -ts recent
----
time->Wed May 20 21:17:21 2026
type=PROCTITLE msg=audit(1779311841.690:60): proctitle=737368643A20636C6F75645F75736572205B707269765D
type=SYSCALL msg=audit(1779311841.690:60): arch=c000003e syscall=2 success=no exit=-13 a0=7fa4ab8e132e a1=0 a2=3ea a3=3 items=0 ppid=990 pid=1234 auid=1002 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=1 comm="sshd" exe="/usr/sbin/sshd" subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 key=(null)
type=AVC msg=audit(1779311841.690:60): avc:  denied  { read } for  pid=1234 comm="sshd" name="lastlog" dev="xvda2" ino=43046 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0
----
time->Wed May 20 21:17:22 2026
type=PROCTITLE msg=audit(1779311842.697:73): proctitle=737368643A20636C6F75645F75736572205B707269765D
type=SYSCALL msg=audit(1779311842.697:73): arch=c000003e syscall=2 success=no exit=-13 a0=7fa1c841232e a1=0 a2=3ea a3=3 items=0 ppid=990 pid=1240 auid=1002 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=2 comm="sshd" exe="/usr/sbin/sshd" subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 key=(null)
type=AVC msg=audit(1779311842.697:73): avc:  denied  { read } for  pid=1240 comm="sshd" name="lastlog" dev="xvda2" ino=43046 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0
----
time->Wed May 20 21:17:24 2026
type=PROCTITLE msg=audit(1779311844.947:82): proctitle=737368643A20636C6F75645F75736572205B707269765D
type=SYSCALL msg=audit(1779311844.947:82): arch=c000003e syscall=2 success=no exit=-13 a0=7ffd248ab370 a1=0 a2=180 a3=0 items=0 ppid=990 pid=1234 auid=1002 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=1 comm="sshd" exe="/usr/sbin/sshd" subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 key=(null)
type=AVC msg=audit(1779311844.947:82): avc:  denied  { read } for  pid=1234 comm="sshd" name="lastlog" dev="xvda2" ino=43046 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0
----
time->Wed May 20 21:17:24 2026
type=PROCTITLE msg=audit(1779311844.947:83): proctitle=737368643A20636C6F75645F75736572205B707269765D
type=SYSCALL msg=audit(1779311844.947:83): arch=c000003e syscall=2 success=no exit=-13 a0=7ffd248aba10 a1=42 a2=180 a3=8 items=0 ppid=990 pid=1234 auid=1002 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=1 comm="sshd" exe="/usr/sbin/sshd" subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 key=(null)
type=AVC msg=audit(1779311844.947:83): avc:  denied  { read write } for  pid=1234 comm="sshd" name="lastlog" dev="xvda2" ino=43046 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0
[root@ip-10-0-1-10 ~]#

```


1. Find the inode it's attempting to write to:
    
    ```
     find / -inum &lt;inode number&gt;
    ```

```
[root@ip-10-0-1-10 ~]# find / -inum &lt;inode number&gt;
[1] 1583
-bash: lt: command not found
[2] 1585
-bash: gt: command not found
[root@ip-10-0-1-10 ~]# -bash: inode: command not found
find: invalid argument `-inum' to `-inum
```

i guess this is what they wanted me to do
Solutino: Looking at your `ausearch` output, you can see `ino=43046` in the AVC denial lines. So the command you need to run is:
```
find / -inum 43046
```

1. View the SELinux context of the directory/file:
    
    ```
     ls -Z /var/log/httpd
    ```
    
2. Restore the proper file context:
    
    ```
     restorecon -Rv /var/log/httpd
    ```
    
3. We should be able to start Apache now:
    
    ```
     systemctl start httpd	
    ```
    
```
[root@ip-10-0-1-10 ~]# find / -inum 43046
/var/log/lastlog
[root@ip-10-0-1-10 ~]# ls -Z /var/log/httpd
[root@ip-10-0-1-10 ~]# restorecon -Rv /var/log/httpd
restorecon reset /var/log/httpd context system_u:object_r:httpd_config_t:s0->system_u:object_r:httpd_log_t:s0
[root@ip-10-0-1-10 ~]# systemctl start httpd
[root@ip-10-0-1-10 ~]#
```
### Resolve SELinux issues preventing viewing web content

1. Attempt to view the web content:
    
    ```
     curl localhost
    ```
    
2. View recent AVC errors:
    
    ```
     ausearch -m avc -ts recent
    ```
    
3. Find the inode of the file/directory:

```
[root@ip-10-0-1-10 ~]# curl localhost
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /index.html
on this server.</p>
</body></html>
[root@ip-10-0-1-10 ~]# ausearch -m avc -ts recent
----
time->Wed May 20 21:32:55 2026
type=PROCTITLE msg=audit(1779312775.449:190): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SYSCALL msg=audit(1779312775.449:190): arch=c000003e syscall=2 success=no exit=-13 a0=55b6aacba030 a1=80000 a2=0 a3=7ffdb979c920 items=0 ppid=1708 pid=1713 auid=4294967295 uid=48 gid=48 euid=48 suid=48 fsuid=48 egid=48 sgid=48 fsgid=48 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1779312775.449:190): avc:  denied  { read } for  pid=1713 comm="httpd" name="index.html" dev="xvda2" ino=18641 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0
[root@ip-10-0-1-10 ~]#

```



`find / -inum &lt;inode number&gt;`

`find / -inum 18641`
    
4. View the context of the file/directory:
    
    ```
     ls -Z /home/cloud_user/html/index.html
    ```

```
[root@ip-10-0-1-10 ~]# find / -inum 18641
/home/cloud_user/html/index.html
[root@ip-10-0-1-10 ~]# ls -Z /home/cloud_user/html/index.html
-rw-rw-r--. cloud_user cloud_user system_u:object_r:user_home_t:s0 /home/cloud_user/html/index.html
[root@ip-10-0-1-10 ~]#
```

4. Install `sealert`:
    
    ```
     yum -y install setroubleshoot setroubleshoot-server
    ```
    
5. Restart the `auditd` service:
    
    ```
     service auditd restart
    ```
    
6. Use `sealert` for more information:
    
    ```
     sealert -a /var/log/audit/audit.log
    ```

```
[root@ip-10-0-1-10 ~]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
[root@ip-10-0-1-10 ~]# sealert -a /var/log/audit/audit.log
100% done
found 4 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/sshd;5d63fa63 (deleted) from write access on the file wtmp.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that sshd;5d63fa63 (deleted) should be allowed write access on the wtmp file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'sshd' --raw | audit2allow -M my-sshd
# semodule -i my-sshd.pp


Additional Information:
Source Context                system_u:system_r:sshd_t:s0-s0:c0.c1023
Target Context                system_u:object_r:var_log_t:s0
Target Objects                wtmp [ file ]
Source                        sshd
Source Path                   /usr/sbin/sshd;5d63fa63 (deleted)
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages
Target RPM Packages
Policy RPM                    selinux-policy-3.13.1-268.el7_9.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     ip-10-0-1-10.ec2.internal
Platform                      Linux ip-10-0-1-10.ec2.internal
                              3.10.0-1160.119.1.el7.x86_64 #1 SMP Tue May 14
                              11:55:25 EDT 2024 x86_64 x86_64
Alert Count                   6
First Seen                    2019-08-26 15:42:39 UTC
Last Seen                     2024-08-06 18:29:43 UTC
Local ID                      d8e91d45-5b70-417a-b26e-2dd3f28ad31c

Raw Audit Messages
type=AVC msg=audit(1722968983.368:272): avc:  denied  { write } for  pid=1326 comm="sshd" name="wtmp" dev="xvda2" ino=44926 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0


type=SYSCALL msg=audit(1722968983.368:272): arch=x86_64 syscall=open success=no exit=EACCES a0=7fa884df37c1 a1=1 a2=59d9c a3=66b26b97 items=0 ppid=1 pid=1326 auid=1002 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=1002 fsgid=0 tty=(none) ses=2 comm=sshd exe=/usr/sbin/sshd;66b26726 (deleted) subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 key=(null)

Hash: sshd,sshd_t,var_log_t,file,write

--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/sshd from read access on the file lastlog.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that sshd should be allowed read access on the lastlog file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'sshd' --raw | audit2allow -M my-sshd
# semodule -i my-sshd.pp


Additional Information:
Source Context                system_u:system_r:sshd_t:s0-s0:c0.c1023
Target Context                system_u:object_r:var_log_t:s0
Target Objects                lastlog [ file ]
Source                        sshd
Source Path                   /usr/sbin/sshd
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           openssh-server-7.4p1-23.el7_9.x86_64
Target RPM Packages
Policy RPM                    selinux-policy-3.13.1-268.el7_9.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     ip-10-0-1-10.ec2.internal
Platform                      Linux ip-10-0-1-10.ec2.internal
                              3.10.0-1160.119.1.el7.x86_64 #1 SMP Tue May 14
                              11:55:25 EDT 2024 x86_64 x86_64
Alert Count                   20
First Seen                    2019-08-26 15:53:07 UTC
Last Seen                     2026-05-20 21:17:24 UTC
Local ID                      0213d868-4ddb-49fe-b3cd-d7dc28f17b03

Raw Audit Messages
type=AVC msg=audit(1779311844.947:82): avc:  denied  { read } for  pid=1234 comm="sshd" name="lastlog" dev="xvda2" ino=43046 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0


type=SYSCALL msg=audit(1779311844.947:82): arch=x86_64 syscall=open success=no exit=EACCES a0=7ffd248ab370 a1=0 a2=180 a3=0 items=0 ppid=990 pid=1234 auid=1002 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=1 comm=sshd exe=/usr/sbin/sshd subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 key=(null)

Hash: sshd,sshd_t,var_log_t,file,read

--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/sshd from 'read, write' accesses on the file lastlog.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that sshd should be allowed read write access on the lastlog file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'sshd' --raw | audit2allow -M my-sshd
# semodule -i my-sshd.pp


Additional Information:
Source Context                system_u:system_r:sshd_t:s0-s0:c0.c1023
Target Context                system_u:object_r:var_log_t:s0
Target Objects                lastlog [ file ]
Source                        sshd
Source Path                   /usr/sbin/sshd
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           openssh-server-7.4p1-23.el7_9.x86_64
Target RPM Packages
Policy RPM                    selinux-policy-3.13.1-268.el7_9.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     ip-10-0-1-10.ec2.internal
Platform                      Linux ip-10-0-1-10.ec2.internal
                              3.10.0-1160.119.1.el7.x86_64 #1 SMP Tue May 14
                              11:55:25 EDT 2024 x86_64 x86_64
Alert Count                   9
First Seen                    2019-08-26 15:53:07 UTC
Last Seen                     2026-05-20 21:17:24 UTC
Local ID                      28d2b104-a5cd-484d-aadf-e2ea95dea2b7

Raw Audit Messages
type=AVC msg=audit(1779311844.947:83): avc:  denied  { read write } for  pid=1234 comm="sshd" name="lastlog" dev="xvda2" ino=43046 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0


type=SYSCALL msg=audit(1779311844.947:83): arch=x86_64 syscall=open success=no exit=EACCES a0=7ffd248aba10 a1=42 a2=180 a3=8 items=0 ppid=990 pid=1234 auid=1002 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=1 comm=sshd exe=/usr/sbin/sshd subj=system_u:system_r:sshd_t:s0-s0:c0.c1023 key=(null)

Hash: sshd,sshd_t,var_log_t,file,read,write

--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/httpd from read access on the file index.html.

*****  Plugin catchall_boolean (89.3 confidence) suggests   ******************

If you want to allow httpd to read user content
Then you must tell SELinux about this by enabling the 'httpd_read_user_content' boolean.

Do
setsebool -P httpd_read_user_content 1

*****  Plugin catchall (11.6 confidence) suggests   **************************

If you believe that httpd should be allowed read access on the index.html file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'httpd' --raw | audit2allow -M my-httpd
# semodule -i my-httpd.pp


Additional Information:
Source Context                system_u:system_r:httpd_t:s0
Target Context                system_u:object_r:user_home_t:s0
Target Objects                index.html [ file ]
Source                        httpd
Source Path                   /usr/sbin/httpd
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           httpd-2.4.6-99.el7_9.1.x86_64
Target RPM Packages
Policy RPM                    selinux-policy-3.13.1-268.el7_9.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     ip-10-0-1-10.ec2.internal
Platform                      Linux ip-10-0-1-10.ec2.internal
                              3.10.0-1160.119.1.el7.x86_64 #1 SMP Tue May 14
                              11:55:25 EDT 2024 x86_64 x86_64
Alert Count                   1
First Seen                    2026-05-20 21:32:55 UTC
Last Seen                     2026-05-20 21:32:55 UTC
Local ID                      8ff1ac06-6b4e-42dc-b577-f4f1402cf45b

Raw Audit Messages
type=AVC msg=audit(1779312775.449:190): avc:  denied  { read } for  pid=1713 comm="httpd" name="index.html" dev="xvda2" ino=18641 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0


type=SYSCALL msg=audit(1779312775.449:190): arch=x86_64 syscall=open success=no exit=EACCES a0=55b6aacba030 a1=80000 a2=0 a3=7ffdb979c920 items=0 ppid=1708 pid=1713 auid=4294967295 uid=48 gid=48 euid=48 suid=48 fsuid=48 egid=48 sgid=48 fsgid=48 tty=(none) ses=4294967295 comm=httpd exe=/usr/sbin/httpd subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: httpd,httpd_t,user_home_t,file,read

[root@ip-10-0-1-10 ~]#
```


4. Lookup the `httpd_read_user_content` boolean:
    
    ```
     getsebool httpd_read_user_content
    ```
    
5. Set the boolean to permit reading user content:
    
    ```
     setsebool -P httpd_read_user_content=1
    ```
    
6. View the web content:
    
    ```
    curl localhost
    ```
    
```
[root@ip-10-0-1-10 ~]# getsebool httpd_read_user_content
httpd_read_user_content --> off
[root@ip-10-0-1-10 ~]# setsebool -P httpd_read_user_content=1
[root@ip-10-0-1-10 ~]# curl localhost
Hello world
[root@ip-10-0-1-10 ~]#
```
## Conclusion

Congratulations, you've completed this hands-on lab!
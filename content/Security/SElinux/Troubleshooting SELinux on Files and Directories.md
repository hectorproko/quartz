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

SELinux is one of those things that quietly breaks services in ways that look like a permissions problem but aren't. In this lab, I worked through a real-world scenario where Apache wouldn't start and a developer's webpage wouldn't serve, both caused by SELinux contexts and booleans rather than standard Linux permissions. The goal was to get comfortable diagnosing these issues using `journalctl`, `ls -lZ`, `restorecon`, and `getsebool`/`setsebool`, instead of just reaching for `chmod` and hoping for the best.
%%## Prerequisites

This lab ran on a pre-provisioned cloud server, no manual setup was required on my end before starting. The environment came with:

- A host instance already running and reachable over SSH (public + private IP provided)
- A `cloud_user` account with credentials supplied by the lab platform
- Apache (`httpd`) already installed, but **not yet started/configured**
- A `developer` user already present on the system, with an `index.html` sitting in their home directory

In other words, the "broken state" (Apache failing to start, SELinux contexts misconfigured) was intentionally pre-set up as the starting point for the lab, I didn't introduce it myself.%%
## The Scenario

A junior sysadmin was setting up a webserver for the development team and ran into SELinux issues he couldn't resolve. Two things needed to get fixed:

1. Apache wouldn't start.
2. Once Apache was running, the developer's `index.html` needed to be moved into their UserDir and served correctly.

## Part 1: Apache Won't Start

### Identifying the Problem

First step, just try to start the service and see what happens.

```bash
systemctl start httpd
```

```
Job for httpd.service failed because the control process exited with error code. See "systemctl status httpd.service" and "journalctl -xe" for details.
```

This failed with a generic message about the control process exiting with an error code, pointing me toward `systemctl status` and `journalctl` for more detail.

```bash
journalctl -xe
```

>**Note:** Even better here is to run `journalctl -xeu httpd` instead of plain `journalctl -xe`.

The relevant lines in the log were:

```
httpd[1355]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name
httpd[1355]: (13)Permission denied: AH00091: httpd: could not open error log file /etc/httpd/...
httpd[1355]: AH00015: Unable to open logs
httpd.service: main process exited, code=exited, status=1/FAILURE
```

So Apache couldn't open its own error log. That's a strong hint this is a context issue, not a "wrong owner" issue.

#### Troubleshooting: Permission Denied on Error Log

**Symptom:** `systemctl start httpd` fails, and the journal shows `Permission denied` when Apache tries to open `/etc/httpd/logs/error_log`.

**Investigation:**
`/etc/httpd/logs/error_log` is a symlink into `/var/log/httpd/error_log`, so I checked the SELinux context on both:

```bash
ls -lZ /etc/httpd/logs/error_log
ls -lZ /var/log/httpd/error_log
```

Both returned:

```
-rw-r--r--. root root unconfined_u:object_r:admin_home_t:s0 error_log
```

**Root cause:** The context `admin_home_t` is wrong for a file living under `/var/log`. It should be using an `httpd_log_t` type. This mismatch is what was blocking Apache from opening the file, regardless of standard Unix permissions.

**Fix:** Restore the correct default context for that path:

```bash
restorecon /var/log/httpd/error_log
```

Confirming the fix:

```bash
ls -lZ /var/log/httpd/error_log
```

```
-rw-r--r--. root root unconfined_u:object_r:httpd_log_t:s0 error_log
```

**Result:** With the context corrected, Apache started without errors:

```bash
systemctl start httpd
systemctl status httpd
```

```
Active: active (running)
```
## Part 2: Serving the Developer's Page from Their Home Directory

With Apache running, the next task was to get `index.html` served out of the developer's home directory using `mod_userdir`.
%%
why this pattern

are you sayning there used to be pages serves with multitenatn and each person could manage their own page?

Yes, exactly that.

Picture an old university or ISP server in the 90s/2000s: one physical machine, dozens or hundreds of unrelated user accounts (students, staff, customers), each with their own home directory like `/home/alice`, `/home/bob`, etc. Apache (or earlier, just plain httpd) ran once on that shared box.

With UserDir enabled, each of those users could:

- Create a `~/public_html` folder in their own home directory
- Drop their own `index.html` (or PHP, CGI scripts, etc.) in there
- Have it instantly live at `http://server.edu/~alice/`, `http://server.edu/~bob/`, etc.

No admin had to:

- Create a new vhost
- Edit Apache's config
- Restart the webserver
- Set up a new directory under `/var/www`

The user just managed their own little corner of the filesystem, and the server-wide UserDir convention exposed it automatically. That's the "multi-tenant, self-service homepage" model — many independent people sharing one server, each fully in control of their own piece of it, with no need to touch shared infrastructure.

It's basically the spiritual ancestor of things like GitHub Pages or a free Geocities/Neocities page — "give me a folder, I'll put my stuff in it, and it's live on the web." The difference is UserDir does it at the OS/filesystem level on a shared Unix box, instead of through a hosting platform's web UI.
%%

### Checking the UserDir Configuration

```bash
cat /etc/httpd/conf.d/userdir.conf
```

The config confirmed Apache expects a `public_html` directory inside each user's home directory (`UserDir public_html`), with specific permission requirements:

- `~user` home directory: `711`
- `~user/public_html`: `755`
- Files inside: world-readable

> [!NOTE]- userdir.conf
> ```bash
> [root@server1 ~]# cat /etc/httpd/conf.d/userdir.conf
> #
> # UserDir: The name of the directory that is appended onto a user's home
> # directory if a ~user request is received.
> #
> # The path to the end user account 'public_html' directory must be
> # accessible to the webserver userid.  This usually means that ~userid
> # must have permissions of 711, ~userid/public_html must have permissions
> # of 755, and documents contained therein must be world-readable.
> # Otherwise, the client will only receive a "403 Forbidden" message.
> #
> <IfModule mod_userdir.c>
>     #
>     # UserDir is disabled by default since it can confirm the presence
>     # of a username on the system (depending on home directory
>     # permissions).
>     #
>     UserDir public_html
> 
>     #
>     # To enable requests to /~user/ to serve the user's public_html
>     # directory, remove the "UserDir public_html" line above, and uncomment
>     # the following line instead:
>     #
>     #UserDir public_html
> </IfModule>
> 
> #
> # Control access to UserDir directories.  The following is an example
> # for a site where these directories are restricted to read-only.
> #
> <Directory "/home/*/public_html">
>     AllowOverride FileInfo AuthConfig Limit Indexes
>     Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
>     Require method GET POST OPTIONS
> </Directory>
> 
> [root@server1 ~]#
> ```

### Setting Up the Directory

Checked whether `public_html` already existed for the `developer` user:

```bash
cd /home/developer
ls -l
```

It didn't, so I created it and set the required permissions:

```bash
mkdir public_html
ls -lZd public_html
```
%%[[ls command#Options]]%%
```
drwxr-xr-x. root root unconfined_u:object_r:httpd_user_content_t:s0 public_html
```

```bash
chmod 0711 /home/developer
chmod 0755 /home/developer/public_html
```

The new directory automatically picked up the `httpd_user_content_t` context, which is what Apache expects to read from.
%%
what did folder newly created not releated to apache have the label httpd_user_content

so basically selinux policy uses pathbase rules to determines what label to give each file created (is not dictated by who creates it or what process uses it)
this related to database (database of path patterns) we restores context from `semanage fcontext -l`

```
/home/[^/]+/public_html(/.*)?    system_u:object_r:httpd_user_content_t:s0
```
%%
### Testing Access

```bash
cd public_html
touch testfile
curl localhost/~developer/testfile
```

This returned a `403 Forbidden` error, even though permissions and context both looked correct.

```
[root@server1 public_html]# curl localhost/~developer/testfile
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /~developer/testfile
on this server.</p>
</body></html>
```
#### Troubleshooting: 403 Forbidden Despite Correct Context

**Symptom:** Apache returns `403 Forbidden` when requesting a file in `public_html`, even though the file has the correct `httpd_user_content_t` context and correct Unix permissions.

**Investigation:**

Checked the audit log for denial entries related to the test file:

```bash
grep testfile /var/log/audit/audit.log
```

Found AVC denials like:

```
avc: denied { getattr } for pid=1377 comm="httpd" path="/home/developer/public_html/testfile" ... scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:httpd_user_content_t:s0 tclass=file permissive=0
```
%%
[[SELinux#Searching Logs with ausearch (Audit Search) ausearch]]
`system_u:system_r:httpd_t:s0` is the label of the **httpd process itself** (the thing doing the accessing), not the file.

That AVC entry was logged because Apache (PID 1377, running under `httpd_t`) attempted a `getattr` (basically a `stat()` call — checking file metadata like size/permissions/existence) on `testfile`, as part of trying to serve it when you ran:

```
curl localhost/~developer/testfile
```
%%

> [!NOTE] SELinux denials come from one of two layers
> 1. **Type enforcement (context-based rules)** - "can `httpd_t` read `httpd_user_content_t`?" → this was already allowed and correctly labeled.
> 2. **Booleans (conditional policy switches)** - extra on/off toggles that layer additional restrictions on top of type enforcement, even when the types check out. Booleans exist for cases where the policy author wants an admin-controlled switch instead of a hard-coded rule *(e.g., "should Apache be allowed to touch home directories at all, even with correct context?")*.

The context was correct, so this pointed toward an SELinux **boolean** rather than a context problem. Checked Apache-related booleans:

```bash
getsebool -a | grep httpd
```

Found `httpd_enable_homedirs --> off`. This boolean specifically controls whether Apache is allowed to serve content out of user home directories at all, separate from file-level contexts.

**Root cause:** SELinux was globally blocking Apache from reading anything under home directories, regardless of file context, because `httpd_enable_homedirs` was disabled.

**Fix:** Enable the boolean:

```bash
setsebool httpd_enable_homedirs on
```

**Result:** ✅ 

```bash
curl localhost/~developer/testfile
# (empty output, file is empty, no error - success)
```

### Moving and Serving index.html

With the boolean fixed, moved the real page into place:

```bash
mv ../index.html .
curl localhost/~developer/index.html
```

This produced another `403 Forbidden`.

```
[root@server1 public_html]# curl localhost/~developer/index.html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /~developer/index.html
on this server.</p>
</body></html>
```
#### Troubleshooting: 403 Forbidden After Moving index.html

**Symptom:** `testfile` (created directly in `public_html`) serves fine, but `index.html` (moved in from the parent directory) still returns `403 Forbidden`.

**Investigation:**

```bash
ls -lZ
```

```
-rw-r--r--. developer developer system_u:object_r:user_home_t:s0       index.html
-rw-r--r--. root      root      unconfined_u:object_r:httpd_user_content_t:s0  testfile
```

**Root cause:** SELinux contexts are tied to where a file was originally created, not where it currently lives. `index.html` was created in `/home/developer`, so it kept the `user_home_t` context even after being moved into `public_html`. Apache can't read `user_home_t` content, which is why `testfile` (created in place) worked but `index.html` (moved in) didn't.

**Fix:** Relabel the file to match its new location's expected context:

```bash
restorecon index.html
ls -lZ
```
%%
[[SELinux]]
%%
```
-rw-r--r--. developer developer system_u:object_r:httpd_user_content_t:s0  index.html
-rw-r--r--. root      root      unconfined_u:object_r:httpd_user_content_t:s0  testfile
```

**Result:**

```bash
curl localhost/~developer/index.html
# This is the developers webpage.
```

The page served correctly.

## Conclusion

Both issues in this lab came down to the same underlying lesson: SELinux tracks its own metadata (contexts and booleans) independently of standard Unix permissions, and that metadata doesn't always update automatically when files are moved or directories are created in unexpected ways. The fix is almost never "loosen permissions further", it's identifying the actual SELinux denial in the audit log, then choosing the precise tool for the job:

- `restorecon` to fix incorrect or stale file/directory contexts
- `setsebool` to enable/disable broader policy behaviors like home directory access

By the end of the lab, Apache was running cleanly and serving the developer's `index.html` correctly from their home directory, with both the startup issue and the UserDir permission issue resolved through SELinux-specific fixes rather than standard `chmod`/`chown` changes.


%%
(https://app.pluralsight.com/hands-on/labs/f5cf55c4-307c-44bf-aafa-d160d859de6c?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

**This lab has been retired.**
This lab is accessible via a direct link but is no longer discoverable in our library. For our most up-to-date content, please visit the [Hands-on page](https://app.pluralsight.com/hands-on/?tab=hands-on-labs&trial=true)

Cloud Server Host instance

Username

cloud_user

Password

14-(&16X

Private ip address of Host instance

10.0.0.30

Public ip address of public instance

34.235.128.51

# Troubleshooting SELinux on Files and Directories

## Introduction

Understanding how to fix potential SELinux issues is important. This lab will present an SELinux problem and allow us to work through the solution, getting us familiar with where to look and how to fix problems.

## The Scenario

A junior sysadmin was trying to set up a webserver for our development team. He's running into some SELinux issues and isn't sure how to fix them.

First of all, Apache won't start. Once that starts correctly, we need to move the developer's `index.html` to their UserDir for Apache, and ensure that the page is served correctly.

### Logging In

Use the credentials provided on the hands-on lab overview page, and log in as `cloud_user`. And once we're in we can just `sudo -i` so we get `root` privileges.

## Identify and Fix the Problem on Startup

Let's try to start Apache right off, to see what happens:

```
# systemctl start httpd
```

We get an error, something about the control process ending with an error code. We do get some advice though, about checking status and using `journalctl`. So let's run that:

```
# journalctl -xe
```

In the output, we're going to see lines similar to this:

```
Jan 09 20:32:46 Server1 httpd[7107]: (13)Permission denied: AH00091: httpd: could not open error log file /etc/httpd/l>
Jan 09 20:32:46 Server1 httpd[7107]: AH00015: Unable to open logs
```


my output

```
Hecti@DESKTOP-7SBLCCA MINGW64 ~
$ ssh cloud_user@34.235.128.51
The authenticity of host '34.235.128.51 (34.235.128.51)' can't be established.
ED25519 key fingerprint is SHA256:f/Eu0iFrUZt4viZYaKKQC3t7Gpd4sIb4crIQYksHbVo.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '34.235.128.51' (ED25519) to the list of known hosts.
(cloud_user@34.235.128.51) Password:
[cloud_user@server1 ~]$ sudo -i
[sudo] password for cloud_user:
[root@server1 ~]# systemctl start httpd
Job for httpd.service failed because the control process exited with error code. See "systemctl status httpd.service" and "journalctl -xe" for details.
[root@server1 ~]# journalctl -xe
-- The start-up result is done.
Jun 25 13:24:36 server1 systemd-logind[495]: New session 1 of user cloud_user.
-- Subject: A new session 1 has been created for user cloud_user
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- Documentation: http://www.freedesktop.org/wiki/Software/systemd/multiseat
--
-- A new session with the ID 1 has been created for the user cloud_user.
--
-- The leading process of the session is 1303.
Jun 25 13:24:36 server1 sshd[1303]: pam_unix(sshd:session): session opened for user cloud_user by (uid=0)
Jun 25 13:24:36 server1 sshd[1303]: pam_lastlog(sshd:session): unable to open /var/log/lastlog: Permission denied
Jun 25 13:24:50 server1 sudo[1330]: cloud_user : TTY=pts/0 ; PWD=/home/cloud_user ; USER=root ; COMMAND=/bin/bash
Jun 25 13:24:50 server1 sudo[1330]: pam_unix(sudo-i:session): session opened for user root by cloud_user(uid=0)
Jun 25 13:25:07 server1 polkitd[498]: Registered Authentication Agent for unix-process:1349:49768 (system bus name
Jun 25 13:25:07 server1 systemd[1]: Starting The Apache HTTP Server...
-- Subject: Unit httpd.service has begun start-up
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
--
-- Unit httpd.service has begun starting up.
Jun 25 13:25:07 server1 httpd[1355]: AH00558: httpd: Could not reliably determine the server's fully qualified doma
Jun 25 13:25:07 server1 httpd[1355]: (13)Permission denied: AH00091: httpd: could not open error log file /etc/http
Jun 25 13:25:07 server1 httpd[1355]: AH00015: Unable to open logs
Jun 25 13:25:07 server1 systemd[1]: httpd.service: main process exited, code=exited, status=1/FAILURE
Jun 25 13:25:07 server1 systemd[1]: Failed to start The Apache HTTP Server.
-- Subject: Unit httpd.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
--
-- Unit httpd.service has failed.
--
-- The result is failed.
Jun 25 13:25:07 server1 systemd[1]: Unit httpd.service entered failed state.
Jun 25 13:25:07 server1 systemd[1]: httpd.service failed.
Jun 25 13:25:07 server1 polkitd[498]: Unregistered Authentication Agent for unix-process:1349:49768 (system bus nam
lines 1968-2003/2003 (END)

```


It looks like we're having a permission problem with the error log, `/var/log/httpd/error_log`. Let's investigate:

```
# ls -lZ /etc/httpd/logs/error_log

-rw-r--r--. 1 root root unconfined_u:object_r:admin_home_t:s0 0 Jan  9 20:17 /etc/httpd/logs/error_log
```

This is just a symbolic link over to a spot in `/var/log`, so let's look at that one too:

```
# ls -lZ /var/log/httpd/error_log

-rw-r--r--. 1 root root unconfined_u:object_r:admin_home_t:s0 0 Jan  9 20:17 /var/log/httpd/error_log
```

The `admin_home` in this isn't the right context for the `/var/log` directory. So let's use `restorecon` to change it:

```
# restorecon /var/log/httpd/error_log
```

Now if we look at the details again, we'll see that the context has changed to `httpd_log`:

```
# ls -lZ /var/log/httpd/error_log

-rw-r--r--. 1 root root unconfined_u:object_r:httpd_log_t:s0 0 Jan  9 20:17 /var/log/httpd/error_log
```

Let's see if we can start Apache now:

```
# systemctl start httpd
```

We get no errors, so we're good to go.

```
[root@server1 ~]# ls -lZ /etc/httpd/logs/error_log
-rw-r--r--. root root unconfined_u:object_r:admin_home_t:s0 /etc/httpd/logs/error_log
[root@server1 ~]# ls -lZ /var/log/httpd/error_log
-rw-r--r--. root root unconfined_u:object_r:admin_home_t:s0 /var/log/httpd/error_log
[root@server1 ~]# restorecon /var/log/httpd/error_log
[root@server1 ~]# ls -lZ /var/log/httpd/error_log
-rw-r--r--. root root unconfined_u:object_r:httpd_log_t:s0 /var/log/httpd/error_log
[root@server1 ~]# systemctl start httpd
[root@server1 ~]#
```

```
[root@server1 ~]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2026-06-25 13:27:24 EDT; 24s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 1372 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─1372 /usr/sbin/httpd -DFOREGROUND
           ├─1373 /usr/sbin/httpd -DFOREGROUND
           ├─1374 /usr/sbin/httpd -DFOREGROUND
           ├─1375 /usr/sbin/httpd -DFOREGROUND
           ├─1376 /usr/sbin/httpd -DFOREGROUND
           └─1377 /usr/sbin/httpd -DFOREGROUND

Jun 25 13:27:24 server1 systemd[1]: Starting The Apache HTTP Server...
Jun 25 13:27:24 server1 httpd[1372]: AH00558: httpd: Could not reliably determine the server's fully qualif...ssage
Jun 25 13:27:24 server1 systemd[1]: Started The Apache HTTP Server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@server1 ~]#
```
## Fix the Problem with Home Directories

Now let's check on what's happening with home directories. Run this to see where they should be:

```
# cat /etc/httpd/conf.d/userdir.conf
```

```
[root@server1 ~]# cat /etc/httpd/conf.d/userdir.conf
#
# UserDir: The name of the directory that is appended onto a user's home
# directory if a ~user request is received.
#
# The path to the end user account 'public_html' directory must be
# accessible to the webserver userid.  This usually means that ~userid
# must have permissions of 711, ~userid/public_html must have permissions
# of 755, and documents contained therein must be world-readable.
# Otherwise, the client will only receive a "403 Forbidden" message.
#
<IfModule mod_userdir.c>
    #
    # UserDir is disabled by default since it can confirm the presence
    # of a username on the system (depending on home directory
    # permissions).
    #
    UserDir public_html

    #
    # To enable requests to /~user/ to serve the user's public_html
    # directory, remove the "UserDir public_html" line above, and uncomment
    # the following line instead:
    #
    #UserDir public_html
</IfModule>

#
# Control access to UserDir directories.  The following is an example
# for a site where these directories are restricted to read-only.
#
<Directory "/home/*/public_html">
    AllowOverride FileInfo AuthConfig Limit Indexes
    Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
    Require method GET POST OPTIONS
</Directory>

[root@server1 ~]#
```

According to the `UserDir` line, we should have a `public_html` directory in the `developer` user's home directory. Is there one? Let's check:

```
# cd /home/developer
# ls -l
```

There isn't, so we've got to create it. We'll also have to set the permissions on it to 755, and set the permissions on `/home/developer` to 711. First, make the directory:

```
# mkdir public_html
```

Check if its context is correct:

```
# ls -lZd public_html
```

We should get `httpd_user_content` in the output. Now we can set the permissions:

```
# chmod 0711 /home/developer
# chmod 0755 /home/developer/public_html
```

Let's create a test file and check that we can see it via Apache:

```
# cd public_html
# touch testfile
# curl localhost/~developer/testfile
```

Oh, we get a _Forbidden_ error. We can check out a log to see what's happening:

```
# grep testfile /var/log/audit/audit.log
```

There are a couple of `avc` errors in there. We know that the context is set properly, so the next step is to look at Booleans. Does Apache have any related to home directories? Let's check:

```
# getsebool -a | grep httpd
```


my output

```
[root@server1 ~]# cd /home/developer/
[root@server1 developer]# ls -l
total 4
-rw-r--r--. 1 developer developer 32 Jun 25 13:19 index.html
[root@server1 developer]# mkdir public_html
[root@server1 developer]# ls -lZd public_html
drwxr-xr-x. root root unconfined_u:object_r:httpd_user_content_t:s0 public_html
[root@server1 developer]# chmod 0711 /home/developer
[root@server1 developer]# chmod 0755 /home/developer/public_html
[root@server1 developer]# cd public_html
[root@server1 public_html]# touch testfile
[root@server1 public_html]# curl localhost/~developer/testfile
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /~developer/testfile
on this server.</p>
</body></html>
[root@server1 public_html]# grep testfile /var/log/audit/audit.log
type=AVC msg=audit(1782408705.483:162): avc:  denied  { getattr } for  pid=1377 comm="httpd" path="/home/developer/public_html/testfile" dev="xvda1" ino=683242 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:httpd_user_content_t:s0 tclass=file permissive=0
type=AVC msg=audit(1782408705.484:163): avc:  denied  { getattr } for  pid=1377 comm="httpd" path="/home/developer/public_html/testfile" dev="xvda1" ino=683242 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:httpd_user_content_t:s0 tclass=file permissive=0
[root@server1 public_html]# getsebool -a | grep httpd
httpd_anon_write --> off
httpd_builtin_scripting --> on
httpd_can_check_spam --> off
httpd_can_connect_ftp --> off
httpd_can_connect_ldap --> off
httpd_can_connect_mythtv --> off
httpd_can_connect_zabbix --> off
httpd_can_network_connect --> off
httpd_can_network_connect_cobbler --> off
httpd_can_network_connect_db --> off
httpd_can_network_memcache --> off
httpd_can_network_relay --> off
httpd_can_sendmail --> off
httpd_dbus_avahi --> off
httpd_dbus_sssd --> off
httpd_dontaudit_search_dirs --> off
httpd_enable_cgi --> on
httpd_enable_ftp_server --> off
httpd_enable_homedirs --> off
httpd_execmem --> off
httpd_graceful_shutdown --> on
httpd_manage_ipa --> off
httpd_mod_auth_ntlm_winbind --> off
httpd_mod_auth_pam --> off
httpd_read_user_content --> off
httpd_run_ipa --> off
httpd_run_preupgrade --> off
httpd_run_stickshift --> off
httpd_serve_cobbler_files --> off
httpd_setrlimit --> off
httpd_ssi_exec --> off
httpd_sys_script_anon_write --> off
httpd_tmp_exec --> off
httpd_tty_comm --> off
httpd_unified --> off
httpd_use_cifs --> off
httpd_use_fusefs --> off
httpd_use_gpg --> off
httpd_use_nfs --> off
httpd_use_openstack --> off
httpd_use_sasl --> off
httpd_verify_dns --> off
[root@server1 public_html]#
```



Aha! There's one called `httpd_enable_homedirs` and it's set to `off`. Let's turn it on and see what happens:

```
# setsebool httpd_enable_homedirs on
```

Now the `curl` should work correctly:

```
# curl localhost/~developer/testfile
```

That file is empty, so there shouldn't be any output at all. No errors means we're good to go. Now, let's get the `index.html` file moved into the `public_html` directory and test:

```
# mv ../index.html .
# curl localhost/~developer/index.html
```

We get another permission error. Let's check on what's going on:

```
# ls -lZ
```

The context is different for the two files, because `index.html` was created in `/home/developer`, not in the `public_html` directory. Let's fix that and check:

```
# restorecon index.html
# ls -lZ
```

Now both files have the `http_user_content` context. Let's try the `curl` command again:

```
# curl localhost/~developer/index.html
```

We should see the contents of `index.html`.

```
[root@server1 public_html]# setsebool httpd_enable_homedirs on
[root@server1 public_html]# curl localhost/~developer/testfile
[root@server1 public_html]# mv ../index.html .
[root@server1 public_html]# curl localhost/~developer/index.html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /~developer/index.html
on this server.</p>
</body></html>
[root@server1 public_html]# ls -lZ
-rw-r--r--. developer developer system_u:object_r:user_home_t:s0 index.html
-rw-r--r--. root      root      unconfined_u:object_r:httpd_user_content_t:s0 testfile
[root@server1 public_html]# restorecon index.html
[root@server1 public_html]# ls -lZ
-rw-r--r--. developer developer system_u:object_r:httpd_user_content_t:s0 index.html
-rw-r--r--. root      root      unconfined_u:object_r:httpd_user_content_t:s0 testfile
[root@server1 public_html]# curl localhost/~developer/index.html
This is the developers webpage.
[root@server1 public_html]#
```
## Conclusion

We managed to get Apache up and running, and we have pages being served correctly out of our developers home directory. Congratulations!
%%
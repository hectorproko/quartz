---
tags:
  - selinux
  - security
linkedin: "False"
quartz: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
hardlinked: "True"
completed: "True"
---
## Introduction

SELinux provides powerful tools like **booleans** and **port labels/types** to adjust security policy without deep policy-writing knowledge, but those tools only cover what the system already knows about. When you deploy your own custom daemons or services, SELinux has no built-in context for them. You need to write a policy from scratch.

In this lab, I work through the full lifecycle of creating a custom SELinux policy for a custom binary called `apachelogger`. The process involves three phases:

1. **Generating** a skeleton policy using `sepolicy`
2. **Diagnosing** what the policy is missing by reading SELinux audit logs with `ausearch`
3. **Refining** the policy by adding the correct interface calls and reapplying it

The goal is to move from a running-but-unconstrained daemon to one that is properly confined by SELinux with only the permissions it actually needs.
<!--
So the most accurate plain-English description would be **"port labeling"** or **"labeling ports with a context"**. The intro sentence could read:

- The **label** is the full context string, e.g. `system_u:object_r:http_port_t:s0`
- The **type** is the meaningful part of that label, e.g. `http_port_t`
- When you run `semanage port -a`, you are **labeling** the port by assigning it a context
  
semanage port -l          # list port types
semanage port -a -t http_port_t -p tcp 8080   # assign a port type
-->
---

## Environment

**OS:** RHEL 9 (AWS cloud instance)
**Tools used:** `sepolicy`, `ausearch`, `audit2allow`, `vim`, `systemctl`, `ps`

---

## Step 1 - Inspect the Custom Daemon

Before writing any policy, I need to understand what the daemon is and where it lives. I start by checking its status and reading its unit file.

```bash
mkdir policy
cd policy
sudo systemctl status apachelogger
```

**Example output:**

```
● apachelogger.service - Simple testing daemon
     Loaded: loaded (/usr/lib/systemd/system/apachelogger.service; disabled; preset: disabled)
     Active: active (running) since Wed 2026-05-20 13:42:08 UTC; 3min 42s ago
   Main PID: 2596 (apachelogger)
      Tasks: 1 (limit: 11750)
     Memory: 196.0K (peak: 452.0K)
        CPU: 7ms
     CGroup: /system.slice/apachelogger.service
             └─2596 /usr/local/bin/apachelogger
```

The daemon is active. I note its binary path: `/usr/local/bin/apachelogger`. I confirm this by reading the unit file directly:

```bash
cat /usr/lib/systemd/system/apachelogger.service
```
<!--
remember service unit (service file, systemd)  points to the execucate
-->
**Example output:**

```ini
[Unit]
Description=Simple testing daemon

[Service]
Type=simple
ExecStart=/usr/local/bin/apachelogger

[Install]
WantedBy=multi-user.target
```

The `ExecStart` line confirms the binary path, this is what I'll pass to `sepolicy generate`.

---

## Step 2 - Generate the Policy Skeleton

`sepolicy generate` builds a skeleton policy from the binary. The `--init` flag tells it the binary is a system daemon (started by `init`/systemd), which shapes the type enforcement rules it creates.

```bash
sudo sepolicy generate --init /usr/local/bin/apachelogger
```

### Troubleshooting: RHUI Repository Error

Running this command on the lab server initially fails with a DNS resolution error:

```
Errors during downloading metadata for repository 'rhel-9-appstream-rhui-rpms':
  - Curl error (6): Couldn't resolve host name for
    https://rhui.REGION.aws.ce.redhat.com/...
dnf.exceptions.RepoError: Failed to download metadata for repo
  'rhel-9-appstream-rhui-rpms': Cannot prepare internal mirrorlist
```

**Why this happens:** Under the hood, `sepolicy generate` calls `dnf` to query system repositories for RPM metadata. This lab environment has no outbound internet access, so DNS resolution for AWS RHUI servers fails entirely.

**Fix:** Temporarily disable the RHUI repos so the command can run in an offline context:

```bash
sudo dnf config-manager --disable "*rhui*"
sudo sepolicy generate --init /usr/local/bin/apachelogger
```

**Example output after fix:**

```
Created the following files:
/home/cloud_user/policy/apachelogger.te   # Type Enforcement file
/home/cloud_user/policy/apachelogger.if   # Interface file
/home/cloud_user/policy/apachelogger.fc   # File Contexts file
/home/cloud_user/policy/apachelogger_selinux.spec  # Spec file
/home/cloud_user/policy/apachelogger.sh   # Setup Script
```

Verifying the files are present:

```bash
ls
```

```
apachelogger.fc  apachelogger.if  apachelogger_selinux.spec  apachelogger.sh  apachelogger.te
```

Each file plays a specific role:

|File|Purpose|
|---|---|
|`apachelogger.te`|Type Enforcement - defines the actual allow rules|
|`apachelogger.if`|Interface - exposes macros for other policies to use|
|`apachelogger.fc`|File Contexts - maps file paths to SELinux labels|
|`apachelogger.sh`|Build script - compiles and loads the policy module|
|`apachelogger_selinux.spec`|RPM spec - packages the policy for distribution|

---

## Step 3 - Apply the Initial Policy

I run the generated build script to compile and load the policy into the kernel:

```bash
sudo ./apachelogger.sh
```
<!--
No. `sepolicy generate` produces this script as a standard output for any policy you generate. The script is auto-generated and named after whatever binary you pass in — in this case `apachelogger.sh` because you ran:
-->

> [!NOTE]- apachelogger.sh
> ```bash
> [cloud_user@web policy]$ sudo cat ./apachelogger.sh
> [sudo] password for cloud_user:
> #!/bin/sh -e
> 
> DIRNAME=`dirname $0`
> cd $DIRNAME
> USAGE="$0 [ --update ]"
> if [ `id -u` != 0 ]; then
> echo 'You must be root to run this script'
> exit 1
> fi
> 
> if [ $# -eq 1 ]; then
>         if [ "$1" = "--update" ] ; then
>                 time=`ls -l --time-style="+%x %X" apachelogger.te | awk '{ printf "%s %s", $6, $7 }'`
>                 rules=`ausearch --start $time -m avc --raw -se apachelogger`
>                 if [ x"$rules" != "x" ] ; then
>                         echo "Found avc's to update policy with"
>                         echo -e "$rules" | audit2allow -R
>                         echo "Do you want these changes added to policy [y/n]?"
>                         read ANS
>                         if [ "$ANS" = "y" -o "$ANS" = "Y" ] ; then
>                               echo "Updating policy"
>                               echo -e "$rules" | audit2allow -R >> apachelogger.te
>                               # Fall though and rebuild policy
>                         else
>                               exit 0
>                         fi
>                 else
>                         echo "No new avcs found"
>                         exit 0
>                 fi
>         else
>                 echo -e $USAGE
>                 exit 1
>         fi
> elif [ $# -ge 2 ] ; then
>         echo -e $USAGE
>         exit 1
> fi
> 
> echo "Building and Loading Policy"
> set -x
> make -f /usr/share/selinux/devel/Makefile apachelogger.pp || exit
> /usr/sbin/semodule -i apachelogger.pp
> 
> # Generate a man page of the installed module
> sepolicy manpage -p . -d apachelogger_t
> # Fixing the file context on /usr/local/bin/apachelogger
> /sbin/restorecon -F -R -v /usr/local/bin/apachelogger
> # Generate a rpm package for the newly generated policy
> 
> pwd=$(pwd)
> rpmbuild --define "_sourcedir ${pwd}" --define "_specdir ${pwd}" --define "_builddir ${pwd}" --define "_srcrpmdir ${pwd}" --define "_rpmdir ${pwd}" --define "_buildrootdir ${pwd}/.build"  -ba apachelogger_selinux.spec
> [cloud_user@web policy]$
> ```

The script does several things automatically: it compiles the `.te` file into a binary policy package (`.pp`), loads it with `semodule`, restores file contexts with `restorecon`, and generates a man page for the new domain. A key excerpt from the script shows the build steps:

```bash
make -f /usr/share/selinux/devel/Makefile apachelogger.pp || exit
/usr/sbin/semodule -i apachelogger.pp
sepolicy manpage -p . -d apachelogger_t
/sbin/restorecon -F -R -v /usr/local/bin/apachelogger
```
<!--[[restorecon (Restore Context)|restorecon]]
why do we use restorecon after loading the poliy .pp? so the context gets restored from the newly appplied policy? YES
loading the policy does **not** automatically relabel files already on disk — the binary still has whatever context it had before
-->
After the policy is loaded, I restart the daemon and confirm it now runs under the correct SELinux type:

```bash
sudo systemctl restart apachelogger
ps -efZ | grep apachelogger
```

**Example output:**

```
system_u:system_r:apachelogger_t:s0  root  2965  1  0 14:39 ?  00:00:00 /usr/local/bin/apachelogger
```

The `apachelogger_t` type is now present in the process label, meaning the daemon is running under its own SELinux domain. However, the policy is set to `permissive` mode for `apachelogger_t`, meaning denials are logged but not enforced yet. This is intentional, as I want to capture what the daemon actually tries to do before enforcing.

---

## Step 4 - Diagnose Access Denials

With the daemon running in permissive mode, I query the audit log to see what it was denied:
<!--
so  you can scope selinux modes to daemons, each daemon can have diffirent selinux modes?
Yes, exactly. SELinux lets you set `permissive` mode on a per-domain basis, so the rest of the system stays in full enforcing mode while just one domain logs-but-doesn't-block.

[[SELinux#^accc96]]
--> 
```bash
sudo ausearch -m AVC -ts recent | grep apachelogger
```
<!--
[[ausearch (Audit Search)|ausearch]]
`-m` stands for "message type."
-ts time start
recenet keyword for  10minutes
[[ausearch (Audit Search)|ausearch]]
--> 
[[AVC (Access Vector Cache)]] 

**Example output (relevant AVC lines):**

```
type=AVC msg=audit(...): avc: denied { open } for pid=2965 comm="apachelogger"
  path="/var/log/httpd/access_log"
  scontext=system_u:system_r:apachelogger_t:s0
  tcontext=system_u:object_r:httpd_log_t:s0 tclass=file permissive=1

type=AVC msg=audit(...): avc: denied { write } for pid=2965 comm="apachelogger"
  name="access_log"
  scontext=system_u:system_r:apachelogger_t:s0
  tcontext=system_u:object_r:httpd_log_t:s0 tclass=file permissive=1

type=AVC msg=audit(...): avc: denied { search } for pid=2965 comm="apachelogger"
  name="httpd"
  scontext=system_u:system_r:apachelogger_t:s0
  tcontext=system_u:object_r:httpd_log_t:s0 tclass=dir permissive=1
```

Reading these AVC messages tells the story clearly:

- **Source context (`scontext`):** `apachelogger_t` - the daemon's domain
- **Target context (`tcontext`):** `httpd_log_t` - Apache log files type
- **Denied actions:** `open`, `write` on the file; `search` on the directory

The daemon is trying to open and write to `/var/log/httpd/access_log`, but its policy gives it no permission to touch anything labeled `httpd_log_t`.

>👀 `permissive=1` means SELinux **logged the denial but did not enforce it**, the action went through anyway.

---

## Step 5 - Find the Right Policy Interfaces

Rather than writing raw `allow` rules, SELinux policy uses reusable **interfaces**, named macros that group related permissions together. I use `audit2allow` to suggest which interfaces apply here:

```bash
sudo ausearch -m AVC -ts recent | audit2allow -R
```
<!--
- `ausearch -m AVC -ts recent` — searches the audit log for **AVC (Access Vector Cache) denial messages** from recent activity. These are the "SELinux blocked this" events.
- The pipe `|` feeds those denial entries into `audit2allow -R` — the `-R` flag means **Reference Policy**, so instead of generating raw `allow` rules, it looks up whether an existing named interface already covers those denied permissions and recommends it by name.
-->
The output suggests two interfaces:

- `apache_manage_log(apachelogger_t)` - grants full management of Apache log files
- `apache_read_log(apachelogger_t)` - grants read-only access to Apache log files

To understand what each one actually does before adding them, I look them up in the SELinux interface library:

```bash
grep -r "apache_manage_log" /usr/share/selinux/devel/include/ | grep .if
grep -r "apache_read_log" /usr/share/selinux/devel/include/ | grep .if
```

Both point to `/usr/share/selinux/devel/include/contrib/apache.if`. Inspecting that file shows their definitions:

**`apache_manage_log`** - allows the domain to fully manage Apache log directories and files:

```
interface(`apache_manage_log',`
        gen_require(`
                type httpd_log_t;
        ')
        logging_search_logs($1)
        manage_dirs_pattern($1, httpd_log_t, httpd_log_t)
        manage_files_pattern($1, httpd_log_t, httpd_log_t)
        read_lnk_files_pattern($1, httpd_log_t, httpd_log_t)
')
```

**`apache_read_log`** - allows the domain to read Apache log files (list directory, read files and symlinks):

```
interface(`apache_read_log',`
        gen_require(`
                type httpd_log_t;
        ')
        logging_search_logs($1)
        allow $1 httpd_log_t:dir list_dir_perms;
        read_files_pattern($1, httpd_log_t, httpd_log_t)
        read_lnk_files_pattern($1, httpd_log_t, httpd_log_t)
')
```

Since `apachelogger` writes to the log file, it needs `apache_manage_log`. I add both to be thorough, covering read and write paths.
<!--
**`apache_manage_log` already includes read**, so `apache_read_log` is the redundant one.
we added both at the end at the bottom
-->
---

## Step 6 - Update the Type Enforcement File

I edit `apachelogger.te` to add the two interface calls:

```bash
sudo vim apachelogger.te
```
<!--
why we adding it to the apachelogger.te?

Because `apachelogger.te` is the **Type Enforcement file** — it's the file that defines what your custom type (`apachelogger_t`) is allowed to do.
-->
I scroll to the bottom of the file and append the two lines:

```
apache_manage_log(apachelogger_t)
apache_read_log(apachelogger_t)
```

After saving, the complete `.te` file looks like this:

```
policy_module(apachelogger, 1.0.0)

########################################
#
# Declarations
#

type apachelogger_t;
type apachelogger_exec_t;
init_daemon_domain(apachelogger_t, apachelogger_exec_t)

permissive apachelogger_t;

########################################
#
# apachelogger local policy
#
allow apachelogger_t self:fifo_file rw_fifo_file_perms;
allow apachelogger_t self:unix_stream_socket create_stream_socket_perms;

domain_use_interactive_fds(apachelogger_t)

files_read_etc_files(apachelogger_t)

miscfiles_read_localization(apachelogger_t)
apache_manage_log(apachelogger_t)
apache_read_log(apachelogger_t)
```

---

## Step 7 - Reapply and Verify

With the updated policy in place, I rebuild and reload it:

```bash
sudo ./apachelogger.sh
sudo systemctl restart apachelogger
```

Finally, I check the audit log one more time to confirm there are no new AVC denials:

```bash
sudo ausearch -m AVC -ts recent
```

**Example output:**

```
<no matches>
```

No denials. The daemon is now properly confined, running under its own `apachelogger_t` domain, with exactly the permissions it needs to access Apache log files and generating no AVC errors.
<!--
no output because the process is able to acces the things now therefore no errors?

Exactly. The audit log is silent because there's nothing to report.
-->
---

## Summary

|Phase|Tool|What it does|
|---|---|---|
|Inspect|`systemctl`, `cat`|Locate the binary and understand the service|
|Generate|`sepolicy generate`|Create a skeleton policy from the binary|
|Apply|`./apachelogger.sh`|Compile and load the policy module|
|Diagnose|`ausearch`, `audit2allow`|Find what the daemon was denied|
|Research|`grep`, `less` on `.if` files|Understand what each interface permits|
|Refine|`vim apachelogger.te`|Add the correct interface calls|
|Verify|`ausearch`|Confirm no remaining denials|

The key insight in this workflow is the loop between **applying**, **checking logs**, and **refining**, as the permissive mode flag on `apachelogger_t` is what makes this iterative approach safe. The daemon runs, its access patterns get recorded, and you can tune the policy without ever breaking the service mid-diagnosis.
<!--
Exactly — it's a feedback loop:

1. **Apply** the policy → load it into the kernel
2. **Run** the daemon → let it do its thing
3. **Check logs** → see what SELinux would have blocked
4. **Refine** the `.te` file → add the missing permissions
5. **Reapply** → go back to step 2

And the reason `permissive` mode is what makes this safe is that at step 3, the daemon **didn't actually fail**. It kept running and doing its job the whole time. Without permissive mode, every missing permission would have caused a real access denial — the daemon would have crashed or malfunctioned while you were still figuring out what it needed.

So permissive mode lets you **observe without disrupting**, build up the complete picture of what the daemon needs, and only lock it down to enforcing once the policy is fully tuned.
__________________________________________________
-->

<!--
####### Draft

`https://app.pluralsight.com/hands-on/labs/487358cb-ebf0-4010-b71d-e4207cb33aa3?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F`

Cloud Server Logging Server

Username
cloud_user

Password
8t4ibV]C

Logging Server Private IP
10.0.1.101

Logging Server Public IP
54.205.237.251
# Working with SELinux Booleans and Ports

## Introduction

SELinux has a number of tools to alter policy without real knowledge of writing policy itself, but when we create our own **custom daemons and processes**, SELinux won't have default settings readily available for us to use. Instead, we'll have to use a combination of tools and troubleshooting to develop a custom policy so that your custom work can work. In this lab, we'll use `sepolicy`, `ausearch`, and `audit2allow` to create a policy, implement it, and then refine the policy through troubleshooting.

## Solution

Log in to the server using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP>
```

## Generate Policy

1. Create the `policy` directory:
    
    ```
    mkdir policy
    ```
    
2. Move to the new directory:
    
    ```
    cd policy
    ```
    
3. Check the status of `apachelogger`:
    
    ```
    sudo systemctl status apachelogger
    ```
    
    Observe it is currently running. Also observe a reference to the service file listed next to `Loaded`. Press **CTRL + C** to exit the log.
    
4. Review the service file:
    
    ```
    cat /usr/lib/systemd/system/apachelogger.service
    ```
    
    Observe from this file the location of `apachelogger` is `/usr/local/bin/apachelogger`.


```
[cloud_user@web ~]$ mkdir policy
[cloud_user@web ~]$ cd policy/
[cloud_user@web policy]$ sudo systemctl status apachelogger
[sudo] password for cloud_user:
● apachelogger.service - Simple testing daemon
     Loaded: loaded (/usr/lib/systemd/system/apachelogger.service; disabled; preset: disabled)
     Active: active (running) since Wed 2026-05-20 13:42:08 UTC; 3min 42s ago
   Main PID: 2596 (apachelogger)
      Tasks: 1 (limit: 11750)
     Memory: 196.0K (peak: 452.0K)
        CPU: 7ms
     CGroup: /system.slice/apachelogger.service
             └─2596 /usr/local/bin/apachelogger

May 20 13:42:08 web systemd[1]: Started Simple testing daemon.
[cloud_user@web policy]$ cat /usr/lib/systemd/system/apachelogger.service
[Unit]
Description=Simple testing daemon

[Service]
Type=simple
ExecStart=/usr/local/bin/apachelogger

[Install]
WantedBy=multi-user.target
[cloud_user@web policy]$
```


1. Create a policy based on this information:
    
    ```
    sudo sepolicy generate --init /usr/local/bin/apachelogger
    ```
    
2. Check that the policy was created in the current directory:
    
    ```
    ls
    ```
    
    You should see a number of files that will generate a policy for the service listed in the output.
    

```
❌[cloud_user@web policy]$ sudo sepolicy generate --init /usr/local/bin/apachelogger
Errors during downloading metadata for repository 'rhel-9-appstream-rhui-rpms':
  - Curl error (6): Couldn't resolve host name for https://rhui.REGION.aws.ce.redhat.com/pulp/mirror/content/dist/rhel9/rhui/9/x86_64/appstream/os [Could not resolve host: rhui.REGION.aws.ce.redhat.com]
Traceback (most recent call last):
  File "/usr/lib/python3.9/site-packages/dnf/repo.py", line 574, in load
    ret = self._repo.load()
  File "/usr/lib64/python3.9/site-packages/libdnf/repo.py", line 331, in load
    return _repo.Repo_load(self)
libdnf._error.Error: Failed to download metadata for repo 'rhel-9-appstream-rhui-rpms': Cannot prepare internal mirrorlist: Curl error (6): Couldn't resolve host name for https://rhui.REGION.aws.ce.redhat.com/pulp/mirror/content/dist/rhel9/rhui/9/x86_64/appstream/os [Could not resolve host: rhui.REGION.aws.ce.redhat.com]

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/bin/sepolicy", line 702, in <module>
    args.func(args)
  File "/bin/sepolicy", line 569, in generate
    mypolicy.gen_writeable()
  File "/usr/lib/python3.9/site-packages/sepolicy/generate.py", line 1304, in gen_writeable
    self.__extract_rpms()
  File "/usr/lib/python3.9/site-packages/sepolicy/generate.py", line 1271, in __extract_rpms
    base.fill_sack(load_system_repo=True)
  File "/usr/lib/python3.9/site-packages/dnf/base.py", line 413, in fill_sack
    self._add_repo_to_sack(r)
  File "/usr/lib/python3.9/site-packages/dnf/base.py", line 142, in _add_repo_to_sack
    repo.load()
  File "/usr/lib/python3.9/site-packages/dnf/repo.py", line 581, in load
    raise dnf.exceptions.RepoError(str(e))
dnf.exceptions.RepoError: Failed to download metadata for repo 'rhel-9-appstream-rhui-rpms': Cannot prepare internal mirrorlist: Curl error (6): Couldn't resolve host name for https://rhui.REGION.aws.ce.redhat.com/pulp/mirror/content/dist/rhel9/rhui/9/x86_64/appstream/os [Could not resolve host: rhui.REGION.aws.ce.redhat.com]
[cloud_user@web policy]$ ls
```

why
That error pops up because under the hood, `sepolicy generate` uses `dnf` to query your system's repositories. Right now, your system has no internet access or its DNS settings are broken, so it can't resolve the AWS Red Hat Update Infrastructure (RHUI) repository URL (`Curl error (6): Couldn't resolve host name`).

The error is that `sepolicy generate` tries to fetch RPM metadata from Red Hat's RHUI repos, but the lab environment can't reach those servers. The fix is to disable those repos temporarily so the command can run offline.

✅ temporarily disable **all** repos for just this command:
```
sudo dnf config-manager --disable "*rhui*"
sudo sepolicy generate --init /usr/local/bin/apachelogger
```

```
[cloud_user@web policy]$ sudo dnf config-manager --disable "*rhui*"
Updating Subscription Management repositories.
Unable to read consumer identity

This system is not registered with an entitlement server. You can use subscription-manager to register.

[cloud_user@web policy]$ sudo sepolicy generate --init /usr/local/bin/apachelogger
Created the following files:
/home/cloud_user/policy/apachelogger.te # Type Enforcement file
/home/cloud_user/policy/apachelogger.if # Interface file
/home/cloud_user/policy/apachelogger.fc # File Contexts file
/home/cloud_user/policy/apachelogger_selinux.spec # Spec file
/home/cloud_user/policy/apachelogger.sh # Setup Script

[cloud_user@web policy]$

```

apachelogger seems like a binary

```
[cloud_user@web policy]$ ls
apachelogger.fc  apachelogger.if  apachelogger_selinux.spec  apachelogger.sh  apachelogger.te
[cloud_user@web policy]$

```
### Troubleshoot Policy

Apply the generated policy, then review SELinux logs to see if any parts of the daemon are blocked by the new policy. Rewrite the policy to allow the appropriate access.

1. Implement the policy:
    
    ```
    sudo ./apachelogger.sh
    ```

```
[cloud_user@web policy]$ sudo cat ./apachelogger.sh
[sudo] password for cloud_user:
#!/bin/sh -e

DIRNAME=`dirname $0`
cd $DIRNAME
USAGE="$0 [ --update ]"
if [ `id -u` != 0 ]; then
echo 'You must be root to run this script'
exit 1
fi

if [ $# -eq 1 ]; then
        if [ "$1" = "--update" ] ; then
                time=`ls -l --time-style="+%x %X" apachelogger.te | awk '{ printf "%s %s", $6, $7 }'`
                rules=`ausearch --start $time -m avc --raw -se apachelogger`
                if [ x"$rules" != "x" ] ; then
                        echo "Found avc's to update policy with"
                        echo -e "$rules" | audit2allow -R
                        echo "Do you want these changes added to policy [y/n]?"
                        read ANS
                        if [ "$ANS" = "y" -o "$ANS" = "Y" ] ; then
                              echo "Updating policy"
                              echo -e "$rules" | audit2allow -R >> apachelogger.te
                              # Fall though and rebuild policy
                        else
                              exit 0
                        fi
                else
                        echo "No new avcs found"
                        exit 0
                fi
        else
                echo -e $USAGE
                exit 1
        fi
elif [ $# -ge 2 ] ; then
        echo -e $USAGE
        exit 1
fi

echo "Building and Loading Policy"
set -x
make -f /usr/share/selinux/devel/Makefile apachelogger.pp || exit
/usr/sbin/semodule -i apachelogger.pp

# Generate a man page of the installed module
sepolicy manpage -p . -d apachelogger_t
# Fixing the file context on /usr/local/bin/apachelogger
/sbin/restorecon -F -R -v /usr/local/bin/apachelogger
# Generate a rpm package for the newly generated policy

pwd=$(pwd)
rpmbuild --define "_sourcedir ${pwd}" --define "_specdir ${pwd}" --define "_builddir ${pwd}" --define "_srcrpmdir ${pwd}" --define "_rpmdir ${pwd}" --define "_buildrootdir ${pwd}/.build"  -ba apachelogger_selinux.spec
[cloud_user@web policy]$
```


1. Restart `apachelogger`:
    
    ```
    sudo systemctl restart apachelogger
    ```
    
2. List `apachelogger` processes:
    
    ```
    ps -efZ | grep apachelogger
    ```
    
    Observe `apachelogger` now has the `apachelogger` type.
    
3. Check if restarting `apachelogger` generated any errors:
    
    ```
    sudo ausearch -m AVC -ts recent
    ```
    
    A lot of output comes back.
    
4. Filter the previous command for `apachelogger`:
    
    ```
    sudo ausearch -m AVC -ts recent | grep apachelogger
    ```
    
    Observe `apachelogger` does not have access to a few areas it needs to have access to.


```
[cloud_user@web policy]$     sudo systemctl restart apachelogger
[cloud_user@web policy]$     ps -efZ | grep apachelogger
system_u:system_r:apachelogger_t:s0 root    2965       1  0 14:39 ?        00:00:00 /usr/local/bin/apachelogger
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 cloud_u+ 2967 2646  0 14:39 pts/0 00:00:00 grep --color=auto apachelogger
[cloud_user@web policy]$     sudo ausearch -m AVC -ts recent
----
time->Wed May 20 14:39:23 2026
type=PROCTITLE msg=audit(1779287963.714:346): proctitle="/usr/local/bin/apachelogger"
type=PATH msg=audit(1779287963.714:346): item=1 name="/var/log/httpd/access_log" inode=33657481 dev=103:04 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_log_t:s0 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779287963.714:346): item=0 name="/var/log/httpd/" inode=33569295 dev=103:04 mode=040700 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_log_t:s0 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779287963.714:346): cwd="/"
type=SYSCALL msg=audit(1779287963.714:346): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=402012 a2=241 a3=1b6 items=2 ppid=1 pid=2965 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="apachelogger" exe="/usr/local/bin/apachelogger" subj=system_u:system_r:apachelogger_t:s0 key=(null)
type=AVC msg=audit(1779287963.714:346): avc:  denied  { open } for  pid=2965 comm="apachelogger" path="/var/log/httpd/access_log" dev="nvme0n1p4" ino=33657481 scontext=system_u:system_r:apachelogger_t:s0 tcontext=system_u:object_r:httpd_log_t:s0 tclass=file permissive=1
type=AVC msg=audit(1779287963.714:346): avc:  denied  { write } for  pid=2965 comm="apachelogger" name="access_log" dev="nvme0n1p4" ino=33657481 scontext=system_u:system_r:apachelogger_t:s0 tcontext=system_u:object_r:httpd_log_t:s0 tclass=file permissive=1
type=AVC msg=audit(1779287963.714:346): avc:  denied  { search } for  pid=2965 comm="apachelogger" name="httpd" dev="nvme0n1p4" ino=33569295 scontext=system_u:system_r:apachelogger_t:s0 tcontext=system_u:object_r:httpd_log_t:s0 tclass=dir permissive=1
[cloud_user@web policy]$     sudo ausearch -m AVC -ts recent | grep apachelogger
type=PROCTITLE msg=audit(1779287963.714:346): proctitle="/usr/local/bin/apachelogger"
type=SYSCALL msg=audit(1779287963.714:346): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=402012 a2=241 a3=1b6 items=2 ppid=1 pid=2965 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="apachelogger" exe="/usr/local/bin/apachelogger" subj=system_u:system_r:apachelogger_t:s0 key=(null)
type=AVC msg=audit(1779287963.714:346): avc:  denied  { open } for  pid=2965 comm="apachelogger" path="/var/log/httpd/access_log" dev="nvme0n1p4" ino=33657481 scontext=system_u:system_r:apachelogger_t:s0 tcontext=system_u:object_r:httpd_log_t:s0 tclass=file permissive=1
type=AVC msg=audit(1779287963.714:346): avc:  denied  { write } for  pid=2965 comm="apachelogger" name="access_log" dev="nvme0n1p4" ino=33657481 scontext=system_u:system_r:apachelogger_t:s0 tcontext=system_u:object_r:httpd_log_t:s0 tclass=file permissive=1
type=AVC msg=audit(1779287963.714:346): avc:  denied  { search } for  pid=2965 comm="apachelogger" name="httpd" dev="nvme0n1p4" ino=33569295 scontext=system_u:system_r:apachelogger_t:s0 tcontext=system_u:object_r:httpd_log_t:s0 tclass=dir permissive=1
type=PROCTITLE msg=audit(1779288028.718:360): proctitle="/usr/local/bin/apachelogger"
type=SYSCALL msg=audit(1779288028.718:360): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=402012 a2=241 a3=1b6 items=2 ppid=1 pid=2965 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="apachelogger" exe="/usr/local/bin/apachelogger" subj=system_u:system_r:apachelogger_t:s0 key=(null)
type=AVC msg=audit(1779288028.718:360): avc:  denied  { search } for  pid=2965 comm="apachelogger" name="httpd" dev="nvme0n1p4" ino=33569295 scontext=system_u:system_r:apachelogger_t:s0 tcontext=system_u:object_r:httpd_log_t:s0 tclass=dir permissive=1
[cloud_user@web policy]$
```


1. Review `audit2allow` to view `apachelogger` suggestions:
    
    ```
    sudo ausearch -m AVC -ts recent | grep audit2allow -R
    ```
    
    Observe the suggestions provided for `apachelogger` permission types: `apache_manage_log` and `apache_read_log`.
    
2. Acquire more information to understand the `apache_manage_log` permission type:
    
    ```
    grep -r "apache_manage_log" /usr/share/selinux/devel/include/ | grep .if
    ```
    
    The output references the `/usr/share/selinux/devel/include/contrib/apache.if` file.
    
3. Review the second permission type:
    
    ```
    grep -r "apache_read_log" /usr/share/selinux/devel/include/ | grep .if
    ```
    
    The output references the same file.
    
4. Open the file:
    
    ```
    less /usr/share/selinux/devel/include/contrib/apache.if
    ```
    
5. Search for `manage_log`:
    
    ```
    /manage_log
    ```
    
    Scroll up and notice that this allows the specific domain to manage Apache files.


```
interface(`apache_manage_log',`
        gen_require(`
                type httpd_log_t;
        ')

        logging_search_logs($1)
        manage_dirs_pattern($1, httpd_log_t, httpd_log_t)
        manage_files_pattern($1, httpd_log_t, httpd_log_t)
        read_lnk_files_pattern($1, httpd_log_t, httpd_log_t)
')
```


1. Search for `read_log`. You may need to scroll up to view matching results:
    
    ```
    /read_log
    ```
    
    Scroll up and observe this type allows the specified domain to read Apache files.

```
interface(`apache_dontaudit_read_log',`
        gen_require(`
                type httpd_log_t;
        ')

        dontaudit $1 httpd_log_t:file read_file_perms;
        dontaudit $1 httpd_log_t:lnk_file read_lnk_file_perms;
')

interface(`apache_read_log',`
        gen_require(`
                type httpd_log_t;
        ')

        logging_search_logs($1)
        allow $1 httpd_log_t:dir list_dir_perms;
        read_files_pattern($1, httpd_log_t, httpd_log_t)
        read_lnk_files_pattern($1, httpd_log_t, httpd_log_t)
')
```


1. Press `q` to close.
    
2. Add these permission types to `apachelogger.te`:

```
[cloud_user@web policy]$     sudo ausearch -m AVC -ts recent | grep audit2allow -R
grep: apachelogger.sh: Permission denied
[cloud_user@web policy]$ sudo ausearch -m AVC -ts recent | grep audit2allow -R
grep: apachelogger.sh: Permission denied
[cloud_user@web policy]$     grep -r "apache_manage_log" /usr/share/selinux/devel/include/ | grep .if
/usr/share/selinux/devel/include/contrib/apache.if:interface(`apache_manage_log',`
[cloud_user@web policy]$     grep -r "apache_read_log" /usr/share/selinux/devel/include/ | grep .if
/usr/share/selinux/devel/include/contrib/apache.if:interface(`apache_read_log',`
[cloud_user@web policy]$ cat /usr/share/selinux/devel/include/contrib/apache.if | grep manage_log
interface(`apache_manage_log',`
[cloud_user@web policy]$ cat /usr/share/selinux/devel/include/contrib/apache.if | grep read_log
interface(`apache_dontaudit_read_log',`
interface(`apache_read_log',`
[cloud_user@web policy]$     less /usr/share/selinux/devel/include/contrib/apache.if
[cloud_user@web policy]$
```





    ```
    sudo vim apachelogger.te
    ```
    
9. Press **i** to enter Insert mode. Scroll down to the bottom of the file, and add both permission types:
    
    > **Note:** When copying and pasting code into Vim from the lab guide, first enter `:set paste` (and then `i` to enter insert mode) to avoid adding unnecessary spaces and hashes. To save and quit the file, press **Escape** followed by `:wq`. To exit the file **without** saving, press **Escape** followed by `:q!`.
    
    ```
    apache_manage_log(apachelogger_t)
    apache_read_log(apachelogger_t)
    ```
    
10. Press **Escape** and type `:wq` to save and exit.
    
```
[cloud_user@web policy]$ sudo vim apachelogger.te
[sudo] password for cloud_user:
[cloud_user@web policy]$ cat apachelogger.te
policy_module(apachelogger, 1.0.0)

########################################
#
# Declarations
#

type apachelogger_t;
type apachelogger_exec_t;
init_daemon_domain(apachelogger_t, apachelogger_exec_t)

permissive apachelogger_t;

########################################
#
# apachelogger local policy
#
allow apachelogger_t self:fifo_file rw_fifo_file_perms;
allow apachelogger_t self:unix_stream_socket create_stream_socket_perms;

domain_use_interactive_fds(apachelogger_t)

files_read_etc_files(apachelogger_t)

miscfiles_read_localization(apachelogger_t)
apache_manage_log(apachelogger_t)
apache_read_log(apachelogger_t)
[cloud_user@web policy]$

```
### Reapply Policy

1. Reapply the rewritten policy:
    
    ```
    sudo ./apachelogger.sh
    ```
    
2. Restart `apachelogger`:
    
    ```
    sudo systemctl restart apachelogger
    ```
    
3. Review the logs again. You should observe no new error logs. Ensure you cross reference the log time with your current time. Any errors observed should be from the first restart:
    
    ```
    sudo ausearch -m AVC -ts recent
    ```
    

```
[cloud_user@web policy]$     sudo ausearch -m AVC -ts recent
<no matches>
[cloud_user@web policy]$
```
## Conclusion

Congratulations — you've completed this hands-on lab!
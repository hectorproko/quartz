---
tags:
  - Linux
  - ubuntu
  - security
  - ufw
  - apparmor
  - hardening
  - auditd
linkedin: "False"
quartz: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
completed: "True"
title: Ubuntu Server Security Hardening
hardlinked: "True"
---
<!--
*implements* [[fail2ban]] mention udner
-->
**Platform:** Pluralsight Hands-On Lab \n
**Environment:** Ubuntu 24.04 (AWS EC2) 

---

## Overview

This lab simulates a real-world pre-audit scenario: I'm part of an operations team at a regional logistics company, and our inventory server is about to face an external security audit. The server was intentionally left in a weak baseline state - SSH accepted password logins with no user restrictions, no firewall was active, there was no privileged action trail, and AppArmor was running but not yet tuned.

My goal was to harden the system across three layers:

1. **SSH & brute-force defense** - key-based auth for an `auditor` account, a locked-down `sshd` drop-in config, and a production-grade `fail2ban` jail
2. **Firewall & audit logging** - a deny-by-default UFW firewall, `auditd` rules targeting sensitive files and privileged commands, and multi-source event correlation
3. **Mandatory access control** - inspecting and toggling AppArmor profiles for `nginx`, observing enforcement in both complain and enforce modes, and practicing the `aa-logprof` refinement workflow

---

## Objective 1 - Harden SSH Access and Configure fail2ban

### Baseline Environment

The server was running Ubuntu 24.04 on AWS EC2 in a weakened state typical of a freshly provisioned instance. SSH was functional for the `ubuntu` account using key-based auth, but several critical controls were missing or inactive:

- [[fail2ban]] was installed but stopped - no brute-force protection in place 
- UFW was disabled - all inbound ports open with no filtering
- [[auditd]] had no custom rules - no trail of privileged actions
- An `auditor` account existed with no SSH key configured
- [[AppArmor (Application Armor)|AppArmor]] was running but profiles were untuned

Before making any changes, I audited the current state of each service to establish a clear before/after baseline.

---

### Generating an SSH Key Pair for the Auditor User

The `auditor` account existed but had no SSH key, only a password. The goal here was to move it to **key-based authentication**, which eliminates password guessing as an attack vector entirely.

I generated an ED25519 key pair (Ed25519 is faster and more secure than RSA for this use case) in my home directory so I could review it before deploying it:

```bash
ssh-keygen -t ed25519 -f ~/auditor_key -N '' -C 'auditor lab key'
```

```
Generating public/private ed25519 key pair.
Your identification has been saved in /home/ubuntu/auditor_key
Your public key has been saved in /home/ubuntu/auditor_key.pub
The key fingerprint is:
SHA256:6gVjBT7X+y5YZNprr4hFLXm7Vyt8vSuBfvNpZ/TDXHI auditor lab key
The key's randomart image is:
+--[ED25519 256]--+
|      .          |
|     . . .       |
|      o o .      |
|       + oo.     |
|      + S=+ .    |
|     . =.oo+ .o E|
|      . oooo..+*o|
|     . +..++++o=*|
|      o ..o=+o=*=|
+----[SHA256]-----+
```

I then created the `.ssh` directory with the exact ownership and permissions SSH requires, and installed the public key:

```bash
sudo install -d -o auditor -g auditor -m 700 /home/auditor/.ssh
sudo install -o auditor -g auditor -m 600 ~/auditor_key.pub /home/auditor/.ssh/authorized_keys
```

^install

Using `install` instead of `mkdir` + `cp` is a clean one-step way to set ownership and permissions **[[atomic transactions|atomically]]**, something worth doing in scripts where you don't want partial states.

Finally, I validated it worked with a live test:

```bash
ssh -i ~/auditor_key -o StrictHostKeyChecking=accept-new auditor@localhost 'whoami && hostname'
```

```
Warning: Permanently added 'localhost' (ED25519) to the list of known hosts.
auditor
ip-172-31-24-10
```

Key-based login confirmed, no password prompt.

---

### Hardening the SSHD Configuration

Rather than editing the main `sshd_config`, I used the drop-in directory `/etc/ssh/sshd_config.d/`. Any file placed here overrides the defaults without touching the main config, which is the Ubuntu-recommended approach and makes auditing config changes easier.

```bash
sudo tee /etc/ssh/sshd_config.d/70-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
AllowUsers ubuntu auditor
LoginGraceTime 30
EOF
```
<!--
[[sshd_config.d load order]]
-->
**What each directive does and why:**

|Directive|Value|Purpose|
|---|---|---|
|`PermitRootLogin`|`no`|Root logins are never needed over SSH; use sudo from a named account|
|`PasswordAuthentication`|`no`|Eliminates brute-force password attacks entirely|
|`MaxAuthTries`|`3`|Cuts off multi-attempt key fishing before fail2ban even needs to act|
|`AllowUsers`|`ubuntu auditor`|Explicit allow-list - any account not named here is denied, even with a valid key|
|`LoginGraceTime`|`30`|Reduces the unauthenticated connection window from the 120s default|

Before reloading, I validated the syntax to avoid locking myself out:

```bash
sudo sshd -t && echo "config ok"
```
<!--
[[sshd#Options]]
Run the left side, and _only if it exits with code 0 (success)_, run the right side.
-->
```
config ok
```

Then reloaded and confirmed the live runtime matched my config:

```bash
sudo systemctl reload ssh
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|maxauthtries|allowusers|logingracetime)'
```

```
logingracetime 30
maxauthtries 3
permitrootlogin no
passwordauthentication no
allowusers ubuntu
allowusers auditor
```

> **Note:** `sshd -T` outputs each `AllowUsers` entry on its own line, both lines together represent the single `AllowUsers ubuntu auditor` directive.

<!--
The order those directives appear in `sshd -T` output is just how the tool formats its dump, not the internal evaluation sequence.
-->
---

### Verifying Allow and Deny Behavior

I tested both the allow path and the deny path explicitly. Good security configs should be proven, not assumed.

**Allow - auditor with key (should succeed):**

```bash
ssh -i ~/auditor_key -o StrictHostKeyChecking=accept-new auditor@localhost 'echo "auditor login ok"'
```

```
auditor login ok
```

**Deny - root (should be blocked by AllowUsers):**

```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new root@localhost 'whoami' || echo 'root login denied as expected'
```

```
root@localhost: Permission denied (publickey).
root login denied as expected
```

**Deny - stranger (unlisted account, should be blocked):**

```bash
sudo useradd -m -s /bin/bash stranger
ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new stranger@localhost 'whoami' || echo 'stranger login denied as expected'
sudo userdel -r stranger
```

```
stranger@localhost: Permission denied (publickey).
stranger login denied as expected
```

I then confirmed the denials were logged:

```bash
sudo tail -n 30 /var/log/auth.log | grep -E '(Accepted|Failed|not allowed|not listed)'
```

```
2026-05-25T15:44:16.574418+00:00 ip-172-31-24-10 sshd[4580]: Accepted publickey for auditor from 127.0.0.1 port 49464 ssh2: ED25519 SHA256:6gVjBT7X...
2026-05-25T15:45:30.477221+00:00 ip-172-31-24-10 sshd[4667]: User root from 127.0.0.1 not allowed because not listed in AllowUsers
2026-05-25T15:46:13.803257+00:00 ip-172-31-24-10 sshd[4680]: User stranger from 127.0.0.1 not allowed because not listed in AllowUsers
```

Both `root` and `stranger` were rejected by the `AllowUsers` check — as confirmed by the auth.log message `not allowed because not listed in AllowUsers`. Both `PermitRootLogin no` and `AllowUsers` would have blocked root independently; the log shows which one reported the denial.

---

### Configuring fail2ban for SSH Defense

[[fail2ban]] watches log sources and automatically bans IPs that exceed a failure threshold. Even with key-only auth, it's a useful defense-in-depth layer, it can catch automated scanners that hammer connection attempts. 

```bash
sudo tee /etc/fail2ban/jail.d/sshd.local > /dev/null <<'EOF'
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime  = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = systemd
EOF
```

These are production values: 3 failures within 10 minutes triggers a 1-hour ban. `ignoreip` exempts localhost so monitoring jobs and scheduled tasks are never accidentally banned, in a real environment, I'd also add the jump host IPs here. 
<!--
**`[sshd]`** — The SSH Jail
**`port = ssh`** tells fail2ban _which port to block_ when it issues a ban.
**`logpath`** is the key one you asked about. `%(sshd_log)s` is a fail2ban variable that resolves to the right log source depending on your OS. On Ubuntu with systemd it typically maps to the systemd journal rather than a flat file. This is where fail2ban reads to _detect_ failures — it's scanning for lines like:

```
sshd: Invalid user foo from 1.2.3.4
sshd: Failed password for root from 1.2.3.4
sshd: User stranger not allowed because not listed in AllowUsers
```

**`backend = systemd`** tells fail2ban _how_ to read that log source. Instead of tailing a flat file like `/var/log/auth.log`, it reads directly from the systemd journal using the journal API.

#### The Full Flow
```
SSH login attempt fails
        ↓
sshd writes event to systemd journal
        ↓
fail2ban reads it via journal API (backend = systemd)
        ↓
fail2ban regex matches the failure pattern
        ↓
failure counter for that source IP increments
        ↓
if counter hits maxretry (3) within findtime (600s)
        ↓
fail2ban inserts a kernel firewall rule blocking that IP on port 22
        ↓
ban automatically expires after bantime (3600s)
```

### What `%(sshd_log)s` Actually Is

It's a **fail2ban internal variable** — think of it like a macro defined inside fail2ban's own config files, not the OS. The `%(...) s` syntax is fail2ban's string interpolation format (similar to Python's `%s`).

Where is it defined? In fail2ban's built-in defaults, somewhere like `/etc/fail2ban/jail.conf`. If you search it:
```bash
grep "sshd_log" /etc/fail2ban/jail.conf
```

You'd find something like:
```ini
sshd_log = /var/log/auth.log
```

or on systemd systems it may just be overridden by the `backend = systemd` setting entirely.
-->

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

```
Status
|- Number of jail:      1
`- Jail list:   sshd
```

I confirmed the runtime values matched the policy:

```bash
sudo fail2ban-client get sshd maxretry   # → 3
sudo fail2ban-client get sshd findtime   # → 600
sudo fail2ban-client get sshd bantime    # → 3600
```

```
sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

Armed and quiet, exactly what you want before any incidents.

---

## Objective 2 - Enable UFW, Configure auditd, and Correlate Security Events

### Enabling UFW with a Deny-by-Default Inbound Policy

A host firewall provides the first line of defense against unsolicited inbound connections. UFW (Uncomplicated Firewall) is a frontend for `iptables`/`nftables` that makes rule management straightforward.

I started by confirming UFW was inactive, then built the ruleset from scratch:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80/tcp       # inventory dashboard
sudo ufw limit 22/tcp       # SSH with rate limiting
sudo ufw --force enable
```

> `sudo ufw default deny incoming` Sets the default rule for all inbound traffic to _drop it_.
> `sudo ufw default allow outgoing` Lets the server initiate outbound connections freely.
> `sudo ufw allow 80/tcp` Punches a hole for HTTP traffic on port 80.
> `sudo ufw limit 22/tcp` Allows SSH, but with *built-in rate limiting*. 

```bash
sudo ufw status verbose
```

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
80/tcp                     ALLOW IN    Anywhere
22/tcp                     LIMIT IN    Anywhere
80/tcp (v6)                ALLOW IN    Anywhere (v6)
22/tcp (v6)                LIMIT IN    Anywhere (v6)
```

> **Why `ufw limit` on SSH?** `limit` rejects connections from a source IP if it makes six or more attempts in the last 30 seconds. This works at the firewall level, complementing the `fail2ban` application-layer defense set up in Objective 1.

I then verified persistence by reloading the firewall and confirming the rules were unchanged:

```bash
sudo ufw reload   # → Firewall reloaded
sudo ufw status verbose   # same rule table as above
```

UFW stores rules in `/etc/ufw/`, so they survive both a reload and a full reboot, there's no window where the host is unprotected.

---

### Defining Audit Rules for Sensitive Events

`auditd` is the Linux kernel audit subsystem. By writing targeted watch rules, I can generate a forensic trail of exactly the events auditors care about: account file modifications, privileged command execution, and authentication log writes.

```bash
sudo tee /etc/audit/rules.d/security-hardening.rules > /dev/null <<'EOF'
# Watch sensitive account files for any write or attribute change
-w /etc/passwd -p wa -k account-changes
-w /etc/shadow -p wa -k account-changes

# Watch the SSH auth log for any write
-w /var/log/auth.log -p wa -k auth-log

# Audit every execution of sudo by tagging it with the 'privileged' key
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/sudo -F key=privileged
EOF
```

The `-k` flag tags each event with a searchable key, which makes querying with [[ausearch (Audit Search)|ausearch]] straightforward later. 

I loaded the rules into the running kernel:

```bash
sudo augenrules --load
sudo auditctl -l
```
<!--Yes, both are `auditd` commands,-->
```
-w /etc/passwd -p wa -k account-changes
-w /etc/shadow -p wa -k account-changes
-w /var/log/auth.log -p wa -k auth-log
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/sudo -F key=privileged
```

All four rules active in the running audit subsystem.

---

### Triggering and Querying Audit Events

To prove the rules actually work, I generated one event of each type and then queried for it.

**Trigger an `account-changes` event** by modifying the `auditor` account's comment field:

```bash
sudo usermod -c "Auditor Account" auditor
```

**Trigger a `privileged` event** by running a sudo command:

```bash
sudo ls /root   # → snap
```

**Trigger an `auth-log` event** with a deliberate failed login:

```bash
su - root -c whoami < /dev/null || echo 'failed su recorded'
```

```
Password: su: Authentication failure
failed su recorded
```

**Query the account-changes trail:**

```bash
sudo ausearch -k account-changes -ts recent | tail -n 25
```

```
time->Mon May 25 16:03:33 2026
type=PATH msg=audit(...): item=0 name="/etc/passwd" inode=33756 ... nametype=NORMAL
type=SYSCALL msg=audit(...): ... comm="usermod" exe="/usr/sbin/usermod" ... key="account-changes"
----
time->Mon May 25 16:03:33 2026
type=PATH msg=audit(...): item=0 name="/etc/shadow" inode=3086 ... nametype=NORMAL
type=SYSCALL msg=audit(...): ... comm="usermod" exe="/usr/sbin/usermod" ... key="account-changes"
```

The audit record shows the exact timestamp, the file touched (`/etc/passwd`, `/etc/shadow`), the process (`usermod`), and the calling user, exactly what an auditor would need.

**Query the privileged sudo trail:**

```bash
sudo ausearch -k privileged -ts recent | tail -n 25
```

```
time->Mon May 25 16:03:41 2026
type=EXECVE msg=audit(...): argc=3 a0="sudo" a1="ls" a2="/root"
type=SYSCALL msg=audit(...): ... exe="/usr/bin/sudo" ... key="privileged"
```

The `EXECVE` record captures the full command line (`sudo ls /root`), not just the binary, useful for reconstructing exactly what was run.

---

### Real-Time Monitoring with journalctl

In production you'd run `journalctl -f` in a second terminal to watch events live. Since this lab uses a single browser terminal, I backgrounded it with a 30-second timeout:

```bash
sudo timeout 30 journalctl -f _COMM=sudo --no-pager &
sudo ls /root
```

```
[1] 6045
May 25 16:09:33 ip-172-31-24-10 sudo[6049]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/ls /root
May 25 16:09:33 ip-172-31-24-10 sudo[6049]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)

[1]+  Exit 124                sudo timeout 30 journalctl -f _COMM=sudo --no-pager
```

`Exit 124` is expected, that's `timeout`'s exit code when it terminates a process after the time limit.

For incident reconstruction after the fact, the retrospective query:

```bash
sudo journalctl _COMM=sudo --since "10 minutes ago" --no-pager | tail -n 20
```

---

### Cross-Referencing Audit Sources

A key audit skill is verifying that the same event appears across multiple independent log sources. Here I cross-referenced `auditd`, `journalctl`, and `/var/log/auth.log` for the same sudo events:

```bash
sudo tail -n 30 /var/log/auth.log | grep -E '(sudo|su:)'
```

```
2026-05-25T16:09:33.998807+00:00 ip-172-31-24-10 sudo: ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/ls /root
2026-05-25T16:09:34.000130+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
```

The same `ls /root` event appears in `auditd` (tagged `privileged`), `journalctl` (via `_COMM=sudo`), and `auth.log`. This overlap is deliberate, if a record exists in one source but not the others, that discrepancy is either a tampered log or a configuration gap, both of which are reportable audit findings.

I also generated a full summary of all audit activity since boot:

```bash
sudo aureport --summary
```

```
Summary Report
======================
Range of time in logs: 05/25/26 13:06:34 - 05/25/26 16:07:48
Number of changes in configuration: 263
Number of changes to accounts, groups, or roles: 12
Number of logins: 2
Number of failed logins: 2
Number of failed authentications: 2
Number of users: 4
Number of keys: 3
Number of events: 1358
```

The 12 account changes and 2 failed logins line up exactly with the test activity I ran.

---

## Objective 3 - Manage AppArmor Profiles and Review Denial Logs

### Inspecting the AppArmor Posture

[[AppArmor (Application Armor)|AppArmor]] is a [[mandatory access control (MAC)]] system that confines processes to a defined set of file and capability permissions. Unlike discretionary access control (which relies on file ownership and `chmod`), MAC rules apply even to root processes.

We already have a pre-staged a custom nginx profile in enforce mode that explicitly denies writes to `/etc/`. I started by understanding what was already there.

```bash
sudo aa-status --json | python3 -c "
import json,sys; d=json.load(sys.stdin)
print('profiles loaded:', sum(len(v) for v in d['profiles'].values()))
print('processes confined:', sum(len(v) for v in d['processes'].values()))"
```

```
profiles loaded: 1295
processes confined: 4
```

Then I read the actual nginx profile to understand its rules:

```bash
sudo cat /etc/apparmor.d/usr.sbin.nginx
```

```
#include <tunables/global>

/usr/sbin/nginx {
    #include <abstractions/base>
    #include <abstractions/nis>

    capability dac_override,
    capability dac_read_search,
    capability net_bind_service,
    capability setgid,
    capability setuid,

    /etc/group r,
    /etc/nginx/** r,
    /etc/passwd r,
    /etc/ssl/openssl.cnf r,

    /run/nginx.pid rw,
    /run/nginx/** rw,

    /usr/sbin/nginx mr,

    /var/log/nginx/*.log w,
    /var/www/** r,

    # Explicitly DENY writes to /etc/ to make denials visible in dmesg
    deny /etc/** w,
}
```

^2b4ffe

The critical line is `deny /etc/** w,`, an explicit deny rule. This means even if the profile mode is switched to complain, this specific path will still block writes. Explicit denies override the mode setting by design.

```bash
sudo aa-status | grep -E '(nginx|profiles are in)'
```

```
51 profiles are in enforce mode.
   /usr/sbin/nginx
6 profiles are in complain mode.
```

nginx confirmed in enforce mode.

---

### Switching to Complain Mode and Probing the Audit Trail

**Complain mode** logs policy violations without blocking them, useful for profiling a new service or testing what a profile would block before enforcing it.

```bash
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
```

```
Setting /etc/apparmor.d/usr.sbin.nginx to complain mode.
```

I first ran the rogue demo script as unconfined root to establish a baseline, outside any profile, it can write anywhere:

```bash
sudo /usr/local/bin/rogue-demo.sh
```

> [!NOTE]- rogue-demo.sh
> ```bash
> #!/bin/bash
> # rogue-demo.sh: attempts a forbidden write to /etc/ so learners can see
> # AppArmor enforcement and auditd capture in action.
> set +e
> echo "rogue process running as $(whoami) at $(date -u +%FT%TZ)" > /etc/rogue-marker 2>/dev/null
> if [ -f /etc/rogue-marker ]; then
>     echo "WRITE SUCCEEDED: /etc/rogue-marker exists"
> else
>     echo "WRITE FAILED: AppArmor or filesystem permissions blocked the write"
> fi
> ```
> 
<!--
`set +e` is a bash shell option that tells the shell **stop exiting on errors**.

By default, or when you run `set -e`, bash will immediately abort a script if any command returns a non-zero exit code (i.e. fails). `set +e` turns that behavior off, meaning the script keeps running even if a command fails.

The `+` and `-` notation is counterintuitive at first:

- `set -e` → **enable** exit-on-error
- `set +e` → **disable** exit-on-error (turn it off)

**Example:**

bash

```bash
set -e
rm /file/that/doesnt/exist   # script dies here

set +e
rm /file/that/doesnt/exist   # error is ignored, script continues
echo "still running"         # this prints
```
-->
```
WRITE SUCCEEDED: /etc/rogue-marker exists
```

After cleaning up the marker file, I ran the same script under the nginx AppArmor profile in complain mode:

```bash
sudo rm -f /etc/rogue-marker
sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
```

`aa-exec` is telling the kernel: **"run this next command under nginx's AppArmor profile."**

```
/usr/local/bin/rogue-demo.sh: line 5: /etc/rogue-marker: Permission denied
WRITE FAILED: AppArmor or filesystem permissions blocked the write
```

> **Key insight:** The write was blocked even in complain mode. This is because the profile contains an explicit `deny /etc/** w,` rule. Explicit deny rules always block, they are not softened by complain mode. Without that explicit deny, complain mode would have logged the violation and allowed the write to proceed.

Checking the audit log for AVC records:

```bash
sudo ausearch -m AVC -ts recent | tail -n 30
```

```
<no matches>
```

This is actually a valid result on many Ubuntu 24.04 systems, the denial worked (the script reported `WRITE FAILED`), but the kernel on this AMI didn't emit a `type=AVC` record. The events are captured in the audit log under a different record type, and `aa-logprof` will read them in the next section. In production, checking multiple sources (`dmesg | grep apparmor`, `journalctl -k | grep apparmor`, raw `grep apparmor /var/log/audit/audit.log`) covers all the bases.

---

### Switching to Enforce Mode and Watching the Block

```bash
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
```

```
Setting /etc/apparmor.d/usr.sbin.nginx to enforce mode.
```

```bash
sudo aa-status | grep -E '/usr/sbin/nginx' | head -3
```

```
   /usr/sbin/nginx
   /usr/sbin/nginx//null-/usr/bin/date
   /usr/sbin/nginx//null-/usr/bin/whoami
```

Now enforce mode is stricter than complain: it blocks anything not explicitly allowed, not just the explicit deny rules. Running the rogue script again:

```bash
sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
```

```
/bin/bash: /usr/local/bin/rogue-demo.sh: Permission denied
```

In enforce mode the shell itself can't even read the script, because `/usr/local/bin/rogue-demo.sh` isn't in the profile's [[#^2b4ffe|allowed paths]]. The error message changed: in complain mode the shell could execute the script but the write was blocked; in enforce mode execution is blocked before the script even starts.
<!--[[#^2b4ffe|allowed paths]]
So the sentence is saying: when running under the `/usr/sbin/nginx` profile in enforce mode, the shell tries to read `/usr/local/bin/rogue-demo.sh` and gets blocked because that path doesn't appear in the profile's rules. You can confirm this by looking at the profile — it only allows paths like `/etc/nginx/**`, `/var/log/nginx/`, `/var/www/`, etc. There's no entry for anything under `/usr/local/bin/`.
-->
```bash
ls -l /etc/rogue-marker 2>&1
```

```
ls: cannot access '/etc/rogue-marker': No such file or directory
```

No file created, the block worked at the kernel level.
<!--
does this fail because it is not part of the allowed path as well?
```
ls: cannot access '/etc/rogue-marker': No such file or directory
```
---------------------------
No — that one is different. The `ls` command is being run by you as `ubuntu` (unconfined), not under the nginx profile. So AppArmor isn't involved there at all.

That output is just a normal filesystem response: the file genuinely doesn't exist, **because AppArmor already blocked the rogue script from creating it** in the first place. It's actually the _proof_ that enforcement worked — the expected side effect of a successful block is that the file was never written.
-->
---

### Refining the Profile with aa-logprof

#### Issue Encountered

[[aa-logprof]] reads the audit log and presents interactive prompts for each AppArmor event recorded, asking whether to allow, deny, or abstract each access. Since `ausearch -m AVC` returned `<no matches>`, I wasn't sure if the tool would find the events.
<!--
es, that's exactly right — and you've got the flow correct. Let me just sharpen one piece:

**`ausearch -m AVC`** queries the audit log for Access Vector Cache denial records — these are the kernel's way of saying "a process tried to do something the MAC policy didn't allow." It's a manual inspection tool, you're just reading what already happened.

**`aa-logprof`** reads that same audit log (`/var/log/audit/audit.log`) and turns those raw denial records into an interactive decision workflow — one prompt per event. For each one it's essentially asking: _"nginx tried to read `/etc/nsswitch.conf` and was denied — what should the profile say about that going forward?"_ You then pick Allow, Deny, use an abstraction include, etc.

The important nuance you touched on: **`aa-logprof` doesn't care if `ausearch` comes back empty.** As you saw in the lab, `ausearch -m AVC` returned `<no matches>`, but `aa-logprof` still found the events because it reads the raw log file directly and parses AppArmor entries that may be stored under different record types than `AVC`.
-->
#### Resolution

Despite the empty `ausearch` result, `aa-logprof` was still able to read the events from `/var/log/audit/audit.log` directly (it uses its own parser, not `ausearch`). The tool launched and presented prompts as expected.

> **Takeaway:** `ausearch -m AVC` returning no results does not mean the audit log is empty, it means the events were stored under a different record type. `aa-logprof` reads the raw log and handles this transparently.

The decision table I applied to each prompt:

|Rule|When the prompt shows...|Key to press|
|---|---|---|
|1|Anything mentioning `/usr/local/bin/rogue-demo.sh`|`D` (Deny)|
|2|A numbered include option (e.g. `[1 - include <abstractions/consoles>]`)|`1`, then `A`|
|3|An `Execute:` line for a helper like `/usr/bin/whoami` or `/usr/bin/date`|`I` (Inherit)|
|4|A `Path:` line for any other system file|`A` (Allow)|
|5|A `Network Family:` line with no numbered include|`A` (Allow)|

After working through all prompts, I saved with `S`. The tool wrote the refinements back to the profile file.

I then confirmed the critical `deny` rule survived the refinement session:
<!--
becuse the policy that denied or triggered the event came from this specific file, our modifications would be saved to that file
-->
```bash
sudo grep -c 'deny /etc/' /etc/apparmor.d/usr.sbin.nginx
```

```
1
```

And ran the rogue script one final time to prove enforce mode still held:

```bash
sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
```

```
/bin/bash: /usr/local/bin/rogue-demo.sh: Permission denied
```

The security boundary was intact after refinement, this is the point of `aa-logprof`: add the access paths a legitimate workload actually needs, without weakening the explicit deny rules that define its security boundary.


---

## Skills Demonstrated


|Tool / Technique|What I Did|
|---|---|
|`ssh-keygen`, `install`|Generated ED25519 key pair and deployed it with correct permissions|
|`sshd_config.d` drop-in|Hardened SSH without touching the main config file|
|`sshd -t`, `sshd -T`|Validated config syntax and confirmed live runtime settings|
|`fail2ban`|Configured a production-grade SSH jail with `maxretry`, `findtime`, `bantime`|
|`ufw`|Deployed a deny-by-default firewall with rate-limited SSH|
|`auditd` / `augenrules`|Wrote and loaded kernel audit rules targeting sensitive files and sudo|
|`ausearch`, `aureport`|Queried audit logs by key, time range, and event type|
|`journalctl`|Watched events live and reconstructed them retrospectively|
|`aa-status`, `aa-complain`, `aa-enforce`|Inspected and toggled AppArmor profile modes|
|`aa-exec`|Tested a process under a specific AppArmor profile without restarting the service|
|`aa-logprof`|Interactively refined a profile against real audit events without weakening deny rules|

---

## Key Takeaways

- **Defense in depth is layered deliberately.** SSH key auth, `fail2ban`, and `ufw limit` each operate at a different layer and cover each other's gaps.
- **Explicit `deny` rules in AppArmor override the profile mode.** A `deny` path blocks in both complain and enforce modes, this is a feature, not a limitation.
- **`ausearch -m AVC` returning `<no matches>` is not a failure.** Ubuntu 24.04 AMIs may route AppArmor events under different record types; the denial still occurs, and `aa-logprof` will still find the events.
- **Log correlation is the core audit skill.** The same privileged event appearing in `auditd`, `journalctl`, and `auth.log` simultaneously is what gives you confidence the audit trail is trustworthy, and lets you spot gaps if one source is missing.





<!--
---
## OBJECTIVE 1

### Harden SSH Access and Validate with fail2ban

You are back on the operations team at a regional logistics company, and the inventory server is about to undergo an external security audit. The server is currently running with default security settings: SSH accepts password authentication with no user restrictions, no firewall is active, there is no audit trail of privileged actions, and AppArmor is running but not yet tuned.

In this capstone guided lab, you will harden SSH access by generating key-based authentication for an `auditor` user, lock down `sshd` with an `AllowUsers` allow-list and a stricter login policy, configure `fail2ban` to defend against brute-force attempts, enable a host firewall with rate-limited SSH, write audit rules that capture the events the auditors will ask about, and demonstrate control over AppArmor mandatory access control profiles on the live `nginx` service.

### Verify the Environment

1. On the lab page, click the **Open Environment** button to launch the lab. The **Ubuntu Console** terminal opens automatically.
    
2. Confirm the bootstrap completed by checking the phase-marker files left in the home directory:
    
    ```
    ls /home/ubuntu/ | grep -E '(READY|COMPLETE)'
    ```
    
    **Expected result:** Five lines, in some order:
    
    - `APT READY`
    - `AUDITOR READY`
    - `APPARMOR READY`
    - `README READY`
    - `SYSTEM_COMPLETE`
    
    If you see fewer than five, bootstrap is still running, so wait a minute and run the command again.
    
3. Read the lab reference note for an orientation to the starting posture:
    
    ```
    cat ~/LAB_README.txt
    ```
    
    **Expected result:** A summary of the starting (deliberately unhardened) security posture, the two pre-staged accounts (`ubuntu` and `auditor`), the pre-staged `rogue-demo.sh` script, and a critical safety reminder to keep `ubuntu` in your `AllowUsers` directive.
    
    > **Note:** Throughout this lab, the `ubuntu` account is your only path back into the instance through the browser terminal. Always include `ubuntu` in any `AllowUsers` directive you write, and never disable its key-based authentication. The `auditor` account is the target of the SSH key-auth exercises in this objective.
    

```
ubuntu@ip-172-31-24-10:~$ ls /home/ubuntu/ | grep -E '(READY|COMPLETE)'
APPARMOR READY
APT READY
AUDITOR READY
README READY
SYSTEM_COMPLETE

ubuntu@ip-172-31-24-10:~$ cat ~/LAB_README.txt
Ubuntu Security Hardening Lab

Starting posture (deliberately unhardened so you have work to do):
  - SSH: ubuntu has key-based access through the browser terminal already.
         No 70-hardening.conf yet.
  - fail2ban: installed, stopped, and disabled (no jails configured).
  - UFW: installed and disabled.
  - auditd: running with NO custom rules (default ruleset only).
  - AppArmor: running with a custom /etc/apparmor.d/usr.sbin.nginx profile
              in ENFORCE mode that denies writes to /etc/.

Pre-staged accounts:
  - ubuntu  : your working account, sudo without password, key-based SSH.
              KEEP THIS ACCOUNT IN AllowUsers throughout the lab.
  - auditor : password is 'auditor-pass', no sudo, no SSH key yet.
              You will configure key-based authentication for this user.

Pre-staged file:
  /usr/local/bin/rogue-demo.sh
    A 5-line shell script that tries to write /etc/rogue-marker and reports
    'WRITE SUCCEEDED' or 'WRITE FAILED'. Used to demonstrate AppArmor
    enforcement and auditd event capture.

Critical safety reminder:
  Keep the ubuntu user in your AllowUsers directive at all times. The
  browser terminal connects via the ubuntu account, so if AllowUsers
  excludes it, you may lose access until the lab is rebuilt.
ubuntu@ip-172-31-24-10:~$
```

### Generate an SSH Key Pair for the auditor User

1. Generate a new ED25519 key pair for the `auditor` user, writing it into your own home directory first so you can review it:
    
    ```
    ssh-keygen -t ed25519 -f ~/auditor_key -N '' -C 'auditor lab key'
    ```
    
    **Expected result:** A summary including:
    
    - `Generating public/private ed25519 key pair.`
    - the path to the saved private key
    - the path to the public key (`~/auditor_key.pub`)
    - the key fingerprint and randomart image
2. Inspect the public key:
    
    ```
    cat ~/auditor_key.pub
    ```
    
    **Expected result:** A single line beginning with `ssh-ed25519`, followed by the base64-encoded public key and the comment `auditor lab key`.
    
3. Create the auditor user's `.ssh` directory with the correct ownership and permissions:
    
    ```
    sudo install -d -o auditor -g auditor -m 700 /home/auditor/.ssh
    ```
    
    **Expected result:** No output.
    
4. Install the public key into auditor's `authorized_keys` with the correct ownership and permissions:
    
    ```
    sudo install -o auditor -g auditor -m 600 ~/auditor_key.pub /home/auditor/.ssh/authorized_keys
    ```
    
    **Expected result:** No output.
    
5. Test key-based login as auditor from localhost:
    
    ```
    ssh -i ~/auditor_key -o StrictHostKeyChecking=accept-new auditor@localhost 'whoami && hostname'
    ```
    
    **Expected result:** A line similar to `Warning: Permanently added 'localhost' (ED25519)...` may appear if this is the first connection. The command then returns `auditor` followed by the lab VM hostname. The session authenticates using the SSH key without prompting for a password.
    

```
ubuntu@ip-172-31-24-10:~$ ssh-keygen -t ed25519 -f ~/auditor_key -N '' -C 'auditor lab key'
Generating public/private ed25519 key pair.
Your identification has been saved in /home/ubuntu/auditor_key
Your public key has been saved in /home/ubuntu/auditor_key.pub
The key fingerprint is:
SHA256:6gVjBT7X+y5YZNprr4hFLXm7Vyt8vSuBfvNpZ/TDXHI auditor lab key
The key's randomart image is:
+--[ED25519 256]--+
|      .          |
|     . . .       |
|      o o .      |
|       + oo.     |
|      + S=+ .    |
|     . =.oo+ .o E|
|      . oooo..+*o|
|     . +..++++o=*|
|      o ..o=+o=*=|
+----[SHA256]-----+
ubuntu@ip-172-31-24-10:~$ cat ~/auditor_key.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKDeXwLQw+/UUssCehsC28lr/qdhqW1snN+kDDZn1Tpu auditor lab key
ubuntu@ip-172-31-24-10:~$ sudo install -d -o auditor -g auditor -m 700 /home/auditor/.ssh
ubuntu@ip-172-31-24-10:~$ sudo install -o auditor -g auditor -m 600 ~/auditor_key.pub /home/auditor/.ssh/authorized_keys
ubuntu@ip-172-31-24-10:~$ ssh -i ~/auditor_key -o StrictHostKeyChecking=accept-new auditor@localhost 'whoami && hostname'
Warning: Permanently added 'localhost' (ED25519) to the list of known hosts.
auditor
ip-172-31-24-10
ubuntu@ip-172-31-24-10:~$
```
### Harden the SSHD Configuration

1. Write a hardening drop-in that enforces an `AllowUsers` allow-list, disables root login, disables password authentication, lowers the per-connection retry limit, and tightens the login grace time:
    
    ```
    sudo tee /etc/ssh/sshd_config.d/70-hardening.conf > /dev/null <<'EOF'
    PermitRootLogin no
    PasswordAuthentication no
    MaxAuthTries 3
    AllowUsers ubuntu auditor
    LoginGraceTime 30
    EOF
    ```
    
    **Expected result:** No output. The hardening drop-in is created at `/etc/ssh/sshd_config.d/70-hardening.conf`.
    
    > **Note:** The drop-in directory `/etc/ssh/sshd_config.d/` is loaded after the main `sshd_config`, so settings here override the defaults without editing the main file. This is the recommended Ubuntu approach for SSH configuration changes. Setting `PasswordAuthentication no` is the single most important hardening directive here because it forces every login to use a key pair and completely removes brute-force password guessing as an attack vector.
    
2. Validate the configuration syntactically before reloading sshd:
    
    ```
    sudo sshd -t && echo "config ok"
    ```
    
    **Expected result:** `config ok`. If `sshd -t` reports an error, the reload would fail. This test catches the problem before it locks you out.
    
3. Reload `sshd` so the new policy takes effect for future connections. Existing sessions stay open.
    
    ```
    sudo systemctl reload ssh
    ```
    
    **Expected result:** No output.
    
4. Confirm the active runtime configuration reflects your changes:
    
    ```
    sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|maxauthtries|allowusers|logingracetime)'
    ```
    
    **Expected result:** Six lines show `permitrootlogin no`, `passwordauthentication no`, `maxauthtries 3`, `logingracetime 30`, `allowusers ubuntu`, and `allowusers auditor`. Note that `sshd -T` displays each `AllowUsers` value on its own line. Both entries together represent the single `AllowUsers ubuntu auditor` directive in your config.
    

```
ubuntu@ip-172-31-24-10:~$ sudo tee /etc/ssh/sshd_config.d/70-hardening.conf > /dev/null <<'EOF'
> PermitRootLogin no
> PasswordAuthentication no
> MaxAuthTries 3
> AllowUsers ubuntu auditor
> LoginGraceTime 30
> EOF
ubuntu@ip-172-31-24-10:~$ sudo sshd -t && echo "config ok"
config ok
ubuntu@ip-172-31-24-10:~$ sudo systemctl reload ssh
ubuntu@ip-172-31-24-10:~$ sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|maxauthtries|allowusers|logingracetime)'
logingracetime 30
maxauthtries 3
permitrootlogin no
passwordauthentication no
allowusers ubuntu
allowusers auditor
ubuntu@ip-172-31-24-10:~$ 
```
### Verify Allow and Deny Behavior

1. Confirm the `auditor` user (in the allow-list) can still authenticate with their key:
    
    ```
    ssh -i ~/auditor_key -o StrictHostKeyChecking=accept-new auditor@localhost 'echo "auditor login ok"'
    ```
    
    **Expected result:** `auditor login ok`.
    
2. Confirm the `root` user is rejected by the new `AllowUsers` policy:
    
    ```
    ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new root@localhost 'whoami' || echo 'root login denied as expected'
    ```
    
    **Expected result:** A line similar to `root@localhost: Permission denied (publickey).` followed by `root login denied as expected`. The error reports only `publickey` (not `publickey,password`) because the hardening drop-in disables password authentication, leaving public-key authentication as the only method `sshd` advertises.
    
3. Create a temporary `stranger` user who is not in the allow-list:
    
    ```
    sudo useradd -m -s /bin/bash stranger
    ```
    
    **Expected result:** No output.
    
4. Confirm `stranger` is also rejected by `AllowUsers`:
    
    ```
    ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new stranger@localhost 'whoami' || echo 'stranger login denied as expected'
    ```
    
    **Expected result:** A line similar to `stranger@localhost: Permission denied (publickey).`, followed by `stranger login denied as expected`. The same `(publickey)` format appears here for the same reason as the root denial above.
    
5. Remove the temporary `stranger` user now that you have validated the deny path:
    
    ```
    sudo userdel -r stranger
    ```
    
    **Expected result:** A harmless warning similar to `userdel: stranger mail spool (/var/mail/stranger) not found`. This is normal on Ubuntu 24.04 because no mail server is installed. The account and its home directory are removed successfully.
    
6. Inspect the SSH authentication log to see the policy decisions on disk:
    
    ```
    sudo tail -n 30 /var/log/auth.log | grep -E '(Accepted|Failed|not allowed|not listed)'
    ```
    
    **Expected result:** A mix of `Accepted publickey for auditor` lines and entries similar to `User root from 127.0.0.1 not allowed because not listed in AllowUsers` and `User stranger from 127.0.0.1 not allowed because not listed in AllowUsers`. Both `root` and `stranger` are rejected by the same `AllowUsers` check, which is evaluated before `PermitRootLogin`.
    

```
ubuntu@ip-172-31-24-10:~$ ssh -i ~/auditor_key -o StrictHostKeyChecking=accept-new auditor@localhost 'echo "auditor login ok"'
auditor login ok
ubuntu@ip-172-31-24-10:~$ ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new root@localhost 'whoami' || echo 'root login denied as expected'
root@localhost: Permission denied (publickey).
root login denied as expected
ubuntu@ip-172-31-24-10:~$ sudo useradd -m -s /bin/bash stranger
ubuntu@ip-172-31-24-10:~$ ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new stranger@localhost 'whoami' || echo 'stranger login denied as expected'
stranger@localhost: Permission denied (publickey).
stranger login denied as expected
ubuntu@ip-172-31-24-10:~$ sudo userdel -r stranger
userdel: stranger mail spool (/var/mail/stranger) not found
ubuntu@ip-172-31-24-10:~$ sudo tail -n 30 /var/log/auth.log | grep -E '(Accepted|Failed|not allowed|not listed)'
2026-05-25T15:44:16.574418+00:00 ip-172-31-24-10 sshd[4580]: Accepted publickey for auditor from 127.0.0.1 port 49464 ssh2: ED25519 SHA256:6gVjBT7X+y5YZNprr4hFLXm7Vyt8vSuBfvNpZ/TDXHI
2026-05-25T15:45:30.477221+00:00 ip-172-31-24-10 sshd[4667]: User root from 127.0.0.1 not allowed because not listed in AllowUsers
2026-05-25T15:46:13.803257+00:00 ip-172-31-24-10 sshd[4680]: User stranger from 127.0.0.1 not allowed because not listed in AllowUsers
ubuntu@ip-172-31-24-10:~$
```

### Configure `fail2ban` for SSH Defense

1. Write a production-style `fail2ban` jail for `sshd`:
    
    ```
sudo tee /etc/fail2ban/jail.d/sshd.local > /dev/null <<'EOF'
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime  = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = systemd
EOF
    ```
    
    **Expected result:** No output.
    
    > **Note:** These are production values, not test values. `maxretry = 3` failures within `findtime = 600` seconds triggers a `bantime = 3600` second block. `ignoreip = 127.0.0.1/8 ::1` always exempts localhost so monitoring traffic and scheduled jobs are never accidentally banned. In a real environment, you would also append your own jump-host IPs to that list.
    
2. Enable and start `fail2ban` so the jail begins watching SSH events immediately:
    
    ```
    sudo systemctl enable --now fail2ban
    ```
    
    **Expected result:** A line similar to `Created symlink /etc/systemd/system/multi-user.target.wants/fail2ban.service`, pointing to the service unit file.
    
3. Confirm the service is running and the `sshd` jail is registered:
    
    ```
    sudo fail2ban-client status
    ```
    
    **Expected result:** A short report with `Number of jail: 1` and `Jail list: sshd`.
    
4. Inspect the operational state of the `sshd` jail:
    
    ```
    sudo fail2ban-client status sshd
    ```
    
    **Expected result:** A table showing `Filter` with `Currently failed`, `Total failed`, and a `File list` (or journal source), plus `Actions` with `Currently banned 0`, `Total banned 0`, and an empty `Banned IP list`. With no malicious traffic yet, the jail is armed but quiet.
    
5. Confirm the `maxretry` runtime parameter matches the policy you wrote:
    
    ```
    sudo fail2ban-client get sshd maxretry
    ```
    
    **Expected result:** `3`, matching your `jail.local` configuration.
    
6. Confirm the `findtime` runtime parameter:
    
    ```
    sudo fail2ban-client get sshd findtime
    ```
    
    **Expected result:** `600`, matching your `jail.local` configuration.
    
7. Confirm the `bantime` runtime parameter:
    
    ```
    sudo fail2ban-client get sshd bantime
    ```
    
    **Expected result:** `3600`, matching your `jail.local` configuration.
    
    > **Note:** In production, when an IP exceeds `maxretry` failures within `findtime` seconds, `fail2ban` inserts a kernel-level firewall block that remains in place for `bantime` seconds. The block is invisible to legitimate users and automatically clears when the ban expires, which is why `fail2ban` is a low-overhead defense-in-depth layer behind SSH key authentication.
    

Fantastic job, you made it happen!

You successfully generated an ED25519 key pair, installed key-based authentication for the auditor user, and hardened `sshd` with a `70-hardening.conf` drop-in that disables root login, disables password authentication, and enforces an `AllowUsers` allow-list. You also validated the policy with positive and negative authentication tests, reviewed deny breadcrumbs in `/var/log/auth.log`, and configured `fail2ban` with production-grade values to defend the system against brute-force attempts.

By practicing `ssh-keygen`, `sshd -t`, `sshd -T`, the `sshd_config.d/` drop-in pattern, and the `fail2ban-client status` and `get` command families, you built the SSH defensive posture every Ubuntu server needs before facing the public internet.

```
ubuntu@ip-172-31-24-10:~$ sudo tee /etc/fail2ban/jail.d/sshd.local > /dev/null <<'EOF'
> [DEFAULT]
> ignoreip = 127.0.0.1/8 ::1
> bantime  = 3600
> findtime = 600
> maxretry = 3
> 
> [sshd]
> enabled = true
> port    = ssh
> logpath = %(sshd_log)s
> backend = systemd
> EOF

ubuntu@ip-172-31-24-10:~$ sudo systemctl enable --now fail2ban
Synchronizing state of fail2ban.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable fail2ban
Created symlink /etc/systemd/system/multi-user.target.wants/fail2ban.service → /usr/lib/systemd/system/fail2ban.service.

ubuntu@ip-172-31-24-10:~$ sudo systemctl enable --now fail2ban
Synchronizing state of fail2ban.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable fail2ban

ubuntu@ip-172-31-24-10:~$ sudo fail2ban-client status
Status
|- Number of jail:      1
`- Jail list:   sshd

ubuntu@ip-172-31-24-10:~$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:

ubuntu@ip-172-31-24-10:~$ sudo fail2ban-client get sshd maxretry
3

ubuntu@ip-172-31-24-10:~$ sudo fail2ban-client get sshd findtime
600

ubuntu@ip-172-31-24-10:~$ sudo fail2ban-client get sshd bantime
3600
ubuntu@ip-172-31-24-10:~$ 
```

## OBJECTIVE 2

### Enable UFW, Configure auditd, and Correlate Security Events

The next layer of the audit-readiness story is a host firewall and a verifiable trail of privileged actions. You will enable `ufw` with a deny-by-default inbound posture, allow only the ports the inventory dashboard requires, apply SSH connection rate limiting, and then configure `auditd` rules that capture sensitive file changes, privileged command execution, and authentication events. You will trigger each event type, then practice the query, summary, and cross-source correlation workflows auditors use to verify the audit trail.

### Enable UFW with Deny-by-Default Inbound

1. Confirm UFW is installed but inactive:
    
    ```
    sudo ufw status
    ```
    
    **Expected result:** `Status: inactive`.
    
2. Set the default inbound policy to deny:
    
    ```
    sudo ufw default deny incoming
    ```
    
    **Expected result:** `Default incoming policy changed to 'deny'`, followed by `(be sure to update your rules accordingly)`.
    
3. Set the default outbound policy to allow so the server can still reach package mirrors, DNS servers, and other external services:
    
    ```
    sudo ufw default allow outgoing
    ```
    
    **Expected result:** `Default outgoing policy changed to 'allow'`.
    
4. Allow HTTP traffic to the inventory dashboard on port `80`:
    
    ```
    sudo ufw allow 80/tcp
    ```
    
    **Expected result:** `Rules updated`, followed by `Rules updated (v6)`.
    
5. Apply SSH rate limiting so repeated connection attempts from the same source are throttled:
    
    ```
    sudo ufw limit 22/tcp
    ```
    
    **Expected result:** `Rules updated` followed by `Rules updated (v6)`.
    
    > **Note:** `ufw limit` rejects connections from an IP if it makes six or more attempts in the last 30 seconds. This is a useful defense-in-depth layer that works at the firewall instead of the application layer, complementing the `fail2ban` policy you configured in the previous objective.
    
6. Enable the firewall (it is safe because your existing terminal session is not affected and SSH is allowed):
    
    ```
    sudo ufw --force enable
    ```
    
    **Expected result:** A short summary including `Firewall is active and enabled on system startup`.
    
7. Inspect the active rule set in verbose mode:
    
    ```
    sudo ufw status verbose
    ```
    
    **Expected result:** A multi-section report. The header shows `Status: active`, `Logging: on (low)`, `Default: deny (incoming), allow (outgoing), deny (routed)`, and `New profiles: skip`. Below that, a four-row table shows the rules you added:
    
    ```
        To                         Action      From
        --                         ------      ----
    80/tcp                     ALLOW IN    Anywhere                  
    22/tcp                     LIMIT IN    Anywhere                  
    80/tcp (v6)                ALLOW IN    Anywhere (v6)             
    22/tcp (v6)                LIMIT IN    Anywhere (v6) 
    ```
    
8. Reload the firewall to confirm your rules are persistent and survive a configuration reload:
    
    ```
    sudo ufw reload
    ```
    
    **Expected result:** `Firewall reloaded`. A reload re-reads the saved rules and reapplies them without fully disabling the firewall, so there is no window where the host is unprotected.
    
9. Confirm the rule set is unchanged after the reload:
    
    ```
    sudo ufw status verbose
    ```
    
    **Expected result:** The same report appears as before the reload: `Status: active`, the same default policies, and the same four-row rule table with `80/tcp ALLOW IN`, `22/tcp LIMIT IN`, and their IPv6 counterparts. The rules persisted intact, proving they are stored on disk in `/etc/ufw/` rather than held only in the running kernel, so they also survive a full reboot.
    
    > **Note:** `ufw reload` is the recommended way to apply UFW rule changes in production because it avoids the brief exposure window that `ufw disable` followed by `ufw enable` would create. Because UFW persists its rules to `/etc/ufw/`, the same rule set is restored automatically at boot, which is why the firewall remains effective across reboots.
    

```
ubuntu@ip-172-31-24-10:~$ sudo ufw status
Status: inactive
ubuntu@ip-172-31-24-10:~$ sudo ufw default deny incoming
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)
ubuntu@ip-172-31-24-10:~$ sudo ufw default deny incoming
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)
ubuntu@ip-172-31-24-10:~$ sudo ufw default allow outgoing
Default outgoing policy changed to 'allow'
(be sure to update your rules accordingly)
ubuntu@ip-172-31-24-10:~$ sudo ufw allow 80/tcp
Rules updated
Rules updated (v6)
ubuntu@ip-172-31-24-10:~$ sudo ufw limit 22/tcp
Rules updated
Rules updated (v6)
ubuntu@ip-172-31-24-10:~$ sudo ufw --force enable
Firewall is active and enabled on system startup
ubuntu@ip-172-31-24-10:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
80/tcp                     ALLOW IN    Anywhere                  
22/tcp                     LIMIT IN    Anywhere                  
80/tcp (v6)                ALLOW IN    Anywhere (v6)             
22/tcp (v6)                LIMIT IN    Anywhere (v6)             

ubuntu@ip-172-31-24-10:~$ sudo ufw reload
Firewall reloaded
ubuntu@ip-172-31-24-10:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
80/tcp                     ALLOW IN    Anywhere                  
22/tcp                     LIMIT IN    Anywhere                  
80/tcp (v6)                ALLOW IN    Anywhere (v6)             
22/tcp (v6)                LIMIT IN    Anywhere (v6)             

ubuntu@ip-172-31-24-10:~$
```

### Define Audit Rules for Sensitive Events

1. Write a custom audit rules file covering the events the auditors will look for:
    
    ```
sudo tee /etc/audit/rules.d/security-hardening.rules > /dev/null <<'EOF'
# Watch sensitive account files for any write or attribute change
-w /etc/passwd -p wa -k account-changes
-w /etc/shadow -p wa -k account-changes

# Watch the SSH auth log for any write
-w /var/log/auth.log -p wa -k auth-log

# Audit every execution of sudo by tagging it with the 'privileged' key
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/sudo -F key=privileged
EOF
    ```
    
    **Expected result:** No output.
    
2. Load the new rules from `/etc/audit/rules.d/` into the running audit daemon:
    
    ```
    sudo augenrules --load
    ```
    
    **Expected result:** `No rules` is replaced after loading. You may see informational output similar to `enabled 1`, `failure 1`, `pid <pid>`, and `rate_limit 0`, indicating the rules engine accepted the new ruleset.
    
3. Confirm the rules are active in the running kernel auditing subsystem:
    
    ```
    sudo auditctl -l
    ```
    
    **Expected result:** Four lines appear, one for each rule above: `-w /etc/passwd -p wa -k account-changes`, `-w /etc/shadow -p wa -k account-changes`, `-w /var/log/auth.log -p wa -k auth-log`, and the `execve` rule tagged `privileged`.
    

```
ubuntu@ip-172-31-24-10:~$ sudo cat /etc/audit/rules.d/security-hardening.rules
# Watch sensitive account files for any write or attribute change
-w /etc/passwd -p wa -k account-changes
-w /etc/shadow -p wa -k account-changes

# Watch the SSH auth log for any write
-w /var/log/auth.log -p wa -k auth-log

# Audit every execution of sudo by tagging it with the 'privileged' key
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/sudo -F key=privileged

ubuntu@ip-172-31-24-10:~$ sudo augenrules --load
No rules
enabled 1
failure 1
pid 2516
rate_limit 0
backlog_limit 8192
lost 0
backlog 0
backlog_wait_time 60000
backlog_wait_time_actual 0
enabled 1
failure 1
pid 2516
rate_limit 0
backlog_limit 8192
lost 0
backlog 3
backlog_wait_time 60000
backlog_wait_time_actual 0
enabled 1
failure 1
pid 2516
rate_limit 0
backlog_limit 8192
lost 0
backlog 5
backlog_wait_time 60000
backlog_wait_time_actual 0

ubuntu@ip-172-31-24-10:~$ sudo auditctl -l
-w /etc/passwd -p wa -k account-changes
-w /etc/shadow -p wa -k account-changes
-w /var/log/auth.log -p wa -k auth-log
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/sudo -F key=privileged
ubuntu@ip-172-31-24-10:~$ 
```
### Trigger and Query Audit Events

1. Change `auditor`'s comment field to generate an `account-changes` event against `/etc/passwd`:
    
    ```
    sudo usermod -c "Auditor Account" auditor
    ```
    
    **Expected result:** No output.
    
2. Generate a `privileged` event by running a `sudo` command as your normal user:
    
    ```
    sudo ls /root
    ```
    
    **Expected result:** A directory listing of `/root` (typically empty, or containing only `snap`).
    
3. Generate an `auth-log` write event by triggering a failed authentication:
    
    ```
    su - root -c whoami < /dev/null || echo 'failed su recorded'
    ```
    
    **Expected result:** A line similar to `su: Authentication failure` (root has no password set), followed by `failed su recorded`.
    
4. Query the audit log for the account-changes events. The output is piped through `tail` so the most recent records stay within the visible terminal window:
    
    ```
    sudo ausearch -k account-changes -ts recent | tail -n 25
    ```
    
    **Expected result:** A multi-line record block showing the `time->` timestamp, `type=SYSCALL`, and `type=PATH` entries identifying `/etc/passwd`, the calling user, and the syscall result. Because the output is bounded with `tail`, the most recent record appears at the bottom of the display. If several records match, the top of the output may be clipped, which is expected.

```
ubuntu@ip-172-31-24-10:~$ sudo usermod -c "Auditor Account" auditor
ubuntu@ip-172-31-24-10:~$ sudo ls /root
snap
ubuntu@ip-172-31-24-10:~$ su - root -c whoami < /dev/null || echo 'failed su recorded'
Password: su: Authentication failure
failed su recorded
ubuntu@ip-172-31-24-10:~$ su - root -c whoami < /dev/null || echo 'failed su recorded'
Password: su: Authentication failure
failed su recorded
ubuntu@ip-172-31-24-10:~$ sudo ausearch -k account-changes -ts recent | tail -n 25
type=SOCKADDR msg=audit(1779724852.729:1419): saddr=100000000000000000000000
type=SYSCALL msg=audit(1779724852.729:1419): arch=c000003e syscall=44 success=yes exit=1084 a0=3 a1=7ffc0ee67580 a2=43c a3=0 items=1 ppid=5973 pid=5987 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=18 comm="auditctl" exe="/usr/sbin/auditctl" subj=unconfined key=(null)
type=CONFIG_CHANGE msg=audit(1779724852.729:1419): auid=1000 ses=18 subj=unconfined op=add_rule key="account-changes" list=4 res=1
----
time->Mon May 25 16:03:33 2026
type=PROCTITLE msg=audit(1779725013.419:1436): proctitle=757365726D6F64002D630041756469746F72204163636F756E740061756469746F72
type=PATH msg=audit(1779725013.419:1436): item=0 name="/etc/passwd" inode=33756 dev=103:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779725013.419:1436): cwd="/home/ubuntu"
type=SYSCALL msg=audit(1779725013.419:1436): arch=c000003e syscall=257 success=yes exit=5 a0=ffffff9c a1=5ec0b9fb02a0 a2=20902 a3=0 items=1 ppid=6002 pid=6003 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=18 comm="usermod" exe="/usr/sbin/usermod" subj=unconfined key="account-changes"
----
time->Mon May 25 16:03:33 2026
type=PROCTITLE msg=audit(1779725013.422:1437): proctitle=757365726D6F64002D630041756469746F72204163636F756E740061756469746F72
type=PATH msg=audit(1779725013.422:1437): item=0 name="/etc/shadow" inode=3086 dev=103:01 mode=0100640 ouid=0 ogid=42 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779725013.422:1437): cwd="/home/ubuntu"
type=SYSCALL msg=audit(1779725013.422:1437): arch=c000003e syscall=257 success=yes exit=6 a0=ffffff9c a1=5ec0b9fb0fc0 a2=20902 a3=0 items=1 ppid=6002 pid=6003 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=18 comm="usermod" exe="/usr/sbin/usermod" subj=unconfined key="account-changes"
----
time->Mon May 25 16:03:33 2026
type=PROCTITLE msg=audit(1779725013.429:1439): proctitle=757365726D6F64002D630041756469746F72204163636F756E740061756469746F72
type=PATH msg=audit(1779725013.429:1439): item=4 name="/etc/passwd" inode=33732 dev=103:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779725013.429:1439): item=3 name="/etc/passwd" inode=33756 dev=103:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779725013.429:1439): item=2 name="/etc/passwd+" inode=33732 dev=103:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779725013.429:1439): item=1 name="/etc/" inode=42 dev=103:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779725013.429:1439): item=0 name="/etc/" inode=42 dev=103:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779725013.429:1439): cwd="/home/ubuntu"
type=SYSCALL msg=audit(1779725013.429:1439): arch=c000003e syscall=82 success=yes exit=0 a0=7ffcef410040 a1=5ec0b9fb02a0 a2=7ffcef40ffb0 a3=100 items=5 ppid=6002 pid=6003 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=18 comm="usermod" exe="/usr/sbin/usermod" subj=unconfined key="account-changes"
ubuntu@ip-172-31-24-10:~$
```



1. Query for the privileged sudo events, again bounding the output with `tail` so the newest records stay on screen:
    
    ```
    sudo ausearch -k privileged -ts recent | tail -n 25
    ```
    
    **Expected result:** One or more record blocks showing the `execve` syscall on `/usr/bin/sudo`, with command-line arguments preserved.
    
    > **Note:** As with the previous query, the most recent record is at the bottom of the display and the top may be clipped if several records matched, which is expected.


```
ubuntu@ip-172-31-24-10:~$ sudo ausearch -k privileged -ts recent | tail -n 25
type=SYSCALL msg=audit(1779725013.402:1431): arch=c000003e syscall=59 success=yes exit=0 a0=608c65a03170 a1=608c659f4e70 a2=608c659c8240 a3=608c659f4e70 items=2 ppid=3927 pid=6001 auid=1000 uid=1000 gid=1000 euid=0 suid=0 fsuid=0 egid=1000 sgid=1000 fsgid=1000 tty=pts1 ses=18 comm="sudo" exe="/usr/bin/sudo" subj=unconfined key="privileged"
----
time->Mon May 25 16:03:41 2026
type=PROCTITLE msg=audit(1779725021.242:1442): proctitle=7375646F006C73002F726F6F74
type=PATH msg=audit(1779725021.242:1442): item=1 name="/lib64/ld-linux-x86-64.so.2" inode=6489 dev=103:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779725021.242:1442): item=0 name="/usr/bin/sudo" inode=2166 dev=103:01 mode=0104755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779725021.242:1442): cwd="/home/ubuntu"
type=EXECVE msg=audit(1779725021.242:1442): argc=3 a0="sudo" a1="ls" a2="/root"
type=SYSCALL msg=audit(1779725021.242:1442): arch=c000003e syscall=59 success=yes exit=0 a0=608c658d20c0 a1=608c65a08010 a2=608c659c8240 a3=608c659fa660 items=2 ppid=3927 pid=6010 auid=1000 uid=1000 gid=1000 euid=0 suid=0 fsuid=0 egid=1000 sgid=1000 fsgid=1000 tty=pts1 ses=18 comm="sudo" exe="/usr/bin/sudo" subj=unconfined key="privileged"
----
time->Mon May 25 16:04:26 2026
type=PROCTITLE msg=audit(1779725066.477:1451): proctitle=7375646F006175736561726368002D6B006163636F756E742D6368616E676573002D747300726563656E74
type=PATH msg=audit(1779725066.477:1451): item=1 name="/lib64/ld-linux-x86-64.so.2" inode=6489 dev=103:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779725066.477:1451): item=0 name="/usr/bin/sudo" inode=2166 dev=103:01 mode=0104755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779725066.477:1451): cwd="/home/ubuntu"
type=EXECVE msg=audit(1779725066.477:1451): argc=6 a0="sudo" a1="ausearch" a2="-k" a3="account-changes" a4="-ts" a5="recent"
type=SYSCALL msg=audit(1779725066.477:1451): arch=c000003e syscall=59 success=yes exit=0 a0=608c659f7130 a1=608c658d5770 a2=608c659c8240 a3=8 items=2 ppid=3927 pid=6016 auid=1000 uid=1000 gid=1000 euid=0 suid=0 fsuid=0 egid=1000 sgid=1000 fsgid=1000 tty=pts1 ses=18 comm="sudo" exe="/usr/bin/sudo" subj=unconfined key="privileged"
----
time->Mon May 25 16:05:52 2026
type=PROCTITLE msg=audit(1779725152.805:1465): proctitle=7375646F006175736561726368002D6B0070726976696C65676564002D747300726563656E74
type=PATH msg=audit(1779725152.805:1465): item=1 name="/lib64/ld-linux-x86-64.so.2" inode=6489 dev=103:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1779725152.805:1465): item=0 name="/usr/bin/sudo" inode=2166 dev=103:01 mode=0104755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779725152.805:1465): cwd="/home/ubuntu"
type=EXECVE msg=audit(1779725152.805:1465): argc=6 a0="sudo" a1="ausearch" a2="-k" a3="privileged" a4="-ts" a5="recent"
type=SYSCALL msg=audit(1779725152.805:1465): arch=c000003e syscall=59 success=yes exit=0 a0=608c659a32f0 a1=608c658d5770 a2=608c659c8240 a3=8 items=2 ppid=3927 pid=6029 auid=1000 uid=1000 gid=1000 euid=0 suid=0 fsuid=0 egid=1000 sgid=1000 fsgid=1000 tty=pts1 ses=18 comm="sudo" exe="/usr/bin/sudo" subj=unconfined key="privileged"
ubuntu@ip-172-31-24-10:~$
```

1. Query the audit log for the `auth-log` key. Unlike the previous two queries, this command omits `-ts recent`, so it searches the full audit history and guarantees the rule-creation event is always in scope.
    
    ```
    sudo ausearch -k auth-log
    ```
    
    **Expected result:** At least one record block tagged `key="auth-log"` appears. Reliably, you will see a `type=CONFIG_CHANGE` record with `op=add_rule key="auth-log"`, which is the audit subsystem recording the moment you loaded the rule with `augenrules --load`. Its presence proves the `auth-log` key is active and that `auditd` is recording against it. You may also see additional records after the system writes to `/var/log/auth.log`, but those writes are produced asynchronously by the logging daemon and may not have been flushed yet, so do not be concerned if the `CONFIG_CHANGE` record is the only block returned.
    
    **Expected result:** At least one record block tagged `key="auth-log"` appears. You should reliably see a `type=CONFIG_CHANGE` record with `op=add_rule key="auth-log"`, which confirms the `auth-log` audit rule was loaded successfully. Additional records related to writes to `/var/log/auth.log` may also appear, but those entries are generated asynchronously by `rsyslog` and may not be written immediately.
    
    > **Note:** `/var/log/auth.log` is written by `rsyslog` as it reads from the systemd journal, so authentication events may not appear immediately in `auditd` output. The deterministic proof that `auth-log` monitoring is working comes from two reliable signals together:
    > 
    > - `sudo auditctl -l` shows that the `/var/log/auth.log` watch is loaded.
    > - The `tail /var/log/auth.log` cross-reference step later in this objective shows the failed `su` attempt recorded in the file itself.
    > 
    > This query is intentionally not piped through `tail` because the record you most want to see — the `CONFIG_CHANGE` rule-add event — is the oldest one and would otherwise be omitted.
    > 
    > If the output is longer than your terminal window, you can scroll back through it using terminal scrollback. Press `Control+b` then `[` to enter scroll mode, use the arrow keys or `Page Up` to scroll, and press `q` to exit scroll mode when you are done.
    
2. Generate a summary report of all audit activity since boot:
    
    ```
    sudo aureport --summary
    ```
    
    **Expected result:** A `Summary Report` appears with a `Range of time in logs:` header followed by categorized audit event counts in `Number of <category>: <count>` format. Categories related to the activities performed in this objective, such as failed authentications, account changes, keys, and events, should contain non-zero values.

```
ubuntu@ip-172-31-24-10:~$ sudo ausearch -k auth-log
----
time->Mon May 25 16:00:52 2026
type=PROCTITLE msg=audit(1779724852.729:1420): proctitle=2F7362696E2F617564697463746C002D52002F6574632F61756469742F61756469742E72756C6573
type=PATH msg=audit(1779724852.729:1420): item=0 name="/var/log/" inode=84647 dev=103:01 mode=040775 ouid=0 ogid=102 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1779724852.729:1420): cwd="/home/ubuntu"
type=SOCKADDR msg=audit(1779724852.729:1420): saddr=100000000000000000000000
type=SYSCALL msg=audit(1779724852.729:1420): arch=c000003e syscall=44 success=yes exit=1084 a0=3 a1=7ffc0ee67580 a2=43c a3=0 items=1 ppid=5973 pid=5987 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=18 comm="auditctl" exe="/usr/sbin/auditctl" subj=unconfined key=(null)
type=CONFIG_CHANGE msg=audit(1779724852.729:1420): auid=1000 ses=18 subj=unconfined op=add_rule key="auth-log" list=4 res=1
ubuntu@ip-172-31-24-10:~$ sudo aureport --summary

Summary Report
======================
Range of time in logs: 05/25/26 13:06:34.659 - 05/25/26 16:07:48.570
Selected time for report: 05/25/26 13:06:34 - 05/25/26 16:07:48.570
Number of changes in configuration: 263
Number of changes to accounts, groups, or roles: 12
Number of logins: 2
Number of failed logins: 2
Number of authentications: 0
Number of failed authentications: 2
Number of users: 4
Number of terminals: 11
Number of host names: 4
Number of executables: 12
Number of commands: 11
Number of files: 8
Number of AVC's: 0
Number of MAC events: 0
Number of failed syscalls: 0
Number of anomaly events: 0
Number of responses to anomaly events: 0
Number of crypto events: 0
Number of integrity events: 0
Number of virt events: 0
Number of keys: 3
Number of process IDs: 400
Number of events: 1358

ubuntu@ip-172-31-24-10:~$ 
```


1. Now watch security events arrive in the systemd journal in real time. Start a live follow of sudo activity in the background. The `timeout 30` wrapper automatically stops the follow after 30 seconds:
    
    ```
    sudo timeout 30 journalctl -f _COMM=sudo --no-pager &
    ```
    
    **Expected result:** A job-control line similar to `[1] 12345` appears, followed by recent sudo journal entries streaming in real time. The follow automatically exits after 30 seconds.
    
2. While the follow is still running, trigger a fresh privileged event in the same terminal:
    
    ```
    sudo ls /root
    ```
    
    <!-- **Expected result:** The `/root` directory listing appears (a single entry, `snap`). Within a second or two, a new journal line for this exact `ls /root` command streams in from the background follow, showing the invoking user, the target `USER=root`, and `COMMAND=/usr/bin/ls /root`. Because the follow is running in the background, this streamed line may appear right after your command output or attached to the next shell prompt, and the interleaving of live log output with your prompt can look slightly messy. That is normal and is exactly what live monitoring looks like in a single terminal: it is the real-time equivalent of running `journalctl -f` in a second terminal while working in the first. 
    
    **Expected result:** The `/root` directory listing appears (`snap`). Within a few seconds, a new sudo journal entry for the `ls /root` command streams in from the background follow, showing the invoking user, `USER=root`, and `COMMAND=/usr/bin/ls /root`. Because the follow is running in the background, the live log output may appear interleaved with your shell prompt. This is normal.
    
3. The follow continues running in the background until the 30-second `timeout` expires. After at least 30 seconds have passed, press Enter once to redraw the prompt.
    
    You will then see the background job's completion notice:
    
    ```
    [1]+  Exit 124                sudo timeout 30 journalctl -f _COMM=sudo --no-pager
    ```
    
    **Expected result:** After you press `Enter`, the shell redraws the prompt and displays any pending background-job notifications. If the message does not appear yet, wait a few more seconds and press `Enter` again.
    
    `Exit 124` is the expected status code returned by `timeout` when it stops a command after the specified time limit.
    
    > **Note:** In production environments, you would typically run `sudo journalctl -f` in one terminal while working in another. Because this lab uses a single browser terminal, the follow is backgrounded with `&` and limited with `timeout` to prevent it from blocking the shell.Live following is what you use during an active incident; the retrospective `journalctl --since` query in the next step is what you use afterward to reconstruct what happened.


```
ubuntu@ip-172-31-24-10:~$ sudo timeout 30 journalctl -f _COMM=sudo --no-pager &
[1] 6045
ubuntu@ip-172-31-24-10:~$ May 25 16:05:52 ip-172-31-24-10 sudo[6029]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:05:52 ip-172-31-24-10 sudo[6029]: pam_unix(sudo:session): session closed for user root
May 25 16:06:33 ip-172-31-24-10 sudo[6036]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/sbin/ausearch -k auth-log
May 25 16:06:33 ip-172-31-24-10 sudo[6036]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:06:33 ip-172-31-24-10 sudo[6036]: pam_unix(sudo:session): session closed for user root
May 25 16:07:48 ip-172-31-24-10 sudo[6039]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/sbin/aureport --summary
May 25 16:07:48 ip-172-31-24-10 sudo[6039]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:07:48 ip-172-31-24-10 sudo[6039]: pam_unix(sudo:session): session closed for user root
May 25 16:09:08 ip-172-31-24-10 sudo[6045]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/timeout 30 journalctl -f _COMM=sudo --no-pager
May 25 16:09:08 ip-172-31-24-10 sudo[6045]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
sudo timeout 30 journalctl -f _COMM=sudo --no-pager &^C
ubuntu@ip-172-31-24-10:~$ sudo ls /root
snap
ubuntu@ip-172-31-24-10:~$ May 25 16:09:33 ip-172-31-24-10 sudo[6049]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/ls /root
May 25 16:09:33 ip-172-31-24-10 sudo[6049]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:09:34 ip-172-31-24-10 sudo[6049]: pam_unix(sudo:session): session closed for user root

[1]+  Exit 124                sudo timeout 30 journalctl -f _COMM=sudo --no-pager
ubuntu@ip-172-31-24-10:~$
```


1. Cross-reference the same events retrospectively, the way you would when reconstructing an incident:
    
    ```
    sudo journalctl _COMM=sudo --since "10 minutes ago" --no-pager | tail -n 20
    ```
    
    **Expected result:** Recent journal entries from sudo activity, including the privileged commands you just ran. Each line shows the invoking user (`ubuntu`), the TTY, the working directory, the target user (`USER=root`), and the full command line.
    
    > **Note:** Sudo is a setuid binary, not a systemd service, so it does not have a unit file you can query with `-u sudo.service`. The journal entries are tagged by their kernel process name instead, which is what `_COMM=sudo` matches.


```
ubuntu@ip-172-31-24-10:~$ sudo journalctl _COMM=sudo --since "10 minutes ago" --no-pager | tail -n 20
May 25 16:09:08 ip-172-31-24-10 sudo[6045]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/timeout 30 journalctl -f _COMM=sudo --no-pager
May 25 16:09:08 ip-172-31-24-10 sudo[6045]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:09:33 ip-172-31-24-10 sudo[6049]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/ls /root
May 25 16:09:33 ip-172-31-24-10 sudo[6049]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:09:34 ip-172-31-24-10 sudo[6049]: pam_unix(sudo:session): session closed for user root
May 25 16:09:38 ip-172-31-24-10 sudo[6045]: pam_unix(sudo:session): session closed for user root
May 25 16:10:58 ip-172-31-24-10 sudo[6152]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/timeout 30 journalctl -f _COMM=sudo --no-pager
May 25 16:10:58 ip-172-31-24-10 sudo[6152]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:11:14 ip-172-31-24-10 sudo[6214]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/journalctl _COMM=sudo --since '10 minutes ago' --no-pager
May 25 16:11:14 ip-172-31-24-10 sudo[6214]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:11:14 ip-172-31-24-10 sudo[6214]: pam_unix(sudo:session): session closed for user root
May 25 16:11:28 ip-172-31-24-10 sudo[6152]: pam_unix(sudo:session): session closed for user root
May 25 16:11:41 ip-172-31-24-10 sudo[6221]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/journalctl _COMM=sudo --since '10 minutes ago' --no-pager
May 25 16:11:41 ip-172-31-24-10 sudo[6221]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:11:41 ip-172-31-24-10 sudo[6221]: pam_unix(sudo:session): session closed for user root
May 25 16:11:46 ip-172-31-24-10 sudo[6225]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/tail -n 30 /var/log/auth.log
May 25 16:11:46 ip-172-31-24-10 sudo[6225]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
May 25 16:11:46 ip-172-31-24-10 sudo[6225]: pam_unix(sudo:session): session closed for user root
May 25 16:12:28 ip-172-31-24-10 sudo[6232]:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/journalctl _COMM=sudo --since '10 minutes ago' --no-pager
May 25 16:12:28 ip-172-31-24-10 sudo[6232]: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
ubuntu@ip-172-31-24-10:~$
```

1. Cross-reference the same events in `/var/log/auth.log`:
    
    ```
    sudo tail -n 30 /var/log/auth.log | grep -E '(sudo|su:)'
    ```
    
    **Expected result:** Entries in `/var/log/auth.log` matching the same events you queried from `auditd` and `journalctl`, demonstrating how a single privileged action appears across three independent log sources. Correlating these sources is a common incident investigation technique.
    
    > **Note:** During an audit, you rely on this overlap deliberately: if a record exists in one log source but not the others, you have either a tampered log or a configuration gap, both of which are reportable findings.
    

Excellent work, you've built the audit trail! You've successfully configured a deny-by-default firewall with `ufw`, opened only the ports required by the inventory dashboard, applied SSH rate limiting to complement your `fail2ban` policy, and confirmed that the rules survive a `ufw reload`.

You also created audit rules that capture account changes, sudo invocations, and `auth.log` writes. You then validated the audit trail by triggering each event type and observing the results both live with `journalctl -f` and retrospectively with `ausearch`, `aureport`, `journalctl`, and `/var/log/auth.log`.

With these skills, you can now answer the two questions every auditor asks: what changed, and who did it?

```
ubuntu@ip-172-31-24-10:~$ sudo tail -n 30 /var/log/auth.log | grep -E '(sudo|su:)'
2026-05-25T16:06:33.805985+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:07:48.570940+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/sbin/aureport --summary
2026-05-25T16:07:48.572359+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:07:48.578757+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:09:08.864847+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/timeout 30 journalctl -f _COMM=sudo --no-pager
2026-05-25T16:09:08.866213+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:09:33.998807+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/ls /root
2026-05-25T16:09:34.000130+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:09:34.003551+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:09:38.871193+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:10:58.526787+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/timeout 30 journalctl -f _COMM=sudo --no-pager
2026-05-25T16:10:58.531653+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:11:14.241382+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/journalctl _COMM=sudo --since '10 minutes ago' --no-pager
2026-05-25T16:11:14.242506+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:11:14.249146+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:11:28.534116+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:11:41.768025+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/journalctl _COMM=sudo --since '10 minutes ago' --no-pager
2026-05-25T16:11:41.769043+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:11:41.775397+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:11:46.368981+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/tail -n 30 /var/log/auth.log
2026-05-25T16:11:46.370143+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:11:46.373859+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:12:28.602052+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/journalctl _COMM=sudo --since '10 minutes ago' --no-pager
2026-05-25T16:12:28.603498+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:12:28.609714+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:13:04.133369+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/tail -n 30 /var/log/auth.log
2026-05-25T16:13:04.134491+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
2026-05-25T16:13:04.137191+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session closed for user root
2026-05-25T16:13:37.392471+00:00 ip-172-31-24-10 sudo:   ubuntu : TTY=pts/1 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/tail -n 30 /var/log/auth.log
2026-05-25T16:13:37.393725+00:00 ip-172-31-24-10 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by ubuntu(uid=1000)
ubuntu@ip-172-31-24-10:~$ 
```

## OBJECTIVE 3

### Manage AppArmor Profiles and Review Denial Logs

The final layer of the audit-readiness story is mandatory access control. The `nginx` service is already running under a custom AppArmor profile in enforce mode that deliberately forbids writes to `/etc/`.

You will inspect the profile, switch it into complain mode to observe how AppArmor logs policy violations, then trigger a deliberately rogue write attempt and inspect the resulting audit trail. After that, you will switch the profile back into enforce mode and watch the same attempt be blocked. You will close out by practicing `aa-logprof`, the interactive workflow used in production to refine AppArmor profiles against real workloads.

### Inspect the AppArmor Posture

1. Get a summary of all loaded AppArmor profiles:
    
    ```
    sudo aa-status --json | python3 -c "import json,sys; d=json.load(sys.stdin); print('profiles loaded:', sum(len(v) for v in d['profiles'].values())); print('processes confined:', sum(len(v) for v in d['processes'].values()))" 2>/dev/null || sudo aa-status
    ```
    
    **Expected result:** Either a two-line summary including `profiles loaded:` and `processes confined:`, or (if Python parsing fails) the standard `aa-status` text report showing the number of loaded profiles, the number in enforce vs complain mode, and a list of confined processes including `/usr/sbin/nginx`.
    
2. Inspect the custom nginx profile that was deployed at lab startup:
    
    ```
    sudo cat /etc/apparmor.d/usr.sbin.nginx
    ```
    
    **Expected result:** The profile file showing `#include` lines, the capability set (`dac_override`, `dac_read_search`, `net_bind_service`, `setgid`, `setuid`), several file permissions for `/etc/nginx/`, `/run/nginx/`, `/var/log/nginx/`, and `/var/www/`, and the explicit `deny /etc/** w,` rule that blocks all writes outside the allowed paths.

```
ubuntu@ip-172-31-24-10:~$ sudo aa-status --json | python3 -c "import json,sys; d=json.load(sys.stdin); print('profiles loaded:', sum(len(v) for v in d['profiles'].values())); print('processes confined:', sum(len(v) for v in d['processes'].values()))" 2>/dev/null || sudo aa-status
profiles loaded: 1295
processes confined: 4
ubuntu@ip-172-31-24-10:~$ sudo cat /etc/apparmor.d/usr.sbin.nginx
#include <tunables/global>

/usr/sbin/nginx {
    #include <abstractions/base>
    #include <abstractions/nis>

    capability dac_override,
    capability dac_read_search,
    capability net_bind_service,
    capability setgid,
    capability setuid,

    /etc/group r,
    /etc/nginx/** r,
    /etc/passwd r,
    /etc/ssl/openssl.cnf r,

    /run/nginx.pid rw,
    /run/nginx/** rw,

    /usr/sbin/nginx mr,

    /var/log/nginx/*.log w,
    /var/www/** r,

    # Explicitly DENY writes to /etc/ to make denials visible in dmesg
    deny /etc/** w,
}
ubuntu@ip-172-31-24-10:~$ 
```

1. Confirm `nginx` is running and confined by the profile:
    
    ```
    sudo aa-status | grep -E '(nginx|profiles are in)'
    ```
    
    **Expected result:** A line similar to `X profiles are in enforce mode.` and `/usr/sbin/nginx` listed in either the enforce or running-confined section.
    
```
ubuntu@ip-172-31-24-10:~$ sudo aa-status | grep -E '(nginx|profiles are in)'
51 profiles are in enforce mode.
   /usr/sbin/nginx
6 profiles are in complain mode.
0 profiles are in prompt mode.
0 profiles are in kill mode.
89 profiles are in unconfined mode.
ubuntu@ip-172-31-24-10:~$ 
```
### Switch to Complain Mode and Probe the Audit Trail

1. Switch the nginx profile from enforce to complain mode:
    
    ```
    sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
    ```
    
    **Expected result:** `Setting /etc/apparmor.d/usr.sbin.nginx to complain mode.`
    
2. Confirm the profile is now in complain mode:
    
    ```
    sudo aa-status | grep -E 'complain mode|enforce mode' | head -2
    ```
    
    **Expected result:** Two summary lines indicating profiles in enforce mode and profiles in complain mode. The nginx profile contributes to the complain count.
    
3. Run the rogue demo script directly as root to set a baseline (this runs outside the nginx profile and is unconfined):
    
    ```
    sudo /usr/local/bin/rogue-demo.sh
    ```
    
    **Expected result:** `WRITE SUCCEEDED: /etc/rogue-marker exists`. Outside the nginx profile, the root user is allowed to write anywhere, so the script succeeds.
    
4. Remove the marker so you have a clean slate for the confined runs:
    
    ```
    sudo rm -f /etc/rogue-marker
    ```
    
    **Expected result:** No output.
    
5. Run the rogue script under the nginx AppArmor profile while the profile is still in complain mode:
    
    ```
    sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
    ```
    
    **Expected result:** `WRITE FAILED: AppArmor or filesystem permissions blocked the write`, preceded by a `/usr/local/bin/rogue-demo.sh: line 5: /etc/rogue-marker: Permission denied` message from the shell.
    
    > **Note:** You might expect complain mode to log and allow the write. The reason it still blocks is the `deny /etc/** w,` rule at the bottom of the profile. Explicit `deny` rules in AppArmor take precedence over the complain-vs-enforce mode setting; they always block, in both modes. This is intentional, because a profile author marks a path as `deny` precisely because the write is never acceptable, mode notwithstanding. Without the deny rule, complain mode would have logged the violation and allowed the write to succeed.


```
ubuntu@ip-172-31-24-10:~$ sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
Setting /etc/apparmor.d/usr.sbin.nginx to complain mode.
ubuntu@ip-172-31-24-10:~$ sudo aa-status | grep -E 'complain mode|enforce mode' | head -2
50 profiles are in enforce mode.
7 profiles are in complain mode.
ubuntu@ip-172-31-24-10:~$ sudo /usr/local/bin/rogue-demo.sh
WRITE SUCCEEDED: /etc/rogue-marker exists
ubuntu@ip-172-31-24-10:~$ sudo rm -f /etc/rogue-marker
ubuntu@ip-172-31-24-10:~$ sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
/usr/local/bin/rogue-demo.sh: line 5: /etc/rogue-marker: Permission denied
WRITE FAILED: AppArmor or filesystem permissions blocked the write
```


1. Inspect the audit log to see what the kernel recorded about the denied write:
    
    ```
    sudo ausearch -m AVC -ts recent | tail -n 30
    ```
    
    **Expected result:** One of two outputs is correct:
    
    - **AVC records present:** A series of records showing `apparmor="DENIED"`, the operation (e.g., `open`), the profile (`/usr/sbin/nginx`), the path (`/etc/rogue-marker`), and the requested permissions. This is the textbook output from a system with full AVC auditing enabled.
        
    - **`<no matches>`:** Also a valid result on this lab AMI and on many production Ubuntu 24.04 systems. The denial still happened (the script reported `WRITE FAILED`), but the kernel did not emit a `type=AVC` audit record for it. AppArmor may be logging the event under a different audit message type, or only to the kernel ring buffer with rate limiting. The events are still being captured somewhere in `/var/log/audit/audit.log`, which you will prove later when `aa-logprof` reads them.
        
    
    > **Note:** Production Ubuntu hosts vary in whether AppArmor denials surface through `ausearch -m AVC`, depending on the `audit=1` kernel parameter, the kernel build, and the audit dispatcher configuration. Experienced operators often check multiple places, including `ausearch -m AVC`, `ausearch -m USER_AVC`, raw `grep apparmor /var/log/audit/audit.log`, `dmesg | grep apparmor`, and `journalctl -k | grep apparmor`, and use whichever one yields results on the specific host.
    
2. Remove the marker to prepare for the enforce-mode test:
    
    ```
    sudo rm -f /etc/rogue-marker
    ```
    
    **Expected result:** No output.
    
```
ubuntu@ip-172-31-24-10:~$ sudo ausearch -m AVC -ts recent | tail -n 30
<no matches>
ubuntu@ip-172-31-24-10:~$ sudo rm -f /etc/rogue-marker
ubuntu@ip-172-31-24-10:~$ 
```

### Switch to Enforce Mode and Watch the Same Write Be Blocked

1. Switch the nginx profile back to enforce mode:
    
    ```
    sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
    ```
    
    **Expected result:** `Setting /etc/apparmor.d/usr.sbin.nginx to enforce mode.`
    
2. Confirm the profile is enforcing:
    
    ```
    sudo aa-status | grep -E '/usr/sbin/nginx' | head -3
    ```
    
    **Expected result:** One or more lines confirming `/usr/sbin/nginx` is in the enforce-mode section of the status output.
    
3. Re-run the rogue script under the now-enforcing nginx profile:
    
    ```
    sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
    ```
    
    **Expected result:** A `/bin/bash: /usr/local/bin/rogue-demo.sh: Permission denied` message. In enforce mode, the profile blocks the shell from even reading the rogue script, because `/usr/local/bin/rogue-demo.sh` is not in the profile's allowed paths. Compared to complain mode (where the write to `/etc/rogue-marker` was blocked by the explicit `deny` rule), enforce mode is stricter still: it blocks everything not explicitly allowed.
    
4. Confirm the marker file does not exist:
    
    ```
    ls -l /etc/rogue-marker 2>&1
    ```
    
    **Expected result:** `ls: cannot access '/etc/rogue-marker': No such file or directory`. The enforce-mode profile blocked the write before the file could be created.
    
5. Inspect the audit log to see what the kernel recorded about the enforce-mode denial:
    
    ```
    sudo ausearch -m AVC -ts recent | tail -n 30
    ```
    
    **Expected result:** Same two-outcome behavior as in complain mode. Either:
    
    - **AVC records present:** Records showing `apparmor="DENIED"`, the operation (e.g., `exec` on the script, or `open` on `/etc/rogue-marker`), the profile `/usr/sbin/nginx`, and the requested permissions. In enforce mode the `DENIED` keyword can appear at every layer the script tried to traverse, not just the final write target.
        
    - **`<no matches>`:** Also valid on this AMI. The denial demonstrably worked (the script could not be executed and `/etc/rogue-marker` does not exist), even though no `type=AVC` record was emitted. As before, the events are present in `/var/log/audit/audit.log` and `aa-logprof` will read them in the next section.
        
    
    > **Note:** Regardless of whether your `ausearch -m AVC` query returns records or `<no matches>`, the previous steps already proved enforce mode is working: the `aa-exec` invocation failed with `Permission denied` and no file was created. The audit-log inspection is about _verifying the trail_ for forensic purposes, not about whether the denial occurred. In a real incident, you would consult multiple log sources to triangulate the events the audit subsystem captured.
    

```
ubuntu@ip-172-31-24-10:~$ sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
Setting /etc/apparmor.d/usr.sbin.nginx to enforce mode.
ubuntu@ip-172-31-24-10:~$ sudo aa-status | grep -E '/usr/sbin/nginx' | head -3
   /usr/sbin/nginx
   /usr/sbin/nginx//null-/usr/bin/date
   /usr/sbin/nginx//null-/usr/bin/whoami
ubuntu@ip-172-31-24-10:~$ sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
/bin/bash: /usr/local/bin/rogue-demo.sh: Permission denied
ubuntu@ip-172-31-24-10:~$ ls -l /etc/rogue-marker 2>&1
ls: cannot access '/etc/rogue-marker': No such file or directory
ubuntu@ip-172-31-24-10:~$ sudo ausearch -m AVC -ts recent | tail -n 30
<no matches>
ubuntu@ip-172-31-24-10:~$
```
### Practice the aa-logprof Refinement Workflow ❌

1. Launch the interactive refinement tool. It reads `/var/log/audit/audit.log`, finds the AppArmor denials your previous tests generated, and walks you through each one, asking how the profile should handle it:
    
    ```
    sudo aa-logprof
    ```
    
    **Expected result:** The tool starts with a banner similar to `Updating AppArmor profiles in /etc/apparmor.d.` and `Reading log entries from /var/log/audit/audit.log.`, then presents the first decision prompt.
    
    > **Note:** `aa-logprof` accepts **single-key responses** without pressing Enter, similar to a text-mode menu. Just press the highlighted letter (`I`, `A`, `D`, `S`, etc.) or the number `1` / `2`, and the tool processes your choice instantly. The bracketed default (`[(D)eny]` or `[1 - include ...]`) is what the tool picks if you press Enter alone.
    
2. The prompts `aa-logprof` shows are not the same every time. The tool replays whatever AppArmor events were recorded in the audit log, so the order and number of prompts can differ from one session to the next. The set of prompts it can display is fixed, though, so instead of following a strict sequence, compare each prompt against the rules below in order and stop at the first matching rule.
    
    > **Note:** Press the indicated key (single keypress, without pressing `Enter`), then repeat the process for the next prompt.


|**Rule**|**When the prompt shows...**|**Press**|
|---|---|---|
|**1**|Anything mentioning `/usr/local/bin/rogue-demo.sh` (the rogue script)|`D` (Deny)|
|**2**|A numbered include option such as `[1 - include <abstractions/consoles>]`, `<abstractions/opencl-pocl>`, or `<abstractions/apache2-common>`|The number (`1`), then `A`|
|**3**|An `Execute:` line for a helper such as `/usr/bin/whoami` or `/usr/bin/date`|`I` (Inherit)|
|**4**|A `Path:` line for any other system file or directory (for example, `/etc/ld.so.cache`, `/dev/tty`, or `/`)|`A` (Allow)|
|**5**|A `Network Family:` line (for example, `inet` / `stream`) with no numbered include option|`A` (Allow)|
output at the bottom

> **Note:** The order matters because some prompts match more than one rule. For example, a `/dev/tty` path may appear with a numbered `[1 - include <abstractions/consoles>]` option. In that case, rule 2 takes precedence over rule 4, so select the include (`1` then `A`) rather than allowing the bare path directly.
    > 
    > The only event you should deny is the rogue script (rule 1), which is listed first so it is never missed.
    
3. Work through every prompt using the rules above until the tool stops presenting events. Along the way, you may notice several normal behaviors: the same path can appear more than once (answer it the same way each time), the tool prints an `Adding ... to profile.` confirmation after each allow, and prompts are grouped under headings such as `Complain-mode changes:` and `Enforce-mode changes:`. Continue until you reach the save screen.
    
4. Once you've answered every prompt, the tool presents the save-changes screen. Press `S` to **Save Changes**.
    
    ```
    = Changed Local Profiles =
    
    The following local profiles were changed. Would you like to save them?
    
     [1 - /usr/sbin/nginx]
    (S)ave Changes / Save Selec(t)ed Profile / [(V)iew Changes] / View Changes b/w (C)lean profiles / Abo(r)t
    ```
    
    **Expected result:** The tool writes your refinements back to `/etc/apparmor.d/usr.sbin.nginx` with a confirmation line similar to `Writing updated profile for /usr/sbin/nginx.`, then exits and returns you to the shell prompt.
    
    > **Note:** `aa-logprof` is the standard workflow for refining AppArmor profiles from recorded audit events. Explicit `deny` rules you added manually are preserved during refinement, so the security boundary remains intact.
    
5. Confirm the original `deny /etc/** w,` rule is still present:
    
    ```
    sudo grep -c 'deny /etc/' /etc/apparmor.d/usr.sbin.nginx
    ```
    
    **Expected result:** `1`. The deny rule survived the refinement session intact, confirming that the profile's security boundary was preserved.
    
6. Run the rogue script one more time to confirm the profile still enforces:
    
    ```
    sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
    ```
    
    **Expected result:** A `Permission denied` message appears when the shell tries to read or execute the script, and no `/etc/rogue-marker` file is created. Mandatory access control still blocks the forbidden operation, confirming that the profile remains intact and `nginx` is still confined.
    

Outstanding work, you've crossed the finish line! You've successfully inspected the live AppArmor posture, switched a profile between complain and enforce modes, watched the same forbidden write be blocked at the kernel level in both modes, learned how to inspect the audit subsystem with `ausearch -m AVC` for the forensic trail, and practiced the `aa-logprof` refinement workflow without weakening the security boundary. These are the same techniques used to deploy and maintain real Ubuntu mandatory access control in production, whether you're onboarding a new service, investigating a security incident, or tightening a profile after a code change.

Congratulations, you have:

- Hardened SSH access and configured `fail2ban` by:
    
    - generating an ED25519 key pair for the `auditor` user,
    - writing a `70-hardening.conf` drop-in that disables root login, disables password authentication, and enforces an `AllowUsers` allow-list,
    - validating the configuration with `sshd -t` and `sshd -T`,
    - proving the policy with positive and negative login tests against `/var/log/auth.log`, and
    - configuring `fail2ban` with production-grade `maxretry`, `findtime`, and `bantime` values.
- Enabled the `ufw` firewall and configured audit logging by:
    
    - setting deny-by-default inbound rules,
    - allowing the dashboard port and rate-limiting SSH,
    - confirming the rules survive a `ufw reload`,
    - writing audit rules for `/etc/passwd`, `/etc/shadow`, `/var/log/auth.log`, and `/usr/bin/sudo`,
    - loading them with `augenrules --load`, and
    - validating the audit trail both live with `journalctl -f` and retrospectively with `ausearch`, `aureport`, `journalctl`, and the auth log.
- Managed AppArmor profiles by inspecting the live nginx profile with `aa-status`, switching between complain and enforce modes with `aa-complain` and `aa-enforce`, inspecting the audit subsystem with `ausearch -m AVC`, and practicing the `aa-logprof` refinement workflow.
    

You're now done with the lab. Confirm by clicking **End Lab**.


![[Pasted image 20260525130136.png]]
![[Pasted image 20260525130247.png]]

got expected results

```
ubuntu@ip-172-31-24-10:~$ sudo grep -c 'deny /etc/' /etc/apparmor.d/usr.sbin.nginx
1
ubuntu@ip-172-31-24-10:~$ sudo aa-exec -p /usr/sbin/nginx -- /usr/local/bin/rogue-demo.sh
/bin/bash: /usr/local/bin/rogue-demo.sh: Permission denied
ubuntu@ip-172-31-24-10:~$ 
```

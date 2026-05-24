---

tags:
  - hardening
  - Linux
  - security
linkedin: "False"
quartz: "False"
refactored: "False"
pluralsight: "True"
hands-on: "True"
hardlinked: "True"
---

**Lab Environment:** 
Ubuntu Server - `172.31.24.10` 
[[Nmap]] - `172.31.24.30` 
**Tools Used:** `nmap`, `vi`, `systemctl`, `apt`, `ufw`

---

## Overview

Security hardening is the process of reducing a system's attack surface by eliminating unnecessary services, enforcing access controls, and applying the principle of least privilege. In this lab, I walked through hardening a Linux Ubuntu server configured to serve as a web server. The goal was to ensure only the services and ports required for its primary function remain active and accessible.

The three core hardening steps covered are:

1. **Baseline the server** - scan open ports to understand the current exposure
2. **Restrict SSH root login** - prevent direct root access over SSH
3. **Remove unnecessary services** - disable and purge LDAP (`slapd`)
4. **Enable the firewall** - explicitly allow only required traffic

---

## Step 1 - Baseline Scan with Nmap

Before making any changes, it is important to take a snapshot of the server's current state. This gives you a reference point to compare against after hardening and helps identify what needs to be addressed.

From a separate machine (the NMAP Console at `172.31.24.30`), I ran a basic Nmap scan against the target server:

```bash
nmap 172.31.24.10
```

**Example Output:**

```
pslearner@ip-172-31-24-30:~$ nmap 172.31.24.10
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 00:02 UTC
Nmap scan report for ip-172-31-24-10.ec2.internal (172.31.24.10)
Host is up (0.00036s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
389/tcp open  ldap
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds
```

**Analysis:** Four ports are currently open:

| Port | Service | Expected?                                |
| ---- | ------- | ---------------------------------------- |
| 22   | SSH     | ✅ Yes - needed for remote administration |
| 80   | HTTP    | ✅ Yes - web server traffic               |
| 443  | HTTPS   | ✅ Yes - encrypted web traffic            |
| 389  | LDAP    | ❌ No - not required for a web server     |
|      |         |                                          |

Port 389 (LDAP) is the anomaly. LDAP is a directory service protocol - useful in authentication infrastructure, but entirely unnecessary on a standalone web server. Leaving it open increases the attack surface unnecessarily.

---

## Step 2 - Disable Direct Root SSH Login

Administrators commonly manage Linux servers via SSH. However, allowing SSH login directly to the `root` account is a significant security risk. If an attacker obtains the root password, they have full, unrestricted control over the system. The safer approach is to require admins to log in as a standard user and escalate privileges using `sudo` only when needed.

The SSH daemon's behavior is controlled by the `/etc/ssh/sshd_config` file. I used `grep` to confirm the current setting, `sed` to modify it in place, and then verified the change, avoiding the need to manually navigate the file in `vi`.

```bash
# Confirm the current setting
sudo grep -i permitrootlogin /etc/ssh/sshd_config

# Replace 'yes' with 'no' in-place
sudo sed -i 's|PermitRootLogin yes|PermitRootLogin no|g' /etc/ssh/sshd_config

# Verify the change was applied
sudo grep -i permitrootlogin /etc/ssh/sshd_config
```

**Example Output:**

```
pslearner@ip-172-31-24-10:~$ sudo grep -i permitrootlogin /etc/ssh/sshd_config
PermitRootLogin yes
# the setting of "PermitRootLogin without-password".

pslearner@ip-172-31-24-10:~$ sudo sed -i 's|PermitRootLogin yes|PermitRootLogin no|g' /etc/ssh/sshd_config

pslearner@ip-172-31-24-10:~$ sudo grep -i permitrootlogin /etc/ssh/sshd_config
PermitRootLogin no
# the setting of "PermitRootLogin without-password".
```


With the configuration file updated, the SSH service must be restarted to apply the changes:

```bash
sudo systemctl restart sshd
```

>**Why `sed` instead of manually editing with `vi`?** Using `sed -i` is faster and scriptable, the same command can be reused in automation playbooks across multiple servers. It's also more self-documenting: if you review shell history later, the `sed` command shows you exactly what was changed, whereas a `vi` entry only tells you the file was opened.

---

## Step 3 - Remove the LDAP Service (slapd)

Disabling a service stops it from running but does not remove it from the system. For services that have no role on this server, the best practice is to fully remove the package to eliminate any residual risk from misconfiguration or future unintended restarts.

### 3a - Identify the LDAP Service

I used `systemctl status` piped through `grep` to locate the **LDAP-related service** without scrolling through the entire service list:

```bash
sudo systemctl status | grep -B 1 -A 1 ldap
```

**Example Output:**

```
             ├─slapd.service
             │ └─2652 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d
             ├─systemd-resolved.service
```

The service responsible for LDAP is `slapd.service`.

### 3b - Disable and Stop the Service

```bash
sudo systemctl disable --now slapd.service
```

The `--now` flag both disables the service from starting at boot **and** stops it immediately. This is equivalent to running `systemctl stop` and `systemctl disable` separately.

**Example Output:**

```
pslearner@ip-172-31-24-10:~$ sudo systemctl disable --now slapd.service
slapd.service is not a native service, redirecting to systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable slapd
```

> The message `redirecting to systemd-sysv-install` is expected, `slapd` was installed as a [[SysV (System V Init)|SysV init]] service rather than a native systemd unit, so systemd delegates the disable action to the legacy compatibility layer.

### 3c - Inspect Package File Locations

Before removing the package, I checked what files and directories `slapd` had placed on the system:

```bash
dpkg -S slapd
```

This showed all file paths associated with the `slapd` package, configuration files, binaries, and library dependencies across the system.

> [!NOTE]- Output
> ```
> slapd: /usr/share/man/man5/slapd-mdb.5.gz
> slapd: /usr/share/man/man5/slapd-ldif.5.gz
> slapd: /usr/share/doc/slapd
> slapd: /usr/share/man/man5/slapd-sock.5.gz
> slapd: /usr/share/doc/slapd/examples/slapd.backup
> slapd: /usr/share/man/man5/slapd-monitor.5.gz
> slapd: /usr/share/man/man5/slapd-relay.5.gz
> slapd: /usr/share/man/man5/slapd-ldap.5.gz
> slapd: /usr/share/apport/package-hooks/slapd.py
> slapd: /usr/sbin/slapd
> slapd: /usr/share/man/man5/slapd-dnssrv.5.gz
> slapd: /usr/share/man/man5/slapd-passwd.5.gz
> slapd: /usr/share/man/man5/slapd.access.5.gz
> slapd: /usr/sbin/slapdn
> slapd: /usr/share/doc/slapd/examples
> slapd: /usr/share/doc/slapd/changelog.Debian.gz
> slapd: /usr/share/lintian/overrides/slapd
> slapd: /usr/share/doc/slapd/examples/slapd.conf
> slapd: /usr/share/man/man5/slapd-asyncmeta.5.gz
> slapd: /lib/systemd/system/slapd.service.d
> slapd: /usr/share/man/man5/slapd-sql.5.gz
> slapd: /etc/apparmor.d/usr.sbin.slapd
> slapd: /usr/share/man/man5/slapd.conf.5.gz
> slapd: /usr/share/doc/slapd/NEWS.Debian.gz
> slapd: /usr/share/doc/slapd/TODO.Debian
> slapd: /usr/share/slapd/ldiftopasswd
> slapd: /etc/default/slapd
> slapd: /usr/share/man/man5/slapd-ndb.5.gz
> slapd: /usr/share/doc/slapd/README.Debian.gz
> slapd: /lib/systemd/system/slapd.service.d/slapd-remain-after-exit.conf
> slapd: /usr/share/man/man5/slapd-meta.5.gz
> slapd: /etc/init.d/slapd
> slapd: /usr/share/man/man5/slapd.backends.5.gz
> slapd: /usr/share/slapd/slapd.init.ldif
> slapd: /usr/share/man/man5/slapd.overlays.5.gz
> slapd: /usr/share/man/man5/slapd-null.5.gz
> slapd: /usr/share/man/man5/slapd-config.5.gz
> slapd: /usr/share/man/man8/slapd.8.gz
> slapd: /usr/share/doc/slapd/copyright
> slapd: /usr/share/slapd
> slapd: /etc/ufw/applications.d/slapd
> slapd: /usr/share/man/man5/slapd.plugin.5.gz
> slapd: /usr/share/man/man5/slapd-perl.5.gz
> slapd: /usr/share/man/man8/slapdn.8.gz
> ```

### 3d - Purge the Package

```bash
sudo apt purge slapd
```

`apt purge` removes both the package binaries **and** its configuration files, unlike `apt remove` which leaves config files behind. I confirmed with `Y` when prompted.

### 3e - Verify Removal

```bash
dpkg -S slapd
```

**Example Output:**

```
dpkg-query: no path found matching pattern *slapd*
```

This confirms that all remnants of the `slapd` package have been removed from the system.

---

## Step 4 - Enable the Firewall (UFW)

Even with unnecessary services removed, a host-based firewall adds a critical layer of defense by explicitly defining what traffic is allowed in and out. Ubuntu's [[UFW (Uncomplicated Firewall)|Uncomplicated Firewall (ufw)]] makes this straightforward.

First, I checked the current firewall status:

```bash
sudo ufw status
```

```
Status: inactive
```

The firewall was not running. I then added rules to allow only the traffic this server legitimately needs:

```bash
# Allow Apache (covers both HTTP port 80 and HTTPS port 443)
sudo ufw allow "Apache Full"

# Allow SSH (so remote administration remains possible)
sudo ufw allow OpenSSH

# Enable the firewall
sudo ufw enable

# Verify the active rules
sudo ufw status
```

**Example Output:**

```
pslearner@ip-172-31-24-10:~$ sudo ufw allow "Apache Full"
Rules updated
Rules updated (v6)

pslearner@ip-172-31-24-10:~$ sudo ufw allow OpenSSH
Rules updated
Rules updated (v6)

pslearner@ip-172-31-24-10:~$ sudo ufw enable
Firewall is active and enabled on system startup

pslearner@ip-172-31-24-10:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
Apache Full                ALLOW       Anywhere
OpenSSH                    ALLOW       Anywhere
Apache Full (v6)           ALLOW       Anywhere (v6)
OpenSSH (v6)               ALLOW       Anywhere (v6)
```


> **Why `Apache Full` instead of just port 80 or 443?** UFW has named application profiles (stored in `/etc/ufw/applications.d/`) that group related rules. `Apache Full` maps to both ports 80 and 443, and `OpenSSH` maps to port 22. Using named profiles is more readable and easier to audit than raw port numbers.

^NamedApplicationProfiles

The rules apply to both IPv4 and IPv6, ensuring the firewall is consistent across both stacks.

---

## Step 5 - Verify with a Final Nmap Scan

Back on the NMAP Console, I re-ran the scan to confirm that the hardening changes are reflected externally:

```bash
nmap 172.31.24.10
```

**Example Output:**

```
pslearner@ip-172-31-24-30:~$ nmap 172.31.24.10
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 01:22 UTC
Nmap scan report for ip-172-31-24-10.ec2.internal (172.31.24.10)
Host is up (0.00059s latency).
Not shown: 997 filtered ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 4.55 seconds
```

Port 389 (LDAP) is gone. Only the three ports required for this server's function remain open. Notice also that the scan now shows `997 filtered ports` instead of `996 closed ports`, this is the UFW firewall actively filtering traffic rather than the OS simply rejecting connections to closed ports.

---

## Summary

|Hardening Action|Command(s) Used|Outcome|
|---|---|---|
|Baseline scan|`nmap 172.31.24.10`|Identified 4 open ports, 1 unnecessary|
|Disable root SSH|`sed -i` on `sshd_config` + `systemctl restart sshd`|Root login over SSH blocked|
|Disable LDAP service|`systemctl disable --now slapd.service`|Service stopped and disabled|
|Remove LDAP package|`apt purge slapd`|Package and config files fully removed|
|Enable firewall|`ufw allow` + `ufw enable`|Only SSH, HTTP, HTTPS permitted|
|Verify hardening|`nmap 172.31.24.10`|Confirmed port 389 closed, firewall active|

Hardening is not a one-time task, it is an ongoing process. This lab covered foundational steps: removing unnecessary exposure, enforcing least privilege on administrative access, and implementing host-based firewall rules. In a production environment, additional steps would include disabling port 80 in favor of HTTPS-only, configuring fail2ban to block brute-force attempts, and auditing user accounts regularly.


<!--

#### DRAFT#####
## OBJECTIVE 1

### Hardening Linux Systems

This challenge will help you understand how to harden a Linux Ubuntu Server (the Linux Console) with the IP address of 172.31.24.10 to function as a Web server. This means that the server should have the appropriate accounts, services, roles, and applications to perform its primary function as a Web server. Throughout this lab, you will ensure anyone who needs to SSH to the server cannot do so directly to the root account, and disable unnecessary services and applications that aren't needed for the server to function properly. Before you begin the configuration of the Linux Server, it's a good idea to get a baseline of the server in its current state.

To do this, you'll use the NMAP to scan the Linux Console to verify which ports and protocols are currently open.

1. Once the environment is started, click **Open environment** and connect to the **NMAP Console** machine to get started.

2. Enter the command `nmap 172.31.24.10` to see which ports are open on the **Linux Console**.![](https://labs.pluralsight.com/labkeep/lab_assets/850e456c-f162-48d4-8f03-0abfed2f8f68)**Analysis:** You can see the open ports on the Linux Console are 22 (ssh), 80 (http), 389 (ldap), and 443 (https). For any Linux Console, it is common practice for administrators to SSH into the server to manage it, so port 22 is required. To function as a web server, port 80 and port 443 are also expected to be open, but port 389 is not required. In a production environment, you may also want to disable port 80, since the traffic is unencrypted, but that is outside the scope of this lab.
    
    Now that you have seen what ports are open on this server, you need to start hardening it!

```
pslearner@ip-172-31-24-30:~$ nmap 172.31.24.10
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 00:02 UTC
Nmap scan report for ip-172-31-24-10.ec2.internal (172.31.24.10)
Host is up (0.00036s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
389/tcp open  ldap
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds
pslearner@ip-172-31-24-30:~$
```


2. Navigate to the **Linux Console** by pressing **CTRL + ALT + SHIFT** (**CTRL + CMD + SHIFT** on Mac), clicking **NMAP Console** in the top left, then selecting **Linux Server**.
    
    Managing Linux Servers is commonly accomplished by administrators connecting to the server via **SSH**. Administrators will routinely need elevated privileges to complete tasks to manage and configure the server. which is done via the `sudo` command. It is a security risk to allow administrators to log directly into the **Root** user, rather than entering `sudo` every time a command needs to be ran with elevated privileges. The first step is to configure the **Linux Console** to not allow administrators to **SSH** directly into the **Root** account.
    
3. Enter the command `sudo vi /etc/ssh/sshd_config` to open the **sshd_config** file.
    
    **Note:** The **sshd_config** file is used to configure **SSH** settings on the **Linux Console**. Settings such as which IP addresses will be listening for **SSH** connections, how users will be able to authenticate, and which algorithms will be used to encrypt the **SSH** traffic.![](https://labs.pluralsight.com/labkeep/lab_assets/9d61ba35-fbf1-4739-8ccc-48b19be4425f)**Analysis**: About halfway down the first screen, you can see that **PermitRootLogin** is configured for **yes**. This would allow administrators to **SSH** directly into the **Linux Console** via the **Root** account, assuming they knew the password.

[[sed command#search replace]]
```
sudo grep -i permitrootlogin /etc/ssh/sshd_config
sudo sed -i 's|PermitRootLogin yes|PermitRootLogin no|g' /etc/ssh/sshd_config
sudo grep -i permitrootlogin /etc/ssh/sshd_config
```

```
pslearner@ip-172-31-24-10:~$ sudo grep -i permitrootlogin /etc/ssh/sshd_config
PermitRootLogin yes
# the setting of "PermitRootLogin without-password".
pslearner@ip-172-31-24-10:~$ sudo sed -i 's|PermitRootLogin yes|PermitRootLogin no|g' /etc/ssh/sshd_config
pslearner@ip-172-31-24-10:~$ sudo grep -i permitrootlogin /etc/ssh/sshd_config
PermitRootLogin no
# the setting of "PermitRootLogin without-password".
pslearner@ip-172-31-24-10:~$
```

2. Press the **down arrow** key until you arrive on the **PermitRootLogin** line, and press **i** to enter **Insert Mode** to manipulate text.
    
3. Change the argument from `yes` to `no`.
    
4. Press **Ctrl + C** to escape **Insert Mode**, and then enter `:wq` to save and quit the **sshd_config** file.
    
5. Enter `sudo systemctl restart sshd` to restart the **SSH** service with the updated changes.
    
    Now that administrators cannot **SSH** to the **Linux Console** using the **Root** account, the next step is to remove unnecessary services. For this example, you will remove the service that is providing **LDAP** capabilities.
    
6. Enter the command `sudo systemctl status` to view all of the services that are running on the **Linux Console**.
    
    **Note:** There will be many services that are running a server, it is up to you as the administrator to determine which services are necessary for the server to perform its intended function. Use the **down arrow** and **up arrow** keys to see all of services running.![](https://labs.pluralsight.com/labkeep/lab_assets/d725eeaa-3ade-4e1b-bf94-03e86c8fa605)**Analysis**: Towards the bottom of the running services, you can see that **slapd.service** is responsible for **LDAP** communication via the references to **LDAP**.

[[grep#display lines below]]

2. Enter **CTRL + C** to escape back to the **CLI**.
    
3. Enter the command `sudo systemctl disable --now slapd.service` to disable the **slapd.service**.
    
    While the service is now disabled, if it isn't necessary for the function of the server, it's best practice to remove it completely.

```
pslearner@ip-172-31-24-10:~$ sudo systemctl status | grep -B 1 -A 1 ldap
           │   │ ├─3571 sudo systemctl status
           │   │ ├─3572 grep --color=auto -B 1 -A 1 ldap
           │   │ ├─3573 sudo systemctl status
--
             ├─slapd.service 
             │ └─2652 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d
             ├─systemd-resolved.service 
             
pslearner@ip-172-31-24-10:~$ sudo systemctl disable --now slapdservice
Failed to disable unit: Unit file slapdservice.service does not exist.
pslearner@ip-172-31-24-10:~$ sudo systemctl disable --now slapd.service
slapd.service is not a native service, redirecting to systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable slapd
pslearner@ip-172-31-24-10:~$ 
```


2. Enter the command `dpkg -S slapd` to see the file locations that reference **slapd**.
    
    **Analysis:** These are all of the files and directories that were manipulated in order for the **LDAP** functionality to work correctly. As you can see, there are a lot of locations affected when **slapd** package was installed and configured.
    
3. Enter the command `sudo apt purge slapd` to uninstall the **slapd** package and its dependencies. Enter `Y` when prompted to continue.
    
4. Once you see confirmation that the **slapd** configuration was removed, enter `dpkg -S slapd` again to confirm that all remnants have been removed.![](https://labs.pluralsight.com/labkeep/lab_assets/e4ceaf2d-9a2b-48b6-8294-a51bf1923a7f)**Analysis:** From the dpkg-query, you receive the message that no path was found matching the pattern **slapd**.
    
    Once you have removed all of the services that are not needed for the server to function, the last thing you need to do is explicitly allow the required services in the server's firewall, and enable the firewall.





5. Enter `sudo ufw status` to view the current status of the firewall.
    
    **Analysis:** The status is showing that the firewall is inactive.
    
6. Enter the command `sudo ufw allow "Apache Full"` to explicitly allow the entire **Apache** application, which is used for both http and https traffic.
    
7. Enter the command `sudo ufw allow OpenSSH` to allow the **OpenSSH** application, which is used for ssh traffic.
    
8. Enter the command `sudo ufw enable` to enable the firewall.
    
9. Enter the command `sudo ufw status` to verify the firewall is enabled.![](https://labs.pluralsight.com/labkeep/lab_assets/4ffe806e-f6b1-4987-a2c2-5685bcc97036)**Analysis** The status is showing as active, and there are **ALLOW** rules for the **Apache** and **OpenSSH** applications on both IPv4 and IPv6.

```
pslearner@ip-172-31-24-10:~$ sudo ufw status
Status: inactive
pslearner@ip-172-31-24-10:~$ sudo ufw allow "Apache Full"
Rules updated
Rules updated (v6)
pslearner@ip-172-31-24-10:~$ sudo ufw allow OpenSSH
Rules updated
Rules updated (v6)
pslearner@ip-172-31-24-10:~$ sudo ufw enable
Firewall is active and enabled on system startup
pslearner@ip-172-31-24-10:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
Apache Full                ALLOW       Anywhere                  
OpenSSH                    ALLOW       Anywhere                  
Apache Full (v6)           ALLOW       Anywhere (v6)             
OpenSSH (v6)               ALLOW       Anywhere (v6)             

pslearner@ip-172-31-24-10:~$
```



7. Connect to the **NMAP Console** to verify that the hardening techniques are visible.
    
8. Enter `nmap 172.31.24.10` to verify that the **Linux Console** is no longer listening for **LDAP** connections.![](https://labs.pluralsight.com/labkeep/lab_assets/4bf889eb-eddd-4e01-a50a-f1a8b6c1b73c)**Analysis:** Port 389 (ldap) is no longer open and listening for connections.

```
pslearner@ip-172-31-24-30:~$ nmap 172.31.24.10
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 01:22 UTC
Nmap scan report for ip-172-31-24-10.ec2.internal (172.31.24.10)
Host is up (0.00059s latency).
Not shown: 997 filtered ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 4.55 seconds
pslearner@ip-172-31-24-30:~$ 
```

Congratulations! You have just finished this challenge. You now have a solid understanding of how to harden a Linux Console.


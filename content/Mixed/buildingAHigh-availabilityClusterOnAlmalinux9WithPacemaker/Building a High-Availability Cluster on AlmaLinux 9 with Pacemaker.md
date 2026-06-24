---
tags:
  - pacemaker
  - AlmaLinux
  - corosync
  - Linux
  - high-availability
  - apache
linkedin: "False"
quartz: "True"
refactored: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
## Overview

High availability (HA) is the practice of designing systems that remain operational even when individual components fail. In this lab, I built a two-node HA cluster using **Pacemaker** and **Corosync** on AlmaLinux 9. The cluster manages a [[Virtual IP (VIP)]] and an Apache web server as **clustered resources**, automatically migrating them to a healthy node if the active one goes down, all without manual intervention. 

This is a foundational pattern used in production Linux environments to eliminate single points of failure for services like web servers, databases, and NFS mounts.

---

## Lab Environment

| Component        | Details                                                  |
| ---------------- | -------------------------------------------------------- |
| OS               | AlmaLinux 9.8 (Olive Jaguar)                             |
| Hypervisor       | VirtualBox                                               |
| Nodes            | 2x VMs - `server1` (10.0.0.4) and `server2` (10.0.0.149) |
| Resources per VM | 1 vCPU, 1 GB RAM, 20 GB disk                             |
| Network adapter  | Bridge                                                   |
| Cluster VIP      | `10.0.0.50` (floats between nodes)                       |

---

## Phase 1 - OS Setup and Hostname Configuration

Both nodes need to know each other by hostname so that cluster communication tools like `pcs` and `corosync` can resolve node names without relying on DNS. I set the hostname on each VM and added static entries to `/etc/hosts`.

```bash
# On server1
hostnamectl set-hostname server1.example.com

# On server2
hostnamectl set-hostname server2.example.com
```
<!--
**`hostnamectl set-hostname`** sets the machine's own identity — it's what the system calls _itself_. This is what shows up in your shell prompt (`[root@server1 ~]#`), in `pcs status` output under "Node List", in logs, and what the cluster uses to identify each member. It writes to `/etc/hostname`.
-->

Then on **each server**, map both hostnames to their IPs:

```bash
echo '10.0.0.4   server1.example.com server1' >> /etc/hosts
echo '10.0.0.149 server2.example.com server2' >> /etc/hosts
```

> **Why set the hostname?** when you run `pcs cluster setup peanut server1.example.com server2.example.com`, `pcs` uses the hostname to identify which node it's currently on, then looks up the other node in `/etc/hosts` to make the network connection. If either is missing, cluster setup fails or nodes appear with the wrong identity.

Verify your interface IPs:

```bash
ip addr show
```

---
%%
### SSH Setup

Remote access is needed to manage both nodes. I also set up key-based authentication between the two nodes, which cluster tools rely on for passwordless communication.

#### Enable SSH on both servers

SSH ships with AlmaLinux 9 by default. Enable it and open the firewall:

```bash
systemctl enable --now sshd
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

#### Set Up Passwordless SSH Between Nodes

Run this on **server1** first:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
ssh-copy-id root@server2.example.com
ssh root@server2.example.com   # should connect without a password prompt
```

Then repeat from **server2**:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
ssh-copy-id root@server1.example.com
ssh root@server1.example.com
```

#### (Optional) Passwordless SSH from Your Host Machine

```bash
ssh-keygen -t ed25519
ssh-copy-id root@10.0.0.4
ssh-copy-id root@10.0.0.149
```

---

### Troubleshooting - SSH Connection Timed Out

**Symptom:** Attempting to SSH from the host using the Host-Only adapter IP failed immediately:

```
ssh root@192.168.56.101
ssh: connect to host 192.168.56.101 port 22: Connection timed out
```

**Root cause:** The Host-Only adapter was not reachable from the host in this environment. I switched each VM to a **Bridge adapter** instead, which placed both VMs on the same physical network as the host and gave them real routable IPs (`10.0.0.4` and `10.0.0.149`).

**Additional fix - sshd_config:** Even after switching adapters, SSH still rejected root login. I had to edit the SSH daemon config to allow it:

```bash
sudo vi /etc/ssh/sshd_config
# (or grep the /etc/ssh directory if your distro splits config across multiple files)
```

Uncomment or add these lines:

```
PasswordAuthentication yes
PermitRootLogin yes
```

Then restart SSH:

```bash
systemctl restart sshd
```

> **Note:** `PermitRootLogin yes` and `PasswordAuthentication yes` are appropriate for a local lab environment. In production, use key-based authentication only and disable root login over SSH.

%%

---

## Phase 2 - Install Cluster Packages

Pacemaker, the Cluster Information Base (CIB) manager, and the `pcs` management CLI all live in the **High Availability** repository, which is not enabled by default on AlmaLinux 9. The following installs everything needed on **both servers**.

```bash
dnf config-manager --enable highavailability
dnf install -y pacemaker pcs resource-agents fence-agents-all
```

After installation, set the `hacluster` service account password (must be identical on both nodes, `pcs` uses it to authenticate between them) and start the `pcsd` daemon:

```bash
echo 'hacluster:Password1' | chpasswd
systemctl enable --now pcsd
```

**Example output on server2:**

```
[root@server2 ~]# echo 'hacluster:Password1' | chpasswd
[root@server2 ~]# systemctl enable --now pcsd
Created symlink /etc/systemd/system/multi-user.target.wants/pcsd.service → /usr/lib/systemd/system/pcsd.service.
```

---

### Troubleshooting - Packages Not Found

**Symptom:** Running `dnf install` before enabling the repo returned "No match for argument":

```
[root@server1 ~]# dnf install -y pacemaker pcs resource-agents fence-agents-all
...
No match for argument: pacemaker
No match for argument: pcs
No match for argument: resource-agents
Error: Unable to find a match: pacemaker pcs resource-agents
```

**✅ Fix:** The High Availability repository must be explicitly enabled first:

```bash
dnf config-manager --enable highavailability
```

Re-running the `dnf install` command afterward succeeds.

---

## Phase 3 - Firewall Configuration

Corosync and Pacemaker communicate over specific ports. Rather than opening individual ports manually, AlmaLinux 9 provides a `high-availability` firewall service definition that covers them all. Run this on **both servers**:

```bash
firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
```

Verify the service is listed:

```bash
firewall-cmd --list-services
```

**Example output:**

```
[root@server1 ~]# firewall-cmd --list-services
cockpit dhcpv6-client high-availability ssh
```

---

## Phase 4 - Create the Cluster

With both nodes prepped, I initialized the cluster from **server1 only**. The `pcs host auth` command establishes trust between the two nodes (both directions) using the `hacluster` credentials set earlier.

```bash
pcs host auth server1.example.com server2.example.com \
  -u hacluster -p Password1
```

>When you run `pcs host auth`, pcsd on server1 contacts pcsd on server2 and exchanges a **token**, essentially a shared secret stored on disk. From that point on, when server1 sends a cluster management command (like "start this resource" or "here's an updated config"), server2's pcsd recognizes the token and accepts it without asking for the `hacluster` password again.

Create the cluster (named `peanut` here, the name is arbitrary):

```bash
pcs cluster setup peanut \
  server1.example.com server2.example.com
```

Start and enable the cluster so it survives reboots:

```bash
pcs cluster start --all
pcs cluster enable --all
```

**Example output:**

```
[root@server1 ~]# pcs host auth server1.example.com server2.example.com \
  -u hacluster -p Password1
server1.example.com: Authorized
server2.example.com: Authorized

[root@server1 ~]# pcs cluster setup peanut \
  server1.example.com server2.example.com
...
Sending 'corosync authkey', 'pacemaker authkey' to 'server1.example.com', 'server2.example.com'
server1.example.com: successful distribution of the file 'corosync authkey'
server2.example.com: successful distribution of the file 'corosync authkey'
Cluster has been successfully set up.

[root@server1 ~]# pcs cluster start --all
server2.example.com: Starting Cluster...
server1.example.com: Starting Cluster...

[root@server1 ~]# pcs cluster enable --all
server1.example.com: Cluster Enabled
server2.example.com: Cluster Enabled
```

---

### Troubleshooting - [[STONITH (Shoot The Other Node in the Head)|STONITH]] / CIB Validation Errors After First `pcs status`

**Symptom:** Running `pcs status` immediately after cluster creation showed STONITH warnings and marked both nodes as UNCLEAN:

```
WARNINGS:
No stonith devices and stonith-enabled is not false
error: Resource start-up disabled since no STONITH resources have been defined
error: CIB did not pass schema validation

Node List:
  * Node server1.example.com: UNCLEAN (offline)
  * Node server2.example.com: UNCLEAN (offline)
```

**Root cause:** Pacemaker requires STONITH (Shoot The Other Node In The Head) fencing by default. Fencing is a mechanism that forcibly powers off or isolates a misbehaving node to prevent data corruption in shared-storage scenarios. Since this is a lab with no shared storage and no fencing hardware, STONITH is disabled in the next phase.

---

## Phase 5 - Disable [[STONITH (Shoot The Other Node in the Head)|STONITH]] and [[Quorum]] Policy

> [!attention]
> **Lab only.** Never disable STONITH in a production cluster that uses shared storage, it exists to prevent split-brain data corruption.

```bash
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
```

After applying these settings, `pcs status` shows both nodes online and clean:

```
[root@server1 ~]# pcs status
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * 2 nodes configured
  * 0 resource instances configured

Node List:
  * Online: [ server1.example.com server2.example.com ]

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

---

## Phase 6 - Create a Clustered Virtual IP Resource

The Virtual IP (VIP) is the address clients connect to. Pacemaker assigns this IP to whichever node is currently active. When that node fails, Pacemaker moves the IP to the other node, transparent to the client.
<!--
transparent it means something is **hidden or invisible** in the sense that it happens without you having to think about it or do anything about it.
-->
Run this on **server1 only**:

```bash
pcs resource create cluster_ip \
  ocf:heartbeat:IPaddr2 \
  ip=10.0.0.50 cidr_netmask=24 \
  op monitor interval=20s
```

- `ocf:heartbeat:IPaddr2` - the Open Cluster Framework resource agent that manages IP address assignment
- `op monitor interval=20s` - Pacemaker checks every 20 seconds that the IP is still assigned

Verify the resource started and that `10.0.0.50` appears as a secondary address on the active node:

```bash
pcs status
ip address show
```

**Example output - IP assigned as secondary on server1's interface:**

```
Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2): Started server1.example.com

2: enp0s9: ...
    inet 10.0.0.4/24 ...
    inet 10.0.0.50/24 brd 10.0.0.255 scope global secondary enp0s9
```

The `secondary` label confirms Pacemaker added the VIP alongside the node's primary address.

---

## Phase 7 - Test IP Failover

To verify the VIP migrates correctly, I started a continuous ping to `10.0.0.50` from server2 and then forced server1 into standby to simulate a failure.

From **server2**, ping the cluster IP:

```bash
ping 10.0.0.50
```

```
[root@server2 ~]# ping 10.0.0.50
64 bytes from 10.0.0.50: icmp_seq=1 ttl=64 time=1.66 ms
64 bytes from 10.0.0.50: icmp_seq=2 ttl=64 time=1.19 ms
...
7 packets transmitted, 7 received, 0% packet loss
```

From **server1**, simulate a failure by putting it in standby:

```bash
pcs node standby server1.example.com
```

Check that the VIP migrated to server2:

```bash
pcs status
```

```
Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2): Started server2.example.com
```

Bring server1 back online:

```bash
pcs node unstandby server1.example.com
```

---

### Troubleshooting - Wrong Standby Command Syntax

**Symptom:** Using `pcs cluster standby` returned a usage error:

```
❌ pcs cluster standby server1.example.com

Usage: pcs cluster ...
    setup <cluster name> (<node name> [addr=<node address>]...)...
```

**Fix:** The correct subcommand is `pcs node standby`, not `pcs cluster standby`:

```bash
✅ pcs node standby server1.example.com
✅ pcs node unstandby server1.example.com
```

---

## Phase 8 - Install and Configure Apache

With IP failover working, I added Apache as a second **cluster-managed resource**. The key principle here is that Pacemaker, not systemd, **controls the service lifecycle**. If systemd manages Apache independently, it could restart the service on the wrong node and conflict with cluster logic.
<!--
Pacemaker needs to be able to start, stop, and restart Apache on demand. If systemd is also set to restart Apache automatically, you get a conflict — Pacemaker tries to stop it as part of a failover and systemd immediately brings it back up. That causes unpredictable behavior and can break the cluster logic entirely.
-->
Install Apache on **both servers**, then immediately disable it from systemd:

```bash
dnf install -y httpd
systemctl stop httpd
systemctl disable httpd
```
<!--
`systemctl disable httpd` removes the **autostart symlinks** for the Apache service — it stops Apache from automatically starting on boot.
-->
Open HTTP through the firewall on **both servers**:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

Create a `/server-status` endpoint, which Pacemaker's Apache resource agent uses to health-check the service:

```bash
cat > /etc/httpd/conf.d/status.conf << 'EOF'
<Location /server-status>
  SetHandler server-status
  Require ip 127.0.0.1
</Location>
EOF
```

Create a unique `index.html` on each server so you can tell which node is actively serving requests:

```bash
# On server1
echo '<h1>Welcome</h1><hr>Server1' > /var/www/html/index.html

# On server2
echo '<h1>Welcome</h1><hr>Server2' > /var/www/html/index.html
```

Validate the Apache config on both servers:

```bash
apachectl configtest
# Expected: Syntax OK
```

---

## Phase 9 - Create the Apache Cluster Resource and Co-location Constraint

Now I register Apache with Pacemaker so the cluster owns it, and then add a co-location constraint so Apache always runs on the same node as the VIP. Without this constraint, Pacemaker might start Apache on server1 while the VIP is on server2, requests would reach the wrong node.<!--
so we ahre using the VIP as a guide to where start the apache
-->

On **server1 only**, create the Apache resource:

```bash
pcs resource create web-server \
  ocf:heartbeat:apache \
  configfile=/etc/httpd/conf/httpd.conf \
  statusurl='http://127.0.0.1/server-status' \
  op monitor interval=20s
```

At this point, the two resources may land on different nodes. Add a co-location constraint to fix them together:

```bash
pcs constraint colocation add web-server with cluster_ip INFINITY
```

The `INFINITY` score means the constraint is mandatory, Pacemaker will never put these two resources on separate nodes.

Confirm both resources share the same node:

```bash
pcs status
```

**Example output - both resources on server2:**

```
Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2): Started server2.example.com
  * web-server  (ocf:heartbeat:apache):  Started server2.example.com
```

---

### Troubleshooting - Wrong Co-location Syntax

**Symptom:** The initial co-location command returned a usage error instead of applying the constraint:

```
❌ pcs constraint colocation add web-server cluster_ip INFINITY

Usage: pcs constraint [constraints]...
    colocation add [<role>] <source resource id> with [<role>]
                   <target resource id> [score] ...
```

**Fix:** The `with` keyword between the two resource names is required:

```bash
✅ pcs constraint colocation add web-server with cluster_ip INFINITY
```

---

## Phase 10 - Test Apache Failover

With both resources co-located and the cluster healthy, I verified end-to-end HTTP failover. The cluster VIP is the single address clients always use, the underlying node can change transparently.

Curl the cluster IP to confirm which server is currently active:

```bash
curl http://10.0.0.50
```

```
<h1>Welcome</h1><hr>Server2
```

![[Pasted image 20260530182627.png]]

Force a failover by putting server2 into standby:

```bash
pcs node standby server2.example.com
```

Curl again, the VIP and Apache have migrated to server1:

```bash
curl http://10.0.0.50
```

```
<h1>Welcome</h1><hr>Server1
```

Bring server2 back and verify the cluster is fully healthy:

```bash
pcs node unstandby server2.example.com
pcs status
```

**Example output - both nodes online, resources on server1:**

```
[root@server2 ~]# pcs status
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * Last updated: Sat May 30 18:37:27 2026 on server2.example.com
  * 2 nodes configured
  * 2 resource instances configured

Node List:
  * Online: [ server1.example.com server2.example.com ]

Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2): Started server1.example.com
  * web-server  (ocf:heartbeat:apache):  Started server1.example.com

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

Both resources remained on server1, where they migrated when server2 went into standby, and the cluster reports no errors. This is expected behavior. Pacemaker uses resource stickiness to avoid unnecessary migrations; once a resource is running cleanly on a node, it stays there even after the previously failed node rejoins. The cluster is healthy and failover is working end to end.

---

## Key Takeaways

- **Pacemaker + Corosync** is the standard Linux HA stack on RHEL-family distros. Pacemaker manages resources; Corosync handles the low-level cluster communication and quorum.
- **Resources** are anything Pacemaker manages - IPs, services, filesystems. The `ocf:heartbeat:*` agents are the standard portable resource types.
- **Constraints** are how you express topology rules. Co-location (`with INFINITY`) means "always together." Order constraints (`pcs constraint order`) control startup sequence.
- **STONITH** exists for a real reason, in production with shared storage, fencing prevents a failed node from corrupting data. Never disable it in a real environment.
- **Disabling systemd for cluster-managed services** is critical. If both Pacemaker and systemd try to manage the same service, you get conflicts and split behavior.

---

## Quick Reference - Common `pcs` Commands

|Action|Command|
|---|---|
|Check cluster status|`pcs status`|
|Put a node in standby|`pcs node standby <node>`|
|Bring a node back|`pcs node unstandby <node>`|
|List resources|`pcs resource status`|
|List constraints|`pcs constraint show`|
|Show cluster config|`pcs config show`|
|Disable a resource|`pcs resource disable <name>`|
|Enable a resource|`pcs resource enable <name>`|

<!--
# HA Cluster Lab — AlmaLinux 9 with Pacemaker

## Lab Environment
- 2x AlmaLinux 9 VMs in VirtualBox
- Each VM: 1 vCPU, 1 GB RAM, 20 GB disk
- Adapter 1: NAT (internet)
- Adapter 2: Host-Only (e.g. 192.168.56.0/24)
- Cluster VIP: 10.0.0.50 (must be free on your subnet)

---
## Phase 1 — VirtualBox & OS Setup (both servers)

Set hostnames:
```bash
# On server1
hostnamectl set-hostname server1.example.com

# On server2
hostnamectl set-hostname server2.example.com
```

Add both nodes to /etc/hosts on EACH server (replace IPs with your actual Host-Only IPs):
```bash
echo '10.0.0.4 server1.example.com server1' >> /etc/hosts
echo '10.0.0.149 server2.example.com server2' >> /etc/hosts
```

Verify your Host-Only IPs:
```bash
ip addr show
```

ended up using bridge adapter
10.0.0.4 server1
10.0.0.149 server2
### SSH setup for both VMs

#### Step 1 — Enable SSH on both servers

SSH is usually installed on AlmaLinux 9 by default, just make sure it's running:
```bash
systemctl enable --now sshd
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

---

#### Step 2 — Find each VM's IP address

On each VM, run:
```bash
ip addr show
```

You're looking for the **Host-Only adapter** IP (usually `enp0s8` or similar, in the `192.168.56.x` range). Note both IPs — you'll need them to connect.

---

#### Step 3 — SSH from your host machine into the VMs

From your **Windows/Mac/Linux host** terminal:
```bash
ssh root@192.168.56.101   # connect to server1
ssh root@192.168.56.102   # connect to server2
```

> If you get a connection refused, double-check the Host-Only adapter IP with `ip addr show` inside the VM.

---

got error connecting from host
ssh root@192.168.56.101
ssh: connect to host 192.168.56.101 port 22: Connection timed out

added bridge adapter to each vm to see if i can connect with that IP
disabled all adapter except bridge

`sudo vi /etc/ssh/sshd_config` *might need to do grep in /ssh for the keywords you need in case of mutiple config files*
had to uncomment PasswordAuthetication yes
and PermitRootLogin yes

10.0.0.4 server1
10.0.0.149 server2

---


did not setup passwordless
#### Step 4 — Set up passwordless SSH between server1 and server2

This is needed so cluster nodes can communicate without prompting. Run this **on server1**:
```bash
# Generate an SSH key pair (press Enter 3 times to accept defaults)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# Copy the public key to server2
ssh-copy-id root@server2.example.com
```

Test it — should connect without a password:
```bash
ssh root@server2.example.com
```

Repeat in the other direction **on server2**:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
ssh-copy-id root@server1.example.com
ssh root@server1.example.com
```

---

#### Step 5 — Optional: SSH from your host without a password

On your **host machine**, generate a key if you don't have one:
```bash
ssh-keygen -t ed25519
```

Then copy it to both VMs:
```bash
ssh-copy-id root@192.168.56.101
ssh-copy-id root@192.168.56.102
```

---

## Phase 2 — Install Cluster Packages (both servers)

```bash
dnf config-manager --enable highavailability
dnf install -y pacemaker pcs resource-agents fence-agents-all
```

```
❌[root@server1 ~]# dnf install -y pacemaker pcs resource-agents fence-agents-all
AlmaLinux 9 - AppStream                                                                                             3.0 MB/s |  10 MB     00:03
AlmaLinux 9 - BaseOS                                                                                                1.8 MB/s | 2.8 MB     00:01
AlmaLinux 9 - Extras                                                                                                 14 kB/s |  22 kB     00:01
No match for argument: pacemaker
No match for argument: pcs
No match for argument: resource-agents
Error: Unable to find a match: pacemaker pcs resource-agents
[root@server1 ~]#
```
✅ `dnf config-manager --enable highavailability`


Set password for the hacluster user (same password on both servers):
```bash
echo 'hacluster:Password1' | chpasswd
```

Enable and start pcsd:
```bash
systemctl enable --now pcsd
```

```
[root@server2 ~]# echo 'hacluster:Password1' | chpasswd
[root@server2 ~]# systemctl enable --now pcsd
Created symlink /etc/systemd/system/multi-user.target.wants/pcsd.service → /usr/lib/systemd/system/pcsd.service.
[root@server2 ~]#
```

---

## Phase 3 — Firewall Configuration (both servers)

```bash
firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
```

Verify:
```bash
firewall-cmd --list-services
```

```
[root@server1 ~]# firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
success
success
[root@server1 ~]# firewall-cmd --list-services
cockpit dhcpv6-client high-availability ssh
[root@server1 ~]#
```

---

## Phase 4 — Create the Cluster (server1 only)

Authenticate both nodes (pcs 0.11+ syntax for AlmaLinux 9):
```bash
pcs host auth server1.example.com server2.example.com \
  -u hacluster -p Password1
```

Create the cluster:
```bash
pcs cluster setup peanut \
  server1.example.com server2.example.com
```

Start and enable the cluster on all nodes:
```bash
pcs cluster start --all
pcs cluster enable --all
```

Verify status:
```bash
pcs status
```

Expected output includes:
- `Online: [ server1.example.com server2.example.com ]`
- corosync, pacemaker, pcsd: active/enabled


```
[root@server1 ~]# pcs host auth server1.example.com server2.example.com \
  -u hacluster -p Password1
server1.example.com: Authorized
server2.example.com: Authorized
[root@server1 ~]# pcs cluster setup peanut \
  server1.example.com server2.example.com
No addresses specified for host 'server1.example.com', using 'server1.example.com'
No addresses specified for host 'server2.example.com', using 'server2.example.com'
Destroying cluster on hosts: 'server1.example.com', 'server2.example.com'...
server1.example.com: Successfully destroyed cluster
server2.example.com: Successfully destroyed cluster
Requesting remove 'pcsd settings' from 'server1.example.com', 'server2.example.com'
server1.example.com: successful removal of the file 'pcsd settings'
server2.example.com: successful removal of the file 'pcsd settings'
Sending 'corosync authkey', 'pacemaker authkey' to 'server1.example.com', 'server2.example.com'
server1.example.com: successful distribution of the file 'corosync authkey'
server1.example.com: successful distribution of the file 'pacemaker authkey'
server2.example.com: successful distribution of the file 'corosync authkey'
server2.example.com: successful distribution of the file 'pacemaker authkey'
Sending 'corosync.conf' to 'server1.example.com', 'server2.example.com'
server1.example.com: successful distribution of the file 'corosync.conf'
server2.example.com: successful distribution of the file 'corosync.conf'
Cluster has been successfully set up.
[root@server1 ~]# pcs cluster start --all
server2.example.com: Starting Cluster...
server1.example.com: Starting Cluster...
[root@server1 ~]# pcs cluster enable --all
server1.example.com: Cluster Enabled
server2.example.com: Cluster Enabled
[root@server1 ~]# pcs status
Cluster name: peanut

WARNINGS:
No stonith devices and stonith-enabled is not false
error: Resource start-up disabled since no STONITH resources have been defined
error: Either configure some or disable STONITH with the stonith-enabled option
error: NOTE: Clusters with shared data need STONITH to ensure data integrity
warning: Node server1.example.com is unclean but cannot be fenced
warning: Node server2.example.com is unclean but cannot be fenced
error: CIB did not pass schema validation
Errors found during check: config not valid

Cluster Summary:
  * Stack: unknown (Pacemaker is running)
  * Current DC: NONE
  * Last updated: Sat May 30 17:59:25 2026 on server1.example.com
  * Last change:  Sat May 30 17:59:09 2026 by hacluster via hacluster on server1.example.com
  * 2 nodes configured
  * 0 resource instances configured

Node List:
  * Node server1.example.com: UNCLEAN (offline)
  * Node server2.example.com: UNCLEAN (offline)

Full List of Resources:
  * No resources

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@server1 ~]#
```


---

## Phase 5 — Disable STONITH and Quorum (server1 only)

> ⚠️ For this lab only. Never do this in production.

```bash
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
```

Confirm warnings are gone:
```bash
pcs status
```

```
[root@server1 ~]# pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore

[root@server1 ~]# pcs status
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * Last updated: Sat May 30 18:01:45 2026 on server1.example.com
  * Last change:  Sat May 30 18:01:37 2026 by root via root on server1.example.com
  * 2 nodes configured
  * 0 resource instances configured

Node List:
  * Online: [ server1.example.com server2.example.com ]

Full List of Resources:
  * No resources

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@server1 ~]#
```

---

## Phase 6 — Cluster an IP Address (server1 only)

Create the clustered virtual IP resource:
```bash
pcs resource create cluster_ip \
  ocf:heartbeat:IPaddr2 \
  ip=10.0.0.50 cidr_netmask=24 \
  op monitor interval=20s
```

Verify the resource started and confirm the IP is active:
```bash
pcs status
ip address show
```

10.0.0.50 should appear as a secondary address on the active node's interface.

```
[root@server1 ~]# pcs resource create cluster_ip \
  ocf:heartbeat:IPaddr2 \
  ip=10.0.0.50 cidr_netmask=24 \
  op monitor interval=20s
[root@server1 ~]# pcs status
ip address show
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * Last updated: Sat May 30 18:07:03 2026 on server1.example.com
  * Last change:  Sat May 30 18:06:51 2026 by root via root on server1.example.com
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ server1.example.com server2.example.com ]

Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2):         Started server1.example.com

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:79:69:58 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.4/24 brd 10.0.0.255 scope global dynamic noprefixroute enp0s9
       valid_lft 171629sec preferred_lft 171629sec
    inet 10.0.0.50/24 brd 10.0.0.255 scope global secondary enp0s9
       valid_lft forever preferred_lft forever
    inet6 2601:58a:8f80:a2c0::f716/128 scope global dynamic noprefixroute
       valid_lft 344431sec preferred_lft 344431sec
    inet6 2601:58a:8f80:a2c0:a261:fba5:8671:4b3d/64 scope global dynamic noprefixroute
       valid_lft 299sec preferred_lft 299sec
    inet6 fe80::f71c:50ba:a0d0:2fde/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
[root@server1 ~]#
```

---

## Phase 7 — Test IP Failover
ing
From server2, start a continuous ping to the cluster IP:
```bash
ping 10.0.0.50
```

```
[root@server2 ~]# ping 10.0.0.50
PING 10.0.0.50 (10.0.0.50) 56(84) bytes of data.
64 bytes from 10.0.0.50: icmp_seq=1 ttl=64 time=1.66 ms
64 bytes from 10.0.0.50: icmp_seq=2 ttl=64 time=1.19 ms
64 bytes from 10.0.0.50: icmp_seq=3 ttl=64 time=1.06 ms
64 bytes from 10.0.0.50: icmp_seq=4 ttl=64 time=1.23 ms
64 bytes from 10.0.0.50: icmp_seq=5 ttl=64 time=1.57 ms
64 bytes from 10.0.0.50: icmp_seq=6 ttl=64 time=1.06 ms
64 bytes from 10.0.0.50: icmp_seq=7 ttl=64 time=1.32 ms

--- 10.0.0.50 ping statistics ---
7 packets transmitted, 7 received, 0% packet loss, time 6009ms
rtt min/avg/max/mdev = 1.055/1.298/1.659/0.219 ms
[root@server2 ~]#
```

From **server1**, put it into standby to simulate failure:

~~pcs cluster standby server1.example.com~~


```
❌[root@server1 ~]# pcs cluster standby server1.example.com

Usage: pcs cluster ...
    setup <cluster name> (<node name> [addr=<node address>]...)...
            [transport knet|udp|udpu
                [<transport options>] [link <link options>]...
                [compression <compression options>] [crypto <crypto options>]
            ] [totem <totem options>] [quorum <quorum options>]

```

```
pcs node standby server1.example.com
```

Verify the IP migrated to **server2**:
```bash
pcs status
```

```
[root@server2 ~]# pcs status
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * Last updated: Sat May 30 18:16:26 2026 on server2.example.com
  * Last change:  Sat May 30 18:16:11 2026 by root via root on server1.example.com
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Node server1.example.com: standby
  * Online: [ server2.example.com ]

Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2):         Started server2.example.com

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@server2 ~]#
```

Bring **server1** back online:

~~pcs cluster unstandby server1.example.com~~

```
pcs node unstandby server1.example.com
```

---

```

[root@server1 ~]# cat /etc/os-release
NAME="AlmaLinux"
VERSION="9.8 (Olive Jaguar)"
ID="almalinux"
ID_LIKE="rhel centos fedora"
VERSION_ID="9.8"
PLATFORM_ID="platform:el9"
PRETTY_NAME="AlmaLinux 9.8 (Olive Jaguar)"
ANSI_COLOR="0;34"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:almalinux:almalinux:9::baseos"
HOME_URL="https://almalinux.org/"
DOCUMENTATION_URL="https://wiki.almalinux.org/"
BUG_REPORT_URL="https://bugs.almalinux.org/"

ALMALINUX_MANTISBT_PROJECT="AlmaLinux-9"
ALMALINUX_MANTISBT_PROJECT_VERSION="9.8"
REDHAT_SUPPORT_PRODUCT="AlmaLinux"
REDHAT_SUPPORT_PRODUCT_VERSION="9.8"
SUPPORT_END=2032-06-01
[root@server1 ~]#
```
## Phase 8 — Install and Configure Apache (both servers)

Install Apache, then stop and disable it — Pacemaker will control it:
```bash
dnf install -y httpd
systemctl stop httpd
systemctl disable httpd
```

Open HTTP through the firewall:
```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

got success for both 

Create the server-status config needed by the cluster health check:
```bash
cat > /etc/httpd/conf.d/status.conf << 'EOF'
<Location /server-status>
  SetHandler server-status
  Require ip 127.0.0.1
</Location>
EOF
```

Create a unique index.html on each server so you can tell which one is serving:
```bash
# On server1
echo '<h1>Welcome</h1><hr>Server1' > /var/www/html/index.html

# On server2
echo '<h1>Welcome</h1><hr>Server2' > /var/www/html/index.html
```

Verify Apache config is valid on both servers:
```bash
apachectl configtest
# Expected: Syntax OK
```

got syntax ok

---

## Phase 9 — Cluster Apache (server1 only)

Create the Apache cluster resource:
```bash
pcs resource create web-server \
  ocf:heartbeat:apache \
  configfile=/etc/httpd/conf/httpd.conf \
  statusurl='http://127.0.0.1/server-status' \
  op monitor interval=20s
```

Check status — web-server and cluster_ip may be on different nodes at this point:
```bash
pcs status
```

Add a co-location constraint so web-server always runs on the same node as cluster_ip:
```bash
❌ pcs constraint colocation add web-server cluster_ip INFINITY
```

✅ fix 
```
pcs constraint colocation add web-server with cluster_ip INFINITY
```

Confirm both resources are now on the same node:
```bash
pcs status
# Both cluster_ip and web-server should show the same node
```

```
[root@server1 ~]# pcs resource create web-server \
  ocf:heartbeat:apache \
  configfile=/etc/httpd/conf/httpd.conf \
  statusurl='http://127.0.0.1/server-status' \
  op monitor interval=20s
[root@server1 ~]# pcs status
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * Last updated: Sat May 30 18:22:13 2026 on server1.example.com
  * Last change:  Sat May 30 18:22:03 2026 by root via root on server1.example.com
  * 2 nodes configured
  * 2 resource instances configured

Node List:
  * Online: [ server1.example.com server2.example.com ]

Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2):         Started server2.example.com
  * web-server  (ocf:heartbeat:apache):  Started server1.example.com

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@server1 ~]# pcs constraint colocation add web-server cluster_ip INFINITY

Usage: pcs constraint [constraints]...
    colocation add [<role>] <source resource id> with [<role>]
                   <target resource id> [score] [options] [id=constraint-id]
        Request <source resource> to run on the same node where pacemaker has
        determined <target resource> should run.  Positive values of score
        mean the resources should be run on the same node, negative values
        mean the resources should not be run on the same node.  Specifying
        'INFINITY' (or '-INFINITY') for the score forces <source resource> to
        run (or not run) with <target resource> (score defaults to "INFINITY").
        A role can be: 'Promoted', 'Unpromoted', 'Started', 'Stopped' (if no
        role is specified, it defaults to 'Started').

[root@server1 ~]# pcs constraint colocation add web-server with cluster_ip INFINITY
[root@server1 ~]#

```


```
[root@server1 ~]# pcs status
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * Last updated: Sat May 30 18:24:11 2026 on server1.example.com
  * Last change:  Sat May 30 18:23:47 2026 by root via root on server1.example.com
  * 2 nodes configured
  * 2 resource instances configured

Node List:
  * Online: [ server1.example.com server2.example.com ]

Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2):         Started server2.example.com
  * web-server  (ocf:heartbeat:apache):  Started server2.example.com

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@server1 ~]#

```


---

## Phase 10 — Test Apache Failover (server1)

Browse to the cluster IP and confirm you see which server services
```bash
curl http://10.0.0.50
```

![[Pasted image 20260530182627.png]]

```
curl http://10.0.0.50
<h1>Welcome</h1><hr>Server2                                                 
```

Put **server2** into standby to force a failover:
```bash
pcs node standby server2.example.com
```

Curl again — should now return Server1's page:
```bash
curl http://10.0.0.50
```

```
curl http://10.0.0.50
<h1>Welcome</h1><hr>Server1
```

Bring server2 back and verify cluster health:
```bash
pcs node unstandby server2.example.com
pcs status
```

```
[root@server2 ~]# pcs node unstandby server2.example.com
[root@server2 ~]# pcs status
Cluster name: peanut
Cluster Summary:
  * Stack: corosync (Pacemaker is running)
  * Current DC: server2.example.com (version 2.1.10-2.el9-5693eaeee) - partition with quorum
  * Last updated: Sat May 30 18:37:27 2026 on server2.example.com
  * Last change:  Sat May 30 18:37:16 2026 by root via root on server2.example.com
  * 2 nodes configured
  * 2 resource instances configured

Node List:
  * Online: [ server1.example.com server2.example.com ]

Full List of Resources:
  * cluster_ip  (ocf:heartbeat:IPaddr2):         Started server1.example.com
  * web-server  (ocf:heartbeat:apache):  Started server1.example.com

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@server2 ~]#
```

---

## Summary of Key Commands

| Action | Command |
|--------|---------|
| Check cluster status | `pcs status` |
| Put a node in standby | `pcs cluster standby <node>` |
| Bring a node back | `pcs cluster unstandby <node>` |
| List resources | `pcs resource status` |
| List constraints | `pcs constraint show` |
| Show cluster config | `pcs config show` |
| Stop all resources | `pcs resource disable <name>` |
| Start all resources | `pcs resource enable <name>` |
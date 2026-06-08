---
tags:
  - draft
  - linux
  - almalinux
  - networking
  - dhcp
  - PXE
linkedin: "True"
quartz: "True"
refactored: "True"
hands-on: "True"
completed: "True"
title: "PXE Lab Series - Lab 2: Configuring a DHCP Server"
hardlinked: "True"
---
**Series:** PXE Network Boot Lab (4-part series)  
**Lab:** 2 of 4  
**OS:** AlmaLinux 9  
**Server:** Server 1  
**IP:** 192.168.56.106 (host-only interface `enp0s8`)

---

## Overview

With DNS running from [[PXE Lab 1 - Configuring BIND DNS|Lab 1]], this lab adds the second foundational service: DHCP. A DHCP server is essential for PXE booting because clients power on with no configuration at all. They need an IP address, they need to know where the DNS server is, and they need to be pointed toward the TFTP server that holds the boot files. A single DHCP response handles all three.

I install ISC DHCP Server on Server 1, configure it to serve the `192.168.56.0/24` subnet with a dynamic pool and a static reservation for Server 2, and verify that Server 2 receives the correct address and DNS configuration.

**What this lab adds to Server 1:**

```
Server 1 (192.168.56.106)
├── DNS  - named   (port 53)  - Lab 1
└── DHCP - dhcpd   (port 67)  - this lab
```

**Prerequisites:**
- [[PXE Lab 1 - Configuring BIND DNS|Lab 1]] complete - Server 1 is running BIND on `192.168.56.106`
- Server 1 has a [[PXE Lab 1 - Configuring BIND DNS#Set the static IP on enp0s8|static IP]] on `enp0s8`
- Root or sudo access

---

## Part 1 - Disable VirtualBox's Built-in DHCP Server

VirtualBox includes its own DHCP server on the host-only network. If it stays active, clients may receive an IP from VirtualBox instead of Server 1, which would break the DNS and PXE configuration entirely. It needs to be disabled before ISC DHCP takes over.

On the host machine, go to **File - Tools - Network Manager**, select the **Host-only Networks** tab, click on your adapter, then open the **DHCP Server** tab and uncheck **Enable Server**.

![[Pasted image 20260603101851.png|500]]

In my case the VirtualBox DHCP server was already disabled, so I could move on. The screenshot confirms the server address shows `0.0.0.0` with the enable checkbox unchecked.

---

## Part 2 - Create Server 2

Server 2 is the DHCP client used to test the setup. It will receive its IP from Server 1 once DHCP is running, and in later labs it acts as a workstation for verifying network services.

### Create a new VM in VirtualBox

- **Name:** server2
- **OS type:** Linux / AlmaLinux 9 (64-bit)
- **RAM:** 1 GB minimum
- **Disk:** 20 GB

For the network adapter, I used **Adapter 2** set to Host-only (same adapter as Server 1's `enp0s8`). Using Adapter 2 instead of Adapter 1 means the interface appears as `enp0s8` inside the VM rather than `enp0s3`, keeping the interface naming consistent with Server 1's layout.

*On Lab 3 I end up [[PXE Lab 3 - Configuring FTP (vsftpd)#Part 1 - Add a NAT Interface to Server 2|adding a NAT interface]]*
### Install AlmaLinux 9 and set the hostname

Do a minimal install. Once booted, set the hostname:

```bash
hostnamectl set-hostname server2.example.vm
```

### Configure enp0s8 to use DHCP

Server 2 should get its IP dynamically. This is the default on a fresh install, but if the interface isn't coming up, force it with NetworkManager:

```bash
nmcli connection modify "Wired connection 1" ipv4.method auto
nmcli connection up "Wired connection 1"
```

At this point, Server 2 will fail to get an IP because Server 1's DHCP isn't running yet. This is expected - the failure confirms Server 2 is correctly looking for a DHCP server on the network.

The error `❌IP configuration could not be reserved, no available address, timeout` is exactly what you want to see here - it means the client is broadcasting correctly.

---

## Part 3 - Install the ISC DHCP Server on Server 1

Back on **Server 1**:

```bash
dnf install -y dhcp-server
```

Verify the service was installed:

```bash
systemctl status dhcpd
```

```
🔴 dhcpd.service - DHCPv4 Server Daemon
     Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; preset: disabled)
     Active: inactive (dead)
```

`inactive (dead)` is expected - the daemon won't start without a valid configuration file.

---

## Part 4 - Configure the DHCP Server

The configuration file defines the network the server is responsible for, the range of IPs it can hand out, and any static reservations for known machines.

```bash
vi /etc/dhcp/dhcpd.conf
```

The default file is nearly empty. Replace it with:

```
# Global options - apply to all subnets
option domain-name-servers 192.168.56.106;
option domain-search "example.vm";
default-lease-time 86400;
max-lease-time 86400;
ddns-update-style none;
authoritative;
log-facility local4;

# Subnet definition
subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.151 192.168.56.254;
}

# Static reservation for Server 2
host server2 {
    hardware ethernet 08:00:27:76:24:c2;
    fixed-address 192.168.56.120;
}
```

^dhcpConf
%%Notice here we define what bind gets from the interfaces, the dns servers, the search thing
[[#Verify the DNS server was received]]%%
**Global directives explained:**

| Directive | Purpose |
|-----------|---------|
| `option domain-name-servers` | Tells clients to use Server 1 as their DNS server |
| `option domain-search` | Allows short names like `server1` instead of `server1.example.vm` |
| `default-lease-time 86400` | IP lease duration of 1 day (86,400 seconds) |
| `max-lease-time 86400` | Maximum lease duration a client can request |
| `ddns-update-style none` | Disables dynamic DNS updates |
| `authoritative` | This server is the official DHCP authority for this network |
| `log-facility local4` | Sends DHCP log messages to the syslog `local4` facility |

**Subnet block explained:**

| Directive | Purpose |
|-----------|---------|
| `subnet` | The network this server will serve |
| `range` | Dynamic IP pool assigned to unknown clients (`.151` to `.254`) |

**Static reservation explained:**

| Directive           | Purpose                             |
| ------------------- | ----------------------------------- |
| `hardware ethernet` | Server 2's MAC address on `enp0s8`  |
| `fixed-address`     | The IP Server 2 will always receive |

The fixed address `192.168.56.120` is outside the dynamic range (`.151-.254`) but still within the subnet. This prevents it from being accidentally handed to another client.

### Get Server 2's MAC address

On **Server 2**, run:

```bash
ip link show enp0s8
```

```
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 08:00:27:76:24:c2 brd ff:ff:ff:ff:ff:ff
```

Copy the `link/ether` value and paste it into the `hardware ethernet` [[#^dhcpConf|line in]] `dhcpd.conf` on Server 1.

---

## Part 5 - Configure the Firewall on Server 1

DHCP uses UDP port 67 (server) and 68 (client). The firewall must allow this traffic:

```bash
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload
firewall-cmd --list-services
```

```
[root@server1 ~]# firewall-cmd --list-services
cockpit dhcp dhcpv6-client dns ssh
```

`dhcp` appears in the list alongside `dns` from [[PXE Lab 1 - Configuring BIND DNS|Lab 1]].

---

## Part 6 - Validate and Start the DHCP Service

### Test the configuration file for syntax errors

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

A clean run produces no error messages and exits without complaints:

```
Internet Systems Consortium DHCP Server 4.4.2b1
...
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcpd/dhcpd.leases
PID file: /var/run/dhcpd.pid
Source compiled to use binary-leases
```

### Start and enable the service

```bash
systemctl start dhcpd
systemctl enable dhcpd
systemctl status dhcpd
```

```
dhcpd.service - DHCPv4 Server Daemon
     Active: active (running) since Wed 2026-06-03 11:45:24 EDT
     Status: "Dispatching packets..."
   Main PID: 2236 (dhcpd)
```

The `Dispatching packets...` status confirms it is actively listening for DHCP requests.

You will also see warnings about `enp0s3` and `enp0s9` in the log output:

```
No subnet declaration for enp0s3 (10.0.2.15).
** Ignoring requests on enp0s3. **
```

These are normal and can be ignored. DHCP only needs to serve the `enp0s8` (host-only) interface.

### Confirm DHCP is listening on UDP port 67

```bash
ss -lun | grep 67
```

```
UNCONN 0  0  0.0.0.0:67  0.0.0.0:*
```

`0.0.0.0:67` means the server is listening on all interfaces for DHCP broadcasts.

---

## Part 7 - Test DHCP from Server 2

### Request a new IP lease

On **Server 2**, bounce the connection to trigger a DHCP request:

```bash
nmcli connection down "Wired connection 1"
nmcli connection up "Wired connection 1"
```
%%
**AlmaLinux 9 note:** `dhclient` was removed from AlmaLinux 9. NetworkManager handles DHCP lease management. The `dhclient` command will return `command not found`.
Old commands from CentOS
%%

After bringing the connection back up, check the assigned address:

```bash
ip a s enp0s8
```

![[Pasted image 20260603123916.png]]

#### Troubleshooting - Server 2 Gets a Dynamic IP Instead of the Reserved Address

After the first test, Server 2 received `192.168.56.151` (from the dynamic range) instead of the expected `192.168.56.120` (the static reservation). This happens when the [[#^dhcpConf|MAC address in]] `dhcpd.conf` does not match the [[#Get Server 2's MAC address|actual MAC of the machine]].

I had copied the wrong MAC initially. The fix was to update the `hardware ethernet` line in `dhcpd.conf` on Server 1, then restart the DHCP service and renew the lease on Server 2:

```bash
# On Server 1
vi /etc/dhcp/dhcpd.conf
# Update: hardware ethernet 08:00:27:76:24:c2;
systemctl restart dhcpd

# On Server 2
nmcli connection down "Wired connection 1"
nmcli connection up "Wired connection 1"

ip a s enp0s8
```

Works ✅ 
![[Pasted image 20260603123916.png]]

```
inet 192.168.56.120/24 brd 192.168.56.255 scope global dynamic noprefixroute enp0s8
```

`192.168.56.120` confirms the static reservation is working correctly.

### Verify the DNS server was received

```bash
cat /etc/resolv.conf
```

```
# Generated by NetworkManager
search example.vm
nameserver 192.168.56.106
```

Server 2 received both the DNS server address and the `example.vm` search domain from Server 1's DHCP response. No manual configuration was needed on Server 2.

### Check the NetworkManager lease file

On AlmaLinux 9, NetworkManager stores lease data in `/var/lib/NetworkManager/` rather than the old `/var/lib/dhclient/` *(ex CentOS)* path:

```bash
ls /var/lib/NetworkManager/
cat /var/lib/NetworkManager/internal-<uuid>-enp0s8.lease
```

```
# This is private data. Do not parse.
ADDRESS=192.168.56.120
```
%%you are in **sever 2** looking at the IP you were leased%%
### Test DNS resolution from Server 2

```bash
dig server1.example.vm
```

```
;; ANSWER SECTION:
server1.example.vm.     10000   IN      A       192.168.56.106

;; SERVER: 192.168.56.106#53(192.168.56.106)
```

The query was answered by Server 1 (`192.168.56.106`), which Server 2 learned about entirely through DHCP. This confirms the full chain is working: DHCP delivered the IP, DHCP delivered the DNS server, and that DNS server resolves the internal zone correctly.

---

## Summary

| Step | What was done |
|------|--------------|
| VirtualBox | Disabled the built-in DHCP server on the host-only network |
| Server 2 | Created a new VM as the DHCP client |
| Install | `dnf install -y dhcp-server` |
| Config | Defined subnet, dynamic range, and static reservation in `dhcpd.conf` |
| Firewall | Opened UDP 67 with `firewall-cmd --add-service=dhcp` |
| Started | `systemctl start dhcpd` and `systemctl enable dhcpd` |
| Tested | Server 2 received `192.168.56.120` and DNS `192.168.56.106` |

**Key files:**

| File | Purpose |
|------|---------|
| `/etc/dhcp/dhcpd.conf` | Main DHCP configuration |
| `/var/lib/NetworkManager/*.lease` | Lease records on the client (AlmaLinux 9) |

**Port reference:**

| Port | Protocol | Service |
|------|----------|---------|
| 67 | UDP | DHCP server |
| 68 | UDP | DHCP client |

---

## What's Next

**[[PXE Lab 3 - Configuring FTP (vsftpd)]]** builds on this setup by installing `vsftpd` on Server 1 and copying two installation ISOs (CentOS 7.2 and AlmaLinux 9) into `/var/ftp/pub/`. This FTP directory is required by [[PXE Lab 4 - Configuring PXE Boot|Lab 4]] (PXE Boot), which fetches the kernel, initrd, and Kickstart file from it over the network.

%%### Post
Part 2 of my PXE home lab series is up. This one covers setting up ISC DHCP Server on AlmaLinux 9.

The interesting part wasn't the installation, it was the details. VirtualBox has its own DHCP server that needs to be disabled first or your clients get addresses from the wrong place. Then there's the static reservation, which only works if the MAC address in your config actually matches the machine. Mine didn't on the first try.

Part 1 covered DNS, which this lab depends on entirely since the DHCP config advertises Server 1 as the DNS server for every client that connects. Parts 3 and 4 are FTP and PXE boot.

https://hectorproko.github.io/quartz/Mixed/pxeLab2-ConfiguringADhcpServer/PXE-Lab-2---Configuring-a-DHCP-Server

`#Linux #Sysadmin #Networking #DHCP #AlmaLinux #Infrastructure `
%%

%%
**OS:** AlmaLinux 9  
**Server:** Server 1  
**IP:** 192.168.56.106 (host-only interface `enp0s8`)

## Prerequisites

- **Lab 1 (DNS) is complete** — Server 1 is running BIND on `192.168.56.106`
- Server 1 has a static IP on `enp0s8` "Wired connection 1"
- You have root or sudo access

> This is **Lab 2 of 3**. PXE boot lab depends on DHCP being set up first.

---

## Part 1 — Disable VirtualBox's Built-in DHCP Server

VirtualBox provides its own DHCP server on the host-only network by default. It must be disabled to avoid conflicts with the ISC DHCP server you're about to configure.

### 1. On your host machine, open VirtualBox

Go to **File → Tools → Network Manager** 

### 2. Select the Host-Only Network

Click on the **Host-only Networks** tab and select your adapter (e.g., `VirtualBox Host-Only Ethernet Adapter`).

### 3. Disable the DHCP server

Click the **DHCP Server** tab and uncheck **Enable Server**, then click **Apply/OK**.

> If you skip this step, clients may get an IP from VirtualBox instead of your server, and testing will be unreliable.

it was already disabled

![[Pasted image 20260603101851.png]]

This affects all VMs on that network, which is exactly what you want — you're replacing VirtualBox's DHCP with your own on Server 1.

---

## Part 2 — Create Server 2

Server 2 will be the DHCP client used to test your setup.

### 1. Create a new VM in VirtualBox

- **Name:** server2
- **OS type:** Linux / AlmaLinux 9 (64-bit)
- **RAM:** 1GB minimum
- **Disk:** 20GB

### 2. Configure the network adapter

- **Adapter 1:** Host-only Network (same adapter as Server 1's `enp0s8`)

I used adapter 2 so the interface would show as enp0s8, adapter 1 is enp0s3

### 3. Install AlmaLinux 9

Do a minimal install. Hostname: `server2.example.vm`

```
hostnamectl set-hostname server2.example.vm
```
### 4. Leave enp0s8 set to DHCP on Server 2

This is the default — don't set a static IP. Server 2 should get its IP dynamically from Server 1's DHCP server.

**1. The interface `enp0s8` has no IP** — it needs to be set to DHCP:
```
nmcli connection modify enp0s8 ipv4.method auto
nmcli connection up enp0s8
```

❌
![[]]

**The problem:** Server 2 is trying to get a DHCP lease but failing — "no available address, timeout". This is because **Server 1's DHCP isn't running yet, or VirtualBox's DHCP is still enabled.**

💡 so is meant to fail because dhcp is not running in server1 yet?
Yes exactly! That's the correct order:
**You can't get a DHCP lease on Server 2 until Server 1's DHCP server is running.**

---

## Part 3 — Install the ISC DHCP Server on Server 1

Back on **Server 1**:

### 1. Install the DHCP server package

```bash
dnf install -y dhcp-server
```

> **AlmaLinux 9 note:** The package is `dhcp-server`, not `dhcp` as it was in CentOS 7.

had the same issue with dns not resolving

![[PXE Lab 1 - Configuring BIND DNS#Troubleshooting Error Downloading]]

### 2. Verify the service exists

```bash
systemctl status dhcpd
```

It will show inactive — that's expected, it's not configured yet.

```
[root@server1 ~]# systemctl status dhcpd
○ dhcpd.service - DHCPv4 Server Daemon
     Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; preset: disabled)
     Active: inactive (dead)
       Docs: man:dhcpd(8)
             man:dhcpd.conf(5)
[root@server1 ~]#
```

---

## Part 4 — Configure the DHCP Server

### 1. Open the DHCP configuration file

```bash
vi /etc/dhcp/dhcpd.conf
```

By default this file is mostly empty. Clear it and replace with the following:

```
# Global options — apply to all subnets
option domain-name-servers 192.168.56.106;
option domain-search "example.vm";
default-lease-time 86400;
max-lease-time 86400;
ddns-update-style none;
authoritative;
log-facility local4;

# Subnet definition
subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.151 192.168.56.254;
}

# Static reservation for Server 2
host server2 {
    hardware ethernet 08:00:27:12:0f:e1;
    fixed-address 192.168.56.120;
}
```

**Global options explained:**

|Directive|Purpose|
|---|---|
|`option domain-name-servers`|Tells clients to use Server 1 as their DNS server|
|`option domain-search`|Allows short names like `server1` instead of `server1.example.vm`|
|`default-lease-time 86400`|IP lease duration — 1 day (86,400 seconds)|
|`max-lease-time 86400`|Maximum lease a client can request|
|`ddns-update-style none`|Disables dynamic DNS updates|
|`authoritative`|This server is the official DHCP server for this network|
|`log-facility local4`|Sends DHCP logs to syslog facility local4|

**Subnet block explained:**

|Directive|Purpose|
|---|---|
|`subnet`|The network the server will serve|
|`range`|Dynamic IP pool for clients (`.151` to `.254`)|

**Static reservation explained:**

|Directive|Purpose|
|---|---|
|`hardware ethernet`|Server 2's MAC address on `enp0s8`|
|`fixed-address`|The IP Server 2 will always receive|

> The fixed address `192.168.56.120` is **outside the dynamic range** (`.151-.254`) but still within the subnet. This prevents it from being handed to another client.

### 2. Get Server 2's MAC address

On **Server 2**, run:

```bash
ip link show enp0s8
```

Look for the `link/ether` value, e.g. `08:00:27:12:0f:e1`. Update the `hardware ethernet` line in `dhcpd.conf` on Server 1 with this value.

![[Pasted image 20260603113413.png]]

---

## Part 5 — Configure the Firewall on Server 1

```bash
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload
```

Verify:

```bash
firewall-cmd --list-services
```

You should see `dhcp` in the list.

```
[root@server1 ~]# firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload
success
success
[root@server1 ~]# firewall-cmd --list-services
cockpit dhcp dhcpv6-client dns ssh
[root@server1 ~]#
```

---

## Part 6 — Validate and Start the DHCP Service

### 1. Test the configuration file for errors

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```


good output
```
[root@server1 ~]# dhcpd -t -cf /etc/dhcp/dhcpd.conf
Internet Systems Consortium DHCP Server 4.4.2b1
Copyright 2004-2019 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
ldap_gssapi_principal is not set,GSSAPI Authentication for LDAP will not be used
Not searching LDAP since ldap-server, ldap-port and ldap-base-dn were not specified in the config file
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcpd/dhcpd.leases
PID file: /var/run/dhcpd.pid
Source compiled to use binary-leases
[root@server1 ~]#
```

### 2. Start and enable the DHCP service

```bash
systemctl start dhcpd
systemctl enable dhcpd
systemctl status dhcpd
```

The status should show `active (running)`.

```
[root@server1 ~]# systemctl start dhcpd
systemctl enable dhcpd
systemctl status dhcpd
Created symlink /etc/systemd/system/multi-user.target.wants/dhcpd.service → /usr/lib/systemd/system/dhcpd.service.
🟢 dhcpd.service - DHCPv4 Server Daemon
     Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; enabled; preset: disabled)
     Active: active (running) since Wed 2026-06-03 11:45:24 EDT; 303ms ago
       Docs: man:dhcpd(8)
             man:dhcpd.conf(5)
   Main PID: 2236 (dhcpd)
     Status: "Dispatching packets..."
      Tasks: 1 (limit: 19336)
     Memory: 5.1M (peak: 5.3M)
        CPU: 15ms
     CGroup: /system.slice/dhcpd.service
             └─2236 /usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid

Jun 03 11:45:24 server1.example.vm dhcpd[2236]:
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: No subnet declaration for enp0s3 (10.0.2.15).
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: ** Ignoring requests on enp0s3.  If this is not what
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    you want, please write a subnet declaration
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    in your dhcpd.conf file for the network segment
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    to which interface enp0s3 is attached. **
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: Sending on   Socket/fallback/fallback-net
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: Server starting service.
Jun 03 11:45:24 server1.example.vm systemd[1]: Started DHCPv4 Server Daemon.
[root@server1 ~]#
```
### 3. Confirm DHCP is listening on UDP port 67

```bash
ss -lun | grep 67
```

Expected output shows something bound to `0.0.0.0:67`.

> **AlmaLinux 9 note:** Use `ss` instead of `netstat`. UDP port 67 is the DHCP server port.

```
[root@server1 ~]# ss -lun | grep 67
UNCONN 0      0             0.0.0.0:67        0.0.0.0:*
[root@server1 ~]#
```

### 4. Check for interface errors in the logs

```bash
journalctl -u dhcpd --no-pager | tail -20
```

You may see a warning like:

```
No subnet declaration for enp0s3
```

This is **normal** — the DHCP server only needs to serve the `enp0s8` (host-only) interface, not the NAT interface.

```
[root@server1 ~]# journalctl -u dhcpd --no-pager | tail -20
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: Wrote 0 leases to leases file.
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: No subnet declaration for enp0s9 (10.0.0.184).
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: ** Ignoring requests on enp0s9.  If this is not what
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    you want, please write a subnet declaration
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    in your dhcpd.conf file for the network segment
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    to which interface enp0s9 is attached. **
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: Listening on LPF/enp0s8/08:00:27:42:bc:22/192.168.56.0/24
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: Sending on   LPF/enp0s8/08:00:27:42:bc:22/192.168.56.0/24
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: No subnet declaration for enp0s3 (10.0.2.15).
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: ** Ignoring requests on enp0s3.  If this is not what
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    you want, please write a subnet declaration
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    in your dhcpd.conf file for the network segment
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:    to which interface enp0s3 is attached. **
Jun 03 11:45:24 server1.example.vm dhcpd[2236]:
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: Sending on   Socket/fallback/fallback-net
Jun 03 11:45:24 server1.example.vm dhcpd[2236]: Server starting service.
Jun 03 11:45:24 server1.example.vm systemd[1]: Started DHCPv4 Server Daemon.
[root@server1 ~]#
```

---

## Part 7 — Test DHCP from Server 2

On **Server 2**:

### 1. Release and renew the IP lease

```bash
dhclient -r enp0s8
dhclient enp0s8
```

- `-r` releases the current lease
- Running `dhclient` again requests a new one from Server 1
![[Pasted image 20260603121153.png]]


That's expected on **AlmaLinux 9**. `dhclient` was removed — it's been replaced by **NetworkManager** for managing DHCP leases.

Use this instead:

```bash
# Release the current IP
nmcli connection down enp0s8

# Request a new IP from the DHCP server
nmcli connection up enp0s8
```

```bash
ip a s enp0s8
```

![[Pasted image 20260603121644.png]]
### 2. Check the assigned IP

```bash
ip a s enp0s8
```

You should see `192.168.56.120` — the fixed address you reserved for Server 2.

❌doesnt get the expected ip

#### Troubleshooting DHCP reserved IP

The IP it got is `192.168.56.151` (dynamic range) instead of `192.168.56.120` (the static reservation). This means the MAC address in your `dhcpd.conf` doesn't match the actual MAC of this machine.

vim /etc/dhcp/dhcpd.conf
```bash
# Static reservation for Server 2
host server2 {
    hardware ethernet 08:00:27:12:0f:e1; #replace with 08:00:27:76:24:c2
    fixed-address 192.168.56.120;
}
```
mac address does not match

```
# On Server 1
systemctl restart dhcpd

# On Server 2
nmcli connection down "Wired connection 1"
nmcli connection up "Wired connection 1"

ip a s enp0s8
```

server 2
![[Pasted image 20260603123916.png]]

✅ You should see `192.168.56.120` — the fixed address you reserved for Server 2.

### 3. Verify the DNS server was received

```bash
cat /etc/resolv.conf
```

Expected:

```
search example.vm
nameserver 192.168.56.106
```

This confirms Server 2 received both the IP and the DNS configuration from Server 1.

![[Pasted image 20260603124514.png]]
``
### 4. View the full lease details

```bash
cat /var/lib/dhclient/dhclient.leases
```

Look for the `enp0s8` lease block. It should show:

- `fixed-address 192.168.56.120`
- `option domain-name-servers 192.168.56.106`
- `option dhcp-server-identifier 192.168.56.106`
- Renewal, rebind, and expiry timestamps

**Lease timing explained:**

|Timing|Meaning|
|---|---|
|`renew` (T1)|Client tries to renew at 50% of lease duration|
|`rebind` (T2)|If renewal fails, client tries any DHCP server|
|`expire` (T3)|Lease expires — client must get a new one|

#### dhclient outdated
On **AlmaLinux 9**, NetworkManager handles DHCP by default instead of the standalone `dhclient` command, so the lease information is stored in a different place.
networkmanager lease files
![[Pasted image 20260603125306.png]]
### 5. Test DNS resolution from Server 2

```bash
dig server1.example.vm
```

This should return `192.168.56.106` — confirming that Server 2 is using Server 1 for DNS (received via DHCP) and can resolve the `example.vm` zone.

dig was not installed
no internet to dnf install
temporary putting bridge adapter
seems like the hostname change kickedin after restart
![[Pasted image 20260603130818.png]]

✅ That's a perfect result! Your DNS is working correctly. Breaking down what it shows:

---

## Summary

|Step|What you did|
|---|---|
|VirtualBox|Disabled the built-in DHCP server on the host-only network|
|Server 2|Created a new VM as the DHCP client|
|Install|`dnf install -y dhcp-server`|
|Config|Defined subnet, dynamic range, and static reservation in `dhcpd.conf`|
|Firewall|Opened UDP 67 with `firewall-cmd --add-service=dhcp`|
|Started|`systemctl start dhcpd` and `systemctl enable dhcpd`|
|Tested|Server 2 received `192.168.56.120` and DNS `192.168.56.106`|

**Key files and locations:**

|File|Purpose|
|---|---|
|`/etc/dhcp/dhcpd.conf`|Main DHCP configuration|
|`/var/lib/dhclient/dhclient.leases`|Lease records on the client|

**Port reference:**

| Port | Protocol | Service     |
| ---- | -------- | ----------- |
| 67   | UDP      | DHCP server |
| 68   | UDP      | DHCP client |

---

## What's Next

[[PXE Lab 3 - Configuring FTP (vsftpd)]] builds directly on this lab. It installs `vsftpd` on Server 1 and copies two installation ISOs (CentOS 7.2 and AlmaLinux 9) into `/var/ftp/pub/`. This FTP directory is required by both Lab 4 (PXE Boot), which borrows the kernel and initrd from it, and by Server 2 as a local package repository.
%%
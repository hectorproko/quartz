---

tags:
  - linux
  - almalinux
  - centos
  - networking
  - PXE
  - tftp
  - kickstart
linkedin: "False"
quartz: "False"
refactored: "False"
hands-on: "True"
completed: "True"
title: "PXE Lab Series - Lab 4: Configuring PXE Network Boot"
hardlinked: "True"
---
**Series:** PXE Network Boot Lab (4-part series)  
**Lab:** 4 of 4  
**OS:** AlmaLinux 9  
**Server:** Server 1  
**IP:** 192.168.56.106 (host-only interface `enp0s8`)

---

## Overview

This is the final lab in the series. All the infrastructure built in Labs 1 through 3 (DNS, DHCP, FTP) comes together here to enable PXE (Preboot eXecution Environment) network booting. By the end of this lab, a client VM with no OS and no boot media can power on, receive an IP from Server 1, download a bootloader over TFTP, present a menu, and install CentOS 7.2 completely unattended using a Kickstart file - without ever touching a CD, USB drive, or boot ISO.

**What PXE boot actually does:**

```
Client powers on - no OS, no disk
    |
    v
NIC broadcasts DHCP Discover
    |
    v
Server 1 responds: IP + next-server (TFTP) + filename (pxelinux.0)
    |
    v
Client downloads pxelinux.0 via TFTP (port 69)
    |
    v
PXE menu appears - user selects install option
    |
    v
Kernel (vmlinuz) + initrd downloaded via TFTP
    |
    v
Installer fetches Kickstart file via FTP - automated install runs
    |
    v
OS installed, VM reboots into new system
```

**Final state of Server 1 after this lab:**

```
Server 1 (192.168.56.106)
├── DNS  - named        (port 53)  - Lab 1
├── DHCP - dhcpd        (port 67)  - Lab 2
├── FTP  - vsftpd       (port 21)  - Lab 3
└── TFTP - tftp.socket  (port 69)  - this lab
```

**Prerequisites:**

| Requirement | From |
|-------------|------|
| Server 1 static IP `192.168.56.106` on `enp0s8` | Lab 1 |
| BIND DNS running, serving `example.vm` | Lab 1 |
| ISC DHCP running, serving `192.168.56.0/24` | Lab 2 |
| vsftpd running on Server 1 | Lab 3 |
| CentOS 7.2 media at `/var/ftp/pub/centos72/` | Lab 3 |
| AlmaLinux 9 media at `/var/ftp/pub/almalinux9/` | Lab 3 |
| `pxelinux.0` and `menu.c32` in `/var/lib/tftpboot/` | Lab 3 |

Verify all services before starting:

```bash
systemctl status named
systemctl status dhcpd
systemctl status vsftpd
```

And confirm the syslinux boot files from Lab 3 are in place:

```bash
ls /var/lib/tftpboot/
```

```
menu.c32  pxelinux.0
```

---

## Step 1 - Install Required Packages

```bash
dnf install -y tftp tftp-server
```

| Package | Role |
|---------|------|
| `tftp` | TFTP client (for local testing) |
| `tftp-server` | TFTP server daemon |

`syslinux` was already installed in Lab 3, so `pxelinux.0` and `menu.c32` are already present.

```
[root@server1 ~]# dnf install -y tftp tftp-server
Package tftp-5.2-40.el9.x86_64 is already installed.
Package tftp-server-5.2-40.el9.x86_64 is already installed.
Nothing to do.
Complete!
```

---

## Step 2 - Copy Kernels and initrds to the TFTP Root

The TFTP root directory is `/var/lib/tftpboot/`. This is where the client VM looks for everything it downloads during the boot process. I copy the kernel and initial RAM disk from each ISO and rename them to avoid filename conflicts.

```bash
cd /var/lib/tftpboot

cp /var/ftp/pub/almalinux9/isolinux/vmlinuz  ./vmlinuz-alma9
cp /var/ftp/pub/almalinux9/isolinux/initrd.img ./initrd-alma9.img

cp /var/ftp/pub/centos72/isolinux/vmlinuz    ./vmlinuz-centos7
cp /var/ftp/pub/centos72/isolinux/initrd.img  ./initrd-centos7.img

ls -l /var/lib/tftpboot
```

```
-r--r--r--. 1 root root 223197216 Jun  3 23:10 initrd-alma9.img
-rw-r--r--. 1 root root  48434768 Jun  3 23:10 initrd-centos7.img
-rw-r--r--. 1 root root     26148 Jun  3 14:03 menu.c32
-rw-r--r--. 1 root root     42722 Jun  3 14:03 pxelinux.0
-r-xr-xr-x. 1 root root  15173808 Jun  3 23:09 vmlinuz-alma9
-rwxr-xr-x. 1 root root   5877760 Jun  3 23:10 vmlinuz-centos7
```

The AlmaLinux 9 initrd is significantly larger (223 MB vs 48 MB for CentOS 7.2) because it includes more hardware support and a newer installer.

---

## Step 3 - Configure DHCP for PXE

The DHCP server is already running from Lab 2. PXE requires two additional fields in the subnet block: `next-server` (the address of the TFTP server) and `filename` (the boot file the client should request first).

```bash
vi /etc/dhcp/dhcpd.conf
```

Add the two PXE directives inside the subnet block, after the `range` line:

```
subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.151 192.168.56.254;
    next-server 192.168.56.106;    # TFTP server = Server 1
    filename "pxelinux.0";         # boot file to download first
}
```

| Directive | Purpose |
|-----------|---------|
| `next-server` | Tells the client where to find the TFTP server |
| `filename` | The first file the client downloads over TFTP |

Validate and restart:

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
systemctl restart dhcpd
ss -lun | grep 67
```

```
[root@server1 ~]# dhcpd -t -cf /etc/dhcp/dhcpd.conf
Source compiled to use binary-leases
[no errors]

[root@server1 ~]# ss -lun | grep 67
UNCONN 0  0  0.0.0.0:67  0.0.0.0:*
```

Clean output from `dhcpd -t` with no errors, and DHCP still listening on port 67.

---

## Step 4 - Open the Firewall for TFTP

```bash
firewall-cmd --permanent --add-service=tftp
firewall-cmd --reload
firewall-cmd --list-services
```

```
cockpit dhcp dhcpv6-client dns ftp ssh tftp
```

All four lab services (dns, dhcp, ftp, tftp) are now visible in the firewall rules.

---

## Step 5 - Start the TFTP Server

TFTP is managed as a socket-activated service on AlmaLinux 9, meaning systemd starts the daemon on demand when the first client connects.

```bash
systemctl start tftp.socket
systemctl enable tftp.socket
ss -lun | grep 69
```

```
UNCONN 0  0  *:69  *:*
```

UDP port 69 is open and listening.

**Complete port reference for all four labs:**

| Port | Protocol | Service |
|------|----------|---------|
| 53 | TCP/UDP | DNS |
| 21 | TCP | FTP |
| 67 | UDP | DHCP server |
| 68 | UDP | DHCP client |
| 69 | UDP | TFTP |

---

## Step 6 - Create the PXE Boot Menu

PXELINUX looks for its configuration in `/var/lib/tftpboot/pxelinux.cfg/`. The `default` file inside this directory is used for any client that does not have a MAC-specific config.

### Create the config directory

```bash
mkdir /var/lib/tftpboot/pxelinux.cfg
```

### Create the default menu

```bash
vi /var/lib/tftpboot/pxelinux.cfg/default
```

```
default menu.c32
prompt 0
timeout 1000
ontimeout local
menu title PXE Boot Menu

label local
  menu label Boot from local disk
  localboot 0xffff

label install-alma9
  menu label Install AlmaLinux 9
  kernel vmlinuz-alma9
  append initrd=initrd-alma9.img ip=dhcp inst.repo=ftp://192.168.56.106/pub/almalinux9

label install-centos7
  menu label Install CentOS 7.2
  kernel vmlinuz-centos7
  append initrd=initrd-centos7.img ip=dhcp inst.repo=ftp://192.168.56.106/pub/centos72
```

| Directive | Meaning |
|-----------|---------|
| `default menu.c32` | Load the graphical boot menu |
| `prompt 0` | Skip the text prompt, go straight to the menu |
| `timeout 1000` | Auto-boot after 100 seconds (units are tenths of seconds) |
| `ontimeout local` | Boot from local disk on timeout |
| `localboot 0xffff` | Boot from the local disk |
| `kernel` | Which kernel file to download from TFTP |
| `append` | Kernel parameters passed at boot time |
| `inst.repo` | Where Anaconda fetches the installation packages |

### Test the default menu - create a client VM

Create a test VM in VirtualBox:

- **RAM:** 2048 MB minimum (AlmaLinux 9 requires more than CentOS 7)
- **Disk:** Optional for initial menu test
- **Network:** Adapter 2 - Host-only (same adapter as Server 1's `enp0s8`)
- **Boot order:** Network first (above Hard Disk in System settings)

#### Troubleshooting - Test VM Does Not Reach the PXE Server

On the first boot attempt, the VM showed a TFTP permission denied error:

![Pasted_image_20260604081948.png]

```
tftp://10.0.2.4/Test.pxe... Permission denied
```

The VM was getting an IP from the NAT interface rather than the host-only network, and the TFTP path was wrong. The issue was the network adapter was set to NAT instead of Host-only. Switching to Host-only on Adapter 2 resolved this.

On the next attempt, the bootloader loaded but the menu failed to render correctly:

![Pasted_image_20260604082352.png]

```
pxelinux.0: 42722 bytes [PXE-NBP]
Failed to load ldlinux.c32
Boot failed
```

`menu.c32` depends on several other `.c32` support libraries that were not present in the TFTP root. The fix was to copy all syslinux modules:

```bash
cp /usr/share/syslinux/*.c32 /var/lib/tftpboot/
```

After copying all the `.c32` files, the boot menu appeared correctly.

---

## Step 7 - Add a MAC-Specific Config

PXELINUX can serve different boot configurations to specific machines by matching the client's MAC address. The filename follows a strict format: `01-` prefix, the MAC address with colons replaced by hyphens, all lowercase.

### Get the client VM's MAC address

In VirtualBox: VM Settings - Network - Adapter 2 - Advanced - copy the MAC address.

![Pasted_image_20260604084339.png]

```
080027D4A5CB
```

### Create the MAC-specific config file

```bash
cd /var/lib/tftpboot/pxelinux.cfg

# Format: 01- + MAC with hyphens + all lowercase
cp default 01-08-00-27-d4-a5-cb
```

> Filename rules: starts with `01-`, colons replaced by hyphens, all lowercase. Example: MAC `08:00:27:60:EF:5B` becomes `01-08-00-27-60-ef-5b`.

This MAC-specific file initially contained the same menu as `default`. The purpose is to later customize it for Kickstart so this specific machine installs automatically without menu interaction.

After creating the file, the test VM only sees the CentOS 7.2 install option:

![Pasted_image_20260604122246.png]

---

## Step 8 - Set Up Kickstart for Automated CentOS 7.2 Install

A Kickstart file is an answer file for the Red Hat Anaconda installer. Instead of clicking through the installation wizard, every choice (disk layout, packages, network, timezone, root password) is specified in advance. The installer reads the file at boot and runs completely unattended.

The file is placed in the FTP directory so the client can fetch it during the PXE process.

### Create the Kickstart file on Server 1

```bash
vi /var/ftp/pub/centos7.ks
```

```
# Installation method
install
url --url="ftp://192.168.56.106/pub/centos72"

# UI mode
text
firstboot --disable

# Locale and keyboard
lang en_US.UTF-8
keyboard --vckeymap=us --xlayouts='us'
timezone America/New_York --utc

# Authentication
auth --enableshadow --passalgo=sha512

# Network
network --bootproto=dhcp --device=enp0s3 --ipv6=auto --activate
network --hostname=server3.example.vm

# Security
rootpw --plaintext changeme
selinux --enforcing

# Disk
ignoredisk --only-use=sda
clearpart --all --initlabel
autopart --type=lvm

# Bootloader
bootloader --location=mbr --boot-drive=sda

# Packages
%packages
@^minimal
@core
kexec-tools
bash-completion
%end

# Post install
%post
echo "Installation complete"
%end

# Reboot after install
reboot
```

> **Tip:** CentOS 7 automatically generates `/root/anaconda-ks.cfg` after every installation, recording all choices made during that install. Copying Server 1's own generated file as a base is the recommended approach for building Kickstart files.

### Set permissions and verify FTP access

```bash
chmod 644 /var/ftp/pub/centos7.ks
curl ftp://192.168.56.106/pub/centos7.ks
```

The curl output should return the full Kickstart file contents, confirming it is accessible via FTP.

### Update the MAC-specific PXE config to use Kickstart

```bash
vi /var/lib/tftpboot/pxelinux.cfg/01-08-00-27-d4-a5-cb
```

Update the `install-centos7` label to reference the Kickstart file:

```
label install-centos7
  menu label Install CentOS 7.2 (Auto)
  kernel vmlinuz-centos7
  append initrd=initrd-centos7.img ip=dhcp ks=ftp://192.168.56.106/pub/centos7.ks
```

The `ks=` parameter replaces `inst.repo=`. When the installer finds a `ks=` argument, it fetches the file and runs in fully automated mode.

### Test the automated install

Boot the client VM and select **Install CentOS 7.2 (Auto)** from the menu. The installer fetches the Kickstart file, partitions the disk, downloads packages from the FTP repo on Server 1, and runs post-install tasks - all without any prompts.

The installation runs post-setup tasks and writes the bootloader:

![Pasted_image_20260604122535.png]

After completion, the VM reboots automatically into the freshly installed system:

![Pasted_image_20260604122711.png]

```
CentOS Linux 7 (Core)
Kernel 3.10.0-693.el7.x86_64 on an x86_64

server3 login:
```

The hostname `server3` confirms the Kickstart `network --hostname=server3.example.vm` directive was applied. The system installed entirely over the network from Server 1.

---

## Troubleshooting

| Problem | Check |
|---------|-------|
| VM gets no IP | `systemctl status dhcpd` - confirm `next-server` and `filename` are in the subnet block |
| PXE menu does not appear | `ss -lun \| grep 69` - confirm TFTP is listening |
| `file not found` on boot | `ls /var/lib/tftpboot` - verify all kernel and initrd files exist |
| `Failed to load ldlinux.c32` | Copy all syslinux modules: `cp /usr/share/syslinux/*.c32 /var/lib/tftpboot/` |
| MAC config not loading | Check filename: `01-` prefix, hyphens not colons, all lowercase |
| Kickstart install fails | `curl ftp://192.168.56.106/pub/centos7.ks` - confirm file is accessible |
| Installer cannot find packages | Confirm the `url` line in the Kickstart file points to the correct FTP path |
| DHCP errors on start | Normal - `enp0s3`/`enp0s9` subnet warnings can be ignored |
| TFTP blocked by firewall | `firewall-cmd --list-services` - confirm `tftp` is listed |

---

## Summary

| Step | What was done |
|------|--------------|
| 1 | Installed `tftp` and `tftp-server` |
| 2 | Copied kernels and initrds from both ISOs to `/var/lib/tftpboot/` |
| 3 | Added `next-server` and `filename` to the existing DHCP subnet block |
| 4 | Opened TFTP port 69 in `firewalld` |
| 5 | Started TFTP with `systemctl start tftp.socket` |
| 6 | Created `pxelinux.cfg/default` with boot menu entries for both images |
| 7 | Created a MAC-specific config file targeting the test VM |
| 8 | Built a Kickstart file and wired it into the MAC-specific PXE config |

**All services running on Server 1:**

| Service | Port | Started in |
|---------|------|-----------|
| DNS (named) | 53 | Lab 1 |
| DHCP (dhcpd) | 67/68 | Lab 2 |
| FTP (vsftpd) | 21 | Lab 3 |
| TFTP (tftp.socket) | 69 | This lab |

**Key files reference:**

| File | Purpose |
|------|---------|
| `/var/lib/tftpboot/pxelinux.0` | PXE bootloader downloaded by the client |
| `/var/lib/tftpboot/pxelinux.cfg/default` | Default boot menu for all clients |
| `/var/lib/tftpboot/pxelinux.cfg/01-*` | Per-machine boot config matched by MAC address |
| `/var/ftp/pub/centos7.ks` | Kickstart file for automated CentOS 7.2 install |
| `/var/ftp/pub/centos72/isolinux/` | CentOS 7.2 kernel and initrd source |
| `/var/ftp/pub/almalinux9/isolinux/` | AlmaLinux 9 kernel and initrd source |

%%
## Prerequisites

Before starting this lab, the following must already be complete:

|Requirement|From|
|---|---|
|Server 1 has a static IP (`192.168.56.106`) on `enp0s8`|Lab 1|
|BIND DNS is running and serving `example.vm`|Lab 1|
|ISC DHCP is installed, configured, and running|Lab 2|
|vsftpd is running on Server 1|Lab 3|
|CentOS 7.2 media available at `/var/ftp/pub/centos72/`|Lab 3|
|AlmaLinux 9 media available at `/var/ftp/pub/almalinux9/`|Lab 3|
|`pxelinux.0` and `menu.c32` already in `/var/lib/tftpboot/`|Lab 3|

Verify all services are up before proceeding:

```bash
systemctl status named
systemctl status dhcpd
systemctl status vsftpd
```

Verify tftpboot already has the syslinux files from Lab 3:

```bash
ls /var/lib/tftpboot/
```

Expected:

```
menu.c32  pxelinux.0
```

```
[root@server1 ~]# ls /var/lib/tftpboot/
menu.c32  pxelinux.0
[root@server1 ~]#
```
---

## Overview

In this lab you will turn Server 1 into a PXE boot server. By the end, a client VM with no OS will boot over the network and install AlmaLinux 9 automatically using a Kickstart file. CentOS 7.2 is also available as a manual install option.

**What runs on Server 1 after this lab:**

```
Server 1 (192.168.56.106)
├── DNS    → named        (port 53)
├── DHCP   → dhcpd        (port 67/68)
├── FTP    → vsftpd       (port 21)
└── TFTP   → tftp.socket  (port 69)  ← new
```

**PXE boot flow:**

```
Client powers on
    ↓
NIC sends DHCP Discover
    ↓
Server 1 DHCP replies with IP + next-server + filename
    ↓
Client downloads pxelinux.0 via TFTP
    ↓
PXE menu loads → user selects install
    ↓
Kernel (vmlinuz) + initrd downloaded via TFTP
    ↓
Kickstart file fetched via FTP → automated install begins
```

---

## Step 1 — Install Required Packages

```bash
dnf install -y tftp tftp-server
```

|Package|Role|
|---|---|
|`tftp`|TFTP client|
|`tftp-server`|TFTP server daemon|

> `syslinux` was already installed in Lab 3 — `pxelinux.0` and `menu.c32` are already in `/var/lib/tftpboot/`.

```
[root@server1 ~]# dnf install -y tftp tftp-server
Last metadata expiration check: 1:50:35 ago on Wed Jun  3 21:17:36 2026.
Package tftp-5.2-40.el9.x86_64 is already installed.
Package tftp-server-5.2-40.el9.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
[root@server1 ~]#
```
---

## Step 2 — Copy Kernels and initrds to TFTP Root

`pxelinux.0` and `menu.c32` are already there from Lab 3. We just need to copy the kernel and initrd from each ISO.

```bash
cd /var/lib/tftpboot

# From AlmaLinux 9 install media
cp /var/ftp/pub/almalinux9/isolinux/vmlinuz ./vmlinuz-alma9
cp /var/ftp/pub/almalinux9/isolinux/initrd.img ./initrd-alma9.img

# From CentOS 7.2 install media
cp /var/ftp/pub/centos72/isolinux/vmlinuz ./vmlinuz-centos7
cp /var/ftp/pub/centos72/isolinux/initrd.img ./initrd-centos7.img

# Verify all files are present
ls -l /var/lib/tftpboot
```

Expected output:

```
initrd-alma9.img
initrd-centos7.img
menu.c32          ← from Lab 3
pxelinux.0        ← from Lab 3
vmlinuz-alma9
vmlinuz-centos7
```

> We rename the kernel and initrd files to avoid conflicts between the two images.

```
[root@server1 tftpboot]# cp /var/ftp/pub/almalinux9/isolinux/vmlinuz ./vmlinuz-alma9
[root@server1 tftpboot]# cp /var/ftp/pub/almalinux9/isolinux/initrd.img ./initrd-alma9.img
[root@server1 tftpboot]# cp /var/ftp/pub/centos72/isolinux/vmlinuz ./vmlinuz-centos7
[root@server1 tftpboot]# cp /var/ftp/pub/centos72/isolinux/initrd.img ./initrd-centos7.img
[root@server1 tftpboot]# ls -l /var/lib/tftpboot
total 285900
-r--r--r--. 1 root root 223197216 Jun  3 23:10 initrd-alma9.img
-rw-r--r--. 1 root root  48434768 Jun  3 23:10 initrd-centos7.img
-rw-r--r--. 1 root root     26148 Jun  3 14:03 menu.c32
-rw-r--r--. 1 root root     42722 Jun  3 14:03 pxelinux.0
-r-xr-xr-x. 1 root root  15173808 Jun  3 23:09 vmlinuz-alma9
-rwxr-xr-x. 1 root root   5877760 Jun  3 23:10 vmlinuz-centos7
[root@server1 tftpboot]#
```

---

## Step 3 — Configure DHCP for PXE

The DHCP server is already running from Lab 2. Add two PXE-specific lines to the existing subnet block.

```bash
vi /etc/dhcp/dhcpd.conf
```

Add `next-server` and `filename` **after** the `range` line inside the subnet block:

```
subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.151 192.168.56.254;
    next-server 192.168.56.106;    # TFTP server = Server 1
    filename "pxelinux.0";         # boot file to download
}
```

```
[root@server1 tftpboot]# cat /etc/dhcp/dhcpd.conf
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
    next-server 192.168.56.106;    # TFTP server = Server 1
    filename "pxelinux.0";         # boot file to download
}

# Static reservation for Server 2
host server2 {
    hardware ethernet 08:00:27:de:58:03;
    fixed-address 192.168.56.120;
}
[root@server1 tftpboot]#
```

|Directive|Purpose|
|---|---|
|`next-server`|Tells the client where the TFTP server is|
|`filename`|The first file the client downloads over TFTP|

Validate and restart:

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
systemctl restart dhcpd
```

```
[root@server1 tftpboot]# dhcpd -t -cf /etc/dhcp/dhcpd.conf
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
[root@server1 tftpboot]# systemctl restart dhcpd
[root@server1 tftpboot]#
```

✅ That's a clean output — no errors. Both commands succeeded:

Confirm DHCP is still listening on UDP 67:

```bash
ss -lun | grep 67
```

> Errors about unconfigured interfaces (`enp0s3`, `enp0s9`) in the logs are normal — DHCP only needs to serve `enp0s8`.

```
[root@server1 tftpboot]# ss -lun | grep 67
UNCONN 0      0             0.0.0.0:67        0.0.0.0:*
[root@server1 tftpboot]#
```
---

## Step 4 — Open the Firewall for TFTP

```bash
firewall-cmd --permanent --add-service=tftp
firewall-cmd --reload
```

Verify:

```bash
firewall-cmd --list-services
```

You should see `tftp` in the list.

```
[root@server1 tftpboot]# firewall-cmd --permanent --add-service=tftp
firewall-cmd --reload
success
success
[root@server1 tftpboot]# firewall-cmd --list-services
cockpit dhcp dhcpv6-client dns ftp ssh tftp
[root@server1 tftpboot]#
```

---

## Step 5 — Start the TFTP Server

```bash
systemctl start tftp.socket
systemctl enable tftp.socket
```

Verify TFTP is listening on UDP port 69:

```bash
ss -lun | grep 69
```

> The entry may show as `:::69` (IPv6 notation) — this is fine, it still serves IPv4 clients.

**Full port summary:**

|Port|Protocol|Service|
|---|---|---|
|53|TCP/UDP|DNS|
|21|TCP|FTP|
|67|UDP|DHCP server|
|68|UDP|DHCP client|
|69|UDP|TFTP ← new|

```
[root@server1 tftpboot]# systemctl start tftp.socket
systemctl enable tftp.socket
Created symlink /etc/systemd/system/sockets.target.wants/tftp.socket → /usr/lib/systemd/system/tftp.socket.
[root@server1 tftpboot]# ss -lun | grep 69
UNCONN 0      0                   *:69              *:*
[root@server1 tftpboot]#
```

---

## Step 6 — Create the PXE Boot Menu

### 6a — Create the config directory

```bash
mkdir /var/lib/tftpboot/pxelinux.cfg
```

> PXELINUX uses a directory for config, unlike ISOLINUX/SYSLINUX which use a single file.

### 6b — Create the default config file

This file is used when no MAC-specific config matches a client.

```bash
vi /var/lib/tftpboot/pxelinux.cfg/default
```

Paste this content:

```
default menu.c32
prompt 0
timeout 1000
ontimeout local
menu title PXE Boot Menu

label local
  menu label Boot from local disk
  localboot 0xffff

label install-alma9
  menu label Install AlmaLinux 9
  kernel vmlinuz-alma9
  append initrd=initrd-alma9.img ip=dhcp inst.repo=ftp://192.168.56.106/pub/almalinux9

label install-centos7
  menu label Install CentOS 7.2
  kernel vmlinuz-centos7
  append initrd=initrd-centos7.img ip=dhcp inst.repo=ftp://192.168.56.106/pub/centos72
```

|Directive|Meaning|
|---|---|
|`default menu.c32`|Load the graphical boot menu|
|`prompt 0`|Don't show a prompt|
|`timeout 1000`|Auto-boot after 100 seconds|
|`ontimeout local`|Default to local disk on timeout|
|`localboot 0xffff`|Boot from local disk|
|`kernel`|Which kernel file to load from TFTP|
|`append`|Parameters passed to the kernel at boot|
|`inst.repo`|Where the installer gets its packages|

### 6c — Test the default menu 👀

Create a test VM in VirtualBox:

- Base memory: 2048 MB minimum (AlmaLinux 9 requires more than CentOS 7)
- No disk required for this test
- Network: Adapter 1 → Host-Only (same network as Server 1)
- Boot order: Network first (move above Hard Disk in System settings)

Boot the VM — you should see the boot menu with three options:

- Boot from local disk
- Install AlmaLinux 9
- Install CentOS 7.2

❌ did not have host-only
![[Pasted image 20260604081948.png]]

some what works
![[Pasted image 20260604082352.png]]

```
cp /usr/share/syslinux/*.c32 /var/lib/tftpboot/
```


---

## Step 7 — Add a MAC-Specific Config (optional)

If you want a specific machine to get a particular install option without selecting from the menu, create a config file named after its MAC address.

### 7a — Get the client VM's MAC address

In VirtualBox: VM Settings → Network → Advanced → copy the MAC address.

going to use the test vm

![[Pasted image 20260604084339.png|500]]

```
080027D4A5CB
```

### 7b — Create the MAC-specific file

```bash
cd /var/lib/tftpboot/pxelinux.cfg

# Replace with your client VM's actual MAC address
cp default 01-08-00-27-d4-a5-cb
```

> Filename rules: starts with `01-`, colons replaced by hyphens, all lowercase. Example: MAC `08:00:27:60:ef:5b` → filename `01-08-00-27-60-ef-5b`

```
[root@server1 pxelinux.cfg]# cat 01-08-00-27-d4-a5-cb
default menu.c32
prompt 0
timeout 1000
ontimeout local
menu title PXE Boot Menu

label local
  menu label Boot from local disk
  localboot 0xffff

label install-centos7
  menu label Install CentOS 7.2
  kernel vmlinuz-centos7
  append initrd=initrd-centos7.img ip=dhcp inst.repo=ftp://192.168.56.106/pub/centos72
[root@server1 pxelinux.cfg]#
```

worked the test machine now only sees centos option
![[Pasted image 20260604122246.png]]


<!--first got [[kernel panic]], it did not happen again-->

let the installtion continue until

![[Pasted image 20260604103048.png]]

### 7c — Verify the directory structure

```bash
tree /var/lib/tftpboot
```

Expected:

```
.
├── initrd-alma9.img
├── initrd-centos7.img
├── menu.c32
├── pxelinux.0
├── pxelinux.cfg
│   ├── 01-08-00-27-60-ef-5b
│   └── default
├── vmlinuz-alma9
└── vmlinuz-centos7
```

[[PXE Lab 1 - Configuring BIND DNS#Troubleshooting Error Downloading]]


did add all the .c32 just in case
```
[root@server1 pxelinux.cfg]# tree /var/lib/tftpboot/
/var/lib/tftpboot/
├── cat.c32
├── chain.c32
├── cmd.c32
├── cmenu.c32
├── config.c32
├── cptime.c32
├── cpu.c32
├── cpuid.c32
├── cpuidtest.c32
├── debug.c32
├── dhcp.c32
├── dir.c32
├── disk.c32
├── dmi.c32
├── dmitest.c32
├── elf.c32
├── ethersel.c32
├── gfxboot.c32
├── gpxecmd.c32
├── hdt.c32
├── hexdump.c32
├── host.c32
├── ifcpu.c32
├── ifcpu64.c32
├── ifmemdsk.c32
├── ifplop.c32
├── initrd-alma9.img
├── initrd-centos7.img
├── kbdmap.c32
├── kontron_wdt.c32
├── ldlinux.c32
├── lfs.c32
├── libcom32.c32
├── libgpl.c32
├── liblua.c32
├── libmenu.c32
├── libutil.c32
├── linux.c32
├── ls.c32
├── lua.c32
├── mboot.c32
├── meminfo.c32
├── menu.c32
├── pci.c32
├── pcitest.c32
├── pmload.c32
├── poweroff.c32
├── prdhcp.c32
├── pwd.c32
├── pxechn.c32
├── pxelinux.0
├── pxelinux.cfg
│   ├── 01-08-00-27-d4-a5-cb
│   └── default
├── reboot.c32
├── rosh.c32
├── sanboot.c32
├── sdi.c32
├── sysdump.c32
├── syslinux.c32
├── vesa.c32
├── vesainfo.c32
├── vesamenu.c32
├── vmlinuz-alma9
├── vmlinuz-centos7
├── vpdtest.c32
├── whichsys.c32
└── zzjson.c32

1 directory, 67 files
```


tried installing the CentOS7 in the test vm but it did not have enought space
deleted the controller: IDE adn put a bigger one

![[Pasted image 20260604095819.png]]

now im getting a [[kernel panic]]


going to recreate the vm with the same mac address in the NIC `080027D4A5CB`



---

## Step 8 — Set Up Kickstart for Automated CentOS 7.2 Install

> All commands in this step are run on **Server 1** (`192.168.56.106`).

A Kickstart file automates the entire installation — no clicking through the installer. The file is created on Server 1 and placed in the FTP directory so the client VM can fetch it during the PXE boot process.

### 8a — Create the Kickstart file


**Note:** CentOS 7 automatically generates an answer file at `/root/anaconda-ks.cfg` after every installation — it records all the choices made during that install. Rather than creating the Kickstart file from scratch, we can copy Server 1's own generated file as a base and modify it for the new machine. This is the recommended approach.

```bash
vi /var/ftp/pub/centos7.ks
```

Paste the following:

```
# Installation method
install
url --url="ftp://192.168.56.106/pub/centos72"

# UI mode
text
firstboot --disable

# Locale and keyboard
lang en_US.UTF-8
keyboard --vckeymap=us --xlayouts='us'
timezone America/New_York --utc

# Authentication
auth --enableshadow --passalgo=sha512

# Network
network --bootproto=dhcp --device=enp0s3 --ipv6=auto --activate
network --hostname=server3.example.vm

# Security
rootpw --plaintext changeme
selinux --enforcing

# Disk
ignoredisk --only-use=sda
clearpart --all --initlabel
autopart --type=lvm

# Bootloader
bootloader --location=mbr --boot-drive=sda

# Packages
%packages
@^minimal
@core
kexec-tools
bash-completion
%end

# Post install
%post
echo "Installation complete"
%end

# Reboot after install
reboot
```

### 8b — Set correct permissions

```bash
chmod 644 /var/ftp/pub/centos7.ks
```

Verify it is accessible via FTP:

bash

```bash
curl ftp://192.168.56.106/pub/centos7.ks
```

```
[root@server1 pxelinux.cfg]# curl ftp://192.168.56.106/pub/centos7.ks
# Installation method
install
url --url="ftp://192.168.56.106/pub/centos72"

# UI mode
text
firstboot --disable

# Locale and keyboard
lang en_US.UTF-8
keyboard --vckeymap=us --xlayouts='us'
timezone America/New_York --utc

# Authentication
auth --enableshadow --passalgo=sha512

# Network
network --bootproto=dhcp --device=enp0s3 --ipv6=auto --activate
network --hostname=server3.example.vm

# Security
rootpw --plaintext changeme
selinux --enforcing

# Disk
ignoredisk --only-use=sda
clearpart --all --initlabel
autopart --type=lvm

# Bootloader
bootloader --location=mbr --boot-drive=sda

# Packages
%packages
@^minimal
@core
kexec-tools
bash-completion
%end

# Post install
%post
echo "Installation complete"
%end

# Reboot after install
reboot
[root@server1 pxelinux.cfg]#

```
### 8c — Update the MAC-specific PXE config to use Kickstart


```bash
vi /var/lib/tftpboot/pxelinux.cfg/01-08-00-27-d4-a5-cb
```

```
default menu.c32
prompt 0
timeout 1000
ontimeout local
menu title PXE Boot Menu

label local
  menu label Boot from local disk
  localboot 0

label install-centos7
  menu label Install CentOS 7.2 (Auto)
  kernel vmlinuz-centos7
  append initrd=initrd-centos7.img ip=dhcp ks=ftp://192.168.56.106/pub/centos7.ks
```

swapping 
`append initrd=initrd-centos7.img ip=dhcp inst.repo=ftp://192.168.56.106/pub/centos72`

### 8d — Test the automated install


1. Boot the client VM (host-only network, network boot first)
2. Select **Install CentOS 7.2 (Auto)** from the menu
3. Press **Tab** — confirm `ks=ftp://...` is shown in the boot line
4. Press **Enter** — the system loads the kernel and initrd, fetches the Kickstart file, and installs without any prompts
5. VM reboots automatically when complete

sucess now instead of prompting me for 
![[Pasted image 20260604103048.png]]

it performs some post-installation steup tasks
![[Pasted image 20260604122535.png]]

and i get final product
![[Pasted image 20260604122711.png]]


---

## Troubleshooting

|Problem|Check|
|---|---|
|VM gets no IP|`systemctl status dhcpd` — confirm `next-server` and `filename` are in the subnet block|
|PXE menu doesn't appear|`ss -lun \| grep 69` — confirm TFTP is listening|
|"file not found" on boot|`ls /var/lib/tftpboot` — verify all kernel and initrd files exist|
|MAC config not loading|Check filename: `01-` prefix, hyphens not colons, all lowercase|
|Kickstart install fails|`curl ftp://192.168.56.106/pub/almalinux9.ks` — confirm file is accessible|
|Installer can't find packages|Confirm both `url` and `repo` lines in Kickstart point to correct FTP paths|
|DHCP errors on start|Normal — `enp0s3`/`enp0s9` subnet warnings can be ignored|
|TFTP blocked by firewall|`firewall-cmd --list-services` — confirm `tftp` is listed|

---

## Summary

|Step|What you did|
|---|---|
|1|Installed `tftp` and `tftp-server` (`syslinux` already done in Lab 3)|
|2|Copied kernels and initrds from both ISOs to `/var/lib/tftpboot/`|
|3|Added `next-server` + `filename` to existing DHCP config|
|4|Opened firewall port for TFTP|
|5|Started TFTP via `systemctl start tftp.socket`|
|6|Created `pxelinux.cfg/default` with menu entries for both images|
|7|Created MAC-specific config for a target machine|
|8|Built an AlmaLinux 9 Kickstart file and wired it into the PXE config|

**All services now running on Server 1:**

|Service|Port|Started by|
|---|---|---|
|DNS (named)|53|Lab 1|
|DHCP (dhcpd)|67/68|Lab 2|
|FTP (vsftpd)|21|Lab 3|
|TFTP (tftp.socket)|69|This lab|

**Key files reference:**

|File|Purpose|
|---|---|
|`/var/lib/tftpboot/pxelinux.0`|PXE bootloader downloaded by client|
|`/var/lib/tftpboot/pxelinux.cfg/default`|Default boot menu for all clients|
|`/var/lib/tftpboot/pxelinux.cfg/01-*`|Per-machine boot config (MAC-based)|
|`/var/ftp/pub/almalinux9.ks`|Kickstart file for automated AlmaLinux 9 install|
|`/var/ftp/pub/almalinux9/isolinux/`|AlmaLinux 9 kernel and initrd source|
|`/var/ftp/pub/centos72/isolinux/`|CentOS 7.2 kernel and initrd source|
%%
---
tags:
  - LDAP
  - Linux
  - security
  - CentOS
  - Security
linkedin: "False"
quartz: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
title: OpenLDAP Server Setup and LDIF Management on CentOS 7
---

%%[[LDAP Configuring an OpenLDA[[OpenLDAP Server Setup and LDIF Management on CentOS 7]]e LDIF File]] %%

## Overview

Authentication and authorization are foundational pillars of any data center environment. When multiple systems need a centralized, standards-based way to manage user identities and access permissions, a **directory service** becomes essential.

**OpenLDAP** is a widely adopted open-source implementation of the Lightweight Directory Access Protocol (LDAP). It gives organizations a flexible, cost-free alternative to commercial directory services while remaining fully compatible with the LDAP standard used across Linux environments, network devices, and enterprise applications.

This article covers two hands-on labs completed end-to-end in a local VirtualBox environment running CentOS 7:

- **Part 1** walks through installing the OpenLDAP server daemon (`slapd`), securing it with TLS, loading the required attribute schemas, and building an organizational directory tree with user accounts.
- **Part 2** builds directly on that server to write and apply custom LDIF files, adding a new Organizational Unit and a new user entry to the live directory.

The two labs are documented together because they run on the same machine and share state. A silent misconfiguration introduced during the TLS setup in Part 1 only surfaces as a startup failure at the beginning of Part 2, a real-world troubleshooting scenario worth preserving.

---

## Environment

|Component|Detail|
|---|---|
|Platform|VirtualBox (local lab)|
|OS|CentOS 7 (bridge adapter)|
|IP Address|`10.0.0.211`|
|LDAP Software|OpenLDAP (`slapd` v2.4.44)|
|Domain|`dadcorp.com`|
|Admin DN|`cn=ldapadm,dc=dadcorp,dc=com`|
|Admin Password|`1234` (lab use only)|

> **Note on the environment:** This lab was originally based on a guided exercise from Pluralsight. Because the hosted lab environment went offline, I replicated the entire setup locally in VirtualBox. All steps and outputs in this article reflect that local environment.

---

## Part 1 - Installing and Configuring OpenLDAP

---

### Step 1 - Fix the CentOS 7 Package Mirrors

![[Pasted image 20260530193221.png]]

Before installing anything, I needed to get the VM online. The CentOS 7 network interface wasn't coming up automatically in bridge adapter mode. I brought it up manually and made the change persistent:

```bash
sudo ip link set enp0s3 up
sudo dhclient enp0s3
```

^08deda

To make this survive reboots, I edited the interface config file:

```bash
sudo vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
```

Setting `ONBOOT=yes` ensures the interface activates at boot without manual intervention. Then I restarted the network service to apply it:

```bash
sudo systemctl restart network
```

With the network up, the next problem was that the **official CentOS 7 mirrors are no longer active**, the project reached end-of-life and the mirror infrastructure was decommissioned. Any `yum` install attempt will fail unless you redirect it to the Vault archive, which hosts frozen copies of all CentOS releases.

The following three commands update every `.repo` file in place: they swap the dead mirror URLs for the vault, uncomment the `baseurl` lines, and comment out the `mirrorlist` lines (which no longer resolve):
<!--
why does the end of life centos is pointing to death mirrors why not point to the archive mirrors, i would think that everything becomes end of life including the last poiting mirror, wountl the lastpoiting mirror just become archived
-->
```bash
sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/*.repo
sed -i 's/^#baseurl/baseurl/g' /etc/yum.repos.d/*.repo
sed -i 's/^mirrorlist/#mirrorlist/g' /etc/yum.repos.d/*.repo
```

This is a required prerequisite for any CentOS 7 VirtualBox lab, without it, `yum` cannot reach any packages.

---

### Step 2 - Install OpenLDAP Packages

With the mirrors fixed, I installed the **four packages** that make up a functional OpenLDAP server:

```bash
sudo yum install compat-openldap openldap-clients openldap-servers nss-pam-ldapd -y
```

| Package            | Purpose                                                                                                                                |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `openldap-servers` | The `slapd` daemon, the actual LDAP server                                                                                             |
| `openldap-clients` | CLI tools: `ldapadd`, `ldapmodify`, `ldapsearch`                                                                                       |
| `compat-openldap`  | Backward-compatibility libraries for older LDAP-linked applications                                                                    |
| `nss-pam-ldapd`    | Plugs LDAP into the OS authentication stack via [[Name Service Switch (NSS)\|NSS]] and [[PAM (Pluggable Authentication Modules)\|PAM]] |
<!--
perfect to connect the does with NSS and PAM
-->
After installation I started the service and enabled it so it starts automatically on every boot:

```bash
systemctl start slapd
systemctl enable slapd
```

**Expected output from `systemctl enable`:**

```
Created symlink from /etc/systemd/system/multi-user.target.wants/slapd.service
to /usr/lib/systemd/system/slapd.service.
```

The symlink is how systemd registers a service to start at boot. Seeing it created confirms `slapd` is now part of the system's startup sequence.

---

#### Troubleshooting - `slapd` Fails to Start (TLS Permission Denied)

`slapd` failed immediately on first start:

```
[root@localhost ~]# systemctl start slapd
Job for slapd.service failed because the control process exited with error code.
See "systemctl status slapd.service" and "journalctl -xe" for details.

[root@localhost ~]# systemctl status slapd
🔴 slapd.service - OpenLDAP Server Daemon
   Active: failed (Result: exit-code)

May 30 20:29:15 localhost.localdomain slapd[1995]: tlsmc_cert_create_hash_symlink:
ERROR: OS error: Permission denied
May 30 20:29:15 localhost.localdomain slapd[1995]: main: TLS init def ctx failed: -1
May 30 20:29:15 localhost.localdomain slapd[1995]: slapd stopped.
```

```
❌ tlsmc_cert_create_hash_symlink: ERROR: OS error: Permission denied
```

This error means the `ldap` user (which `slapd` runs as) cannot access the certificates directory. My first attempt was to fix ownership directly:

```bash
chown -R ldap:ldap /etc/openldap/certs/
chmod 700 /etc/openldap/certs/
```

This did not resolve it. The underlying cause was **SELinux**, even with correct file ownership, SELinux was blocking `slapd` from accessing the certs path under its default enforcing policy.

I confirmed this by temporarily disabling SELinux enforcement:

```bash
setenforce 0
systemctl start slapd
```

`slapd` started successfully. For the purposes of this lab I left SELinux in permissive mode to proceed, a deeper look at SELinux policy configuration is outside the scope of this exercise and will be covered in dedicated SELinux hand-ons.

<!--
`slapd` started successfully, confirming SELinux was the blocker. The correct fix is to restore the SELinux security context on the certs directory and enable the relevant boolean, then re-enable enforcement:

```bash
setsebool -P allow_ypbind 1
restorecon -R /etc/openldap/certs/
setenforce 1
systemctl restart slapd
```

In my lab environment, `slapd` still failed to start after applying the SELinux fix, likely due to a pre-existing stale config in the certs directory. For the purposes of this lab I left SELinux in permissive mode (`setenforce 0`) to proceed. In a production environment this would need to be properly resolved before going live.

-->

---

### Step 3 - Initialize the LDAP Database

OpenLDAP's modern configuration model, called **OLC (Online LDAP Configuration)** or `cn=config`, stores the server's configuration as **LDIF (LDAP Data Interchange Format)** files on disk under `/etc/openldap/slapd.d/`. When the service is running, you modify that configuration by sending LDIF-formatted changes through `ldapmodify`, you are essentially using the LDAP protocol to configure the server that is running it. When the service is down (as we saw at the start of Part 2), those same files can be edited directly on disk.

LDIF is also the format used to manage **directory data**, the OUs, users, and other entries that live in the directory itself. These are two distinct things that happen to share the same format: server configuration lives under `cn=config` and is modified with root OS credentials (`-Y EXTERNAL`), while directory data lives under your domain suffix (`dc=dadcorp,dc=com`) and is modified with the LDAP admin credentials (`-x -D cn=ldapadm...`).
<!--
The files on disk under `/etc/openldap/slapd.d/` are just how that `cn=config` tree is persisted to disk.
-->
I have a set of LDIF files in `~/ldifs/` that define the initial configuration and directory structure for the `dadcorp.com` domain:

```bash
cd ldifs
ls
```

```
base.ldif  dadcorp.key  dadcorp.pem  dbinit.ldif  key.ldif  monitor.ldif  pem.ldif  users.ldif
```

> [!NOTE]- base.ldif
> ```
> [root@localhost ldifs]# cat base.ldif
> dn: dc=dadcorp,dc=com
> dc: dadcorp
> objectClass: top
> objectClass: domain
> 
> dn: cn=ldapadm,dc=dadcorp,dc=com
> objectClass: organizationalRole
> cn: ldapadm
> description: LDAP Manager
> 
> dn: ou=Sales,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: Sales
> 
> dn: ou=People,ou=Sales,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: People
> 
> dn: ou=Engineering,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: Engineering
> 
> dn: ou=People,ou=Engineering,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: People
> 
> dn: ou=Printers,ou=Engineering,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: Printers
> 
> dn: ou=Admin,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: Admin
> 
> dn: ou=People,ou=Admin,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: People
> 
> dn: ou=Printers,ou=Admin,dc=dadcorp,dc=com
> objectClass: organizationalUnit
> ou: Printers
> ```

> [!NOTE]- key.ldif
> ```
> [root@localhost ldifs]# cat key.ldif
> dn: cn=config
> changetype: modify
> replace: olcTLSCertificateKeyFile
> olcTLSCertificateKeyFile: /etc/openldap/certs/dadcorp.key
> ```

> [!NOTE]- monitor.ldif
> ```
> [root@localhost ldifs]# cat monitor.ldif
> dn: olcDatabase={1}monitor,cn=config
> changetype: modify
> replace: olcAccess
> olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read by dn.base="cn=ldapadm,dc=dadcorp,dc=com" read by * none
> ```

> [!NOTE]- pem.ldif
> ```
> [root@localhost ldifs]# cat pem.ldif
> dn: cn=config
> changetype: modify
> replace: olcTLSCertificateFile
> olcTLSCertificateFile: /etc/openldap/certs/dadcorp.pem
> ```

> [!NOTE]- users.ldif
> ```
> [root@localhost ldifs]# cat users.ldif
> dn: uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com
> objectClass: account
> objectClass: posixAccount
> objectClass: shadowAccount
> objectClass: top
> cn: Tony Stark
> uidNumber: 5000
> gidNumber: 5000
> userPassword: {crypt}$6$yi5/qY.s$odjLQeKpb8wvS8vSe4OsoqEOO7Uxime9HBrswd6/xBOgVmviSlK8HPIr8zWCIxx2SuCRzVNdp3ssh7wMTbENf/
> gecos: Tony Stark
> loginShell: /bin/bash
> homeDirectory: /home/ldap/tstark
> shadowExpire: -1
> shadowWarning: 7
> shadowMin: 0
> shadowMax: 99999
> 
> dn: uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com
> objectClass: account
> objectClass: posixAccount
> objectClass: shadowAccount
> objectClass: top
> cn: Bruce Banner
> uidNumber: 5001
> gidNumber: 5001
> userPassword: {crypt}!!
> gecos: Bruce Banner
> loginShell: /bin/bash
> homeDirectory: /home/ldap/bbanner
> shadowExpire: -1
> shadowWarning: 7
> shadowMin: 0
> shadowMax: 99999
> 
> dn: uid=srogers,ou=People,ou=Admin,dc=dadcorp,dc=com
> objectClass: account
> objectClass: posixAccount
> objectClass: shadowAccount
> objectClass: top
> cn: Steve Rogers
> uidNumber: 5002
> gidNumber: 5002
> userPassword: {crypt}!!
> gecos: Steve Rogers
> loginShell: /bin/bash
> homeDirectory: /home/ldap/srogers
> shadowExpire: -1
> shadowWarning: 7
> shadowMin: 0
> shadowMax: 99999
> 
> dn: uid=labprinter,ou=Printers,ou=Engineering,dc=dadcorp,dc=com
> objectClass: account
> objectClass: posixAccount
> objectClass: shadowAccount
> objectClass: top
> cn: Lab Printer
> uidNumber: 5003
> gidNumber: 5003
> userPassword: {crypt}$6$4WVSHR5l8yXNNr$NgnnPfOTXPr0Za3mZyrX.8xlyzv9OqVhzK5FX.42dHHt6KPp26VW84xBdpozepQ7t7JlRq541rytfp2iXfZPL1
> gecos: Lab Printer
> loginShell: /bin/false
> homeDirectory: /tmp
> shadowExpire: -1
> shadowWarning: 7
> shadowMin: 0
> shadowMax: 99999
> 
> dn: uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com
> objectClass: account
> objectClass: posixAccount
> objectClass: shadowAccount
> objectClass: top
> cn: Front Desk Printer
> uidNumber: 5004
> gidNumber: 5004
> userPassword: {crypt}$6$czWl0/JLZqK/np$me/VQQvKomnPwurgsJwN/wwix7.pwPr0Kg4WP5JaW9sFel96OYPddAGz5CltNmdpZjp5iyMICuCuvLjYHDYyB.
> gecos: Front Desk Printer
> loginShell: /bin/false
> homeDirectory: /tmp
> shadowExpire: -1
> shadowWarning: 7
> ```

The `dbinit.ldif` file configures three critical database parameters:

| Parameter   | What it sets                                                        |
| ----------- | ------------------------------------------------------------------- |
| `olcSuffix` | The root DN of the **directory tree**, the domain being served      |
| `olcRootDN` | The administrator's Distinguished Name (the **LDAP admin account**) |
| `olcRootPW` | The administrator's hashed **password**                             |
|             |                                                                     |
> The `olcRootDN` and `olcRootPW` values set here do not create a real directory entry, they tell the database engine to treat `cn=ldapadm,dc=dadcorp,dc=com` as a built-in superuser that bypasses access controls. This DN can authenticate and operate on the directory without needing to exist as an actual entry yet. Think of it as a master key defined at the engine level. *(Used in [[#Step 5 - Load Required Schemas and Build the Directory Structure|Step 5]])*

```bash
cat dbinit.ldif
```

> [!NOTE]- dbinit.ldif
> ```
> dn: olcDatabase={2}hdb,cn=config
> changetype: modify
> replace: olcSuffix
> olcSuffix: dc=dadcorp,dc=com
> 
> dn: olcDatabase={2}hdb,cn=config
> changetype: modify
> replace: olcRootDN
> olcRootDN: cn=ldapadm,dc=dadcorp,dc=com
> 
> dn: olcDatabase={2}hdb,cn=config
> changetype: modify
> replace: olcRootPW
> olcRootPW: {SSHA}Hh1Dk9Zcq2TOm7WxJrDzbBXwI2HilWXs
> ```

The placeholder `{SSHA}changeme` must be replaced with a real hashed password. I generated one using `slappasswd`, which is the built-in OpenLDAP password hashing utility:

```bash
slappasswd -s 1234 -n
```

```
{SSHA}Hh1Dk9Zcq2TOm7WxJrDzbBXwI2HilWXs
```

> **Tip:** Notice that the hash output runs right up against the shell prompt (`[root@server ldifs]#`) with no newline between them. When copying, it is easy to accidentally grab the trailing `[` from the next prompt line. That extra character will corrupt the hash, this becomes a real issue in the next step.

I opened `dbinit.ldif` in `vim`, replaced `{SSHA}changeme` with the generated hash, and saved. Then I applied both the **database config** and the **monitor config**:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif
ldapmodify -Y EXTERNAL -H ldapi:/// -f monitor.ldif
```

The `-Y EXTERNAL` flag authenticates using the OS-level identity of the running process (root), which is the required method for modifying `slapd`'s own configuration. The `-H ldapi:///` flag connects over the local Unix socket rather than a TCP port.
dont 
<!--
**`-H ldapi:///`** — this one first because it sets the context.

This tells the `ldap` client to connect through a **Unix socket** (a local file-based connection) instead of going over the network. The `ldapi` scheme means "LDAP over IPC (Inter-Process Communication)". No port, no network — the client and server are just talking through a file on the same machine.

**`-Y EXTERNAL`** — this is the authentication mechanism.

When you connect over that Unix socket, the kernel itself knows who opened the connection. EXTERNAL tells `slapd` to ask the OS "who is this?" instead of asking for a username and password. Since you're running the command as root, the OS reports back `uid=0, gid=0` — and `slapd` trusts that.

"I'm connecting locally through a Unix socket, and instead of a password I'm telling the server: just ask the OS who I am."
-----------------------------------------------------------------
ldapmodify is like client side thing conneciton to ldap server to send LDIF file defintions and we are using the scockt as the endpoint of the server because we are in teh same machine
-----------------------------------------------------------------
we doing this because our client side ldapmodify is in the same machine as the server

If you were managing an LDAP server remotely, `ldapmodify` would run on your local machine and connect over the network to the server:

bash

```bash
ldapmodify -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 \
  -H ldap://10.0.0.211:389 -f changes.ldif
```

- **`-H ldap://10.0.0.211:389`** — connect to the remote server by IP over the standard LDAP port
- **`-x -D ... -w ...`** — now you need simple bind with credentials because the OS socket trick doesn't work across machines

In that scenario `-Y EXTERNAL` isn't an option at all — the kernel on your machine can't vouch for you to a server on a different machine. You have to prove identity the traditional way with a DN and password.
-->

**Output for `dbinit.ldif`:**

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"
```

**Output for `monitor.ldif`:**

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}monitor,cn=config"
```

Three entries modified for the database, one for monitor, matching the three stanzas in `dbinit.ldif` and the single stanza in `monitor.ldif`.

---

### Step 4 - Enable TLS

To secure client-server communications, I configured TLS using a pre-generated certificate (`dadcorp.pem`) and private key (`dadcorp.key`). The `pem.ldif` and `key.ldif` files point `slapd` to those files:
<!--
**`.pem` is the certificate (public), `.key` is the private key.**

|File|What it is|
|---|---|
|`dadcorp.pem`|The **certificate** — contains the public key and the server's identity, signed by a CA|
|`dadcorp.key`|The **private key** — stays on the server, never shared|

Here's how they work together during a TLS handshake:

1. A client connects to the LDAP server
2. The server sends `dadcorp.pem` to the client — _"here's who I am"_
3. The client verifies the certificate (checks the signature, the domain name, expiry, etc.)
4. The client uses the public key inside that certificate to encrypt a session key
5. The server uses `dadcorp.key` to decrypt it — **only the server can do this** because only it has the private key
6. Both sides now share a secret session key and the encrypted channel is established

The private key is what proves the server actually owns that certificate. Anyone can have a copy of `dadcorp.pem` — but only the real server can decrypt what gets encrypted with the public key inside it.

This is also why the permissions mattered:

- `dadcorp.pem` — `644`, world readable is fine, it's meant to be shared
- `dadcorp.key` — `640`, only owner and the `ldap` group can read it, never world readable
  
--------------
**The client has a list of trusted CAs already installed on the OS** — things like DigiCert, Let's Encrypt, Comodo, etc. When it receives `dadcorp.pem` it checks: _"was this signed by someone I already trust?"_

But in this lab, `dadcorp.pem` is self-signed.`
we need to set the client to accept self-sgiend certificates

Looking back at both lab notes — **no, we didn't explicitly set `TLS_REQCERT never`** in `/etc/openldap/ldap.conf`.

in our case we dont have that set up so soemthing to look into

[root@localhost ldifs]# cat /etc/openldap/ldap.conf
#
# LDAP Defaults
#

# See ldap.conf(5) for details
# This file should be world readable but not world writable.

#BASE   dc=example,dc=com
#URI    ldap://ldap.example.com ldap://ldap-master.example.com:666

#SIZELIMIT      12
#TIMELIMIT      15
#DEREF          never

TLS_CACERTDIR   /etc/openldap/certs

# Turning this off breaks GSSAPI used with krb5 when rdns = false
SASL_NOCANON    on
[root@localhost ldifs]#
-->
```bash
cat pem.ldif
```

```ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/dadcorp.pem
```

```bash
cat key.ldif
```

```ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/dadcorp.key
```

Both files reference paths under `/etc/openldap/certs/`, so I copied the cert and key there first:

```bash
cp dadcorp.* /etc/openldap/certs/
```

Then applied the TLS configuration:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f key.ldif
ldapmodify -Y EXTERNAL -H ldapi:/// -f pem.ldif
```

**Output:**

```
# key.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"
ldap_modify: Other (e.g., implementation specific) error (80) ❌

# pem.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"
```

> [!attention]
> `key.ldif` returned ❌error `(80)` - `LDAP_OTHER`, a generic catch-all OpenLDAP uses when the failure doesn't fit a more specific code. The root cause was not confirmed in this lab. What matters is the consequence: the modify **did not go through**, meaning `olcTLSCertificateKeyFile` was never updated and stayed at its default placeholder value `/etc/openldap/certs/password`.
> 
> After applying any TLS-related LDIF, always verify the change actually landed with:
> ```
> grep -i TLS /etc/openldap/slapd.d/cn\=config.ldif
> ```
> 
> ```
> olcTLSCACertificatePath: /etc/openldap/certs
olcTLSCertificateKeyFile: /etc/openldap/certs/password ❌
olcTLSCertificateFile: /etc/openldap/certs/dadcorp.pem
> ```
> 

> [!tip]
> This silent misconfiguration is not visible until the next cold start, which is exactly what happens at the beginning of Part 2, where `slapd` crashes on reboot and we fix it by editing the config file directly with `vim`. Direct editing of files under `/etc/openldap/slapd.d/` is discouraged since OpenLDAP manages those files, but I resorted to this bypass for time's sake.
> 

<!--
**`grep`** — searches for a pattern inside a file and prints matching lines

**`-i`** — case-insensitive, so it matches `TLS`, `tls`, `Tls`, etc.

**`TLS`** — the pattern being searched for

**`/etc/openldap/slapd.d/cn\=config.ldif`** — the file being searched. This is the live config file where `slapd` stores its runtime settings in LDIF format. The `\=` is just escaping the `=` sign so the shell doesn't misinterpret it.

**So in plain English:** "show me every line in `slapd`'s config file that contains the word TLS"
-->

A successful `slaptest -u` will not catch this, it validates syntax, not whether the configured file paths are correct.

```bash
slaptest -u
```

```
config file testing succeeded
```


---

### Step 5 - Load Required Schemas and Build the Directory Structure

LDAP **schemas** define what object classes and attributes are valid in the directory *(what they're named, what data type they hold, whether they're required or optional)*. Before adding user entries with Linux account attributes, three standard schemas must be loaded:

|Schema|Purpose|
|---|---|
|`cosine`|Core organizational attributes (common names, addresses, etc.)|
|`nis`|POSIX account attributes (`uidNumber`, `gidNumber`, `loginShell`, etc.)|
|`inetorgperson`|Internet-era person attributes (email, photo, etc.)|
<!--
when you say object class it mean we create objects of type that object class?
_____________________________________________

**Schemas** are the rulebooks. They _define_ object classes and attributes — what they're named, what data type they hold, whether they're required or optional.

**Object classes** and **attributes** are what the schemas define. So:

- `cosine` is a **schema** — it defines things like the `organizationalRole` object class and attributes like `description`
- `nis` is a **schema** — it defines the `posixAccount` and `shadowAccount` object classes and attributes like `uidNumber`, `gidNumber`, `loginShell`
- `inetorgperson` is a **schema** — it defines the `inetOrgPerson` object class and attributes like `mail`, `givenName`
-->
```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

**Output (same pattern for each):**

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=cosine,cn=schema,cn=config"
```

---

In LDAP, a directory isn't a flat list of users, it's a tree. Before you can add any users, you need to create the containers they live in. Those containers are called **[[Organizational Unit (OU)|Organizational Units (OUs)]]**.
<!--
Think of it like this: before you can create the file `bbanner.txt` inside `Engineering/People/`, the folders `Engineering` and `People` inside it need to exist first.
-->

`base.ldif` builds that structure for `dadcorp.com` from scratch. It also creates the root domain entry (`dc=dadcorp,dc=com`) itself and the admin account (`cn=ldapadm`), because in LDAP even the top of the tree is an entry that has to be explicitly created. None of this is user data yet, it's purely the skeleton of the directory.

```
dadcorp.com
├── cn=ldapadm
├── Sales
│   └── People
├── Engineering
│   ├── People
│   └── Printers
└── Admin
    ├── People
    └── Printers
```

Only after this structure exists can users be added into their respective places. This command authenticates as ldapadm using **simple bind** (`-x`) rather than `-Y EXTERNAL`, because `base.ldif` writes to the directory data tree, not the server config. Note that ldapadm doesn't exist as a real directory entry yet at this point, it was defined as a **virtual root DN** in `dbinit.ldif` ([[#Step 3 - Initialize the LDAP Database|Step 3]])., and `base.ldif` is what gives it an actual entry in the tree.
<!--
so in this lab we dont seem to use the actual ldapadm in the directory because we never assign it a password we keep using the virtual root DN
-->
```bash
ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f base.ldif
```
<!--
**Bind** is just LDAP's term for authentication — it's how a client identifies itself to the server before performing operations.

**Simple bind** specifically means: authenticate by sending a DN and a password in plaintext over the connection. That's it. The `-x` flag in the command tells `ldapmodify`/`ldapadd` to use simple bind, and `-D` and `-w` provide the DN and password respectively:

bash

```bash
ldapadd -x          # use simple bind
        -w 1234     # password
        -D cn=ldapadm,dc=dadcorp,dc=com   # who you're authenticating as
```

It's called "simple" to distinguish it from the other LDAP authentication mechanism — **SASL (Simple Authentication and Security Layer)**, which is what `-Y EXTERNAL` uses.
-->
---

#### Troubleshooting - `ldap_bind: Invalid credentials (49)`

This command failed twice with the same error:

```
❌ ldap_bind: Invalid credentials (49)
```

![[Pasted image 20260511124750.png]] 
_Screenshot: Invalid credentials error in terminal_

**Root cause:** When copying the `slappasswd` output in Step 3, I accidentally captured the trailing `[` character from the next shell prompt. The stored password hash in `dbinit.ldif` was malformed, so every bind attempt with password `1234` failed to match.

**Fix:** I re-applied `dbinit.ldif` with the correctly copied hash to overwrite the bad entry:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif
```

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"
```

After re-applying the correct hash, the `ldapadd` succeeded:

```
adding new entry "dc=dadcorp,dc=com"
adding new entry "cn=ldapadm,dc=dadcorp,dc=com"
adding new entry "ou=Sales,dc=dadcorp,dc=com"
adding new entry "ou=People,ou=Sales,dc=dadcorp,dc=com"
adding new entry "ou=Engineering,dc=dadcorp,dc=com"
adding new entry "ou=People,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "ou=Printers,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "ou=Admin,dc=dadcorp,dc=com"
adding new entry "ou=People,ou=Admin,dc=dadcorp,dc=com"
adding new entry "ou=Printers,ou=Admin,dc=dadcorp,dc=com"
```

This establishes the full organizational hierarchy for `dadcorp.com`: three top-level departments (Sales, Engineering, Admin), each with a `People` OU, and two of them with a `Printers` OU.

```
dadcorp.com
├── cn=ldapadm
├── Sales
│   └── People
├── Engineering
│   ├── People
│   └── Printers
└── Admin
    ├── People
    └── Printers
```

---

### Step 6 - Add Users and Verify

With the directory structure in place, I added the user accounts defined in `users.ldif`:

```bash
ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f users.ldif
```
<!--
By Step 6, `base.ldif` has already run so `cn=ldapadm,dc=dadcorp,dc=com` exists as a real entry in the directory. However the authentication is **still going through the virtual root DN mechanism** — not the real entry.

Here's why: when OpenLDAP receives a bind request for `cn=ldapadm,dc=dadcorp,dc=com`, it checks `olcRootDN`/`olcRootPW` from the engine config first. That's the password `1234` we set in `dbinit.ldif`. The real directory entry created by `base.ldif` is an `organizationalRole` object — it never had a `userPassword` attribute set on it.

in this lab we dont use the real ldapadm at all, because doesnt have password? seems correct
-->
**Output:**

```
adding new entry "uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com"
adding new entry "uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "uid=srogers,ou=People,ou=Admin,dc=dadcorp,dc=com"
adding new entry "uid=labprinter,ou=Printers,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com"
```

Three user accounts and two printer service accounts were added to their respective OUs.

To confirm the directory was correctly configured and queryable, I searched for a specific user using `ldapsearch`:

```bash
ldapsearch -h localhost -D cn=ldapadm,dc=dadcorp,dc=com -w 1234 \
  -b dc=dadcorp,dc=com "uid=bbanner"
```

The `-b` flag sets the **search base**, the point in the directory tree to start searching from. The final argument `"uid=bbanner"` is the **search filter**, narrowing results to entries where the `uid` attribute equals `bbanner`.

**Output:**

```
# bbanner, People, Engineering, dadcorp.com
dn: uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Bruce Banner
uidNumber: 5001
gidNumber: 5001
homeDirectory: /home/ldap/bbanner
loginShell: /bin/bash
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: bbanner

# numEntries: 1
```

The entry returned with the correct OU path (`ou=People,ou=Engineering`) and all expected attributes, confirming the directory is correctly set up and accepting queries.

---

## Part 2 - Writing and Applying LDIF Files

---

#### Troubleshooting - `slapd` Fails to Start After Reboot (`TLS init def ctx failed: -1`)

After taking a snapshot and powering the VM back on to begin Part 2, `slapd` failed to start:

```
[root@localhost ~]# systemctl status slapd
🔴 slapd.service - OpenLDAP Server Daemon
   Active: failed (Result: exit-code)

May 30 22:04:05 localhost.localdomain slapd[1751]: main: TLS init def ctx failed: -1
May 30 22:04:05 localhost.localdomain slapd[1751]: slapd stopped.
```

**Root cause:** This is the deferred consequence of the `key.ldif` error `(80)` from [[#Step 4 - Enable TLS|Step 4 - Enable TLS]]. During Part 1, `slapd` was already running when TLS was configured and was never restarted, so it kept using its in-memory state. On a cold start, it tried to open the TLS key file from its saved config, and that path was wrong.

I verified this by inspecting the active config directly:

```bash
grep -i TLS /etc/openldap/slapd.d/cn\=config.ldif
```

```
olcTLSCACertificatePath: /etc/openldap/certs
olcTLSCertificateKeyFile: /etc/openldap/certs/password
olcTLSCertificateFile: /etc/openldap/certs/dadcorp.pem
```

`olcTLSCertificateKeyFile` was still pointing to `/etc/openldap/certs/password`, the default placeholder that was never overwritten because the `key.ldif` apply failed.

Since `slapd` was down I couldn't use `ldapmodify`, so I edited the config file directly:

```bash
vim /etc/openldap/slapd.d/cn\=config.ldif
```

I also fixed the file ownership so the `ldap` user could read the key:

```bash
chown root:ldap /etc/openldap/certs/dadcorp.key
chown root:ldap /etc/openldap/certs/dadcorp.pem
chmod 640 /etc/openldap/certs/dadcorp.key
chmod 644 /etc/openldap/certs/dadcorp.pem
```

The key gets `640` (owner read/write, group read), the `ldap` user can read it through group membership, but it is never world-readable. After saving the corrected path:

```bash
systemctl start slapd
systemctl status slapd
```

```
🟢 Active: active (running)
```

> **Lesson learned:** `slaptest -u` validates config file syntax, not whether the file paths it references actually exist. A failed `ldapmodify` that returns error `(80)` can leave a stale value in the config that appears fine until the next cold start. Always verify the actual config values after a TLS-related change.

---

### Step 7 - Inspect the Existing Directory

Before writing any new LDIF files it is useful to query the full directory tree to understand its current state. This also gives you a real entry to use as a template.

```bash
ldapsearch -x -h localhost -b dc=dadcorp,dc=com
```

The `-x` flag uses simple (unauthenticated) bind, since no password is passed here, this is an anonymous search. The server returns only publicly readable attributes.

**Output (truncated to show structure):**

```
# dadcorp.com
dn: dc=dadcorp,dc=com
objectClass: top
objectClass: domain

# Sales, dadcorp.com
dn: ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit

# People, Sales, dadcorp.com
dn: ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit

# Engineering, dadcorp.com
dn: ou=Engineering,dc=dadcorp,dc=com
objectClass: organizationalUnit

# People, Engineering, dadcorp.com
dn: ou=People,ou=Engineering,dc=dadcorp,dc=com

# Printers, Engineering, dadcorp.com
dn: ou=Printers,ou=Engineering,dc=dadcorp,dc=com

# Admin, dadcorp.com
dn: ou=Admin,dc=dadcorp,dc=com

# tstark, People, Sales, dadcorp.com
dn: uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
...

# fd_printer, Printers, Admin, dadcorp.com
dn: uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
...

# numEntries: 16
```

The task for Part 2 is to:

1. Add a new `ou=Printers` under `ou=Sales` (it currently doesn't exist there)
2. Add a new user `sales_printer` inside that new OU

---

### Step 8 - Create the `Printers` OU LDIF Under Sales

I created `ou.ldif` to define the new Organizational Unit:

```bash
vi ou.ldif
cat ou.ldif
```

```ldif
dn: ou=Printers,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: printers
```

The `dn` line is the Distinguished Name, the full unique path of this entry in the directory tree. Every entry must have a DN. `objectClass: organizationalUnit` tells the server what kind of entry this is and which attributes are valid for it.

Applied it with `ldapmodify -a`, where the `-a` flag means **add** (equivalent to `ldapadd`):

```bash
ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f ou.ldif
```

```
adding new entry "ou=Printers,ou=Sales,dc=dadcorp,dc=com"
```

---

#### Troubleshooting - `ldap_add: No such object (32)`

My first attempt to add the printer user failed before I had the OU correctly named:

```
[root@localhost ~]# ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 \
  -H ldapi:/// -f printer.ldif
adding new entry "uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com"
❌ldap_add: No such object (32)
        matched DN: ou=Sales,dc=dadcorp,dc=com
```

The error `No such object (32)` means the parent container the new entry tries to live in doesn't exist. The error message helpfully shows the last DN that _did_ match, `ou=Sales,dc=dadcorp,dc=com`, confirming the `ou=Printers` child was missing.

I searched the Sales branch to see what was there:

```bash
ldapsearch -x -h localhost -b ou=Sales,dc=dadcorp,dc=com
```

```
# prints, Sales, dadcorp.com
dn: ou=prints,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: printers
ou: prints
```

The OU existed but had been created with the name `ou=prints` instead of `ou=Printers`. I deleted the wrong one and recreated it correctly:

```bash
ldapdelete -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 \
  -H ldapi:/// "ou=prints,ou=Sales,dc=dadcorp,dc=com"
```

Then applied the corrected `ou.ldif` shown above.

---

### Step 9 - Create the `sales_printer` User Entry

To build the new user LDIF, I used the `fd_printer` entry from the directory as a template since it is the same type of object (a printer service account). I created `printer.ldif` and edited it with the new values:
<!--[[#Step 7 - Inspect the Existing Directory|Step 7]]
Good catch — "template" is misleading because it implies a separate file. What actually happened was:

1. We ran `ldapsearch -x -h localhost -b dc=dadcorp,dc=com` in Step 7 to dump the whole directory
2. We found the `fd_printer` entry in that output
3. We **copied that entry directly from the terminal output** into a new `printer.ldif` file
4. Then edited the values to match `sales_printer`
-->
```bash
vim printer.ldif
```

**Final `printer.ldif`:**

```ldif
dn: uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Sales Desk Printer
uidNumber: 6001
gidNumber: 6001
gecos: Sales Desk Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
```

Key changes from the template entry:

- `dn` updated to place it under `ou=Printers,ou=Sales`
- `cn`, `gecos`, and `uid` set to `sales_printer` / `Sales Desk Printer`
- `uidNumber` and `gidNumber` set to `6001`
- `userPassword` line removed entirely (to be set in a later step)
- `loginShell: /bin/false` - printer service accounts should not have an interactive shell

Applied it:

```bash
ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f printer.ldif
```

```
adding new entry "uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com"
```

---

### Step 10 - Verify the New Entry

I queried the directory to confirm the new entry was added correctly.

**Note on search syntax:** The search filter is a separate argument from the base DN. Putting the `uid` inside the `-b` value is a common mistake, it treats `uid=sales_printer` as part of the tree path rather than as a filter, which returns nothing.

```bash
# Wrong - uid embedded in base DN:
ldapsearch -x -h localhost -b dc=dadcorp,dc=com,uid=sales_printer

# Correct - uid as a filter:
ldapsearch -x -h localhost -b dc=dadcorp,dc=com "uid=sales_printer"
```

**Output:**

```
# sales_printer, Printers, Sales, dadcorp.com
dn: uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Sales Desk Printer
uidNumber: 6001
gidNumber: 6001
gecos: Sales Desk Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
uid: sales_printer

# numEntries: 1
```

The entry appears at the correct path with all expected attributes.

---

## Final Directory Tree

The complete `dadcorp.com` directory at the end of both labs:

```
dc=dadcorp,dc=com
├── cn=ldapadm                        (LDAP admin account)
│
├── ou=Sales
│   ├── ou=People
│   │   └── uid=tstark
│   └── ou=Printers                   ← added in Part 2
│       └── uid=sales_printer         ← added in Part 2
│
├── ou=Engineering
│   ├── ou=People
│   │   └── uid=bbanner
│   └── ou=Printers
│       └── uid=labprinter
│
└── ou=Admin
    ├── ou=People
    │   └── uid=srogers
    └── ou=Printers
        └── uid=fd_printer
```

---

## Key Takeaways

**LDIF is the universal interface for OpenLDAP.** Every operation, setting database parameters, configuring TLS, loading schemas, adding OUs, and creating user accounts, flows through LDIF files. Understanding the format is the core skill.

**Two authentication modes serve two different purposes.** `-Y EXTERNAL` with `ldapi:///` uses your OS identity (root) and is required for modifying `slapd`'s own configuration. Simple bind (`-x`) with a DN and password is used for directory data operations. Using the wrong one for a given operation will result in an access error.

**A passing `slaptest` does not mean all paths are valid.** The tool tests config syntax only. A `key.ldif` error `(80)` during TLS setup left a wrong file path in the config that wasn't caught until the next cold start. After any TLS configuration change, verify the actual saved values with `grep -i TLS /etc/openldap/slapd.d/cn\=config.ldif`.

**Copy terminal output carefully.** When `slappasswd` outputs a hash with no trailing newline, the shell prompt immediately follows it on the same line. Capturing the `[` or the full prompt string into a password field causes silent authentication failures that are difficult to trace without knowing the root cause.

**Parent containers must exist before child entries.** LDAP enforces referential integrity, you cannot add `uid=sales_printer,ou=Printers,ou=Sales,...` if `ou=Printers,ou=Sales,...` does not already exist. Always verify the parent OU is present and correctly named before adding entries beneath it.

**Schemas must be loaded before the object classes that depend on them.** Attempting to add entries with `posixAccount` or `shadowAccount` attributes before loading the `nis` and `cosine` schemas will fail with an `objectClass` error.


**Recently implemented an OpenLDAP Server from scratch on CentOS 7** to get hands-on experience with the LDAP protocol.

I successfully set up a fully functional OpenLDAP directory server, configured the base directory structure, and secured it with TLS. I then created multiple users and organizational units using LDIF files, added them to the directory, and performed various operations including modifying user attributes, searching the directory, and managing entries.

This project helped me deeply understand how LDAP works in practice — from server setup and schema management to real directory operations like adding, modifying, and querying user data.



%%
### Post

Recently implemented an OpenLDAP Server from scratch to get hands-on experience with the LDAP protocol.

I successfully set up a fully functional OpenLDAP directory server, configured the base directory structure, and secured it with TLS. I then created multiple users and organizational units using LDIF files, added them to the directory, and performed various operations including modifying user attributes, searching the directory, and managing entries.

This project helped me deeply understand how LDAP works in practice, from server setup and schema management to real directory operations like adding, modifying, and querying user data.

`#OpenLDAP #LDAP #DirectoryServices #Linux #IdentityAndAccessManagement #Cybersecurity`
%%



<!--
[[LDAP Configuring an OpenLDAP Server]] + [[LDAP Create LDIF File]] 

# LDAP Configuring an OpenLDAP Server
![[Pasted image 20260530193221.png]]
%%[[Building a High-Availability Cluster on AlmaLinux 9 with Pacemaker#SSH Setup]]%%
Put centos in bridge adapter but is not geetting IP

```
sudo ip link set enp0s3 up
sudo dhclient enp0s3
```
to make permanent
`sudo vi /etc/sysconfig/network-scripts/ifcfg-enp0s3`
ONBOOT=yes
`sudo systemctl restart network`

IP 10.0.0.211

That's exactly the issue I mentioned earlier — the CentOS 7 mirrors are dead. Run these three commands to point `yum` at the vault archive:
```
sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/*.repo
sed -i 's/^#baseurl/baseurl/g' /etc/yum.repos.d/*.repo
sed -i 's/^mirrorlist/#mirrorlist/g' /etc/yum.repos.d/*.repo
```
[[#^664e14|for this to work]]
## Overview

Authentication and authorization are fundamental pillars of any data center environment. When systems need a centralized, standards-based way to manage user identities and access permissions, a directory service becomes essential. **OpenLDAP** is a widely adopted open-source implementation of the [[Lightweight Directory Access Protocol (LDAP)]], making it a go-to choice for organizations that want control and flexibility without licensing costs.

In this lab, I walked through installing and configuring an OpenLDAP server on a Linux host, securing it with TLS, loading the necessary schemas, and populating it with an organizational directory structure and user accounts.

---

## Environment

| Component     | Detail                         |
| ------------- | ------------------------------ |
| OS            | CentOS / RHEL (using `yum`)    |
| LDAP Software | OpenLDAP (`slapd`)             |
| Domain        | `dadcorp.com`                  |
| Admin DN      | `cn=ldapadm,dc=dadcorp,dc=com` |

---

## Step 1 - Install the LDAP Software

The first step is installing the required OpenLDAP packages. This includes the server daemon (`slapd`), compatibility libraries, client tools, and the NSS/PAM integration module so the system can actually use LDAP for authentication.

```bash
sudo yum install compat-openldap openldap-clients openldap-servers nss-pam-ldapd -y
```

^664e14

Once installed, I started and enabled the `slapd` service so it runs on boot:

```bash
systemctl start slapd
systemctl enable slapd
```

```
[root@localhost ~]# sudo yum install compat-openldap openldap-clients openldap-servers nss-pam-ldapd -y
Package 1:compat-openldap-2.3.43-5.el7.x86_64 already installed and latest version
Package openldap-clients-2.4.44-25.el7_9.x86_64 already installed and latest version
Package openldap-servers-2.4.44-25.el7_9.x86_64 already installed and latest version
Package nss-pam-ldapd-0.8.13-25.el7.x86_64 already installed and latest version
Nothing to do
❌[root@localhost ~]# systemctl start slapd
Job for slapd.service failed because the control process exited with error code. See "systemctl status slapd.service" and "journalctl -xe" for details.
[root@localhost ~]# systemctl status slapd
● slapd.service - OpenLDAP Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/slapd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sat 2026-05-30 20:29:15 EDT; 13s ago
     Docs: man:slapd
           man:slapd-config
           man:slapd-hdb
           man:slapd-mdb
           file:///usr/share/doc/openldap-servers/guide.html
  Process: 1995 ExecStart=/usr/sbin/slapd -u ldap -h ${SLAPD_URLS} $SLAPD_OPTIONS (code=exited, status=1/FAILURE)
  Process: 1980 ExecStartPre=/usr/libexec/openldap/check-config.sh (code=exited, status=0/SUCCESS)

May 30 20:29:14 localhost.localdomain runuser[1983]: pam_unix(runuser:session): session closed for user ldap
May 30 20:29:14 localhost.localdomain slapd[1995]: @(#) $OpenLDAP: slapd 2.4.44 (Feb 23 2022 17:11:27) $
                                                           mockbuild@x86-01.bsys.centos.org:/builddir/build/BUILD...lapd
May 30 20:29:15 localhost.localdomain slapd[1995]: tlsmc_cert_create_hash_symlink: ERROR: OS error: Permission denied
May 30 20:29:15 localhost.localdomain slapd[1995]: main: TLS init def ctx failed: -1
May 30 20:29:15 localhost.localdomain slapd[1995]: slapd stopped.
May 30 20:29:15 localhost.localdomain slapd[1995]: connections_destroy: nothing to destroy.
May 30 20:29:15 localhost.localdomain systemd[1]: slapd.service: control process exited, code=exited status=1
May 30 20:29:15 localhost.localdomain systemd[1]: Failed to start OpenLDAP Server Daemon.
May 30 20:29:15 localhost.localdomain systemd[1]: Unit slapd.service entered failed state.
May 30 20:29:15 localhost.localdomain systemd[1]: slapd.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
[root@localhost ~]#
```

```
❌ tlsmc_cert_create_hash_symlink: ERROR: OS error: Permission denied
```

The `ldap` user doesn't have the right permissions on the certs directory. Fix it with:
```bash
chown -R ldap:ldap /etc/openldap/certs/
chmod 700 /etc/openldap/certs/
```
did not fix it

✅ disabled [[SELinux]]
```
setenforce 0
```

**If it starts**, then SELinux was the problem. Fix it properly with:

```bash
setsebool -P allow_ypbind 1
restorecon -R /etc/openldap/certs/
```

Then re-enable SELinux and restart:

```bash
setenforce 1
systemctl restart slapd
```

```
[root@localhost ~]# setsebool -P allow_ypbind 1
[root@localhost ~]# restorecon -R /etc/openldap/certs/
[root@localhost ~]# setenforce 1
[root@localhost ~]# systemctl restart slapd
Job for slapd.service failed because the control process exited with error code. See "systemctl status slapd.service" and "journalctl -xe" for details.
[root@localhost ~]# systemctl status slapd
● slapd.service - OpenLDAP Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/slapd.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sat 2026-05-30 20:35:07 EDT; 6s ago
     Docs: man:slapd
           man:slapd-config
           man:slapd-hdb
           man:slapd-mdb
           file:///usr/share/doc/openldap-servers/guide.html
  Process: 2215 ExecStart=/usr/sbin/slapd -u ldap -h ${SLAPD_URLS} $SLAPD_OPTIONS (code=exited, status=1/FAILURE)
  Process: 2186 ExecStartPre=/usr/libexec/openldap/check-config.sh (code=exited, status=0/SUCCESS)
 Main PID: 2124 (code=exited, status=0/SUCCESS)

May 30 20:35:07 localhost.localdomain runuser[2206]: pam_unix(runuser:session): session opened for user ldap by (uid=0)
May 30 20:35:07 localhost.localdomain runuser[2208]: pam_unix(runuser:session): session opened for user ldap by (uid=0)
May 30 20:35:07 localhost.localdomain runuser[2210]: pam_unix(runuser:session): session opened for user ldap by (uid=0)
May 30 20:35:07 localhost.localdomain runuser[2212]: pam_unix(runuser:session): session opened for user ldap by (uid=0)
May 30 20:35:07 localhost.localdomain runuser[2212]: pam_unix(runuser:session): session closed for user ldap
May 30 20:35:07 localhost.localdomain slapd[2215]: @(#) $OpenLDAP: slapd 2.4.44 (Feb 23 2022 17:11:27) $
                                                           mockbuild@x86-01.bsys.centos.org:/builddir/build/BUILD...lapd
May 30 20:35:07 localhost.localdomain systemd[1]: slapd.service: control process exited, code=exited status=1
May 30 20:35:07 localhost.localdomain systemd[1]: Failed to start OpenLDAP Server Daemon.
May 30 20:35:07 localhost.localdomain systemd[1]: Unit slapd.service entered failed state.
May 30 20:35:07 localhost.localdomain systemd[1]: slapd.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
[root@localhost ~]#
```

ended up just `setenforce 0`


**Expected output:**

```
Created symlink from /etc/systemd/system/multi-user.target.wants/slapd.service to /usr/lib/systemd/system/slapd.service.
```

The symlink creation confirms `slapd` is now set to start automatically on every boot.

---

copying all files from the lab server
```
 scp -r cloud_user@34.207.249.105:/home/cloud_user/. .
```

## Step 2 - Initialize the LDAP Database

With the service running, the next step is configuring the database backend. OpenLDAP uses LDIF (LDAP Data Interchange Format) files to apply configuration changes. The lab environment came pre-staged with a set of LDIF files in the `~/ldifs/` directory.

```bash
cd ldifs
ls
```

**Contents of the `ldifs/` directory:**

```
base.ldif  dadcorp.key  dadcorp.pem  dbinit.ldif  key.ldif  monitor.ldif  pem.ldif  users.ldif
```

The `dbinit.ldif` file sets three critical database parameters: the domain suffix (`olcSuffix`), the administrator Distinguished Name (`olcRootDN`), and the administrator password (`olcRootPW`).

```bash
cat dbinit.ldif
```

```ldif
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=dadcorp,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=dadcorp,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}changeme
```

The placeholder password `{SSHA}changeme` needs to be replaced with a real hashed password. I generated one using `slappasswd`:

```bash
slappasswd -s 1234 -n
```

**Output:**

```
{SSHA}Hh1Dk9Zcq2TOm7WxJrDzbBXwI2HilWXs
```

I copied this hash and updated the `olcRootPW` field inside `dbinit.ldif` using `vim`. With the file updated, I applied both the database configuration and the monitor configuration:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif
ldapmodify -Y EXTERNAL -H ldapi:/// -f monitor.ldif
```

**Expected output:**

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"
```

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}monitor,cn=config"
```

The `-Y EXTERNAL` flag tells `ldapmodify` to authenticate using the OS-level credentials (running as root), which is the appropriate method for modifying the server's own configuration.

---

## Step 3 - Enable TLS

To secure communications between LDAP clients and the server, I configured TLS using a pre-generated certificate and key pair (`dadcorp.pem` and `dadcorp.key`).

First, I reviewed what each LDIF file would configure:

```bash
cat pem.ldif
```

```ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/dadcorp.pem
```

```bash
cat key.ldif
```

```ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/dadcorp.key
```

Both LDIF files reference paths under `/etc/openldap/certs/`, so I copied the certificate files there first:

```bash
cp dadcorp.* /etc/openldap/certs/
```

Then applied both TLS configurations:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f key.ldif
ldapmodify -Y EXTERNAL -H ldapi:/// -f pem.ldif
```

```bash
#ldapmodify -Y EXTERNAL -H ldapi:/// -f key.ldif
TERNAL -H ldapi:/// -f pem.ldifSASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"
❌ldap_modify: Other (e.g., implementation specific) error (80)
```

> **Note:** The `key.ldif` apply returned an error code `(80)`, a generic implementation-specific error. This can occur when file permissions on the key aren't set correctly for the `ldap` user. The `pem.ldif` apply succeeded cleanly, and the subsequent config test passed, confirming the overall TLS configuration was accepted.

Finally, I validated the configuration with:

```bash
slaptest -u
```

**Output:**

```
config file testing succeeded
```

---

## Step 4 - Load the Required Schemas

LDAP relies on schemas to define what object classes and attributes are valid in the directory. For user account management, three standard schemas are needed: `cosine`, `nis`, and `inetorgperson`.

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

**Expected output (repeated for each schema):**

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=cosine,cn=schema,cn=config"
```

With schemas loaded, I populated the organizational structure using `base.ldif`. This command authenticates as the LDAP admin using simple bind (`-x`) with the password set earlier:

```bash
ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f base.ldif
```

### Troubleshooting - `ldap_bind: Invalid credentials (49)`

After running the `ldapadd` command, I received the following error twice:

```
ldap_bind: Invalid credentials (49)
```

![[Pasted image 20260511124750.png]] _Screenshot: Invalid credentials error in terminal_

**Root cause:** When copying the generated SSHA hash from the terminal output, I accidentally included the trailing `[` bracket that the shell appended to the prompt continuation. This caused the stored password hash to be malformed, so the bind failed.

**Fix:** I regenerated and correctly copied the hash, then re-applied `dbinit.ldif` to overwrite the corrupted password entry:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif
```

After re-applying the correct hash, the `ldapadd` command succeeded:

```
adding new entry "dc=dadcorp,dc=com"
adding new entry "cn=ldapadm,dc=dadcorp,dc=com"
adding new entry "ou=Sales,dc=dadcorp,dc=com"
adding new entry "ou=People,ou=Sales,dc=dadcorp,dc=com"
adding new entry "ou=Engineering,dc=dadcorp,dc=com"
adding new entry "ou=People,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "ou=Printers,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "ou=Admin,dc=dadcorp,dc=com"
adding new entry "ou=People,ou=Admin,dc=dadcorp,dc=com"
adding new entry "ou=Printers,ou=Admin,dc=dadcorp,dc=com"
```

This established the full organizational hierarchy for `dadcorp.com`, with three top-level departments (Sales, Engineering, Admin), each containing a `People` OU and some containing a `Printers` OU.

---

## Step 5 - Add Users and Verify

With the directory structure in place, I added the user accounts defined in `users.ldif`:

```bash
ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f users.ldif
```

**Output:**

```
adding new entry "uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com"
adding new entry "uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "uid=srogers,ou=People,ou=Admin,dc=dadcorp,dc=com"
adding new entry "uid=labprinter,ou=Printers,ou=Engineering,dc=dadcorp,dc=com"
adding new entry "uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com"
```

Three user accounts (Tony Stark, Bruce Banner, Steve Rogers) and two printer objects were added to their respective OUs.

To verify the directory was working correctly, I queried for a specific user using `ldapsearch`:

```bash
ldapsearch -h localhost -D cn=ldapadm,dc=dadcorp,dc=com -w 1234 -b dc=dadcorp,dc=com "uid=bbanner"
```

**Output:**

```
# bbanner, People, Engineering, dadcorp.com
dn: uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Bruce Banner
uidNumber: 5001
gidNumber: 5001
homeDirectory: /home/ldap/bbanner
loginShell: /bin/bash
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: bbanner

# numEntries: 1
```

The user was returned with the correct OU placement and attributes, confirming the directory is properly configured and queryable.

---

## Directory Structure Summary

The final `dadcorp.com` LDAP directory tree looks like this:

```
dc=dadcorp,dc=com
├── cn=ldapadm          (Admin account)
├── ou=Sales
│   └── ou=People
│       └── uid=tstark
├── ou=Engineering
│   ├── ou=People
│   │   └── uid=bbanner
│   └── ou=Printers
│       └── uid=labprinter
└── ou=Admin
    ├── ou=People
    │   └── uid=srogers
    └── ou=Printers
        └── uid=fd_printer
```

---

## Key Takeaways

- **LDIF files are the core mechanism** for configuring and populating OpenLDAP. Every change, from database settings to TLS certificates to user accounts, flows through LDIF modifications.
- **`-Y EXTERNAL` vs `-x` authentication**: Server configuration changes use SASL EXTERNAL (root OS credentials), while directory data operations use simple bind (`-x`) with the LDAP admin DN and password.
- **Password hashing matters**: Always use `slappasswd` to generate SSHA hashes, and be careful when copying the output to avoid capturing extra characters from the terminal prompt.
- **Schemas must be loaded before data**: Attempting to add entries with `posixAccount` or `shadowAccount` attributes before loading the `nis` and `cosine` schemas would fail.
- **`slaptest -u` is your friend**: Running a config test after applying TLS or other structural changes quickly confirms whether `slapd` can actually read and use its configuration.


%%
## Draft
**Cloud Server of the LDAP Server**

Username 
cloud_user

Password
RIn-9Ma(

Private IP of the LDAP Server
10.0.1.10

Public IP of the LDAP Server
34.200.254.136
## Lab Overview

Authentication and authorization are important in a data center environment. While there are many directory access services that can do the job, OpenLDAP is a popular open-source one. In this lab, we'll go over installing and configuring OpenLDAP.

## LDAP: Configure an OpenLDAP Server

## Introduction

In this lab, we'll go over installing and configuring OpenLDAP to make sure it is authenticated and authorized correctly.

## Solution

Log in to the lab-provided server using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP>
```

Then, become the `root` user:

```
sudo -i
```

### Install the LDAP Software

1. Install our package:
    
    ```
    sudo yum install compat-openldap openldap-clients openldap-servers nss-pam-ldapd -y
    ```
    
2. Start `slapd`:
    
    ```
    systemctl start slapd
    ```
    
3. Enable `slapd`:
    
    ```
    systemctl enable slapd
    ```
    

### Initialize the LDAP Database


1. Change our directory over to `ldifs`:
    👀i used root user to it wsa installed in the root users home
    ```
    cd ldifs
    ```
    
2. List out the `ldifs` contents:
    
    ```
    ls
    ```
    
3. We need to `cat` out the following:
    [[find#find command with multiple patterns]]
    ```
    cat dbinit.ldif
    ```

```
[root@server ~]# systemctl start slapd
[root@server ~]# systemctl enable slapd
Created symlink from /etc/systemd/system/multi-user.target.wants/slapd.service to /usr/lib/systemd/system/slapd.service.
[root@server ~]# cd ldifs/
[root@server ldifs]# ls
base.ldif  dadcorp.key  dadcorp.pem  dbinit.ldif  key.ldif  monitor.ldif  pem.ldif  users.ldif
[root@server ldifs]# cat dbinit.ldif
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=dadcorp,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=dadcorp,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}changeme
[root@server ldifs]#

```


4. We find that the `olcRootPW` section says `{SSHA}changeme`. Let's update the password to `1234` as our example:
    
    ```
    slappasswd -s 1234 -n
    ```
    
5. Copy the password that is generated, and then open the `dbinit.ldif` file with `vim`:
    
    ```
    vim dbinit.ldif
    ```
    
6. In here, change our `olcRootPW` to the copied password.
7. Save and quit with `:wq`.
8. Initialize the database:
    
    ```
    ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif
    ```
    
9. Initialize the monitor:
    
    ```
    ldapmodify -Y EXTERNAL -H ldapi:/// -f monitor.ldif
    ```
    

```
[root@server ldifs]# slappasswd -s 1234 -n
{SSHA}Hh1Dk9Zcq2TOm7WxJrDzbBXwI2HilWXs[root@server ldifs]# vim dbinit.ldif

[root@server ldifs]# ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

[root@server ldifs]# ldapmodify -Y EXTERNAL -H ldapi:/// -f monitor.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}monitor,cn=config"

[root@server ldifs]#
```
### Enable TLS

1. We need to `cat` the `pem.ldif` and `key.ldif` files:
    
    ```
    cat pem.ldif
    ```
    
    ```
    cat key.ldif
    ```
    
2. We find that we need to move both the file and certificate into the `/etc/openldap/certs/dadcorp.pem`:
    
    ```
    cp dadcorp.* /etc/openldap/certs/
    ```
    
3. With the files moved, we can continue:
    
    ```
    ldapmodify -Y EXTERNAL -H ldapi:/// -f key.ldif
    ```
    
    ```
    ldapmodify -Y EXTERNAL -H ldapi:/// -f pem.ldif
    ```
    
4. Run a test:
    
    ```
    slaptest -u
    ```
    
    We learn that the test succeeded.

```
[root@server ldifs]# cat pem.ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/dadcorp.pem
[root@server ldifs]# cat key.ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/dadcorp.key
[root@server ldifs]# ls
base.ldif  dadcorp.key  dadcorp.pem  dbinit.ldif  key.ldif  monitor.ldif  pem.ldif  users.ldif
[root@server ldifs]# cp dadcorp.* /etc/openldap/certs/
[root@server ldifs]# ldapmodify -Y EXTERNAL -H ldapi:/// -f key.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"
ldap_modify: Other (e.g., implementation specific) error (80)

[root@server ldifs]# ldapmodify -Y EXTERNAL -H ldapi:/// -f pem.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

[root@server ldifs]# slaptest -u
config file testing succeeded
[root@server ldifs]#

```

### Load the Schemas Required to use LDAP

1. Run the following:
    
    ```
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
    ```
    
    ```
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
    ```
    
    ```
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
    ```
    
2. Enter the following to add our Organizational Units diagram. Make sure to use our `1234` password from earlier:
    
    ```
    ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f base.ldif
    ```


```
[root@server ldifs]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=cosine,cn=schema,cn=config"

[root@server ldifs]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=nis,cn=schema,cn=config"

[root@server ldifs]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=inetorgperson,cn=schema,cn=config"

[root@server ldifs]# ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f base.ldif
❌ldap_bind: Invalid credentials (49)
[root@server ldifs]# ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f base.ldif
❌ldap_bind: Invalid credentials (49)
[root@server ldifs]#

```

![[Pasted image 20260511124750.png]]
*Solution, copied `[` by mistake*
**had to** `ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif`

✅ works now
```
[root@server ldifs]# ldapmodify -Y EXTERNAL -H ldapi:/// -f dbinit.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

[root@server ldifs]# ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f base.ldif
adding new entry "dc=dadcorp,dc=com"

adding new entry "cn=ldapadm,dc=dadcorp,dc=com"

adding new entry "ou=Sales,dc=dadcorp,dc=com"

adding new entry "ou=People,ou=Sales,dc=dadcorp,dc=com"

adding new entry "ou=Engineering,dc=dadcorp,dc=com"

adding new entry "ou=People,ou=Engineering,dc=dadcorp,dc=com"

adding new entry "ou=Printers,ou=Engineering,dc=dadcorp,dc=com"

adding new entry "ou=Admin,dc=dadcorp,dc=com"

adding new entry "ou=People,ou=Admin,dc=dadcorp,dc=com"

adding new entry "ou=Printers,ou=Admin,dc=dadcorp,dc=com"

[root@server ldifs]#
```


1. Add our users:
    
    ```
    ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f users.ldif
    ```
    
2. Verify that it worked correctly:
    
    ```
    ldapsearch -h localhost -D cn=ldapadm,dc=dadcorp,dc=com -w 1234 -b dc=dadcorp,dc=com "uid=bbanner"
    ```
    
    We receive the information for the `bbanner` user.
3. If we want to double-check, we can check in the `users` and make sure the information for `bbanner` is the same:
    
    ```
    cat users.ldif
    ```
    

```
[root@server ldifs]# ldapadd -x -w 1234 -D cn=ldapadm,dc=dadcorp,dc=com -f users.ldif
adding new entry "uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com"

adding new entry "uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com"

adding new entry "uid=srogers,ou=People,ou=Admin,dc=dadcorp,dc=com"

adding new entry "uid=labprinter,ou=Printers,ou=Engineering,dc=dadcorp,dc=com"

adding new entry "uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com"

[root@server ldifs]# ldapsearch -h localhost -D cn=ldapadm,dc=dadcorp,dc=com -w 1234 -b dc=dadcorp,dc=com "uid=bbanner"
# extended LDIF
#
# LDAPv3
# base <dc=dadcorp,dc=com> with scope subtree
# filter: uid=bbanner
# requesting: ALL
#

# bbanner, People, Engineering, dadcorp.com
dn: uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Bruce Banner
uidNumber: 5001
gidNumber: 5001
userPassword:: e2NyeXB0fSEh
gecos: Bruce Banner
loginShell: /bin/bash
homeDirectory: /home/ldap/bbanner
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: bbanner

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
[root@server ldifs]# cat users.ldif
dn: uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Tony Stark
uidNumber: 5000
gidNumber: 5000
userPassword: {crypt}$6$yi5/qY.s$odjLQeKpb8wvS8vSe4OsoqEOO7Uxime9HBrswd6/xBOgVmviSlK8HPIr8zWCIxx2SuCRzVNdp3ssh7wMTbENf/
gecos: Tony Stark
loginShell: /bin/bash
homeDirectory: /home/ldap/tstark
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999

dn: uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Bruce Banner
uidNumber: 5001
gidNumber: 5001
userPassword: {crypt}!!
gecos: Bruce Banner
loginShell: /bin/bash
homeDirectory: /home/ldap/bbanner
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999

dn: uid=srogers,ou=People,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Steve Rogers
uidNumber: 5002
gidNumber: 5002
userPassword: {crypt}!!
gecos: Steve Rogers
loginShell: /bin/bash
homeDirectory: /home/ldap/srogers
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999

dn: uid=labprinter,ou=Printers,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Lab Printer
uidNumber: 5003
gidNumber: 5003
userPassword: {crypt}$6$4WVSHR5l8yXNNr$NgnnPfOTXPr0Za3mZyrX.8xlyzv9OqVhzK5FX.42dHHt6KPp26VW84xBdpozepQ7t7JlRq541rytfp2iXfZPL1
gecos: Lab Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999

dn: uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Front Desk Printer
uidNumber: 5004
gidNumber: 5004
userPassword: {crypt}$6$czWl0/JLZqK/np$me/VQQvKomnPwurgsJwN/wwix7.pwPr0Kg4WP5JaW9sFel96OYPddAGz5CltNmdpZjp5iyMICuCuvLjYHDYyB.
gecos: Front Desk Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
[root@server ldifs]#
```
%%


# LDAP Create LDIF File

👀 i think we merged this and another lab into one

[LDAP: Create an LDIF File](https://app.pluralsight.com/hands-on/labs/174c0378-3d25-4c22-9e74-b76a96ff2d50?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

Using the VM from [[LDAP Configuring an OpenLDAP Server]]


turned of the server and created snapht  now i get
```

[root@localhost ~]# systemctl status slapd
● slapd.service - OpenLDAP Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/slapd.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sat 2026-05-30 22:04:05 EDT; 1min 53s ago
     Docs: man:slapd
           man:slapd-config
           man:slapd-hdb
           man:slapd-mdb
           file:///usr/share/doc/openldap-servers/guide.html
  Process: 1751 ExecStart=/usr/sbin/slapd -u ldap -h ${SLAPD_URLS} $SLAPD_OPTIONS (code=exited, status=1/FAILURE)
  Process: 1716 ExecStartPre=/usr/libexec/openldap/check-config.sh (code=exited, status=0/SUCCESS)

May 30 22:04:05 localhost.localdomain runuser[1746]: pam_unix(runuser:session): session opened for user ldap by (uid=0)
May 30 22:04:05 localhost.localdomain runuser[1748]: pam_unix(runuser:session): session opened for user ldap by (uid=0)
May 30 22:04:05 localhost.localdomain slapd[1751]: @(#) $OpenLDAP: slapd 2.4.44 (Feb 23 2022 17:11:27) $
                                                           mockbuild@x86-01.bsys.centos.org:/builddir/build/BUILD...lapd
May 30 22:04:05 localhost.localdomain slapd[1751]: main: TLS init def ctx failed: -1
May 30 22:04:05 localhost.localdomain slapd[1751]: slapd stopped.
May 30 22:04:05 localhost.localdomain slapd[1751]: connections_destroy: nothing to destroy.
May 30 22:04:05 localhost.localdomain systemd[1]: slapd.service: control process exited, code=exited status=1
May 30 22:04:05 localhost.localdomain systemd[1]: Failed to start OpenLDAP Server Daemon.
May 30 22:04:05 localhost.localdomain systemd[1]: Unit slapd.service entered failed state.
May 30 22:04:05 localhost.localdomain systemd[1]: slapd.service failed.
Hint: Some lines were ellipsized, use -l to show in full.

```

selinux in permissive did not fix it

🧐 The error `main: TLS init def ctx failed: -1` is the key. When you shut down the VM, `slapd` now starts as the `ldap` user — and that user likely can't read the certificate/key files you copied in Lab 1. This is the most common cause of this exact error after a reboot.

**1. Check the current permissions:**
```bash
ls -la /etc/openldap/certs/dadcorp.*
```

You'll probably see the files are owned by `root` with no group read access.

```
[root@localhost ~]# ls -la /etc/openldap/certs/dadcorp.*
-rw-r--r--. 1 root root 1704 May 30 21:19 /etc/openldap/certs/dadcorp.key
-rw-r--r--. 1 root root 1265 May 30 21:19 /etc/openldap/certs/dadcorp.pem
```

**2. Fix ownership and permissions:**
```bash
chown root:ldap /etc/openldap/certs/dadcorp.key
chown root:ldap /etc/openldap/certs/dadcorp.pem
chmod 640 /etc/openldap/certs/dadcorp.key
chmod 644 /etc/openldap/certs/dadcorp.pem
```

The key needs `640` (owner read/write, group read only) — the `ldap` user must be able to read it but it should never be world-readable.

**3. Also confirm the key path was actually saved in the config** (the `key.ldif` apply hit error `(80)` in Lab 1, so it's worth verifying):
```bash
grep -i TLS /etc/openldap/slapd.d/cn\=config.ldif
```

You should see both `olcTLSCertificateFile` and `olcTLSCertificateKeyFile` entries. If `olcTLSCertificateKeyFile` is missing, you'll need to re-apply `key.ldif` after fixing permissions and getting `slapd` back up.

```
[root@localhost ~]# grep -i TLS /etc/openldap/slapd.d/cn\=config.ldif
olcTLSCACertificatePath: /etc/openldap/certs
olcTLSCertificateKeyFile: /etc/openldap/certs/password
olcTLSCertificateFile: /etc/openldap/certs/dadcorp.pem
[root@localhost ~]#
```

There's your problem. The key file path is wrong — it's pointing to `/etc/openldap/certs/password` instead of `/etc/openldap/certs/dadcorp.key`. This is a leftover default value because the `key.ldif` apply failed with error `(80)` back in Lab 1 and never actually saved correctly.

Since `slapd` is down you can't use `ldapmodify`, so edit the config file directly:
```bash
vim /etc/openldap/slapd.d/cn\=config.ldif
```

Find this line:

```
olcTLSCertificateKeyFile: /etc/openldap/certs/password
```

Change it to:

```
olcTLSCertificateKeyFile: /etc/openldap/certs/dadcorp.key
```

Save and quit `:wq`, then start it back up:

bash

```bash
systemctl start slapd
systemctl status slapd
```

✅ You should see `Active: active (running)` this time. Let me know what you get.


## Lab Overview

Authentication and authorization are essential things to have in a data center environment. While there are many Directory Access services that can do the job, OpenLDAP is a popular open-source one. In this lab, we'll go over using LDIF files to add Organizational Units and users.


# LDAP: Create an LDIF File

## Introduction

In this lab, we're working with a recently set up OpenLDAP server to test its suitability and set up some automation. For this section of the automation, we need to write two LDIF files. The first is required to add a new OU under the Sales OU named "printers". The second is to add a new user named "sales_printer" with a `uidNumber` and `gidNumber` set to `6001` that will have additional configuration done at a later time.

The LDAP server is already up and running, and the LDAP Administrator username is `ldapadm`, and the password is `1234`.

> [!attention]- Why the error now and not during Lab 1 [[LDAP Configuring an OpenLDAP Server]]
> Because `slapd` was **already running** the entire time during Lab 1 and was never restarted.
> 
> Here's the sequence of what actually happened:
> 
> 1. `slapd` started fine before TLS was configured — the placeholder path `/etc/openldap/certs/password` exists by default so it caused no issues at that point
> 2. `key.ldif` failed with error `(80)` — the key path was **never updated** in the config
> 3. `pem.ldif` succeeded — the cert file was updated correctly
> 4. `slaptest -u` passed — it tests config syntax, not whether the file paths actually resolve
> 5. Lab continued normally because `slapd` was still the same running process in memory — it never had to re-read the broken TLS config
> 
> When you shut down the VM and brought it back up, `slapd` tried to **cold start** for the first time with that config, actually attempted to open `/etc/openldap/certs/password`, found nothing there, and crashed.
> 
> This is a good real-world lesson worth adding to the Lab 1 article — the `key.ldif` error `(80)` wasn't truly harmless, it left a silent misconfiguration that only surfaced on reboot. Want me to add a troubleshooting subsection to the Lab 1 article capturing this?

### Generate the LDIF and Add a New OU

Add in our new OU:

1. Generate the LDIF:
    
    ```
    ldapsearch -x -h localhost -b dc=dadcorp,dc=com
    ```

```
[root@localhost ~]#     ldapsearch -x -h localhost -b dc=dadcorp,dc=com
# extended LDIF
#
# LDAPv3
# base <dc=dadcorp,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# dadcorp.com
dn: dc=dadcorp,dc=com
dc: dadcorp
objectClass: top
objectClass: domain

# ldapadm, dadcorp.com
dn: cn=ldapadm,dc=dadcorp,dc=com
objectClass: organizationalRole
cn: ldapadm
description: LDAP Manager

# Sales, dadcorp.com
dn: ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Sales

# People, Sales, dadcorp.com
dn: ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# Engineering, dadcorp.com
dn: ou=Engineering,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Engineering

# People, Engineering, dadcorp.com
dn: ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# Printers, Engineering, dadcorp.com
dn: ou=Printers,ou=Engineering,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Printers

# Admin, dadcorp.com
dn: ou=Admin,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Admin

# People, Admin, dadcorp.com
dn: ou=People,ou=Admin,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# Printers, Admin, dadcorp.com
dn: ou=Printers,ou=Admin,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Printers

# tstark, People, Sales, dadcorp.com
dn: uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Tony Stark
uidNumber: 5000
gidNumber: 5000
userPassword:: e2NyeXB0fSQ2JHlpNS9xWS5zJG9kakxRZUtwYjh3dlM4dlNlNE9zb3FFT083VXh
 pbWU5SEJyc3dkNi94Qk9nVm12aVNsSzhIUElyOHpXQ0l4eDJTdUNSelZOZHAzc3NoN3dNVGJFTmYv
gecos: Tony Stark
loginShell: /bin/bash
homeDirectory: /home/ldap/tstark
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: tstark

# bbanner, People, Engineering, dadcorp.com
dn: uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Bruce Banner
uidNumber: 5001
gidNumber: 5001
userPassword:: e2NyeXB0fSEh
gecos: Bruce Banner
loginShell: /bin/bash
homeDirectory: /home/ldap/bbanner
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: bbanner

# srogers, People, Admin, dadcorp.com
dn: uid=srogers,ou=People,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Steve Rogers
uidNumber: 5002
gidNumber: 5002
userPassword:: e2NyeXB0fSEh
gecos: Steve Rogers
loginShell: /bin/bash
homeDirectory: /home/ldap/srogers
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: srogers

# labprinter, Printers, Engineering, dadcorp.com
dn: uid=labprinter,ou=Printers,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Lab Printer
uidNumber: 5003
gidNumber: 5003
userPassword:: e2NyeXB0fSQ2JDRXVlNIUjVsOHlYTk5yJE5nbm5QZk9UWFByMFphM21aeXJYLjh
 4bHl6djlPcVZoeks1RlguNDJkSEh0NktQcDI2Vlc4NHhCZHBvemVwUTd0N0psUnE1NDFyeXRmcDJp
 WGZaUEwx
gecos: Lab Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: labprinter

# fd_printer, Printers, Admin, dadcorp.com
dn: uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Front Desk Printer
uidNumber: 5004
gidNumber: 5004
userPassword:: e2NyeXB0fSQ2JGN6V2wwL0pMWnFLL25wJG1lL1ZRUXZLb21uUHd1cmdzSndOL3d
 3aXg3LnB3UHIwS2c0V1A1SmFXOXNGZWw5Nk9ZUGRkQUd6NUNsdE5tZHBaanA1aXlNSUN1Q3V2TGpZ
 SERZeUIu
gecos: Front Desk Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
uid: fd_printer

# search result
search: 2
result: 0 Success

# numResponses: 16
# numEntries: 15
[root@localhost ~]#
```



1. Look for the `# People, Sales, dadcorp.com` section.
    
2. Copy all four lines of this section:
    
    ```
    # People, Sales, dadcorp.com
    dn: ou=People,ou=Sales,dc=dadcorp,dc=com
    objectClass: organizationalUnit
    ou: People
    ```
    
3. Next, we need to create and edit our new `ou.ldif` file:
    
    ```
    vim ou.ldif
    ```
    
4. Paste the four lines that we copied.
    
5. Remove the `# People, Sales, dadcorp.com` line and update it to read as follows:
    
    ```
dn: ou=prints,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: printers
    ```
    
6. Save and quit with `:wq!`.
    
7. Generate the OU entry:
    
    ```
    ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f ou.ldif
    ```


```
[root@localhost ~]# ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f ou.ldif
adding new entry "ou=prints,ou=Sales,dc=dadcorp,dc=com"

[root@localhost ~]#
```


1. To make sure that this worked correctly, run the following:
    
    ```
    ldapsearch -x -h localhost -b ou=Sales,dc=dadcorp,dc=com
    ```

```
[root@localhost ~]#     ldapsearch -x -h localhost -b ou+Sales,dc=dadcorp,dc=com
# extended LDIF
#
# LDAPv3
# base <ou+Sales,dc=dadcorp,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 34 Invalid DN syntax
text: invalid DN

# numResponses: 1
[root@localhost ~]#     ldapsearch -x -h localhost -b ou=Sales,dc=dadcorp,dc=com
# extended LDIF
#
# LDAPv3
# base <ou=Sales,dc=dadcorp,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# Sales, dadcorp.com
dn: ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Sales

# People, Sales, dadcorp.com
dn: ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# tstark, People, Sales, dadcorp.com
dn: uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Tony Stark
uidNumber: 5000
gidNumber: 5000
userPassword:: e2NyeXB0fSQ2JHlpNS9xWS5zJG9kakxRZUtwYjh3dlM4dlNlNE9zb3FFT083VXh
 pbWU5SEJyc3dkNi94Qk9nVm12aVNsSzhIUElyOHpXQ0l4eDJTdUNSelZOZHAzc3NoN3dNVGJFTmYv
gecos: Tony Stark
loginShell: /bin/bash
homeDirectory: /home/ldap/tstark
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: tstark

# prints, Sales, dadcorp.com
dn: ou=prints,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: printers
ou: prints

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 4
[root@localhost ~]#
```

### Generate the LDIF and Add a New User

Create our new user for the sale's printer:

1. Perform an `ldapsearch`:
    
    ```
    ldapsearch -x -h localhost -b dc=dadcorp,dc=com
    ```

```
[root@localhost ~]# ldapsearch -x -h localhost -b dc=dadcorp,dc=com
# extended LDIF
#
# LDAPv3
# base <dc=dadcorp,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# dadcorp.com
dn: dc=dadcorp,dc=com
dc: dadcorp
objectClass: top
objectClass: domain

# ldapadm, dadcorp.com
dn: cn=ldapadm,dc=dadcorp,dc=com
objectClass: organizationalRole
cn: ldapadm
description: LDAP Manager

# Sales, dadcorp.com
dn: ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Sales

# People, Sales, dadcorp.com
dn: ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# Engineering, dadcorp.com
dn: ou=Engineering,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Engineering

# People, Engineering, dadcorp.com
dn: ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# Printers, Engineering, dadcorp.com
dn: ou=Printers,ou=Engineering,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Printers

# Admin, dadcorp.com
dn: ou=Admin,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Admin

# People, Admin, dadcorp.com
dn: ou=People,ou=Admin,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# Printers, Admin, dadcorp.com
dn: ou=Printers,ou=Admin,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Printers

# tstark, People, Sales, dadcorp.com
dn: uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Tony Stark
uidNumber: 5000
gidNumber: 5000
userPassword:: e2NyeXB0fSQ2JHlpNS9xWS5zJG9kakxRZUtwYjh3dlM4dlNlNE9zb3FFT083VXh
 pbWU5SEJyc3dkNi94Qk9nVm12aVNsSzhIUElyOHpXQ0l4eDJTdUNSelZOZHAzc3NoN3dNVGJFTmYv
gecos: Tony Stark
loginShell: /bin/bash
homeDirectory: /home/ldap/tstark
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: tstark

# bbanner, People, Engineering, dadcorp.com
dn: uid=bbanner,ou=People,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Bruce Banner
uidNumber: 5001
gidNumber: 5001
userPassword:: e2NyeXB0fSEh
gecos: Bruce Banner
loginShell: /bin/bash
homeDirectory: /home/ldap/bbanner
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: bbanner

# srogers, People, Admin, dadcorp.com
dn: uid=srogers,ou=People,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Steve Rogers
uidNumber: 5002
gidNumber: 5002
userPassword:: e2NyeXB0fSEh
gecos: Steve Rogers
loginShell: /bin/bash
homeDirectory: /home/ldap/srogers
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: srogers

# labprinter, Printers, Engineering, dadcorp.com
dn: uid=labprinter,ou=Printers,ou=Engineering,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Lab Printer
uidNumber: 5003
gidNumber: 5003
userPassword:: e2NyeXB0fSQ2JDRXVlNIUjVsOHlYTk5yJE5nbm5QZk9UWFByMFphM21aeXJYLjh
 4bHl6djlPcVZoeks1RlguNDJkSEh0NktQcDI2Vlc4NHhCZHBvemVwUTd0N0psUnE1NDFyeXRmcDJp
 WGZaUEwx
gecos: Lab Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: labprinter

# fd_printer, Printers, Admin, dadcorp.com
dn: uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Front Desk Printer
uidNumber: 5004
gidNumber: 5004
userPassword:: e2NyeXB0fSQ2JGN6V2wwL0pMWnFLL25wJG1lL1ZRUXZLb21uUHd1cmdzSndOL3d
 3aXg3LnB3UHIwS2c0V1A1SmFXOXNGZWw5Nk9ZUGRkQUd6NUNsdE5tZHBaanA1aXlNSUN1Q3V2TGpZ
 SERZeUIu
gecos: Front Desk Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
uid: fd_printer

# prints, Sales, dadcorp.com
dn: ou=prints,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: printers
ou: prints

# search result
search: 2
result: 0 Success

# numResponses: 17
# numEntries: 16
[root@localhost ~]#
```

1. Find the entry that starts with `# fd_printer, Printers, Admin, dadcorp.com`, then copy the entire entry.

```
# fd_printer, Printers, Admin, dadcorp.com
dn: uid=fd_printer,ou=Printers,ou=Admin,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Front Desk Printer
uidNumber: 5004
gidNumber: 5004
userPassword:: e2NyeXB0fSQ2JGN6V2wwL0pMWnFLL25wJG1lL1ZRUXZLb21uUHd1cmdzSndOL3d
 3aXg3LnB3UHIwS2c0V1A1SmFXOXNGZWw5Nk9ZUGRkQUd6NUNsdE5tZHBaanA1aXlNSUN1Q3V2TGpZ
 SERZeUIu
gecos: Front Desk Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
uid: fd_printer
```

2. Next, we need to edit the `printer.ldif` file:
    
    ```
    vim printer.ldif
    ```
    
3. Paste in the section we just copied.
    
4. Remove the first line of `# fd_printer, Printers, Admin, dadcorp.com`.
    
5. Change the following lines to read as follows while leaving all unmentioned lines as their defaults:
    
    ```bash
    dn: uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com
    cn: Sales Desk Printer 
    
    uidNumber: 6001
    
    gidNumber: 6001 
    
    userPassword: <DELETE FULL LINE>
    
    gecos: Sales Desk Printer
    
    uid: sales_printer
    ```
    
6. Save and quit with `:wq!`.
    
7. Add our new entry:
    
    ```
    ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f printer.ldif
    ```

```
[root@localhost ~]# ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f printer.ldif
adding new entry "uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com"
❌ldap_add: No such object (32)
        matched DN: ou=Sales,dc=dadcorp,dc=com

```

```
[root@localhost ~]# ldapsearch -x -h localhost -b ou=Sales,dc=dadcorp,dc=com
# extended LDIF
#
# LDAPv3
# base <ou=Sales,dc=dadcorp,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# Sales, dadcorp.com
dn: ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: Sales

# People, Sales, dadcorp.com
dn: ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: People

# tstark, People, Sales, dadcorp.com
dn: uid=tstark,ou=People,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Tony Stark
uidNumber: 5000
gidNumber: 5000
userPassword:: e2NyeXB0fSQ2JHlpNS9xWS5zJG9kakxRZUtwYjh3dlM4dlNlNE9zb3FFT083VXh
 pbWU5SEJyc3dkNi94Qk9nVm12aVNsSzhIUElyOHpXQ0l4eDJTdUNSelZOZHAzc3NoN3dNVGJFTmYv
gecos: Tony Stark
loginShell: /bin/bash
homeDirectory: /home/ldap/tstark
shadowExpire: -1
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
uid: tstark

# prints, Sales, dadcorp.com
dn: ou=prints,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: printers
ou: prints

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 4
[root@localhost ~]#
```

💡 The OU exists but it was created with the wrong name — `ou=prints` instead of `ou=Printers`.

Delete the bad OU:
```
ldapdelete -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// "ou=prints,ou=Sales,dc=dadcorp,dc=com"
```

✅ fix
```
[root@localhost ~]# vi ou.ldif
[root@localhost ~]# cat ou.ldif
dn: ou=Printers,ou=Sales,dc=dadcorp,dc=com
objectClass: organizationalUnit
ou: printers
[root@localhost ~]#
```

Apply it:
```
root@localhost ~]# ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f ou.ldif
adding new entry "ou=Printers,ou=Sales,dc=dadcorp,dc=com"
```

**Redoing**
```
[root@localhost ~]# ldapmodify -a -x -D "cn=ldapadm,dc=dadcorp,dc=com" -w 1234 -H ldapi:/// -f printer.ldif
adding new entry "uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com"
```

2. Perform a search for our new entry:
    
    ```
    ldapsearch -x -h localhost -b dc=dadcorp,dc=com,uid=sales_printer
    ```
    
    Our user will appear.
    
```
❌[root@localhost ~]# ldapsearch -x -h localhost -b dc=dadcorp,dc=com,uid=sales_printer
# extended LDIF
#
# LDAPv3
# base <dc=dadcorp,dc=com,uid=sales_printer> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 32 No such object

# numResponses: 1
[root@localhost ~]#
```

✅ The search syntax is wrong — the uid is part of the base DN instead of being a filter. Run it this way:
```
ldapsearch -x -h localhost -b dc=dadcorp,dc=com "uid=sales_printer"
```

```
[root@localhost ~]# ldapsearch -x -h localhost -b dc=dadcorp,dc=com "uid=sales_printer"
# extended LDIF
#
# LDAPv3
# base <dc=dadcorp,dc=com> with scope subtree
# filter: uid=sales_printer
# requesting: ALL
#

# sales_printer, Printers, Sales, dadcorp.com
dn: uid=sales_printer,ou=Printers,ou=Sales,dc=dadcorp,dc=com
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: Sales Desk Printer
uidNumber: 6001
gidNumber: 6001
gecos: Sales Desk Printer
loginShell: /bin/false
homeDirectory: /tmp
shadowExpire: -1
shadowWarning: 7
uid: sales_printer

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
[root@localhost ~]#
```
## Conclusion

Congratulations! You've completed the lab!
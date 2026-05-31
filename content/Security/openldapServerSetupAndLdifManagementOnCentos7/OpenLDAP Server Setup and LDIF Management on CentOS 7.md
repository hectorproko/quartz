---
tags:
  - LDAP
  - Linux
  - security
  - CentOS
linkedin: "False"
quartz: "False"
refactored: "False"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
<!--
[[LDAP Configuring an OpenLDAP Server]] + [[LDAP Create LDIF File]] 
-->
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
ldap_bind: Invalid credentials (49)
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

in this lab we dont use the real ldapadm at all, because doesnt have password?
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

**Root cause:** This is the deferred consequence of the `key.ldif` error `(80)` from Step 4. During Part 1, `slapd` was already running when TLS was configured and was never restarted, so it kept using its in-memory state. On a cold start, it tried to open the TLS key file from its saved config, and that path was wrong.

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

### Step 8 - Create the `Printers` OU Under Sales

#### Write the OU LDIF

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
ldap_add: No such object (32)
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
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

The goal of this hands-on lab is to change the Apache port and give it a proper SELinux label. Keep in mind, changing the port of a web server also requires changes to SELinux and the firewall. I first changed the listening port for Apache in the Apache configuration file, switching from port 80 to a custom port. After that, I configured SELinux for the new port, closed the old port (80) with firewalld, and opened the newly configured Apache port. Finally, I tested the changes to confirm everything worked.
%%If SELinux is in **Enforcing** mode and you change a service to a non-default port, you must explicitly tell SELinux about it — otherwise it'll block the bind even if the firewall and service config are correct.%%
## Environment Details

- **OS:** CentOS Linux 7 (Core)
- **Public IP:** `18.206.239.229`
- **Private IP:** `10.0.1.167`

## Solution

### Step 1: Change the Apache Port from 80 to 61297

1. Open the Apache configuration file:
    
    ```bash
    sudo vim /etc/httpd/conf/httpd.conf
    ```
    
2. Comment out the default port and set the new one:
    
    ```bash
    #Listen 80
    Listen 61297
    ```
    
3. Stop the Apache service:
    
    ```bash
    sudo systemctl stop httpd
    ```
    
4. Configure SELinux to allow the new port:
    
    ```bash
    sudo semanage port -a -t http_port_t -p tcp 61297
    ```
    *As root, add a new SELinux port rule that labels TCP port 61297 with the `http_port_t` type, so Apache is allowed to bind to it.*

%%kjk
how can port be assinged type like process or file
[[semanage (SELinux Management)#Options|semanage]]
**Files and processes:** SELinux contexts are stored as literal metadata.

- Files have the context as an **extended attribute (xattr)** on the inode itself — you can see it with `ls -Z`.
- Processes get their context from the program that exec'd them, also tracked via the kernel — visible with `ps -Z`.

In both cases, there's a real kernel object (inode, task_struct) that the context is attached to.

**Ports are different:** A TCP/UDP port number is not a file and has no inode. There's no xattr to attach to. Instead, SELinux maintains a separate **policy table** (the port context table) that maps port number + protocol → type. That's exactly what `semanage port -l` is showing you — it's not metadata stored "on" the port, it's a lookup table the kernel's network hooks (via `netlabel`/`selinux` socket hooks) consult whenever a process tries to `bind()` to that port.
[[SELinux#SELinux Contexts]]
%% 

5. Start the Apache service:
    
    ```bash
    sudo systemctl start httpd
    ```
    

### Step 2: Update Firewall Rules

1. Remove the old port (80) from firewalld:
    
    ```bash
    sudo firewall-cmd --permanent --remove-port=80/tcp
    ```
    
2. Add the new port (61297):
    
    ```bash
    sudo firewall-cmd --permanent --add-port=61297/tcp
    ```
    
3. Reload firewalld to apply changes:
    
    ```bash
    sudo firewall-cmd --reload
    ```
    

> [!NOTE]- Terminal Session Reference
> ```bash
> $ ssh cloud_user@18.206.239.229
> The authenticity of host '18.206.239.229 (18.206.239.229)' can't be established.
> ED25519 key fingerprint is SHA256:y/R3VByB5nFlKEk7xqPDn8AIx2k1wrsl8N6UEUtjdRk.
> This key is not known by any other names
> Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
> Warning: Permanently added '18.206.239.229' (ED25519) to the list of known hosts.
> (cloud_user@18.206.239.229) Password:
> [cloud_user@ip-10-0-1-167 ~]$ sudo -i
> [sudo] password for cloud_user:
> [root@ip-10-0-1-167 ~]# vi /etc/httpd/conf/httpd.conf
> [root@ip-10-0-1-167 ~]# cat /etc/httpd/conf/httpd.conf | grep "Listen"
> # Listen: Allows you to bind Apache to specific IP addresses and/or
> # Change this to Listen on specific IP addresses as shown below to
> #Listen 12.34.56.78:80
> Listen 61297
> [root@ip-10-0-1-167 ~]# sudo systemctl stop httpd
> [root@ip-10-0-1-167 ~]# sudo semanage port -a -t http_port_t -p tcp 61297
> [root@ip-10-0-1-167 ~]# sudo systemctl start httpd
> [root@ip-10-0-1-167 ~]# sudo firewall-cmd --permanent --remove-port=80/tcp
> success
> [root@ip-10-0-1-167 ~]# sudo firewall-cmd --permanent --add-port=61297/tcp
> success
> [root@ip-10-0-1-167 ~]# sudo firewall-cmd --reload
> success
> ```

### Step 3: Verify the Changes

Tested the new configuration using a private/incognito browser window, pointing at the public IP and new port:

```
http://18.206.239.229:61297
```

![[Pasted image 20260625142124.png]]


## Conclusion

This lab reinforced how SELinux and firewalld both need to be updated in tandem whenever a service's listening port changes on a CentOS/RHEL system - simply editing the Apache config isn't enough on its own. Successfully changed Apache's port from 80 to 61297, applied the correct SELinux context, updated firewall rules, and verified the new port was reachable.

<!--
take snapshot of the page before you start lab
page displays when we use the port below then it does not display
I get the whole page witht he picture

(https://app.pluralsight.com/hands-on/labs/016ea985-5091-42e5-b7e5-c5addbc23d14?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

Username

cloud_user

Password

SAn&w1Qp

Private ip address of public instance

10.0.1.167

Public ip address of public instance

18.206.239.229

# Change Apache Port and Give It a Proper Selinux Label

## Introduction

The goal of this hands-on lab is to change the Apache port and give it a proper SELinux label. Keep in mind, when changing the port of the web server, this will also require changes to SELinux and firewall. We will first change the listening port for Apache in the Apache configuration file. In this lab environment, we will change from port 80 to a different port number. Once the port changes are made, we will configure SELinux for the new port. We will also close the previous port with firewalld (port 80) and open the newly configured Apache port. Once completed, we will test if the changes were successful.

## How to Log in to Cloud Server

Log in to the server using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP_ADDRESS> -p 61613
```

**Note:** *When copying and pasting code into Vim from the lab guide, first enter `:set paste` (and then `i` to enter insert mode) to avoid adding unnecessary spaces and hashes. To save and quit the file, press **Escape** followed by `:wq`. To exit the file _without_ saving, press **Escape** followed by `:q!`.

## Solution

### Change Apache port from 80 to 61297

1. Go to Apache configuration file:
    
    ```
    sudo vim /etc/httpd/conf/httpd.conf
    ```
    
2. Edit the Apache configuration file:
    
    ```
    #Listen 80
    Listen 61297
    ```
    
3. Stop Apache service:
    
    ```
    sudo systemctl stop httpd
    ```
    
4. Configure SELinux for the new port:
    
    ```
    sudo semanage port -a -t http_port_t -p tcp 61297
    ```
    
5. Start Apache service:
    
    ```
    sudo systemctl start httpd
    ```
    
6. Close port 80 using firewalld:
    
    ```
    sudo firewall-cmd --permanent --remove-port=80/tcp
    ```
    
7. Add the new port using firewalld:
    
    ```
    sudo firewall-cmd --permanent --add-port=61297/tcp
    ```
    
8. Reload firewalld:
    
    ```
    sudo firewall-cmd --reload
    ```


```
Hecti@DESKTOP-7SBLCCA MINGW64 ~
$ ssh cloud_user@18.206.239.229
The authenticity of host '18.206.239.229 (18.206.239.229)' can't be established.
ED25519 key fingerprint is SHA256:y/R3VByB5nFlKEk7xqPDn8AIx2k1wrsl8N6UEUtjdRk.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '18.206.239.229' (ED25519) to the list of known hosts.
(cloud_user@18.206.239.229) Password:
[cloud_user@ip-10-0-1-167 ~]$ sudo -i
[sudo] password for cloud_user:
[root@ip-10-0-1-167 ~]# vi /etc/httpd/conf/httpd.conf
[root@ip-10-0-1-167 ~]# cat /etc/httpd/conf/httpd.conf | grep  "Listen"
# Listen: Allows you to bind Apache to specific IP addresses and/or
# Change this to Listen on specific IP addresses as shown below to
#Listen 12.34.56.78:80
Listen 61297
[root@ip-10-0-1-167 ~]#
[root@ip-10-0-1-167 ~]# sudo systemctl stop httpd
[root@ip-10-0-1-167 ~]# sudo semanage port -a -t http_port_t -p tcp 61297
[root@ip-10-0-1-167 ~]# sudo systemctl stat httpd
Unknown operation 'stat'.
[root@ip-10-0-1-167 ~]# sudo systemctl start httpd
[root@ip-10-0-1-167 ~]#     sudo firewall-cmd --permanent --remove-port=80/tcp
success
[root@ip-10-0-1-167 ~]# sudo firewall-cmd --permanent --add-port=61297/tcp
success
[root@ip-10-0-1-167 ~]# sudo firewall-cmd --reload
success
[root@ip-10-0-1-167 ~]#
```

1. Test with a private/incognito browser using the public ip address of your cloud server and new port number:
    
    ```
    http://Server_IP:61297
    ```

18.206.239.229

![[Pasted image 20260625142124.png]]
## Conclusion

Congratulations — you've completed this hands-on lab!

```
[root@ip-10-0-1-167 ~]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```
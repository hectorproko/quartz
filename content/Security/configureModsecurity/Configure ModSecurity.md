---
tags:
  - "#WAF"
linkedin: "False"
quartz: "False"
refactored: "False"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
title: Configure ModSecurity on Apache with OWASP CRS
---
## Introduction

The goal of this lab is to configure ModSecurity for the Apache web server on a CentOS 7 instance. A few things to keep in mind before starting:

- `firewalld` is running with ports **61613**, **80**, and **65535** open
- Port **61613** is used for SSH
- Apache is running on port **80**
- The back end is running on port **65535**
- SELinux is in enforcing mode

The objectives are to install ModSecurity from the repositories, load it into Apache, and configure it to use the [[OWASP Core Rule Set (CRS)]] to inspect and filter incoming traffic.


### Server

- OS: CentOS Linux 7 (Core)
- Private IP: 10.0.1.125
- Public IP: 54.209.182.126


---

## Install `mod_security`

ModSecurity is a web application firewall (WAF) module for Apache. Installing it from the CentOS repositories pulls in the module along with a default configuration file.

```bash
sudo yum install mod_security
```

---

## Configure the OWASP Core Rule Set (CRS)

The OWASP CRS is a set of generic attack detection rules for use with ModSecurity. Cloning the official repository gives access to a maintained, community-driven ruleset that protects against common threats like SQL injection and cross-site scripting.

1. Create a directory to store the CRS files:
    
    ```bash
    sudo mkdir /etc/httpd/crs
    ```
    
2. Navigate into the new directory:
    
    ```bash
    cd /etc/httpd/crs
    ```
    
3. Install Git if it is not already present:
    
    ```bash
    sudo yum install git
    ```
    
4. Clone the OWASP ModSecurity CRS repository:
    
    ```bash
    sudo git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git
    ```

5. Navigate into the cloned repository:
    
    ```bash
    cd /etc/httpd/crs/owasp-modsecurity-crs/
    ```
    
6. Create the active configuration file by copying the example template:
    
    ```bash
    sudo cp crs-setup.conf.example crs-setup.conf
    ```
    
---

## Inform Apache of the Changes

Apache needs to be told where to find the CRS configuration and rules. This is done by adding an `IfModule` block to the main Apache configuration file so the includes are only processed when `mod_security` is loaded.

1. Open the Apache configuration file:
    
    ```bash
    sudo vim /etc/httpd/conf/httpd.conf
    ```
    
2. Append the following block at the bottom of the file:
    
    ```apache
    <IfModule security2_module>
       Include /etc/httpd/crs/owasp-modsecurity-crs/crs-setup.conf
       Include /etc/httpd/crs/owasp-modsecurity-crs/rules/*.conf
    </IfModule>
    ```
    

3. Verify the block was written correctly using `grep`:
    
    ```bash
    grep -A 4 security2_module /etc/httpd/conf/httpd.conf
    ```
    
    **Expected output:**
    
    ```
    <IfModule security2_module>
       Include /etc/httpd/crs/owasp-modsecurity-crs/crs-setup.conf
       Include /etc/httpd/crs/owasp-modsecurity-crs/rules/*.conf
    </IfModule>
    ```
    

---

## Restart Apache and Test ModSecurity

With the configuration in place, Apache needs to be restarted to load the new rules. The tests below confirm that ModSecurity is actively blocking suspicious traffic while still serving legitimate requests.

1. Restart the Apache service:
    
    ```bash
    sudo systemctl restart httpd
    ```
    
2. Simulate a vulnerability scanner using `curl` with the `Nessus` user-agent string. ModSecurity's OWASP rules flag this agent and should return a `403 Forbidden` response:
    
    ```bash
    curl -i http://<SERVER_IP_ADDRESS>/index.html -A Nessus
    ```
    %%[[curl#Options]]%%
    **Expected output:**
    
    ```
    HTTP/1.1 403 Forbidden
    Date: Fri, 03 Jul 2026 13:13:01 GMT
    Server: Apache/2.4.6 (CentOS)
    Content-Length: 212
    Content-Type: text/html; charset=iso-8859-1
    
    <!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
    <html><head>
    <title>403 Forbidden</title>
    </head><body>
    <h1>Forbidden</h1>
    <p>You don't have permission to access /index.html on this server.</p>
    </body></html>
    ```
    
3. Open `http://<SERVER_IP_ADDRESS>/index.html` in a browser. A normal browser request should load the page successfully, confirming that legitimate traffic is not blocked.
    
    ![[Pasted image 20260703091610.png]]
    

%%### Troubleshooting - Apache Fails to Restart

After adding the `IfModule` block and restarting, Apache may fail to start:

```
[root@ip-10-0-1-125 crs]# sudo systemctl restart httpd
Job for httpd.service failed because the control process exited with error code.
See "systemctl status httpd.service" and "journalctl -xe" for details.
```

Checking the service status reveals the cause:

```bash
systemctl status httpd
```

```
Active: failed (Result: exit-code)
...
httpd[13926]: httpd: Syntax error on line 355 of /etc/httpd/conf/httpd.conf...tory
```

**Root cause:** The `Include` directive in `httpd.conf` was pointing to `crs-setup.conf`, but the actual file in the repository was still named `crs-setup.conf.example`. The `cp` command had not been run from inside the repository directory, so the renamed file was never created.

**Fix:** Navigate into the repository and create the config file from the example:

```bash
cd /etc/httpd/crs/owasp-modsecurity-crs/
sudo cp crs-setup.conf.example crs-setup.conf
sudo systemctl restart httpd
```

Confirm Apache is running:

```bash
systemctl status httpd
```

```
Active: active (running) since Fri 2026-07-03 09:11:42 EDT; 13s ago
```
%%
---

## Conclusion

This lab covered installing and configuring ModSecurity as a WAF for Apache on CentOS 7. The OWASP CRS was cloned from the official repository, activated via the Apache configuration, and validated by confirming that requests with a known malicious user-agent string were blocked with a `403 Forbidden` response while normal browser traffic continued to work as expected.

<!--
(https://app.pluralsight.com/hands-on/labs/e838a9ae-5f0c-47f3-a2db-59eb0c805466?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

Username

cloud_user

Password

yAJ-oaU9

Private ip address of public instance

10.0.1.125

Public ip address of public instance

54.209.182.126

```

Hecti@DESKTOP-7SBLCCA MINGW64 ~
$ cloud_user@54.209.182.126
bash: cloud_user@54.209.182.126: command not found

Hecti@DESKTOP-7SBLCCA MINGW64 ~
$ ssh cloud_user@54.209.182.126 -p 61613
ssh: connect to host 54.209.182.126 port 61613: Connection timed out

Hecti@DESKTOP-7SBLCCA MINGW64 ~
$ ssh cloud_user@54.209.182.126 -p 61613
The authenticity of host '[54.209.182.126]:61613 ([54.209.182.126]:61613)' can't be established.
ED25519 key fingerprint is SHA256:hE47FLxbtg2v5y102Z20jUYP26OJRVYrdPRR4mJf/0Q.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[54.209.182.126]:61613' (ED25519) to the list of known hosts.
(cloud_user@54.209.182.126) Password:
[cloud_user@ip-10-0-1-125 ~]$ cat /etc/os-release
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

[cloud_user@ip-10-0-1-125 ~]$
```
# Configure ModSecurity

## Introduction

The goal for this lab is to configure ModSecurity for the Apache web server. There are a few things you need to keep in mind before you start the lab. Take into consideration that firewalld is up and running and that ports 61613, 80, and 65535 are open. Port 61613 is your SSH port where you will connect. Apache is running on port 80, and the back end is functioning on port 65535. SELinux is in enforcing mode. The objective of the lab is to install ModSecurity from the repositories, load it, and instruct Apache to use it. Lastly, install ModSecurity to use OWASP rules in order to apply them against traffic.

## Solution

**Note:** For this lab, the use of a standalone terminal app with ssh support is best as the Instant Terminal does not permit port 61613. The instance does take a minute or so to be ready to be connected to via ssh.

Log in to the lab server via SSH using the credentials provided:

```bash
ssh cloud_user@<SERVER_IP_ADDRESS> -p 61613
```

### Install `mod_security`

1. Install `mod_security` from the repositories:
    
    ```
    sudo yum install mod_security
    ```
    

### Configure OWASP Core Rule Set (CRS)

1. Make a `crs` directory:
    
    ```
    sudo mkdir /etc/httpd/crs
    ```
    
2. Navigate to the new directory:
    
    ```
    cd /etc/httpd/crs
    ```
    
3. Install Git:
    
    ```
    sudo yum install git
    ```
    
4. Clone a Git repository for OWASP CRS:
    
    ```
    sudo git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git
    ```
    
5. Configure the new repository:
    
    ```
    sudo cd /etc/httpd/crs/owasp-modsecurity-crs/
    ```
    
6. Make a copy of `crs-setup.conf.example` and rename it to `crs-setup.conf`:
    
    ```
    sudo cp crs-setup.conf.example crs-setup.conf
    ```
    
```
[root@ip-10-0-1-125 cloud_user]# sudo mkdir /etc/httpd/crs
[root@ip-10-0-1-125 cloud_user]# cd /etc/httpd/crs
[root@ip-10-0-1-125 crs]# sudo yum install git
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * epel: d2lzkl7pfhq30w.cloudfront.net
Package git-1.8.3.1-25.el7_9.x86_64 already installed and latest version
Nothing to do
[root@ip-10-0-1-125 crs]#     sudo git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git
Cloning into 'owasp-modsecurity-crs'...
remote: Enumerating objects: 10486, done.
remote: Total 10486 (delta 0), reused 0 (delta 0), pack-reused 10486 (from 1)
Receiving objects: 100% (10486/10486), 3.33 MiB | 0 bytes/s, done.
Resolving deltas: 100% (7687/7687), done.
[root@ip-10-0-1-125 crs]# sudo cp crs-setup.conf.example crs-setup.conf
cp: cannot stat ‘crs-setup.conf.example’: No such file or directory
[root@ip-10-0-1-125 crs]#
```
### Inform Apache of the Changes

1. Open the configuration file:
    
    ```
    sudo vim /etc/httpd/conf/httpd.conf
    ```
    
2. Insert at the bottom of the file:
    
    ```bash
<IfModule security2_module>
   Include /etc/httpd/crs/owasp-modsecurity-crs/crs-setup.conf
   Include /etc/httpd/crs/owasp-modsecurity-crs/rules/*.conf
</IfModule>
    ```
    
3. Save and close:
    
    ```
    ESC
    :wq
    ENTER
    ```
    
%%[[grep#display lines below]]%%

```
[root@ip-10-0-1-125 crs]# sudo vim /etc/httpd/conf/httpd.conf
[root@ip-10-0-1-125 crs]# grep  -A 4 security2_module /etc/httpd/conf/httpd.conf
<IfModule security2_module>
   Include /etc/httpd/crs/owasp-modsecurity-crs/crs-setup.conf
   Include /etc/httpd/crs/owasp-modsecurity-crs/rules/*.conf
</IfModule>
[root@ip-10-0-1-125 crs]#
```
### Restart Apache and Run a Few Tests to Confirm `mod_security` Is Working Properly

1. Restart the Apache service:
    
    ```
    sudo systemctl restart httpd
    ```
    
2. Run a test (replace `<SERVER_IP_ADDRESS>` with the server IP address on the lab page):
    
    ```bash
    curl -i http://<SERVER_IP_ADDRESS>/index.html -A Nessus
    curl -i http://54.209.182.126/index.html -A Nessus #public ip
    curl -i http://10.0.1.125/index.html -A Nessus #private ip
    ```
    
    You should receive a `403 Forbidden` error.
    
3. Enter `http://<SERVER_IP_ADDRESS>/index.html` into a new browser tab. We should see it's a functional site.
    

Using public IP
my laptop
```

Hecti@DESKTOP-7SBLCCA MINGW64 /
$ curl -i http://54.209.182.126/index.html -A Nessus
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:21 --:--:--     0
curl: (28) Failed to connect to 54.209.182.126 port 80 after 21044 ms: Timed out

Hecti@DESKTOP-7SBLCCA MINGW64 /
```

inside instance
```
[root@ip-10-0-1-125 crs]# sudo systemctl restart httpd
Job for httpd.service failed because the control process exited with error code. See "systemctl status httpd.service" and "journalctl -xe" for details.
[root@ip-10-0-1-125 crs]# curl -i http://54.209.182.126/index.html -A Nessus
curl: (7) Failed connect to 54.209.182.126:80; Connection refused
[root@ip-10-0-1-125 crs]#
```

private ip failed as well
```
[root@ip-10-0-1-125 crs]# curl -i http://10.0.1.125/index.html -A Nessus #private ip
curl: (7) Failed connect to 10.0.1.125:80; Connection refused
[root@ip-10-0-1-125 crs]#
```

cause the httpd is not on
```
[root@ip-10-0-1-125 crs]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Fri 2026-07-03 08:53:45 EDT; 6min ago
     Docs: man:httpd(8)
           man:apachectl(8)
  Process: 13922 ExecStop=/bin/kill -WINCH ${MAINPID} (code=exited, status=0/SUCCESS)
  Process: 13926 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
 Main PID: 13926 (code=exited, status=1/FAILURE)

Jul 03 08:53:44 ip-10-0-1-125.ec2.internal systemd[1]: Stopped The Apache HTTP Server.
Jul 03 08:53:44 ip-10-0-1-125.ec2.internal systemd[1]: Starting The Apache HTTP Server...
Jul 03 08:53:45 ip-10-0-1-125.ec2.internal httpd[13926]: httpd: Syntax error on line 355 of /etc/httpd/conf/httpd.conf...tory
Jul 03 08:53:45 ip-10-0-1-125.ec2.internal systemd[1]: httpd.service: main process exited, code=exited, status=1/FAILURE
Jul 03 08:53:45 ip-10-0-1-125.ec2.internal systemd[1]: Failed to start The Apache HTTP Server.
Jul 03 08:53:45 ip-10-0-1-125.ec2.internal systemd[1]: Unit httpd.service entered failed state.
Jul 03 08:53:45 ip-10-0-1-125.ec2.internal systemd[1]: httpd.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
[root@ip-10-0-1-125 crs]#
```

the incldue module path is pointing to file with different name 
```
[root@ip-10-0-1-125 crs]# ls /etc/httpd/crs/owasp-modsecurity-crs/crs-setup.conf.example
/etc/httpd/crs/owasp-modsecurity-crs/crs-setup.conf.example
```
it has sample

we are pointing to one without no example
```
<IfModule security2_module>
   Include /etc/httpd/crs/owasp-modsecurity-crs/crs-setup.conf
   Include /etc/httpd/crs/owasp-modsecurity-crs/rules/*.conf
</IfModule>
```

now httpd is up
```
[root@ip-10-0-1-125 owasp-modsecurity-crs]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2026-07-03 09:11:42 EDT; 13s ago
     Docs: man:httpd(8)
           man:apachectl(8)
  Process: 13922 ExecStop=/bin/kill -WINCH ${MAINPID} (code=exited, status=0/SUCCESS)
 Main PID: 28141 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─28141 /usr/sbin/httpd -DFOREGROUND
           ├─28151 /usr/sbin/httpd -DFOREGROUND
           ├─28152 /usr/sbin/httpd -DFOREGROUND
           ├─28153 /usr/sbin/httpd -DFOREGROUND
           ├─28154 /usr/sbin/httpd -DFOREGROUND
           └─28155 /usr/sbin/httpd -DFOREGROUND

Jul 03 09:11:42 ip-10-0-1-125.ec2.internal systemd[1]: Starting The Apache HTTP Server...
Jul 03 09:11:42 ip-10-0-1-125.ec2.internal systemd[1]: Started The Apache HTTP Server.
[root@ip-10-0-1-125 owasp-modsecurity-crs]#

```

figure out why the naming stated exapmle iwht you renamed it in you cp


getting expected forbidden message (both instance and laptop)
```
[root@ip-10-0-1-125 owasp-modsecurity-crs]# curl -i http://54.209.182.126/index.html -A Nessus
HTTP/1.1 403 Forbidden
Date: Fri, 03 Jul 2026 13:13:01 GMT
Server: Apache/2.4.6 (CentOS)
Content-Length: 212
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /index.html
on this server.</p>
</body></html>
[root@ip-10-0-1-125 owasp-modsecurity-crs]#

```

when i put in the browswer i do get the page
![[Pasted image 20260703091610.png]]
## Conclusion

Congratulations on successfully completing this hands-on lab!
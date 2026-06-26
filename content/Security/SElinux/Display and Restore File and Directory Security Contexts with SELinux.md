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
%%[[SELinux]]%%
## Introduction

This hands-on lab focuses on troubleshooting and resolving SELinux filesystem context issues. Knowing how to discover and fix SELinux context problems is a key skill when administering SELinux-enabled systems. By the end of this lab, I was able to view security contexts on files and directories and apply corrected contexts to resolve an access issue.

**Environment:** CentOS Linux 7, accessed via SSH to a cloud server instance.

## What I Did and Why

The goal was to diagnose why a web API endpoint was returning a `403 Forbidden` error despite normal file permissions looking fine, and to fix it using SELinux tools rather than just permissions. This matters because in SELinux-enforced environments, correct file permissions alone aren't enough. The security context also has to match what the service (in this case, Apache/httpd) is allowed to access.

### Step 1: Identify the Problem

I started by confirming the symptom and checking whether SELinux was actually enforcing.

```
curl localhost/web-api/index.html
```

Output:

```
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /web-api/index.html
on this server.</p>
</body></html>
```

This returned a `403 Forbidden` error.

```
getenforce
```

Output:

```
Enforcing
```

This confirmed SELinux was set to `Enforcing`, meaning it was actively blocking the request rather than this being a plain permissions issue.

#### Troubleshooting: Comparing Security Contexts

To find the actual cause, I compared the SELinux context of a known-working directory against the one throwing the error.

```
ls -Z /var/www
```

Output:

```
drwxr-xr-x. root root system_u:object_r:httpd_sys_script_exec_t:s0 cgi-bin
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 html
```

The `html` directory showed the correct context for serving web content: `httpd_sys_content_t`.

```
ls -Z /var/www/html/web-api
```

Output:

```
-rw-r--r--. root root system_u:object_r:admin_home_t:s0 index.html
```

The `index.html` file showed a context of `admin_home_t`, which is not a context Apache is permitted to serve. This was the root cause of the `403` error.

### Step 2: Fix the Security Context

With the cause identified, I used `restorecon` to reset the directory and file to the correct default context defined by SELinux policy.

```
sudo restorecon -R /var/www/html/web-api
```

> **Note:** This step prompts for the `cloud_user` password.

### Step 3: Verify the Fix

```
ls -Z /var/www/html/web-api
```

Output: 

```
-rw-r--r--. root root system_u:object_r:httpd_sys_content_t:s0 index.html
```

The context updated to `httpd_sys_content_t`, matching the working directory from [[#Step 1 Identify the Problem|Step 1]].

```
curl localhost/web-api/index.html
```

Output:

```
This is the index file for the new web-api, placeholder text...
```

Instead of the `403 Forbidden` error, the request now returned the expected placeholder content from the index file.

## Conclusion

This lab reinforced that SELinux context issues can look identical to permissions issues from the outside (both return `403 Forbidden`), but require a different diagnostic path: checking `getenforce`, comparing contexts with `ls -Z`, and resolving mismatches with `restorecon` instead of `chmod` or `chown`. This is a useful pattern for troubleshooting access issues on any SELinux-enforced system.

<!--
(https://app.pluralsight.com/hands-on/labs/fb4ec61f-df6c-41f9-b007-b791bf330362?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

**This lab has been retired.**
This lab is accessible via a direct link but is no longer discoverable in our library. For our most up-to-date content, please visit the [Hands-on page](https://app.pluralsight.com/hands-on/?tab=hands-on-labs&trial=true).

---

Cloud Server instance

Username

cloud_user

Password

zVRCj !_ 1

Private IP address of instance

10.0.1.100

Public IP address of instance
13.218.109.161


# Display and Restore File and Directory Security Contexts with SELinux

## Introduction

This hands-on lab allows you to practice troubleshooting and resolving SELinux filesystem context issues. Being able to discover and resolve SELinux context issues is a key concept when working with SELinux. At the end of this activity, you will understand how to view and apply new security contexts to files and directories.

## Solution

Log in to the server using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP_ADDRESS>
```

```
Hecti@DESKTOP-7SBLCCA MINGW64 ~
$ ssh cloud_user@13.218.109.161
The authenticity of host '13.218.109.161 (13.218.109.161)' can't be established.
ED25519 key fingerprint is SHA256:NfRXPpGV29Kei6Pvw8ywBo/Xe59mnwCRl+yyYXBJZTU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '13.218.109.161' (ED25519) to the list of known hosts.
(cloud_user@13.218.109.161) Password:
[cloud_user@ip-10-0-1-100 ~]$ cat /etc/os-release
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

[cloud_user@ip-10-0-1-100 ~]$

```
### Check the Directory and Index File Security Context

1. Run the following:
    
    ```
    curl localhost/web-api/index.html
    ```
    
    You should receive a `403 Forbidden` message.
    
2. Run the following:
    
    ```
    getenforce
    ```
    
    You should receive an output of `Enforcing`, verifying that SELinux is enforcing.
    
3. Run the following:
    
    ```
    ls -Z /var/www
    ```
    
    In the `html` file, you should see the correct context to serve data: `httpd_sys_content_t`.
    
4. Run the following:
    
    ```
    ls -Z /var/www/html/web-api
    ```
    
    In this file and directory, you should see the context of `admin_home_t`, which needs to be fixed.
    
```
[cloud_user@ip-10-0-1-100 ~]$ curl localhost/web-api/index.html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /web-api/index.html
on this server.</p>
</body></html>
[cloud_user@ip-10-0-1-100 ~]$ getenforce
Enforcing
[cloud_user@ip-10-0-1-100 ~]$ ls -Z /var/www
drwxr-xr-x. root root system_u:object_r:httpd_sys_script_exec_t:s0 cgi-bin
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 html
[cloud_user@ip-10-0-1-100 ~]$ ls -Z /var/www/html/web-api
-rw-r--r--. root root system_u:object_r:admin_home_t:s0 index.html
[cloud_user@ip-10-0-1-100 ~]$

```
### Restore the Appropriate Security Context to the API Directory

1. To resolve the issue, run the following:
    
    ```
    sudo restorecon -R /var/www/html/web-api
    ```
    
    > **Note:** You will be asked to enter the password for `cloud_user` at this step. Use the same password from the lab **Credentials** section that you used at the beginning of this lab.
    
2. To check the context, run the following:
    
    ```
    ls -Z /var/www/html/web-api
    ```
    
    You should now see the correct context of `httpd_sys_content_t`.
    
3. Verify your update is working correctly by running the following:
    
    ```
    curl localhost/web-api/index.html
    ```
    
    Instead of an "Access Denied" message, you should now see `This is the index file for the new web-api, placeholder text...`.
    
```
[cloud_user@ip-10-0-1-100 ~]$ sudo restorecon -R /var/www/html/web-api
[sudo] password for cloud_user:
[cloud_user@ip-10-0-1-100 ~]$ ls -Z /var/www/html/web-api
-rw-r--r--. root root system_u:object_r:httpd_sys_content_t:s0 index.html
[cloud_user@ip-10-0-1-100 ~]$ curl localhost/web-api/index.html
This is the index file for the new web-api, placeholder text...
[cloud_user@ip-10-0-1-100 ~]$

```
## Conclusion

Congratulations — you've completed this hands-on lab!
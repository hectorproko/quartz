---
tags:
  - Podman
  - RHEL
linkedin: "True"
quartz: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
**Platform:** Red Hat Enterprise Linux 9.7 (Plow) 
**Tool:** Podman (rootless container engine)  
**Goal:** Launch, interact with, and cleanly remove a containerized Apache HTTP server

---

## Introduction

This hands-on lab was my first real dive into container technology on a live RHEL system. The goal was straightforward: get a container running, poke around inside it, and clean everything up afterward. What made it interesting was that things didn't go perfectly from the start, and troubleshooting those issues taught me just as much as the lab itself.

---

## Environment Verification

Before doing anything else, I confirmed the operating system I was working on. This is a good habit, knowing exactly what environment you're in prevents a lot of "why isn't this working?" moments later.

```bash
cat /etc/os-release
```

**Output:**

```
NAME="Red Hat Enterprise Linux"
VERSION="9.7 (Plow)"
ID="rhel"
PRETTY_NAME="Red Hat Enterprise Linux 9.7 (Plow)"
```

> **Why this matters:** RHEL 9 uses `dnf` as its package manager and ships with Podman as the preferred container runtime, no Docker daemon required.

---
<!--
## Troubleshooting: Broken Repo & Missing Podman

### The Problem

My first attempt at using `podman` hit an immediate wall:

```bash
podman ps -a
```

**Output:**

```
bash: podman: command not found...
Failed to search for file: Failed to download gpg key for repo 'mariadb-main':
Curl error (37): Couldn't read a file:// file for
file:///etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY
[Couldn't open file /etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY]
```

Two problems were happening at the same time:

1. **Podman wasn't installed** - it wasn't in my PATH yet.
2. **The DNF package manager was broken** - a misconfigured MariaDB repository was referencing a GPG key file that didn't exist on disk.

### Why These Two Errors Are Connected

RHEL has a helpful plugin called **Command Not Found**. When you type a command it doesn't recognize, it searches the package repositories to suggest what to install. Because the MariaDB repo was misconfigured, that background search crashed with a GPG key error, making it look like there were two unrelated failures when really one broken repo was causing both.

### The Fix

**Step 1 - Disable the broken MariaDB repo:**

```bash
sudo dnf config-manager --set-disabled mariadb-main
```

> **Alternative:** If `config-manager` isn't available, manually edit `/etc/yum.repos.d/` and set `enabled=0` in the MariaDB repo file.

**Step 2 - Install Podman:**

```bash
sudo dnf install -y podman
```

With the broken repo out of the way, the install completed successfully and `podman` was available on the PATH.

---

-->

## Step 1 - Check for Existing Containers and Images

I started by getting a baseline: are there any containers or images already on this system?

```bash
podman ps -a
```

**Output:**

```
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS  PORTS  NAMES
```

```bash
podman images
```

**Output:**

```
REPOSITORY  TAG  IMAGE ID  CREATED  SIZE
```

Both commands returned empty tables, a clean slate. The `-a` flag on `podman ps` is important because it shows **all** containers, **not just running ones**. Without it, stopped containers would be invisible.

---

## Step 2 - Search for an Apache Image

Next, I searched for an `httpd-24` container image to use as my web server.

```bash
podman search httpd-24 | more
```

**Output (excerpt):**

```
NAME                                     DESCRIPTION
registry.redhat.io/rhel8/httpd-24        Platform for running Apache httpd 2.4...
registry.redhat.io/ubi9/httpd-24         Platform for running Apache httpd 2.4...
docker.io/centos/httpd-24-centos8        (no description)
docker.io/library/httpd                  The Apache HTTP Server Project
```

> **Pro Tip:** Red Hat's own registries (`registry.redhat.io`, `registry.access.redhat.com`) offer officially supported, security-patched images for production use. For this lab, we're using the CentOS image to demonstrate container isolation, the container OS doesn't have to match the host OS.

---

## Step 3 - Run the Container

Now the main event. I launched the container in **detached mode** (`-d`) with a **pseudo-TTY** (`-t`) so I could exec into it later:

```bash
podman run -dt docker.io/centos/httpd-24-centos8
```

**Output:**

```
Trying to pull docker.io/centos/httpd-24-centos8:latest...
Getting image source signatures
Copying blob 3c72a8ed6814 done |
Copying blob a1d926117d46 done |
...
Writing manifest to image destination
2bfce50fca393f2b75b82ff204e81570259f2ce12f5ad114a2905bdb0444e831
```

I did **not** run `podman pull` first, Podman pulled the image automatically when it didn't find it locally. The long string at the end is the full container ID (SHA256 hash).

> **Key concept, Detached mode (`-d`):** The container runs in the background instead of taking over your terminal. This is how you'd run a real server, you don't want to babysit it.

---

## Step 4 - Verify the Container Is Running

```bash
podman ps -a
```

**Output:**

```
CONTAINER ID  IMAGE                                    COMMAND              CREATED         STATUS        PORTS               NAMES
2bfce50fca39  docker.io/centos/httpd-24-centos8:latest /usr/bin/run-http... 50 seconds ago  Up 51 seconds 8080/tcp, 8443/tcp  cool_mccarthy
```

A few things worth noting here:

- The container ID is shortened to `2bfce50fca39` - you only need enough characters to be unique.
- Status shows `Up 51 seconds` - it's actively running.
- Podman auto-assigned the name `cool_mccarthy` (it generates random names when you don't specify one with `--name`).
- Ports `8080/tcp` and `8443/tcp` are listed, but they are **not yet published to the host** - more on this shortly.

```bash
podman images
```

**Output:**

```
REPOSITORY                        TAG     IMAGE ID      CREATED      SIZE
docker.io/centos/httpd-24-centos8 latest  7d2fe0e482ba  5 years ago  441 MB
```

The image is now cached locally. Future `podman run` commands using this image won't need to pull it again.

---

## Step 5 - Exec Into the Container

This is where containers get interesting. I opened an interactive Bash shell **inside** the running container:

```bash
podman exec -it 2bfce50fca39 /bin/bash
```

The prompt changed to `bash-4.4$`, I'm now **inside the container**'s filesystem and process space.

### Check the Container's OS

```bash
cat /etc/redhat-release
```

**Output (host, before exec):**

```
Red Hat Enterprise Linux release 9.7 (Plow)
```

**Output (inside container):**

```
CentOS Linux release 8.2.2004 (Core)
```

This demonstrates one of the most powerful concepts in containerization: **the container has its own OS identity, completely isolated from the host.** The host is RHEL 9.7; the container thinks it's CentOS 8.2. Same kernel underneath, entirely different userspace.

### Test the Apache Web Server

```bash
curl http://localhost:8080
```

**Output (excerpt):**

```html
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE HTML>
<html lang="en">
<head>
  <title>Apache HTTP Server Test Page powered by CentOS</title>
  ...
</head>
<body>
  <h1>Test Page</h1>
  <p>This page is used to test the proper operation of the Apache HTTP server...</p>
</body>
</html>
```

Apache is running and serving its default test page on port 8080, **from inside the container.** The `curl` command worked here because I was inside the container's own network namespace.

```bash
exit
```

---

## Step 6 - Port Isolation Explained

Back on the host, I tried the same `curl` command:

```bash
curl http://localhost:8080
```

**Output:**

```
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

This is **expected behavior**, not an error in my setup. Podman containers are network-isolated by default. Even though the container listens on port 8080 internally, that port is not exposed to the host unless you explicitly publish it with the `-p` flag at run time:

```bash
# Example of how to publish ports (not run in this lab)
podman run -dt -p 8080:8080 docker.io/centos/httpd-24-centos8
```

> **Why this matters for security:** By default, a container can't be reached from outside, you have to intentionally open the door. This is the "deny by default" security posture that makes containers so much safer than bare-metal deployments.

---

## Step 7 - Clean Up

Good container hygiene means removing what you're no longer using. I stopped the container, removed it, and then removed the image.

### Stop the Container

```bash
podman stop 2bfce50fca39
```

**Output:**

```
2bfce50fca39
```

```bash
podman ps -a
```

**Output:**

```
CONTAINER ID  IMAGE                                    COMMAND              CREATED       STATUS                    PORTS               NAMES
2bfce50fca39  docker.io/centos/httpd-24-centos8:latest /usr/bin/run-http...  5 min ago    Exited (0) 7 seconds ago  8080/tcp, 8443/tcp  cool_mccarthy
```

Status now shows `Exited (0)` - a clean shutdown (exit code 0 means no errors).

### Remove the Container

```bash
podman rm 2bfce50fca39
```

**Output:**

```
2bfce50fca39
```

```bash
podman ps -a
```

**Output:**

```
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS  PORTS  NAMES
```

Container is gone. Note that removing a container does **not** remove the image it was built from.

### Remove the Image

```bash
podman rmi docker.io/centos/httpd-24-centos8:latest
```

**Output:**

```
Untagged: docker.io/centos/httpd-24-centos8:latest
Deleted: 7d2fe0e482baf01e8a54f6c633bb2b5a89b3f35d278458c5ed73f3fdcc5646aa
```

> **Note:** During this step I hit an error by accidentally running `podman rmi podman rmi httpd-24-centos8:latest` (duplicated the command). The correct syntax only needs the image name once. Always double-check your command before hitting Enter, especially with destructive operations.

```bash
podman images
```

**Output:**

```
REPOSITORY  TAG  IMAGE ID  CREATED  SIZE
```

Clean slate restored.

---

## Key Takeaways

| Concept                 | What I Learned                                                           |
| ----------------------- | ------------------------------------------------------------------------ |
| **Podman vs Docker**    | Podman is daemonless and rootless-friendly, no background service needed |
| **Container Isolation** | A CentOS 8 container ran on a RHEL 9 host with no conflicts              |
| **Network Isolation**   | Ports must be explicitly published (`-p`) to be reachable from the host  |
| **Auto-pull**           | `podman run` pulls the image if it's not cached locally                  |
| **Cleanup Order**       | Stop container → remove container → remove image                         |

---

## Commands Reference

```bash
# Check OS version
cat /etc/os-release

# List all containers (including stopped)
podman ps -a

# List locally cached images
podman images

# Search for an image
podman search <image-name>

# Run a container (detached, with TTY)
podman run -dt <image>

# Run a container with port publishing
podman run -dt -p <host-port>:<container-port> <image>

# Open a shell inside a running container
podman exec -it <container-id> /bin/bash

# Stop a container
podman stop <container-id>

# Remove a container
podman rm <container-id>

# Remove an image
podman rmi <image-name>:<tag>

# Fix a broken DNF repo
sudo dnf config-manager --set-disabled <repo-name>
```

---

_Lab completed on RHEL 9.7 | Podman container runtime | Apache httpd-24-centos8_




%%
[[podman]]
### Post
Just finished my first hands-on lab with Podman on RHEL 9, and I’m impressed. If you’ve ever used Docker, the learning curve is almost nonexistent. The commands are nearly identical, podman run, podman ps, podman images, exec into a container, curling a web server, stopping, and cleaning up all feel very familiar.

What really stands out are the daemonless and rootless capabilities. They offer clear advantages in terms of reliability and security, and I’m starting to see why many organizations are preferring Podman over Docker.

Looking forward to diving deeper!

`#Podman #Containers #RHEL #Linux #CloudNative #DevOps `

https://hectorproko.github.io/quartz/Podman/Running-My-First-Podman-Container-on-RHEL
%%

%%


```
[cloud_user@0e1f427e773c ~]$ `cat /etc/os-release`
NAME="Red Hat Enterprise Linux"
VERSION="9.7 (Plow)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="9.7"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Red Hat Enterprise Linux 9.7 (Plow)"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:redhat:enterprise_linux:9::baseos"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9"
BUG_REPORT_URL="https://issues.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 9"
REDHAT_BUGZILLA_PRODUCT_VERSION=9.7
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="9.7"
[cloud_user@0e1f427e773c ~]$

```

```
[cloud_user@0e1f427e773c ~]$ sudo dnf module install container-tools
[sudo] password for cloud_user:
Red Hat Enterprise Linux 9 for x86_64 - AppStream from RHUI (RPMs)                                                    56 kB/s | 4.5 kB     00:00
Red Hat Enterprise Linux 9 for x86_64 - AppStream from RHUI (RPMs)                                                   113 MB/s |  89 MB     00:00
Red Hat Enterprise Linux 9 for x86_64 - BaseOS from RHUI (RPMs)                                                       62 kB/s | 4.1 kB     00:00
Red Hat Enterprise Linux 9 for x86_64 - BaseOS from RHUI (RPMs)                                                      109 MB/s | 113 MB     00:01
Red Hat Enterprise Linux 9 Client Configuration                                                                       10 kB/s | 1.5 kB     00:00
Red Hat Enterprise Linux 9 Client Configuration                                                                       43 kB/s | 4.1 kB     00:00
Last metadata expiration check: 0:00:01 ago on Wed 13 May 2026 10:03:40 PM UTC.
Error: Problems in request:
missing groups or modules: container-tools

[cloud_user@0e1f427e773c ~]$ sudo dnf install podman
Last metadata expiration check: 0:00:20 ago on Wed 13 May 2026 10:03:40 PM UTC.
Dependencies resolved.
```
## Lab Overview

Every journey has to start somewhere. In this lab, we jump right into the deep end and look at how to launch and interact with our first container! Once complete, you'll know how to start, interact, and stop a container.

# Running Your First Podman Container on RHEL

## Introduction

Every journey has to start somewhere. In this lab, we jump right into the deep end and look at how to launch and interact with our first container! Once complete, you'll know how to start, interact, and stop a container.

## Solution

Log in to the server using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP_ADDRESS>
```

### Run Your First Podman Container

1. Before we launch our first Podman container, let's check to see if any containers are running. As `cloud_user`:
    
    ```
    podman ps -a
    ```
    
    As `root`:
    
    ```
    sudo podman ps -a
    ```
    
2. Ensure no containers are running. We use the `-a` or `--all` option to display both running and non-running containers. Let's check our container images:
    
    ```
    podman images
    ```
    
    As you can see, we don't have any images yet.
    
3. We'd like to run an `httpd-24` container. Let's search for one:
    
    ```
    podman search httpd-24 | more
    ```
    
4. Let's use the `docker.io/centos/httpd-24-centos8` image. We are running our container in detached mode, with a `tty` to run commands:
    
    ```
    podman run -dt docker.io/centos/httpd-24-centos8
    ```
    
    Note that we didn't have to pull the container image to run it. When we ran the container, `podman` pulled the container image for us.

**my output**
that image did not appear int he search
```
[cloud_user@0e1f427e773c ~]$ podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
[cloud_user@0e1f427e773c ~]$ sudo podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
[cloud_user@0e1f427e773c ~]$ podman images
REPOSITORY  TAG         IMAGE ID    CREATED     SIZE
[cloud_user@0e1f427e773c ~]$ podman search httpd-24 | more
NAME                                     DESCRIPTION
registry.redhat.io/rhel8/httpd-24        Platform for running Apache httpd 2.4 or bui...
registry.redhat.io/ubi8/httpd-24         Platform for running Apache httpd 2.4 or bui...
registry.redhat.io/rhel9/httpd-24        Platform for running Apache httpd 2.4 or bui...
registry.redhat.io/ubi9/httpd-24         Platform for running Apache httpd 2.4 or bui...
registry.redhat.io/rhel10/httpd-24       Platform for running Apache httpd 2.4 or bui...
registry.redhat.io/ubi10/httpd-24        Platform for running Apache httpd 2.4 or bui...
registry.redhat.io/rhscl/httpd-24-rhel7  Platform for running Apache httpd 2.4 or bui...
docker.io/alfiilham26/httpd-24
docker.io/dcpilla/httpd-24
docker.io/yikeshuizhudan/httpd-24
docker.io/glfngocanh/httpd-24
docker.io/rasskazovda/httpd-24
docker.io/pabloapolo/httpd-24
docker.io/herzer/httpd-24
docker.io/aliijaz/httpd-24
docker.io/tomhac3k/httpd-24
docker.io/gwojcieszczuk/httpd-24
docker.io/kasunrajapakse/httpd-24
docker.io/library/httpd                  The Apache HTTP Server Project
docker.io/danielpenagos/httpd-24
docker.io/dgolovin/httpd-24
docker.io/at1969/httpd-24
docker.io/rmallaya/httpd-24
docker.io/mtaru/httpd-24
docker.io/ld5016/httpd-24
docker.io/paketobuildpacks/httpd
docker.io/paketobuildpacks/php-httpd
docker.io/manageiq/httpd                 Container with httpd, built on CentOS for Ma...
docker.io/cilium/demo-httpd
docker.io/shshin4370/httpd-24
docker.io/oryd/hydra-oidc-httpd
docker.io/vulhub/httpd
[cloud_user@0e1f427e773c ~]$ podman search httpd-24 | grep mtaru
docker.io/mtaru/httpd-24
[cloud_user@0e1f427e773c ~]$ podman search httpd-24 | grep centos
[cloud_user@0e1f427e773c ~]$ podman run -dt docker.io/centos/httpd-24-centos8
Trying to pull docker.io/centos/httpd-24-centos8:latest...
Getting image source signatures
Copying blob 3c72a8ed6814 done   |
Copying blob a1d926117d46 done   |
Copying blob 42c269fe6f7b done   |
Copying blob afdf2ecdaa2f done   |
Copying blob 07a0da26acf9 done   |
Copying blob 74241943e2c2 done   |
Copying blob 9e6c9d2db631 done   |
Copying blob a33035987d8c done   |
Copying blob eefe613be3f4 done   |
Copying config 7d2fe0e482 done   |
Writing manifest to image destination
2bfce50fca393f2b75b82ff204e81570259f2ce12f5ad114a2905bdb0444e831
[cloud_user@0e1f427e773c ~]$
```

    
    
    
1. Let's check our containers and copy the `CONTAINER ID`:
    
    ```
    podman ps -a
    ```
    
2. Let's check our container images:
    
    ```
    podman images
    ```
    
    We see that we have our `httpd-24-centos8:latest` image now.
    
3. The container is running, let's interact with it. First, view the operating system our server using:
    
    ```
    cat /etc/redhat-release
    ```
    
    We can see it's RHEL 8.
    
4. Open a Bash shell in our container and paste in the `CONTAINER ID`, copied in our earlier step:
    
    ```
    podman exec -it <container_ID> /bin/bash
    ```
    
    This will give us an interactive session, showing the prompt as `bash-4.4$`.
    
5. Check what version of the operating system our container identifies as:
    
    ```
    cat /etc/redhat-release
    ```
    
    The container identifies itself as CentOS 8.
    
6. Try accessing the Apache web server running on port `8080` using `curl`:
    
    ```
    curl http://localhost:8080
    ```
    
    We see that we get the CentOS Apache server test page, albeit in text.
    
7. Exit our container shell:
    
    ```
    exit
    ```


```
[cloud_user@0e1f427e773c ~]$ podman ps -a
CONTAINER ID  IMAGE                                     COMMAND               CREATED         STATUS         PORTS               NAMES
2bfce50fca39  docker.io/centos/httpd-24-centos8:latest  /usr/bin/run-http...  50 seconds ago  Up 51 seconds  8080/tcp, 8443/tcp  cool_mccarthy
[cloud_user@0e1f427e773c ~]$ podman images
REPOSITORY                         TAG         IMAGE ID      CREATED      SIZE
docker.io/centos/httpd-24-centos8  latest      7d2fe0e482ba  5 years ago  441 MB
[cloud_user@0e1f427e773c ~]$ cat /etc/redhat-release
Red Hat Enterprise Linux release 9.7 (Plow)
[cloud_user@0e1f427e773c ~]$ podman exec -it 2bfce50fca39 /bin/bash
bash-4.4$ cat /etc/redhat-release
CentOS Linux release 8.2.2004 (Core)
bash-4.4$ curl http://localhost:8080
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE HTML>
<html lang="en">
  <head>
    <title>Apache HTTP Server Test Page powered by CentOS</title>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no"/>
    <link rel="shortcut icon" href="http://www.centos.org/favicon.ico"/>
    <link rel="stylesheet" media="all" href="noindex/common/css/bootstrap.min.css"/>
    <link rel="stylesheet" media="all" href="noindex/common/css/styles.css"/>
  </head>
  <body>
    <header class="container">
      <section class="row">
        <div class="header-graphic v3-banner platform-banner centos-banner" role="banner">
          <div class="graphic-inner">
            <div class="graphic-inner2">
              <div class="banner-title"><span>Apache HTTP Server</span></div>
            </div>
          </div>
        </div>
      </section>
    </header>
    <main class="container">
      <h1>Test Page</h1>
      <p class="lead">This page is used to test the proper operation of the <a href="http://apache.org">Apache HTTP server</a> after it has been installed. If you can read this page it means that this site is working properly. This server is powered by <a href="http://centos.org">CentOS</a>.</p>
      <hr/>
      <section class="row">
        <div class="col-md-6">
          <h2>Just visiting?</h2>
                                <p class="lead">The website you just visited is either experiencing problems or is undergoing routine maintenance.</p>
          <p>If you would like to let the administrators of this website know that you've seen this page instead of the page you expected, you should send them e-mail. In general, mail sent to the name "webmaster" and directed to the website's domain should reach the appropriate person.</p>
          <p>For example, if you experienced problems while visiting www.example.com, you should send e-mail to "webmaster@example.com".</p>
        <h2>Important note:</h2>
        <p class="lead">The CentOS Project has nothing to do with this website or its content, it just provides the software that makes the website run.</p>
        <p>If you have issues with the content of this site, contact the owner of the domain, not the CentOS project. Unless you intended to visit CentOS.org, the CentOS Project does not have anything to do with this website, the content or the lack of it.</p>
        <p>For example, if this website is www.example.com, you would find the owner of the example.com domain at the following WHOIS server: <a href="http://www.internic.net/whois.html">http://www.internic.net/whois.html</a></p>
        </div>
        <div class="col-md-6">
          <h2>Are you the Administrator?</h2>
          <p>You should add your website content to the directory <code>/var/www/html/</code>.</p>
          <p>To prevent this page from ever being used, follow the instructions in the file <code>/etc/httpd/conf.d/welcome.conf</code>.</p>
          <h2>Promoting Apache and CentOS</h2>
          <p>You are free to use the images below on Apache and CentOS Linux powered HTTP servers. Thanks for using Apache and CentOS!</p>
          <p>
            <a href="http://httpd.apache.org/">
              <img src="noindex/common/images/pb-apache.png" alt="[ Powered by Apache ]"/>
            </a>
            <a href="http://www.centos.org/">
              <img src="noindex/common/images/pb-centos.png" alt="[ Powered by CentOS Linux ]"/>
            </a>
          </p>
        </div>
        <div class="col-md-6">
        </div>
        <div class="col-md-6">
          <h2>The CentOS Project</h2>
          <p>The CentOS Linux distribution is a stable, predictable, manageable and reproduceable platform derived from the sources of Red Hat Enterprise Linux (RHEL).</p>
          <p>Additionally to being a popular choice for web hosting, CentOS also provides a rich platform for open source communities to build upon. For more information please visit the <a href="http://www.centos.org/">CentOS website</a>.</p>
        </div>
      </section>
      <hr/>
    </main>
    <footer class="container">
      <p>© 2019 The CentOS Project | <a href="https://www.centos.org/legal/">Legal</a> | <a href="https://www.centos.org/legal/privacy/">Privacy</a></p>
    </footer>
  </body>
</html>
bash-4.4$ exit
exit
[cloud_user@0e1f427e773c ~]$
```


1. Back in our server, try accessing the Apache web server running on port `8080` in our container using `curl`:
    
    ```
    curl http://localhost:8080
    ```
    
    We see that we can't access the Apache web server running in the container. This is because we haven't published those ports to the host. We're going to get into that in an upcoming lesson.
    
```
[cloud_user@0e1f427e773c ~]$ curl http://localhost:8080
curl: (7) Failed to connect to localhost port 8080: Connection refused
[cloud_user@0e1f427e773c ~]$
```

### Clean Up

1. Let's stop our container:
    
    ```
    podman stop <container_ID>
    ```
    
2. Let's check our containers again:
    
    ```
    podman ps -a
    ```
    
    We can see our container is stopped now. We're done with our container.
    
3. Remove the container. Be sure to paste in your container's ID:
    
    ```
    podman rm <container_ID>
    ```
    
4. Verify the container has been removed:
    
    ```
    podman ps -a
    ```
    
5. Let's check our container images again:
    
    ```
    podman images
    ```
    
6. Remove our `httpd-24-centos8:latest` image:
    
    ```
    podman rmi httpd-24-centos8:latest
    ```
    
7. Check our container images again:
    
    ```
    podman images
    ```
    

We're all cleaned up!


```
[cloud_user@0e1f427e773c ~]$ podman stop 2bfce50fca39
2bfce50fca39
[cloud_user@0e1f427e773c ~]$ podman ps -a
CONTAINER ID  IMAGE                                     COMMAND               CREATED        STATUS                    PORTS               NAMES
2bfce50fca39  docker.io/centos/httpd-24-centos8:latest  /usr/bin/run-http...  5 minutes ago  Exited (0) 7 seconds ago  8080/tcp, 8443/tcp  cool_mccarthy
[cloud_user@0e1f427e773c ~]$ podman rm 2bfce50fca39
2bfce50fca39
[cloud_user@0e1f427e773c ~]$ podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
[cloud_user@0e1f427e773c ~]$ podman images
REPOSITORY                         TAG         IMAGE ID      CREATED      SIZE
docker.io/centos/httpd-24-centos8  latest      7d2fe0e482ba  5 years ago  441 MB

[cloud_user@0e1f427e773c ~]$ podman rmi podman rmi httpd-24-centos8:latest
Untagged: docker.io/centos/httpd-24-centos8:latest
Deleted: 7d2fe0e482baf01e8a54f6c633bb2b5a89b3f35d278458c5ed73f3fdcc5646aa
Error: 2 errors occurred:
        * podman: image not known
        * rmi: image not known

[cloud_user@0e1f427e773c ~]$ podman rmi httpd-24-centos8:latest
Error: httpd-24-centos8:latest: image not known

[cloud_user@0e1f427e773c ~]$ podman images
REPOSITORY  TAG         IMAGE ID    CREATED     SIZE
[cloud_user@0e1f427e773c ~]$

```
## Conclusion

Great going, Cloud Guru! We just ran our first Podman container, interacted with it, and cleaned up after ourselves.


# error 

cloud_user@ip-10-0-1-4: ~ $     podman ps -a
bash: podman: command not found...
Failed to search for file: Failed to download gpg key for repo 'mariadb-main': Curl error (37): Couldn't read a file:// file for file:///etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY [Couldn't open file /etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY]
cloud_user@ip-10-0-1-4: ~ $ podman ps -a
bash: podman: command not found...
Failed to search for file: Failed to download gpg key for repo 'mariadb-main': Curl error (37): Couldn't read a file:// file for file:///etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY [Couldn't open file /etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY]
cloud_user@ip-10-0-1-4: ~ $

check the relationship with keylime read the callout
> [!NOTE]- FIX
> It looks like you've hit two separate issues here: **Podman isn't installed** (or isn't in your PATH), and your **YUM/DNF package manager is broken** because of a missing MariaDB GPG key.
> 
> When you type a command that doesn't exist, RHEL tries to search the repositories to see if it can suggest a package to install. Because the MariaDB repo is misconfigured, that search is crashing.
> 
> ---
> 
> ### Step 1: Fix the Package Manager
> 
> You need to disable the broken MariaDB repo so you can actually install Podman.
> 
> 1. **List your repos to find the exact name:**
>     
>     Bash
>     
>     ```
>     sudo dnf repolist
>     ```
>     
> 2. **Disable the broken MariaDB repo:** (Based on your error, it's likely named `mariadb-main`)
>     
>     Bash
>     
>     ```
>     sudo dnf config-manager --set-disabled mariadb-main
>     ```
>     
>     _If `config-manager` isn't found, you can manually edit the file in `/etc/yum.repos.d/` and change `enabled=1` to `enabled=0`._
>     
> 
> ---
> 
> ### Step 2: Install Podman
> 
> Now that the package manager isn't crashing, you can install the container tools.
> 
> Bash
> 
> ```
> sudo dnf install -y podman
> ```
> 
> Once that finishes, your `podman ps -a` command should work perfectly.
> 
> ---
> 
> ### Why is this happening?
> 
> - **The GPG Error:** Your system is trying to verify the "honesty" of the MariaDB packages using a security key (GPG). The repo file says the key is located at `/etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY`, but that file doesn't actually exist on your disk.
>     
> - **Command Not Found:** In RHEL/CentOS, there is a plugin called `Command Not Found`. When you typed `podman`, the system tried to be helpful and look up which package provides it. To do that, it refreshed the repo metadata, hit the broken MariaDB link, and threw the Curl error (37).
>     
> 
> ### Pro-Tip for your Lab
> 
> Since you are working with **Keylime** and **PQC** in this lab environment, you might actually need the entire "Container Tools" module, which includes Podman, Buildah, and Skopeo. You can install them all at once with:
> 
> Bash
> 
> ```
> sudo dnf module install -y container-tools
> ```
> 
> Give the repo disable command a shot first—it should clear up that annoying "Failed to search for file" error immediately!

%%


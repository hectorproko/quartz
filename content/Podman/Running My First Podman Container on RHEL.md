---
tags:
  - Podman
  - RHEL
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


<!--
### Post
Just finished my first hands-on lab with Podman on RHEL 9, and I’m impressed. If you’ve ever used Docker, the learning curve is almost nonexistent. The commands are nearly identical, podman run, podman ps, podman images, exec into a container, curling a web server, stopping, and cleaning up all feel very familiar.

What really stands out are the daemonless and rootless capabilities. They offer clear advantages in terms of reliability and security, and I’m starting to see why many organizations are preferring Podman over Docker.

Looking forward to diving deeper!

#Podman #Containers #RHEL #Linux #CloudNative #DevOps 

https://hectorproko.github.io/quartz/Podman/Running-My-First-Podman-Container-on-RHEL
-->


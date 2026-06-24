---
tags:
  - Podman
  - RHEL
quartz: "True"
linkedin: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---

## Overview

One of the key advantages Podman has over Docker is its native support for **pods**,  a concept borrowed directly from Kubernetes. A pod is a group of one or more containers that share the same network namespace, meaning they can communicate with each other over `localhost`. This is particularly useful for multi-container applications like WordPress, which requires both a web server and a database running together.

In this lab, I deployed a WordPress application as a Podman pod in two different security contexts:

- A **rootless pod** (`wp-pod`) -  running as a regular user for improved security
- A **rootfull pod** (`root-wp-pod`) -  running as root, which unlocks features like `pause/unpause`

I also practiced the full pod lifecycle: creating, inspecting, stopping, restarting, and cleaning up pods and their containers.

---

## Environment

This lab was performed on a **Red Hat Enterprise Linux 10.1 (Coughlan)** instance running on AWS EC2 (`t2.medium`). Podman comes pre-installed on RHEL, which makes it the natural container runtime choice in Red Hat environments.

```bash
$ cat /etc/os-release
NAME="Red Hat Enterprise Linux"
VERSION="10.1 (Coughlan)"
ID="rhel"
PRETTY_NAME="Red Hat Enterprise Linux 10.1 (Coughlan)"
```

---

## Part 1 -  Running a Rootless Pod

### Why Rootless?

Running containers without root privileges is a security best practice. If a container is ever compromised, the attacker cannot gain root access to the host. Podman makes this easy and seamless.

### Step 1: Verify a Clean Starting State

Before creating anything, I confirmed there were no existing pods or containers.

```bash
$ podman ps -a --pod
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS  PORTS  NAMES  POD ID  PODNAME

$ podman pod ps
POD ID  NAME  STATUS  CREATED  INFRA ID  # OF CONTAINERS
```

>The `--pod` flag tells `podman ps` to include two extra columns in the output: **POD ID** and **PODNAME**.

Both commands returned empty tables,  a clean slate.

### Step 2: Create the Pod

I created a pod named `wp-pod` and mapped port `80` inside the pod to port `8080` on the host. Port mapping is defined at the **pod level**, not the container level,  this is an important difference from standalone containers.

```bash
$ podman pod create --name wp-pod -p 8080:80
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1
```

Immediately after creation, Podman automatically creates an **infra container** to hold the pod's network namespace. This is what allows all containers in the pod to share the same network.

```bash
$ podman ps -a --pod
CONTAINER ID  IMAGE  COMMAND  CREATED         STATUS   PORTS                 NAMES               POD ID        PODNAME
a4bd5bd79862               19 seconds ago  Created  0.0.0.0:8080->80/tcp  31d1c0b8b575-infra  31d1c0b8b575  wp-pod
```

> **Key insight:** The infra container is created automatically. It exists solely to maintain the shared network namespace for the pod,  you never interact with it directly.

### Step 3: Add the MariaDB Database Container

I added a `mariadb` container to the pod, passing in database credentials via environment variables. Using `--pod=wp-pod` attaches it to the pod's shared network.

```bash
$ podman run -d --restart=always --pod=wp-pod \
  -e MYSQL_ROOT_PASSWORD="dbpass" \
  -e MYSQL_DATABASE="wp" \
  -e MYSQL_USER="wordpress" \
  -e MYSQL_PASSWORD="wppass" \
  --name=wp-db \
  docker.io/library/mariadb:latest
```

Podman pulled the image and started the container:

```
Trying to pull docker.io/library/mariadb:latest...
Getting image source signatures
Copying blob 9728cf7b6840 done   |
...
Writing manifest to image destination
98eae259ed310d92cfbccce6ade528f3f1d0a20381d86cd2ec63815a9ea18d49
```

Confirming `wp-db` is now part of the pod:

```bash
$ podman ps -a --pod
CONTAINER ID  IMAGE                             COMMAND   CREATED        STATUS        PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                              8 minutes ago  Up 4 minutes  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest  mariadbd  4 minutes ago  Up 4 minutes  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
```

### Step 4: Add the WordPress Web Container

Next, I added the WordPress container. Notice that `WORDPRESS_DB_HOST` is set to `127.0.0.1`,  because all containers in a pod share the same network namespace, the database is reachable on localhost.

```bash
$ podman run -d --restart=always --pod=wp-pod \
  -e WORDPRESS_DB_NAME="wp" \
  -e WORDPRESS_DB_USER="wordpress" \
  -e WORDPRESS_DB_PASSWORD="wppass" \
  -e WORDPRESS_DB_HOST="127.0.0.1" \
  --name wp-web \
  docker.io/library/wordpress:latest
```

All three containers (infra, wp-db, wp-web) are now running inside `wp-pod`:

```bash
$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            12 minutes ago  Up 8 minutes   0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              8 minutes ago   Up 8 minutes   0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  37 seconds ago  Up 37 seconds  0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod
```

### Step 5: Verify the Web Server is Accessible

I used `curl` to confirm the WordPress server was responding on port 8080. The `-s` flag suppresses the progress output. A successful response returns an empty body (WordPress redirects to setup), but an exit code of `0` confirms the connection succeeded.

```bash
$ curl -s http://localhost:8080

$ echo $?
0
```

Exit code `0` = success. The WordPress setup page was also verified by navigating to the public IP on port 8080 in a browser.

![[Pasted image 20260514103953.png|400]]

---

## Part 2 -  Running a Rootfull Pod

### Why Rootfull?

Some Podman operations,  like **pausing** a container,  require root because they depend on kernel cgroup features that are restricted for unprivileged users. The rootfull pod demonstrates these capabilities.

I switched to root and repeated the pod creation process, this time mapping port `80` to `8081`:

```bash
$ sudo -i

$ podman pod create --name root-wp-pod -p 8081:80
dfe66840c335294a3d47927da4c63179c5c4c3daaadfed5f84e910b8060d0340

$ podman pod ps
POD ID        NAME         STATUS   CREATED        INFRA ID      # OF CONTAINERS
dfe66840c335  root-wp-pod  Created  7 seconds ago  55ba8e7f8155  1
```

After adding both `root-wp-db` (MariaDB) and `wp-web` (WordPress) containers using the same approach as the rootless pod, the rootfull pod was up and verified:

```bash
$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
55ba8e7f8155                                                            9 minutes ago   Up 5 minutes   0.0.0.0:8081->80/tcp            dfe66840c335-infra  dfe66840c335  root-wp-pod
099860142a69  docker.io/library/mariadb:latest    mariadbd              5 minutes ago   Up 5 minutes   0.0.0.0:8081->80/tcp, 3306/tcp  root-wp-db          dfe66840c335  root-wp-pod
94adf6dd7eed  docker.io/library/wordpress:latest  apache2-foregroun...  32 seconds ago  Up 32 seconds  0.0.0.0:8081->80/tcp            wp-web              dfe66840c335  root-wp-pod

$ curl -s http://localhost:8081
$ echo $?
0
```

---

## Part 3 -  Managing the Rootless Pod (`wp-pod`)

### Stop, Start, and Restart

One of the key benefits of pods is that lifecycle commands apply to **all containers at once**. Rather than stopping each container individually, a single command manages the whole pod.

```bash
# Stop all containers in the pod
$ podman pod stop wp-pod

$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS                     PORTS                           NAMES
a4bd5bd79862  ...                                                        30 minutes ago  Exited (0) 10 seconds ago  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              26 minutes ago  Exited (0) 10 seconds ago  ...                             wp-db
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  18 minutes ago  Exited (0) 10 seconds ago  ...                             wp-web
```

All three containers transitioned to `Exited` simultaneously.

```bash
# Start them all back up
$ podman pod start wp-pod
wp-pod

# Restart (stop + start in one command)
$ podman pod restart wp-pod
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1
```

After restart, all containers show fresh uptime:

```bash
$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS
a4bd5bd79862  ...                                                        32 minutes ago  Up 10 seconds  0.0.0.0:8080->80/tcp
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              28 minutes ago  Up 10 seconds  0.0.0.0:8080->80/tcp, 3306/tcp
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  20 minutes ago  Up 9 seconds   0.0.0.0:8080->80/tcp
```

### Inspecting the Pod

`podman pod inspect` gives a detailed JSON view of the pod's configuration,  creation time, port bindings, cgroup paths, and the list of containers with their IDs and states.

```bash
$ podman pod inspect wp-pod | more
[
     {
          "Id": "31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1",
          "Name": "wp-pod",
          "Created": "2026-05-14T12:58:16.711215479Z",
          "State": "Running",
          "InfraConfig": {
               "PortBindings": {
                    "80/tcp": [{ "HostIp": "0.0.0.0", "HostPort": "8080" }]
               }
          },
          "NumContainers": 3,
          "Containers": [
               { "Id": "a4bd5bd7...", "Name": "31d1c0b8b575-infra", "State": "running" },
               { "Id": "98eae259...", "Name": "wp-db",              "State": "running" },
               { "Id": "06ee38a6...", "Name": "wp-web",             "State": "running" }
          ]
     }
]
```

`podman pod top` shows **all processes running** across every container in the pod,  useful for verifying services are actually running:

```bash
$ podman pod top wp-pod
USER      PID  PPID  %CPU  ELAPSED          TTY  TIME  COMMAND
0         1    0     0.000 5m0.709282336s   ?    0s    /catatonit -P
mysql     1    0     0.665 5m0.710179973s   ?    2s    mariadbd
root      1    0     0.000 4m59.71151296s   ?    0s    apache2 -DFOREGROUND
www-data  14   1     0.000 4m59.711601996s  ?    0s    apache2 -DFOREGROUND
www-data  15   1     0.000 4m59.711669913s  ?    0s    apache2 -DFOREGROUND
```

> **What I see here:** The infra container runs `catatonit` (a minimal init process), MariaDB runs as the `mysql` user, and Apache (WordPress) runs as `root` then forks worker processes as `www-data`. All of this is visible from a single command.

### Killing and Removing the Pod

When it's time to tear down a pod, `podman pod kill` sends a signal to **stop all** containers (forcing an immediate stop), and `podman pod rm` **removes the pod** and all its containers in one shot.

```bash
$ podman pod kill wp-pod
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1

$ podman pod ps
POD ID        NAME    STATUS  CREATED         INFRA ID      # OF CONTAINERS
31d1c0b8b575  wp-pod  Exited  39 minutes ago  a4bd5bd79862  3

$ podman pod rm wp-pod
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1

$ podman pod ps
POD ID  NAME  STATUS  CREATED  INFRA ID  # OF CONTAINERS
```

Pod and all containers are gone.

---

## Part 4 -  Managing the Rootfull Pod (`root-wp-pod`)

### Pause and Unpause (Rootfull Only)

This is a feature exclusive to rootfull containers. Pausing freezes all processes in the pod using Linux [[cgroups]],  the containers remain in memory but stop consuming CPU. This is useful for temporarily suspending workloads without losing their state.

```bash
$ podman pod pause root-wp-pod
dfe66840c335294a3d47927da4c63179c5c4c3daaadfed5f84e910b8060d0340

$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               STATUS   NAMES
55ba8e7f8155  ...                                                        Paused   dfe66840c335-infra
099860142a69  docker.io/library/mariadb:latest    mariadbd              Paused   root-wp-db
94adf6dd7eed  docker.io/library/wordpress:latest  apache2-foregroun...  Paused   wp-web
```

All three containers show `Paused`. Resuming is just as simple:

```bash
$ podman pod unpause root-wp-pod
dfe66840c335294a3d47927da4c63179c5c4c3daaadfed5f84e910b8060d0340
```

### Live Resource Statistics

`podman pod stats` streams real-time CPU and memory usage for every container in the pod,  similar to `docker stats` but at the pod level.

```bash
$ podman pod stats root-wp-pod
```

```
POD           CID           NAME                CPU %   MEM USAGE / LIMIT    MEM %   NET IO              BLOCK IO           PIDS
dfe66840c335  55ba8e7f8155  dfe66840c335-infra  0.00%   233.5kB / 3.828GB    0.01%   2.852kB / 1.842kB   -- / --            1
dfe66840c335  099860142a69  root-wp-db          0.26%   129.5MB / 3.828GB    3.38%   2.852kB / 1.842kB   6.085MB / 25.59MB  8
dfe66840c335  94adf6dd7eed  wp-web              0.06%   35.2MB / 3.828GB     0.92%   2.852kB / 1.842kB   0B / 84.74MB       7
```

> **What I see:** The database (`root-wp-db`) is the heaviest consumer at ~129 MB RAM and 0.26% CPU, while the WordPress Apache server uses only ~35 MB. The infra container is nearly invisible at 233 KB,  just holding the network namespace open.

---

## Part 5 -  Cleanup and Storage Reclamation

After stopping the pods, I used a two-step cleanup process to reclaim disk space.
<!--
`podman pod rm` deletes the pod **and all its containers** completely. Nothing is left behind from the pod itself.

What `prune` cleans up is different — the **container images** that were pulled to run those containers.
-->

### Step 1: `podman pod prune` -  Remove Stopped Pods

```bash
$ podman pod prune
WARNING! This will remove all stopped/exited pods..
Are you sure you want to continue? [y/N] y
```

This removes pod metadata and container layers, but images remain cached.

### Step 2: `podman system prune -a` -  Remove Everything Else

```bash
$ podman system prune -a
WARNING! This command removes:
        - all stopped containers
        - all networks not used by at least one container
        - all images without at least one container associated with them
        - all build cache

Are you sure you want to continue? [y/N] y
Deleted Images
8431e30f98d8c6e37655557914320bdc56eabdda0f18501fd31b4cfc9942d7ec
fe25111d923f83698454de718683c488c1166becc290c607df37b540d6ecfaa1
Total reclaimed space: 1.108GB
```

Verifying everything is clean:

```bash
$ podman system df
TYPE           TOTAL  ACTIVE  SIZE  RECLAIMABLE
Images         0      0       0B    0B (0%)
Containers     0      0       0B    0B (0%)
Local Volumes  0      0       0B    0B (0%)
```

1.1 GB reclaimed,  back to a clean state.

---

## Commands Summary

| Command                                                 | Purpose                                            |
| ------------------------------------------------------- | -------------------------------------------------- |
| `podman pod create --name <name> -p <host>:<container>` | Create a new pod with port mapping                 |
| `podman pod ps`                                         | List all pods                                      |
| `podman ps -a --pod`                                    | List all containers and their pod membership       |
| `podman run --pod=<name> ...`                           | Start a container inside a pod                     |
| `podman pod stop <name>`                                | Gracefully stop a pod and all its containers       |
| `podman pod start <name>`                               | Start a stopped pod and all its containers         |
| `podman pod restart <name>`                             | Restart a pod and all its containers               |
| `podman pod kill <name>`                                | Force-kill a pod and all its containers            |
| `podman pod rm <name>`                                  | Remove a pod and all its containers                |
| `podman pod pause <name>`                               | Pause all processes in a pod (rootfull only)       |
| `podman pod unpause <name>`                             | Resume a paused pod                                |
| `podman pod inspect <name>`                             | Show detailed JSON configuration of a pod          |
| `podman pod top <name>`                                 | Show all running processes across a pod            |
| `podman pod stats <name>`                               | Stream live CPU/memory stats for a pod             |
| `podman pod prune`                                      | Remove all stopped pods                            |
| `podman system prune -a`                                | Remove all unused containers, images, and networks |
| `podman system df`                                      | Show disk usage by images, containers, and volumes |

---

## Key Takeaways

- **Pods share a network namespace.** Containers within the same pod communicate over `localhost`, which is why `WORDPRESS_DB_HOST=127.0.0.1` works without any service discovery.
- **Port mapping belongs to the pod.** You define `-p` at pod creation time, not on individual containers. All containers in the pod share those published ports.
- **Rootless = better security.** Running pods without root limits the blast radius if a container is compromised.
- **Pause/unpause requires root.** Freezing processes via cgroups is a privileged operation not available to rootless containers.
- **Podman pods mirror Kubernetes pods.** The mental model,  shared network, infra container, multi-container lifecycle,  maps directly to how Kubernetes pods work, making this excellent preparation for k8s.

<!--
### Post

Continuing my Podman series, I recently published a hands-on guide on managing Pods with Podman on RHEL.

In this post, I explore how Podman implements the pod concept (one or more containers sharing the same namespace), just like Kubernetes. I walked through setting up a complete multi-container application, WordPress with a MariaDB database, running inside a Pod, tested in both rootless and rootful modes.

Coming from a Kubernetes and Docker background, the workflow felt very natural. What stood out to me was how rootful pods unlock extra capabilities (like pausing all processes in the pod) due to direct access to kernel cgroups, a great example of seeing theoretical concepts come to life.

Full article here: [https://hectorproko.github.io/quartz/Podman/Managing-Pods-Using-Podman-on-RHEL](https://hectorproko.github.io/quartz/Podman/Managing-Pods-Using-Podman-on-RHEL)

#Podman #Linux #Containers  #RHEL #DevOps
-->


%%

## Lab Overview

Unlike Docker’s single-container concept, Podman brings the ability to run multiple containers in a pod. In this lab, we will examine how to manage multiple containers as part of a pod. Upon completion of this lab, you will be able to use Podman to manage pods.

# Managing Pods Using Podman on RHEL

## Introduction

Unlike Docker’s single-container concept, Podman brings the ability to run multiple containers in a pod. In this lab, we will examine how to manage multiple containers as part of a pod. Upon completion of this lab, you will be able to use Podman to manage pods.

**Let's build some WordPress pods!**

In order to explore managing pods with Podman, we're going to stand up one rootless WordPress pod and one rootfull WordPress pod. We'll take a look at the most common `podman pod` operations used to manage Podman pods, and get hands-on practice doing it.

**Let's go!**

## Solution

When the lab starts, open an SSH connection to your lab instance, replacing `PUBLIC_IP_ADDRESS` with either the _Public IP_ or _DNS hostname_ of the instance:

```
ssh cloud_user@<PUBLIC_IP_ADDRESS>
```

The 'cloud_user' password has been provided with the credentials and instance information within the hands-on lab.

### Run a Rootless Pod

1. Check for existing rootless containers and pods using the `--pod` option to show the ID and name of the pod that the containers belong to:
    
    ```
    podman ps -a --pod
    ```
    
    There should not be any containers or pods listed.
    
2. Check for any existing pods:
    
    ```
    podman pod ps
    ```
    
    There should not be any pods listed.
    
3. Create a pod named `wp-pod`, with port `80` in the pod published to `8080` on the host:
    
    ```
    podman pod create --name wp-pod -p 8080:80
    ```
    
4. Check for any existing pods again:
    
    ```
    podman pod ps
    ```
    
    The `wp-pod` you just created should now be listed.
    
5. Check for any existing containers again:
    
    ```
    podman ps -a --pod
    ```
    
    The `infra` container created by default with the `wp-pod` should now be listed. The pod's published port are also listed, confirming that port `80` has been published to `8080`.
    
6. Add a `mariadb` container named `wp-db` (with a short name of `mariadb`) to the `wp-pod` pod with the following parameters and environment variables:
    
    - It is set to `--restart=always` 
    - The MYSQL_ROOT_PASSWORD is `dbpass` 
    - The MYSQL_DATABASE is `wp`
    - The MYSQL_USER is `wordpress`
    - The MYSQL_PASSWORD is `wppass`
    
    ```
  ❌  podman run -d --restart=always --pod=wp-pod -e MYSQL_ROOT_PASSWORD="dbpass" -e MYSQL_DATABASE="wp" -e MYSQL_USER="wordpress" -e MYSQL_PASSWORD="wppass" --name=wp-db mariadb
    ```

1. Check for containers again:
    
    ```
    podman ps -a --pod
    ```
    
    The `mariadb` container named `wp-db` you just started should now be listed.


2. Add a Wordpress container named `wp-web` (with a short name of `wordpress`) to the `wp-pod` pod with the following parameters and environment variables:
    
    - It is set to `--restart=always`
    - The WORDPRESS_DB_NAME is `wp`
    - The WORDPRESS_DB_USER is `wordpress`
    - The WORDPRESS_DB_PASSWORD is `wppass`
    - The WORDPRESS_DB_HOST is `127.0.0.1`
    
    ```
    podman run -d --restart=always --pod=wp-pod -e WORDPRESS_DB_NAME="wp" -e WORDPRESS_DB_USER="wordpress" -e WORDPRESS_DB_PASSWORD="wppass" -e WORDPRESS_DB_HOST="127.0.0.1" --name wp-web wordpress
    ```
  
3. Check for containers again:
    
    ```
    podman ps -a --pod
    ```
    
    The `wordpress` container named `wp-web` you just started should now be listed.

it is there as per the previous output


1. Using `curl`, check that the webserver on the Wordpress server is accessible via the published ports and is working properly:
    
    ```
    curl -s http://localhost:8080
    ```
    
    Nothing should be returned.
    
2. Immediately check the exit code:
    
    ```
    echo $?
    ```
    
    The exit code that is returned should be `0`, confirming that everything is working as expected.


1. In a web browser, navigate to the Public IP address or DNS hostname for the lab instance (provided with the lab credentials) on port 8080, and verify that the Wordpress page displays.
    

### Run a Rootfull Pod

1. Become the `root` user:
    
    ```
    sudo -i
    ```
    
    When prompted, enter the password for 'cloud_user' that was provided with the lab credentials.
    
2. Check for existing rootfull containers and pods using the `--pod` option:
    
    ```
    podman ps -a --pod
    ```
    
    There should not be any containers or pods listed.
    
3. Check for any existing pods:
    
    ```
    podman pod ps
    ```
    
    There should not be any pods listed.
    
4. Create a pod named `root-wp-pod`, with port `80` in the pod published to `8081` on the host:
    
    ```
    podman pod create --name root-wp-pod -p 8081:80
    ```
    
5. Check for any existing pods again:
    
    ```
    podman pod ps
    ```
    
    The `root-wp-pod` you just created should now be listed.
    
6. Check for any existing containers again:
    
    ```
    podman ps -a --pod
    ```
    
    The `infra` container created by default with the `root-wp-pod` should now be listed. The pod's published port are also listed, confirming that port `80` has been published to `8081`.
    
7. Add a `mariadb` container named `root-wp-db` (with a short name of `mariadb`) to the `root-wp-pod` pod with the following parameters and environment variables:
    
    - It is set to `--restart=always`
    - The MYSQL_ROOT_PASSWORD is `dbpass`
    - The MYSQL_DATABASE is `wp`
    - The MYSQL_USER is `wordpress`
    - The MYSQL_PASSWORD is `wppass`
    
    ```
    podman run -d --restart=always --pod=root-wp-pod -e MYSQL_ROOT_PASSWORD="dbpass" -e MYSQL_DATABASE="wp" -e MYSQL_USER="wordpress" -e MYSQL_PASSWORD="wppass" --name=root-wp-db mariadb
    ```
    
8. Check for containers again:
    
    ```
    podman ps -a --pod
    ```
    
    The `mariadb` container named `root-wp-db` you just started should now be listed.
    
9. Add a Wordpress container named `root-wp-web` (with a short name of `wordpress`) to the `root-wp-pod` pod with the following parameters and environment variables:
    
    - It is set to `--restart=always`
    - The WORDPRESS_DB_NAME is `wp`
    - The WORDPRESS_DB_USER is `wordpress`
    - The WORDPRESS_DB_PASSWORD is `wppass`
    - The WORDPRESS_DB_HOST is `127.0.0.1`
    
    ```
    podman run -d --restart=always --pod=root-wp-pod -e WORDPRESS_DB_NAME="wp" -e WORDPRESS_DB_USER="wordpress" -e WORDPRESS_DB_PASSWORD="wppass" -e WORDPRESS_DB_HOST="127.0.0.1" --name wp-web wordpress
    ```
    
10. Check for containers again:
    
    ```
    podman ps -a --pod
    ```
    
    The `wordpress` container named `root-wp-web` you just started should now be listed.
    
11. Using `curl`, check that the webserver on the Wordpress server is accessible via the published ports and is working properly:
    
    ```
    curl -s http://localhost:8081
    ```
    
    Nothing should be returned.
    
12. Immediately check the exit code:
    
    ```
    echo $?
    ```
    
    The exit code that is returned should be `0`, confirming that everything is working as expected.
    
13. In a web browser, navigate to the Public IP address or DNS hostname for the lab instance (provided with the lab credentials) on port 8081, and verify that the Wordpress page displays.
    
14. Log out as the `root` user:
    
    ```
    exit
    ```
    

### Manage a Rootless Pod

#### Stop, Start, and Restart the `wp-pod` Pod and Its Containers

1. Check for any rootless containers and pods that are currently running, using the `podman pod ps` and `podman ps -a --pod` commands. The `wp-pod` pod and its containers — `wp-web`, `wp-db`, and `infra` — should all be listed.
    
2. Stop the pod, along with all of its containers, using the `podman pod stop` command:
    
    ```
    podman pod stop wp-pod
    ```
    
3. Check the status of the containers and pods:
    
    ```
    podman ps -a --pod
    ```
    
    The `wp-pod` pod and the `wp-web`, `wp-db`, and `infra` containers should all be listed as `exited`, or stopped.
    
4. Start the pod and its containers again, using the `podman pod start` command:
    
    ```
    podman pod start wp-pod
    ```
    
5. Check the status of the containers and pods again:
    
    ```
    podman ps -a --pod
    ```
    
    The `wp-pod` pod and the `wp-web`, `wp-db`, and `infra` containers should all be listed as `up` or `running`.
    
6. Restart the pod and its containers, using the `podman pod restart` command:
    
    ```
    podman pod restart wp-pod
    ```
    
7. Check the status of the containers and pods again:
    
    ```
    podman ps -a --pod
    ```
    
    The `wp-pod` pod and the `wp-web`, `wp-db`, and `infra` containers should all be listed as `up` or `running`.
    

#### Get Information About the `wp-pod` Pod and Its Containers

1. Get information about the pod using the `podman pod inspect` command:
    
    ```
    podman pod inspect wp-pod | more
    ```
    
    This provides detailed information about the `wp-pod` pod, such as when it was created, its current status, its containers, and the IDs, names, and states of those containers.
    
2. View the pod's processes, using the `podman pod top` command:
    
    ```
    podman pod top wp-pod
    ```
    
    This returns all the processes associated with the pod.
    

#### Clean Up the `wp-pod` Pod and Its Containers

1. Check for any existing rootless containers and pods, using the `podman pod ps` and `podman ps -a --pod` commands. The `wp-pod` pod and its containers — `wp-web`, `wp-db`, and `infra` — should all be listed.
    
2. Kill the `wp-pod` pod and all of its containers:
    
    ```
    podman pod kill wp-pod
    ```
    
3. Check the status of the pod and its containers, using the `podman pod ps` and `podman ps -a --pod` commands. The `wp-pod` pod and its containers — `wp-web`, `wp-db`, and `infra` — should all be listed as `exited`.
    
4. Remove the `wp-pod` pod and all of its containers:
    
    ```
    podman pod rm wp-pod
    ```
    
5. Check the status of the pod and its containers again, using the `podman pod ps` and `podman ps -a --pod` commands. The `wp-pod` pod and its containers — `wp-web`, `wp-db`, and `infra` — should all be listed as `removed`.
    
6. Check the storage utilization for the system:
    
    ```
    podman system df
    ```
    
    It shows that there is some space that is reclaimable.
    
7. Clean up and reclaim the available storage space, using the `podman pod prune` command:
    
    ```
    podman system prune -a
    ```
    
    When prompted, enter `y` to continue.
    
8. Check the storage utilization again:
    
    ```
    podman system df
    ```
    
    It now shows that there is no longer any reclaimable space.
    

### Manage a Rootfull Pod

#### Pause and Resume the `root-wp-pod` Pod and Its Containers

1. Become the `root` user:
    
    ```
    sudo -i
    ```
    
    When prompted, enter the password for 'cloud_user' that was provided with the lab credentials.
    
2. Check for any rootfull containers and pods that are currently running, using the `podman pod ps` and `podman ps -a --pod` commands. The `root-wp-pod` pod and its containers — `root-wp-web`, `root-wp-db`, and `infra` — should all be listed.
    
3. Pause the pod, along with all of its containers, using the `podman pod pause` command:
    
    ```
    podman pod pause root-wp-pod
    ```
    
4. Check the status of the pod and its containers again, using the `podman pod ps` and `podman ps -a --pod` commands. The `root-wp-pod` pod and its containers — `root-wp-web`, `root-wp-db`, and `infra` — should all be listed as `paused`.
    
5. Unpause the pod, along with all of its containers, using the `podman pod unpause` command:
    
    ```
    podman pod unpause root-wp-pod
    ```
    
6. Check the status of the pod and its containers again, using the `podman pod ps` and `podman ps -a --pod` commands. The `root-wp-pod` pod and the `root-wp-web`, `root-wp-db`, and `infra` containers should all be listed as `up` or `running`.
    

#### Get Information About the `root-wp-pod` Pod and Its Containers

1. Get performance statistics about the pod using the `podman pod stats` command:
    
    ```
    podman pod stats root-wp-pod
    ```
    
    This provides near real-time resource utilization for the `root-wp-pod` pod and its containers, such as CPU percentage, memory usage, etc.
    
2. Press **Ctrl+C** to exit the resource utilization view.
    

#### Stop and Clean Up the `root-wp-pod` Pod and Its Containers

1. Stop the pod and its containers, using the `podman pod stop` command:
    
    ```
    podman pod stop root-wp-pod
    ```
    
2. Check for any reclaimable space:
    
    ```
    podman system df
    ```
    
    It shows that there is some space that is reclaimable.
    
3. Clean up and reclaim the available storage space, using the `podman pod prune` command:
    
    ```
    podman pod prune
    ```
    
    When prompted, enter `y` to continue.
    
4. Check the storage utilization again:
    
    ```
    podman system df
    ```
    
    It only cleaned up space being utilized in `Containers` and `Local Volumes`, but not `Images`.
    
5. Clean up and reclaim the remaining available storage space, using the `podman system prune` command:
    
    ```
    podman system prune -a
    ```
    
    When prompted, enter `y` to continue.
    
6. Check the storage utilization again:
    
    ```
    podman system df
    ```
    
    It now shows that there is no longer any reclaimable space.
    
7. Check the status of the pod and its containers one last time, using the `podman pod ps` and `podman ps -a --pod` commands. There are no pods or containers listed.
    

## Conclusion

Congratulations — you've completed this hands-on lab!

# REDOING aWS

AWS Redhat has podman installed

**ec2 instance**
Instance ID
C i-0d8aa1f5bc8e32ce3

IPv6 address

Private IPv4 addresses
172.31.41.38

Public DNS
ec2-13-222-203-88.compute-1.amazonaws.com | open address 7

Hostname type
IP name: ip-172-31-41-38.ec2.internal

Answer private resource DNS name
IPv4 (A)

Auto-assigned IP address
13.222.203.88 [Public IP]

Public IPv4 address
13.222.203.88 | open address 7

Instance state
Running

Private IP DNS name (IPv4 only)
G ip-172-31-41-38.ec2.internal

Instance type
t2.medium

VPC ID
vpc-0e07f47f32342fb65 7

Elastic IP addresses

AWS Compute Optimizer finding

need to open these ports
![[Pasted image 20260514105512.png]]


```
[ec2-user@ip-172-31-41-38 ~]$ cat /etc/os-release
NAME="Red Hat Enterprise Linux"
VERSION="10.1 (Coughlan)"
ID="rhel"
ID_LIKE="centos fedora"
VERSION_ID="10.1"
PLATFORM_ID="platform:el10"
PRETTY_NAME="Red Hat Enterprise Linux 10.1 (Coughlan)"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:redhat:enterprise_linux:10.1"
HOME_URL="https://www.redhat.com/"
VENDOR_NAME="Red Hat"
VENDOR_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10"
BUG_REPORT_URL="https://issues.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 10"
REDHAT_BUGZILLA_PRODUCT_VERSION=10.1
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="10.1"
[ec2-user@ip-172-31-41-38 ~]$
```

my output
```
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES       POD ID      PODNAME
[ec2-user@ip-172-31-41-38 ~]$ podman pod ps
POD ID      NAME        STATUS      CREATED     INFRA ID    # OF CONTAINERS

[ec2-user@ip-172-31-41-38 ~]$ podman pod create --name wp-pod -p 8080:80
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1

[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE       COMMAND     CREATED         STATUS      PORTS                 NAMES               POD ID        PODNAME
a4bd5bd79862                          19 seconds ago  Created     0.0.0.0:8080->80/tcp  31d1c0b8b575-infra  31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$
```

creating mariadb

```
podman run -d --restart=always --pod=wp-pod \
  -e MYSQL_ROOT_PASSWORD="dbpass" \
  -e MYSQL_DATABASE="wp" \
  -e MYSQL_USER="wordpress" \
  -e MYSQL_PASSWORD="wppass" \
  --name=wp-db \
  docker.io/library/mariadb:latest
```

not i see the infra container

```
[ec2-user@ip-172-31-41-38 ~]$ podman run -d --restart=always --pod=wp-pod \
  -e MYSQL_ROOT_PASSWORD="dbpass" \
  -e MYSQL_DATABASE="wp" \
  -e MYSQL_USER="wordpress" \
  -e MYSQL_PASSWORD="wppass" \
  --name=wp-db \
  docker.io/library/mariadb:latest
Trying to pull docker.io/library/mariadb:latest...
Getting image source signatures
Copying blob 9728cf7b6840 done   |
Copying blob b40150c1c271 done   |
Copying blob 79b59e88f775 done   |
Copying blob 3a08fbfd5bee done   |
Copying blob d5c8079c2cfb done   |
Copying blob 70e28d1970b2 done   |
Copying blob be8de6e97fbd done   |
Copying blob 4f12367291a4 done   |
Copying config 8431e30f98 done   |
Writing manifest to image destination
98eae259ed310d92cfbccce6ade528f3f1d0a20381d86cd2ec63815a9ea18d49

[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                             COMMAND     CREATED        STATUS        PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                8 minutes ago  Up 4 minutes  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest  mariadbd    4 minutes ago  Up 4 minutes  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$
```

creating the **wordpress** container


```
[ec2-user@ip-172-31-41-38 ~]$ podman run -d --restart=always --pod=wp-pod \
  -e WORDPRESS_DB_NAME="wp" \
  -e WORDPRESS_DB_USER="wordpress" \
  -e WORDPRESS_DB_PASSWORD="wppass" \
  -e WORDPRESS_DB_HOST="127.0.0.1" \
  --name wp-web \
  docker.io/library/wordpress:latest
Trying to pull docker.io/library/wordpress:latest...
Getting image source signatures
Copying blob d0b8ca23768a done   |
Copying blob 57fb71246055 done   |
Copying blob 218ef6f07cc8 done   |
Copying blob 55c83a3a3836 done   |
Copying blob 4c445e431149 done   |
Copying blob 44505f1315cf done   |
Copying blob e61b9621690d done   |
Copying blob ba5f325922e0 done   |
Copying blob 331aed9a0eed done   |
Copying blob 7e5a7f2359a5 done   |
Copying blob 8f09c012430d done   |
Copying blob 8e4127ce68fd done   |
Copying blob 8dccb04f243a done   |
Copying blob 963d38cf195b done   |
Copying blob 4f4fb700ef54 done   |
Copying blob 05971c21802f done   |
Copying blob c200b78618ef done   |
Copying blob 40afff4f5c63 done   |
Copying blob d419ca11f42d done   |
Copying blob 0bdfc03ec09a done   |
Copying blob 7baa44f26561 done   |
Copying blob d75a21f56232 done   |
Copying blob a3b1586265b1 done   |
Copying blob 0576cb355e51 done   |
Copying config fe25111d92 done   |
Writing manifest to image destination
06ee38a6aba0dbd33c2e03141254809fc26d8e50b5bd0e754b3f66381673e979
[ec2-user@ip-172-31-41-38 ~]$
```

```
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            12 minutes ago  Up 8 minutes   0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              8 minutes ago   Up 8 minutes   0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  37 seconds ago  Up 37 seconds  0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod

[ec2-user@ip-172-31-41-38 ~]$ curl -s http://localhost:8080

[ec2-user@ip-172-31-41-38 ~]$ echo $?
0
[ec2-user@ip-172-31-41-38 ~]$
```

public ip does not match because i restarted the ec2 and the ip  reset
![[Pasted image 20260514103953.png]]
#### run a rootfull pod

```
[ec2-user@ip-172-31-41-38 ~]$ sudo -i

[root@ip-172-31-41-38 ~]# podman ps -a --pod
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES       POD ID      PODNAME
[root@ip-172-31-41-38 ~]# podman pod ps
POD ID      NAME        STATUS      CREATED     INFRA ID    # OF CONTAINERS

[root@ip-172-31-41-38 ~]# podman pod create --name root-wp-pod -p 8081:80
dfe66840c335294a3d47927da4c63179c5c4c3daaadfed5f84e910b8060d0340

[root@ip-172-31-41-38 ~]# podman pod ps
POD ID        NAME         STATUS      CREATED        INFRA ID      # OF CONTAINERS
dfe66840c335  root-wp-pod  Created     7 seconds ago  55ba8e7f8155  1

[root@ip-172-31-41-38 ~]# podman ps -a --pod
CONTAINER ID  IMAGE       COMMAND     CREATED         STATUS      PORTS                 NAMES               POD ID        PODNAME
55ba8e7f8155                          34 seconds ago  Created     0.0.0.0:8081->80/tcp  dfe66840c335-infra  dfe66840c335  root-wp-pod
[root@ip-172-31-41-38 ~]#
```

**mariadb**



```    
podman run -d --restart=always \
  --pod=root-wp-pod -e \
  MYSQL_ROOT_PASSWORD="dbpass" \
  -e MYSQL_DATABASE="wp" \
  -e MYSQL_USER="wordpress" \
  -e MYSQL_PASSWORD="wppass" \
  --name=root-wp-db docker.io/library/mariadb:latest
```

```
[root@ip-172-31-41-38 ~]# podman ps -a --pod
CONTAINER ID  IMAGE                             COMMAND     CREATED             STATUS             PORTS                           NAMES               POD ID        PODNAME
55ba8e7f8155                                                6 minutes ago       Up About a minute  0.0.0.0:8081->80/tcp            dfe66840c335-infra  dfe66840c335  root-wp-pod
099860142a69  docker.io/library/mariadb:latest  mariadbd    About a minute ago  Up About a minute  0.0.0.0:8081->80/tcp, 3306/tcp  root-wp-db          dfe66840c335  root-wp-pod
[root@ip-172-31-41-38 ~]#
```

adding **wordpress** container

```
podman run -d --restart=always --pod=root-wp-pod \
  -e WORDPRESS_DB_NAME="wp" \
  -e WORDPRESS_DB_USER="wordpress" \
  -e WORDPRESS_DB_PASSWORD="wppass" -e \
  WORDPRESS_DB_HOST="127.0.0.1" \
  --name wp-web docker.io/library/wordpress:latest
```

```
[root@ip-172-31-41-38 ~]# podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
55ba8e7f8155                                                            9 minutes ago   Up 5 minutes   0.0.0.0:8081->80/tcp            dfe66840c335-infra  dfe66840c335  root-wp-pod
099860142a69  docker.io/library/mariadb:latest    mariadbd              5 minutes ago   Up 5 minutes   0.0.0.0:8081->80/tcp, 3306/tcp  root-wp-db          dfe66840c335  root-wp-pod
94adf6dd7eed  docker.io/library/wordpress:latest  apache2-foregroun...  32 seconds ago  Up 32 seconds  0.0.0.0:8081->80/tcp            wp-web              dfe66840c335  root-wp-pod
[root@ip-172-31-41-38 ~]# curl -s http://localhost:8081
[root@ip-172-31-41-38 ~]# echo $?
0
[root@ip-172-31-41-38 ~]# exit
logout
[ec2-user@ip-172-31-41-38 ~]$
```

![[Pasted image 20260514105158.png]]

#### Manage a Rootless Pod

```
[ec2-user@ip-172-31-41-38 ~]$ podman pod ps
POD ID        NAME        STATUS      CREATED         INFRA ID      # OF CONTAINERS
31d1c0b8b575  wp-pod      Running     29 minutes ago  a4bd5bd79862  3
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            29 minutes ago  Up 26 minutes  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              26 minutes ago  Up 26 minutes  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  18 minutes ago  Up 18 minutes  0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$ podman pod stop wp-pod
wp-pod
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS                     PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            30 minutes ago  Exited (0) 10 seconds ago  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              26 minutes ago  Exited (0) 10 seconds ago  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  18 minutes ago  Exited (0) 10 seconds ago  0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$ podman pod start wp-pod
wp-pod
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            31 minutes ago  Up 30 seconds  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              28 minutes ago  Up 30 seconds  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  20 minutes ago  Up 30 seconds  0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$ podman pod restart wp-pod
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            32 minutes ago  Up 10 seconds  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              28 minutes ago  Up 10 seconds  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  20 minutes ago  Up 9 seconds   0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$
```

getting information about **wp-pod**

```
[ec2-user@ip-172-31-41-38 ~]$ podman pod inspect wp-pod | more
[
     {
          "Id": "31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1",
          "Name": "wp-pod",
          "Created": "2026-05-14T12:58:16.711215479Z",
          "CreateCommand": [
               "podman",
               "pod",
               "create",
               "--name",
               "wp-pod",
               "-p",
               "8080:80"
          ],
          "ExitPolicy": "continue",
          "State": "Running",
          "Hostname": "",
          "CreateCgroup": true,
          "CgroupParent": "user.slice",
          "CgroupPath": "user.slice/user-1000.slice/user@1000.service/user.slice/user-libpod_pod_31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1.slic
e",
          "CreateInfra": true,
          "InfraContainerID": "a4bd5bd79862f551be6b4762e96d04e44bd2b9bdb2f4e8ae91becdc2ae13c8e4",
          "InfraConfig": {
               "PortBindings": {
                    "80/tcp": [
                         {
                              "HostIp": "0.0.0.0",
                              "HostPort": "8080"
                         }
                    ]
               },
               "HostNetwork": false,
               "StaticIP": "",
               "StaticMAC": "",
               "NoManageResolvConf": false,
               "DNSServer": null,
               "DNSSearch": null,
               "DNSOption": null,
               "NoManageHostname": false,
               "NoManageHosts": false,
               "HostAdd": null,
               "HostsFile": "",
               "Networks": null,
               "NetworkOptions": null,
               "pid_ns": "private",
               "userns": "host",
               "uts_ns": "private"
          },
          "SharedNamespaces": [
               "ipc",
               "net",
               "uts"
          ],
          "NumContainers": 3,
          "Containers": [
               {
                    "Id": "a4bd5bd79862f551be6b4762e96d04e44bd2b9bdb2f4e8ae91becdc2ae13c8e4",
                    "Name": "31d1c0b8b575-infra",
                    "State": "running"
               },
               {
                    "Id": "98eae259ed310d92cfbccce6ade528f3f1d0a20381d86cd2ec63815a9ea18d49",
                    "Name": "wp-db",
                    "State": "running"
               },
               {
                    "Id": "06ee38a6aba0dbd33c2e03141254809fc26d8e50b5bd0e754b3f66381673e979",
                    "Name": "wp-web",
                    "State": "running"
               }
          ],
          "LockNumber": 0
     }
]
[ec2-user@ip-172-31-41-38 ~]$ podman pod top wp-pod
USER        PID         PPID        %CPU        ELAPSED          TTY         TIME        COMMAND
0           1           0           0.000       5m0.709282336s   ?           0s          /catatonit -P
mysql       1           0           0.665       5m0.710179973s   ?           2s          mariadbd
root        1           0           0.000       4m59.71151296s   ?           0s          apache2 -DFOREGROUND
www-data    14          1           0.000       4m59.711601996s  ?           0s          apache2 -DFOREGROUND
www-data    15          1           0.000       4m59.711669913s  ?           0s          apache2 -DFOREGROUND
www-data    16          1           0.000       4m59.711732474s  ?           0s          apache2 -DFOREGROUND
www-data    17          1           0.000       4m59.711796211s  ?           0s          apache2 -DFOREGROUND
www-data    18          1           0.000       4m59.711856816s  ?           0s          apache2 -DFOREGROUND
[ec2-user@ip-172-31-41-38 ~]$
```

**clean up** the **wp-pod**

```
[ec2-user@ip-172-31-41-38 ~]$ podman pod ps
POD ID        NAME        STATUS      CREATED         INFRA ID      # OF CONTAINERS
31d1c0b8b575  wp-pod      Running     38 minutes ago  a4bd5bd79862  3
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS        PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            39 minutes ago  Up 6 minutes  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              35 minutes ago  Up 6 minutes  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  27 minutes ago  Up 6 minutes  0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$ podman pod kill wp-pod
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1
[ec2-user@ip-172-31-41-38 ~]$ podman pod ps
POD ID        NAME        STATUS      CREATED         INFRA ID      # OF CONTAINERS
31d1c0b8b575  wp-pod      Exited      39 minutes ago  a4bd5bd79862  3
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS                       PORTS                           NAMES               POD ID        PODNAME
a4bd5bd79862                                                            39 minutes ago  Exited (137) 22 seconds ago  0.0.0.0:8080->80/tcp            31d1c0b8b575-infra  31d1c0b8b575  wp-pod
98eae259ed31  docker.io/library/mariadb:latest    mariadbd              36 minutes ago  Exited (137) 22 seconds ago  0.0.0.0:8080->80/tcp, 3306/tcp  wp-db               31d1c0b8b575  wp-pod
06ee38a6aba0  docker.io/library/wordpress:latest  apache2-foregroun...  27 minutes ago  Exited (137) 22 seconds ago  0.0.0.0:8080->80/tcp            wp-web              31d1c0b8b575  wp-pod
[ec2-user@ip-172-31-41-38 ~]$ podman pod rm wp-pod
31d1c0b8b5759ffbafd3d41138fde8432de2383484e3013c37354ae46435d9c1
[ec2-user@ip-172-31-41-38 ~]$ podman pod ps
POD ID      NAME        STATUS      CREATED     INFRA ID    # OF CONTAINERS
[ec2-user@ip-172-31-41-38 ~]$ podman ps -a --pod
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES       POD ID      PODNAME
[ec2-user@ip-172-31-41-38 ~]$ podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         2           0           1.108GB     1.108GB (100%)
Containers     0           0           0B          0B (0%)
Local Volumes  0           0           0B          0B (0%)
[ec2-user@ip-172-31-41-38 ~]$ podman pod prune
WARNING! This will remove all stopped/exited pods..
Are you sure you want to continue? [y/N] y
[ec2-user@ip-172-31-41-38 ~]$ podman system prune -a
WARNING! This command removes:
        - all stopped containers
        - all networks not used by at least one container
        - all images without at least one container associated with them
        - all build cache

Are you sure you want to continue? [y/N] y
Deleted Images
8431e30f98d8c6e37655557914320bdc56eabdda0f18501fd31b4cfc9942d7ec
fe25111d923f83698454de718683c488c1166becc290c607df37b540d6ecfaa1
Total reclaimed space: 1.108GB
[ec2-user@ip-172-31-41-38 ~]$ podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         0           0           0B          0B (0%)
Containers     0           0           0B          0B (0%)
Local Volumes  0           0           0B          0B (0%)
[ec2-user@ip-172-31-41-38 ~]$
```

#### Manage a Rootfull Pod

```
[ec2-user@ip-172-31-41-38 ~]$ sudo -i
[root@ip-172-31-41-38 ~]# podman pod ps
POD ID        NAME         STATUS      CREATED         INFRA ID      # OF CONTAINERS
dfe66840c335  root-wp-pod  Running     28 minutes ago  55ba8e7f8155  3
[root@ip-172-31-41-38 ~]# podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS         PORTS                           NAMES               POD ID        PODNAME
55ba8e7f8155                                                            28 minutes ago  Up 23 minutes  0.0.0.0:8081->80/tcp            dfe66840c335-infra  dfe66840c335  root-wp-pod
099860142a69  docker.io/library/mariadb:latest    mariadbd              23 minutes ago  Up 23 minutes  0.0.0.0:8081->80/tcp, 3306/tcp  root-wp-db          dfe66840c335  root-wp-pod
94adf6dd7eed  docker.io/library/wordpress:latest  apache2-foregroun...  19 minutes ago  Up 19 minutes  0.0.0.0:8081->80/tcp            wp-web              dfe66840c335  root-wp-pod
[root@ip-172-31-41-38 ~]# podman pod pause root-wp-pod
dfe66840c335294a3d47927da4c63179c5c4c3daaadfed5f84e910b8060d0340
[root@ip-172-31-41-38 ~]# podman pod ps
POD ID        NAME         STATUS      CREATED         INFRA ID      # OF CONTAINERS
dfe66840c335  root-wp-pod  Paused      28 minutes ago  55ba8e7f8155  3
[root@ip-172-31-41-38 ~]# podman ps -a --pod
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS      PORTS                           NAMES               POD ID        PODNAME
55ba8e7f8155                                                            28 minutes ago  Paused      0.0.0.0:8081->80/tcp            dfe66840c335-infra  dfe66840c335  root-wp-pod
099860142a69  docker.io/library/mariadb:latest    mariadbd              24 minutes ago  Paused      0.0.0.0:8081->80/tcp, 3306/tcp  root-wp-db          dfe66840c335  root-wp-pod
94adf6dd7eed  docker.io/library/wordpress:latest  apache2-foregroun...  19 minutes ago  Paused      0.0.0.0:8081->80/tcp            wp-web              dfe66840c335  root-wp-pod
[root@ip-172-31-41-38 ~]# podman pod unpause root-wp-pod
dfe66840c335294a3d47927da4c63179c5c4c3daaadfed5f84e910b8060d0340
[root@ip-172-31-41-38 ~]#
```

getting information about **root-wp-pod**

*auto updates*

    podman pod stats root-wp-pod
```
POD           CID           NAME                CPU %       MEM USAGE/ LIMIT   MEM %       NET IO             BLOCK IO           PIDS
dfe66840c335  55ba8e7f8155  dfe66840c335-infra  0.00%       233.5kB / 3.828GB  0.01%       2.852kB / 1.842kB  -- / --            1
dfe66840c335  099860142a69  root-wp-db          0.26%       129.5MB / 3.828GB  3.38%       2.852kB / 1.842kB  6.085MB / 25.59MB  8
dfe66840c335  94adf6dd7eed  wp-web              0.06%       35.2MB / 3.828GB   0.92%       2.852kB / 1.842kB  0B / 84.74MB       7
```

stop clean **root-wp-pod**

```
[root@ip-172-31-41-38 ~]# podman pod stop root-wp-pod
root-wp-pod
[root@ip-172-31-41-38 ~]# podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         2           2           1.108GB     0B (0%)
Containers     3           0           57.91kB     57.91kB (100%)
Local Volumes  2           2           232.2MB     0B (0%)
[root@ip-172-31-41-38 ~]# podman pod prune
WARNING! This will remove all stopped/exited pods..
Are you sure you want to continue? [y/N] y
dfe66840c335294a3d47927da4c63179c5c4c3daaadfed5f84e910b8060d0340
[root@ip-172-31-41-38 ~]# podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         2           0           1.108GB     1.108GB (100%)
Containers     0           0           0B          0B (0%)
Local Volumes  0           0           0B          0B (0%)
[root@ip-172-31-41-38 ~]# podman system prune -a
WARNING! This command removes:
        - all stopped containers
        - all networks not used by at least one container
        - all images without at least one container associated with them
        - all build cache

Are you sure you want to continue? [y/N] y
Deleted Images
8431e30f98d8c6e37655557914320bdc56eabdda0f18501fd31b4cfc9942d7ec
fe25111d923f83698454de718683c488c1166becc290c607df37b540d6ecfaa1
Total reclaimed space: 1.108GB
[root@ip-172-31-41-38 ~]# podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         0           0           0B          0B (0%)
Containers     0           0           0B          0B (0%)
Local Volumes  0           0           0B          0B (0%)
[root@ip-172-31-41-38 ~]# podman pod ps
POD ID      NAME        STATUS      CREATED     INFRA ID    # OF CONTAINERS
[root@ip-172-31-41-38 ~]# podman ps -a --pod
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES       POD ID      PODNAME
[root@ip-172-31-41-38 ~]#
```

%%


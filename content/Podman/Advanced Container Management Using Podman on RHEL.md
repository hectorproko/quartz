---
tags:
  - Podman
  - RHE
quartz: "True"
linkedin: "False"
pluralsight: "True"
---
## Overview

In this lab I went beyond the basics of running containers and practiced the day-to-day management skills that are actually needed in production environments. The lab covered four key areas: **persistent storage**, **container networking**, **health monitoring**, and **system resource management**. Each section built on the last, and by the end I had a realistic multi-container web server setup running on RHEL with shared volumes, published ports, health checks, and a cleaned-up system.

---

## Environment

**Platform:** Red Hat Enterprise Linux (RHEL 10.1) on AWS EC2 (t2.medium)  
**Tool:** Podman 5.6.0  

The environment used Podman running in **rootless mode**,  meaning containers run without root privileges, which is a key security advantage Podman has over Docker. All container data was stored under the user's home directory rather than in system-level paths.

---

## Part 1 - Container Storage

### Why Storage Matters

By default, any data written inside a container is lost when the container is removed. Persistent storage solves this by attaching external storage,  either a host directory or a named volume,  to the container's filesystem. I practiced both approaches.

---

### Method 1: Bind Mount (Host Directory)

**What I did:** I created a directory on the host, put some test files in it, and mounted it directly into three nginx containers.

**Why:** This is useful when you want to share files from the host machine with one or more containers. Changes made on the host are immediately visible inside the containers and vice versa.

```bash
# Create the directory and test files on the host
mkdir ~/html
echo "Test File 1" > ~/html/test1.txt
echo "Test File 2" > ~/html/test2.txt
echo "Test File 3" > ~/html/test3.txt

# Launch 3 nginx containers with the host directory mounted
# The :z flag relabels the volume for SELinux compatibility on RHEL
podman run -dt --name web1 --volume ~/html:/usr/share/nginx/html:z nginx
podman run -dt --name web2 --volume ~/html:/usr/share/nginx/html:z nginx
podman run -dt --name web3 --volume ~/html:/usr/share/nginx/html:z nginx

# Verify the files are accessible inside each container
podman exec -i web1 curl -s http://localhost/test1.txt
podman exec -i web2 curl -s http://localhost/test2.txt
podman exec -i web3 curl -s http://localhost/test3.txt
```

> **Note on the `:z` flag:** On RHEL, SELinux enforces strict access controls. Without `:z`, the container process would be denied access to the mounted directory because it wouldn't carry the right SELinux context label. The `:z` flag tells Podman to relabel the volume so all containers sharing it can access it safely.

**Output:**

```
Resolved "nginx" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/nginx:latest...
Getting image source signatures
Copying blob 030365c1a354 done   |
...
61c1837d575da66f01342d29b780a671354aeb62511fe0967a0e73234ec072c1

$ podman exec -i web1 curl -s http://localhost/test1.txt
Test File 1

$ podman exec -i web2 curl -s http://localhost/test2.txt
Test File 2

$ podman exec -i web3 curl -s http://localhost/test3.txt
Test File 3
```

Each container served its own file through nginx,  confirming the host directory was successfully mounted inside all three.

After testing I cleaned up:

```bash
podman stop -a   # Stop all running containers
podman rm -a     # Remove all containers
podman ps -a     # Verify no containers remain
```

**Output after cleanup:**

```
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
```

An empty table means all containers are gone,  but the files in `~/html` are still on the host, unaffected.

---

### Method 2: Named Volume

**What I did:** I created a Podman-managed named volume called `webvol`, attached it to three nginx containers, then copied files into it through one container and confirmed all three containers could see them.

**Why:** Named volumes are managed by Podman and stored in a standard location (`~/.local/share/containers/storage/volumes/`). They're easier to reference by name, more portable, and better suited for container-to-container data sharing than bind mounts tied to specific host paths.

```bash
# Create the named volume
podman volume create webvol

# Verify it exists
podman volume ls

# Launch 3 containers using the named volume
podman run -dt --name web1 --volume webvol:/usr/share/nginx/html:z nginx
podman run -dt --name web2 --volume webvol:/usr/share/nginx/html:z nginx
podman run -dt --name web3 --volume webvol:/usr/share/nginx/html:z nginx

# Copy files into the volume via web1
podman cp ~/html/test1.txt web1:/usr/share/nginx/html
podman cp ~/html/test2.txt web1:/usr/share/nginx/html
podman cp ~/html/test3.txt web1:/usr/share/nginx/html

# Verify all 3 containers see the same files
podman exec web1 ls -la /usr/share/nginx/html/
podman exec web2 ls -la /usr/share/nginx/html/
podman exec web3 ls -la /usr/share/nginx/html/
```

**Output - all three containers show the same directory listing:**

```
total 20
drwxr-xr-x. 2 root root  91 May 14 15:40 .
drwxr-xr-x. 3 root root  18 May 13 19:05 ..
-rw-r--r--. 1 root root 497 May 13 12:43 50x.html
-rw-r--r--. 1 root root 896 May 13 12:43 index.html
-rw-r--r--. 1 root root  12 May 14 15:34 test1.txt
-rw-r--r--. 1 root root  12 May 14 15:34 test2.txt
-rw-r--r--. 1 root root  12 May 14 15:35 test3.txt
```

All three containers shared the same `webvol` volume, so writing files through one container made them immediately visible in all others. This confirms the named volume is acting as shared persistent storage.

---

## Part 2 - Container Networking

### Why Port Publishing Matters

By default, containers are isolated,  their internal ports are not accessible from the host or the outside world. Port publishing (`--publish`) creates a mapping between a host port and a container port, making services accessible from outside the container.

**What I did:** I launched the same three nginx containers again, this time publishing each one on a unique host port (8081, 8082, 8083) so they could be reached directly on the host.

```bash
podman run -dt --name web1 --publish 8081:80 --volume webvol:/usr/share/nginx/html:z nginx
podman run -dt --name web2 --publish 8082:80 --volume webvol:/usr/share/nginx/html:z nginx
podman run -dt --name web3 --publish 8083:80 --volume webvol:/usr/share/nginx/html:z nginx

# Verify containers and their port mappings
podman ps -a
podman port -a

# Access each web server directly from the host
curl -s http://localhost:8081/test1.txt
curl -s http://localhost:8082/test2.txt
curl -s http://localhost:8083/test3.txt
```

**Output - `podman ps -a` shows the port bindings:**

```
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS         PORTS                 NAMES
1bd9514338fd  docker.io/library/nginx:latest  nginx -g daemon o...  20 seconds ago  Up 20 seconds  0.0.0.0:8081->80/tcp  web1
c349ed716ddf  docker.io/library/nginx:latest  nginx -g daemon o...  14 seconds ago  Up 14 seconds  0.0.0.0:8082->80/tcp  web2
272314006cf5  docker.io/library/nginx:latest  nginx -g daemon o...  7 seconds ago   Up 7 seconds   0.0.0.0:8083->80/tcp  web3
```

**Output - `podman port -a` lists the port mappings:**

```
1bd9514338fd    80/tcp -> 0.0.0.0:8081
c349ed716ddf    80/tcp -> 0.0.0.0:8082
272314006cf5    80/tcp -> 0.0.0.0:8083
```

**Output - curl from the host reaches each container:**

```
$ curl -s http://localhost:8081/test1.txt
Test File 1

$ curl -s http://localhost:8082/test2.txt
Test File 2

$ curl -s http://localhost:8083/test3.txt
Test File 3
```

The `0.0.0.0` in the port binding means the containers are listening on all network interfaces of the host,  so they'd be reachable not just on `localhost` but also via the EC2 public IP (assuming the security group allows it).

---

## Part 3 - Container Health Checks

### Why Health Checks Matter

Just because a container is _running_ doesn't mean its application is actually _working_. A health check defines a command Podman runs periodically inside the container to verify the service is functioning. If the command fails, the container is marked unhealthy.

**What I did:** I relaunched the three web containers with a health check that runs `curl http://localhost` inside the container. If nginx is up and responding, the curl succeeds (exit code 0). If not, it exits with code 1, triggering an unhealthy status.

```bash
podman run -dt --name web1 \
  --publish 8081:80 \
  --volume webvol:/usr/share/nginx/html:z \
  --health-cmd 'curl http://localhost || exit 1' \
  --health-interval=0 nginx

podman run -dt --name web2 \
  --publish 8082:80 \
  --volume webvol:/usr/share/nginx/html:z \
  --health-cmd 'curl http://localhost || exit 1' \
  --health-interval=0 nginx

podman run -dt --name web3 \
  --publish 8083:80 \
  --volume webvol:/usr/share/nginx/html:z \
  --health-cmd 'curl http://localhost || exit 1' \
  --health-interval=0 nginx

# Manually trigger the health check on each container
podman healthcheck run web1
podman healthcheck run web2
podman healthcheck run web3
```

> **Note on `--health-interval=0`:** Setting the interval to `0` disables automatic periodic health check execution. This means the check only runs when you manually trigger it with `podman healthcheck run`. In production you'd set a real interval (e.g., `--health-interval=30s`) so Podman continuously monitors container health.

**Output - checking exit codes to confirm health:**

When `podman healthcheck run` returns silently with no error, the service passed. I confirmed this by checking the exit code with `echo $?`:

```
$ podman healthcheck run web1
$ echo $?
0

$ podman healthcheck run web2
$ echo $?
0

$ podman healthcheck run web3
$ echo $?
0
```

An exit code of `0` means the health check command (`curl http://localhost`) succeeded,  nginx responded correctly inside the container. A non-zero exit code would indicate the container is unhealthy.

You can also observe the health status in `podman ps -a`,  while health checks are still initializing, it shows `(starting)`. Once passed, it changes to `(healthy)`.

```
CONTAINER ID  ...  STATUS                        PORTS                 NAMES
216af5b5b26e  ...  Up About a minute (starting)  0.0.0.0:8081->80/tcp  web1
cb8fdb778bb0  ...  Up About a minute (starting)  0.0.0.0:8082->80/tcp  web2
ca8349cf4b4f  ...  Up 19 seconds (starting)      0.0.0.0:8083->80/tcp  web3
```

---

## Part 4 - System Management

### Inspecting the Podman Environment

With containers running, I explored tools for understanding the overall state of the Podman host.

**System info** gives a full picture of the **runtime environment** - kernel version, OCI runtime, security settings, network backend, storage driver, and more:

```bash
podman system info | more
```

**Key details from my output:**

```yaml
host:
  arch: amd64
  distribution:
    distribution: rhel
    version: "10.1"
  networkBackend: netavark        # Modern network backend replacing CNI
  ociRuntime:
    name: crun                    # Lightweight OCI runtime written in C
  security:
    rootless: true                # Confirms containers run without root
    selinuxEnabled: true          # SELinux is active - explains the :z flag
  os: linux
version:
  Version: 5.6.0
```

The `rootless: true` and `selinuxEnabled: true` entries confirm why the `:z` volume flag was needed earlier,  SELinux is enforcing access controls, and rootless mode means containers never run as the system root user.

---

### Monitoring Disk Usage

**What I did:** I used `podman system df` to track how much disk space images, containers, and volumes were consuming,  including what could be reclaimed.

```bash
podman system df        # Summary view
podman system df -v     # Verbose/detailed view
```

**Output - before pulling an extra image:**

```
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         1           1           164.9MB     0B (0%)
Containers     3           3           89.46kB     0B (0%)
Local Volumes  1           3           1.429kB     0B (0%)
```

Everything is active,  nothing can be reclaimed yet.

**After pulling the `httpd` image** (which is unused by any container):

```
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         2           1           204MB       120.2MB (59%)
Containers     3           3           89.46kB     0B (0%)
Local Volumes  1           3           1.429kB     0B (0%)
```

Now 120.2MB is reclaimable because the httpd image isn't attached to any running container.

**Verbose mode** shows per-image and per-container breakdown:

```bash
[ec2-user@ip-172-31-46-11 ~]$ podman system df -v
Images space usage:

REPOSITORY               TAG         IMAGE ID      CREATED     SIZE        SHARED SIZE  UNIQUE SIZE  CONTAINERS
docker.io/library/nginx  latest      6f8edba05e38  21 hours    164.9MB     81.04MB      83.81MB      3
docker.io/library/httpd  latest      c194ed9b9e8f  5 days      120.2MB     81.04MB      39.18MB      0

Containers space usage:

CONTAINER ID  IMAGE         COMMAND               LOCAL VOLUMES  SIZE        CREATED     STATUS      NAMES
216af5b5b26e  6f8edba05e38  nginx -g daemon off;  1              29.82kB     10 minutes  running     web1
cb8fdb778bb0  6f8edba05e38  nginx -g daemon off;  1              29.82kB     10 minutes  running     web2
ca8349cf4b4f  6f8edba05e38  nginx -g daemon off;  1              29.82kB     9 minutes   running     web3

Local Volumes space usage:

VOLUME NAME  LINKS       SIZE
webvol       3           1.429kB
```

The `SHARED SIZE` column is interesting,  both images share 81MB of base layers. Podman only stores those layers once on disk, which is why the combined size isn't simply 164.9MB + 120.2MB.
<!--
did not understand these verbose thing
-->
---

### Pruning Unused Resources

```bash
podman system prune -a
```

**Output:**

```
WARNING! This command removes:
        - all stopped containers
        - all networks not used by at least one container
        - all images without at least one container associated with them
        - all build cache

Are you sure you want to continue? [y/N] y
Deleted Images
c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a
Total reclaimed space: 120.2MB
```

**After prune:**

```
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         1           1           164.9MB     0B (0%)
Containers     3           3           89.46kB     0B (0%)
Local Volumes  1           3           1.429kB     0B (0%)
```

The unused `httpd` image was removed and 120.2MB was freed. The nginx image remained because it's still being used by the running containers.

---

### Monitoring Events

Podman logs all container and image activity as events. I used `podman events` to review recent activity.

```bash
# View all events from the last 10 minutes
podman events --since 10m

# Filter for only prune events
podman events --since 15m --filter event=prune
```

**Output - recent events:**

```
2026-05-14 15:55:18 UTC container health_status cb8fdb778bb0 (name=web2, health_status=healthy)
2026-05-14 15:55:28 UTC container health_status ca8349cf4b4f (name=web3, health_status=healthy)
2026-05-14 16:01:16 UTC image pull-error  httpd ^C
2026-05-14 16:02:18 UTC image pull c194ed9b9e8f docker.io/library/httpd:latest
2026-05-14 16:03:30 UTC image untag c194ed9b9e8f docker.io/library/httpd:latest
2026-05-14 16:03:30 UTC image remove c194ed9b9e8f
```

**The prune filter returned nothing** - Found out  `podman system prune` is a meta-command that internally triggers individual `remove` events for each resource it cleans up. There is no single `prune` event type in the log. To see what was removed, I had to filter for `event=remove` instead:

```bash
podman events --since 15m --filter event=remove
```

```
2026-05-14 16:03:30 UTC image remove c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a
```

---

## Key Takeaways

|Concept|What I Learned|
|---|---|
|**Bind Mounts**|Mount a host directory directly into containers,  fast and simple for development|
|**Named Volumes**|Podman-managed persistent storage,  better for production, easily shared across containers|
|**SELinux + `:z` flag**|Required on RHEL to allow containers access to mounted volumes under SELinux enforcement|
|**Port Publishing**|`--publish host:container` exposes container services on the host network|
|**Health Checks**|Containers can report healthy/unhealthy based on a custom command; exit code 0 = healthy|
|**`podman system df`**|Shows disk usage with reclaimable space broken out per image/container/volume|
|**`podman system prune`**|Cleans unused images, stopped containers, and networks,  does not log a `prune` event|
|**`podman events`**|Full audit trail of all Podman activity,  health checks, image pulls, removes, and more|
|**Rootless Mode**|Podman runs containers as a normal user a core security advantage over Docker|

---

## Skills Demonstrated

- Podman container lifecycle management (run, stop, rm, ps)
- Persistent storage with bind mounts and named volumes
- Multi-container shared storage patterns
- Container port publishing and host-level networking
- Health check configuration and manual validation
- Podman system monitoring, disk usage analysis, and cleanup
- Event log filtering and auditing
- Operating on RHEL with SELinux-aware configuration


[[Lab Advanced Container Management Using Podman on RHEL]]

<!--Original Draft
### Post

Continuing with my Podman series, in this hands-on lab I dove into deeper concepts.

I worked on persistent storage using both bind mounts and named volumes, published container ports (fun fact: I always thought -p stood for “port”, but it actually publishes them), implemented healthchecks to properly verify the application status, learned how to inspect and monitor the Podman host, tracked disk space consumption of images, containers, and volumes, and cleaned them up to reclaim space when needed. I also explored how to check the audit trail of Podman activity.

All of this was done in rootless mode, focusing on real-world operational skills.

https://hectorproko.github.io/quartz/Podman/Advanced-Container-Management-Using-Podman-on-RHEL


#Podman #RHEL #Containers #DevOps #Linux



---------------------------------------------------------------


# DRAFT

# Advanced Container Management Using Podman on RHEL

### Lab Overview

Let’s take our container skills to the next level! In this lab, we’re going to step up our container management by adding storage, networking, monitoring, and more. Upon completion of this lab, you will be able to perform the foundational container management activities required for day-to-day container management.
### Introduction

Take your container skills to the next level! In this lab, you will add storage, networking, monitoring, and more. Upon completion of this lab, you will be able to perform the foundational container management activities required for day-to-day container management.

### Solution

When the lab starts, open an SSH connection to your lab instance. Replace `PUBLIC_IP_ADDRESS` with either the _Public IP_ or _DNS_ of the instance:

```
ssh cloud_user@<PUBLIC_IP_ADDRESS>
```

The `cloud_user` password has been provided with the credentials and instance information within the hands-on lab.

### Working With Container Storage

For this objective, you will work with Podman container storage.

#### Container Storage Using a Directory

1. Create a directory named `~/html`:
    
    ```
    mkdir ~/html
    ```
    
2. Create 3 text files in your new `~/html` directory. Name these files `test1.txt`, `test2.txt`, and `test3.txt`. Make the contents "Test File 1", "Test File 2", and "Test File 3". Use the following command to create the first file:
    
    ```
    echo "Test File 1" > ~/html/test1.txt
    ```
    
3. Create the second file:
    
    ```
    echo "Test File 2" > ~/html/test2.txt
    ```
    
4. Create the third file:
    
    ```
    echo "Test File 3" > ~/html/test3.txt
    ```
    
5. Next, start 3 `nginx` containers. Name them `web1`, `web2`, and `web3`. Attach the `~/html` directory to the container at `/usr/share/nginx/html/`. Use the following command to start the first container:
    
    ```
    podman run -dt --name web1 --volume ~/html:/usr/share/nginx/html:z nginx
    ```
    
6. Start the second container:
    
    ```
    podman run -dt --name web2 --volume ~/html:/usr/share/nginx/html:z nginx
    ```
    
7. Start the third container:
    
    ```
    podman run -dt --name web3 --volume ~/html:/usr/share/nginx/html:z nginx
    ```
    
8. Run a `curl` command inside each container to test. Pull the first text file:
    
    ```
    podman exec -i web1 curl -s http://localhost/test1.txt
    ```
    
9. Pull the second file:
    
    ```
    podman exec -i web2 curl -s http://localhost/test2.txt
    ```
    
10. Pull the third file:
    
    ```
    podman exec -i web3 curl -s http://localhost/test3.txt
    ```
    
    You should see all 3 files displayed.
    
11. Stop the 3 containers:
    
    ```
    podman stop -a
    ```
    
12. Remove the 3 containers:
    
    ```
    podman rm -a
    ```
    
13. Check to ensure you have no containers:
    
    ```
    podman ps -a
    ```
    

my output:
```
[ec2-user@ip-172-31-46-11 ~]$     mkdir ~/html
[ec2-user@ip-172-31-46-11 ~]$     echo "Test File 1" > ~/html/test1.txt
[ec2-user@ip-172-31-46-11 ~]$     echo "Test File 2" > ~/html/test2.txt
[ec2-user@ip-172-31-46-11 ~]$     echo "Test File 3" > ~/html/test3.txt
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web1 --volume ~/html:/usr/share/nginx/html:z nginx
Resolved "nginx" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/nginx:latest...
Getting image source signatures
Copying blob 030365c1a354 done   |
Copying blob 57fb71246055 done   |
Copying blob 834a05acfff4 done   |
Copying blob 60ad0c2ccfc6 done   |
Copying blob 1bb34ee717a4 done   |
Copying blob 9f7bde970101 done   |
Copying blob 735e1c628373 done   |
Copying config 6f8edba05e done   |
Writing manifest to image destination
61c1837d575da66f01342d29b780a671354aeb62511fe0967a0e73234ec072c1
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web2 --volume ~/html:/usr/share/nginx/html:z nginx
59969daa442658a76ca75b92181688c8c926084e2d4ba3267c1d6cae5819521f
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web3 --volume ~/html:/usr/share/nginx/html:z nginx
775a7939711ed1e26acc7f36ab594488d843a9bc73c29aa90d26dbd807220aab
[ec2-user@ip-172-31-46-11 ~]$     podman exec -i web1 curl -s http://localhost/test1.txt
Test File 1
[ec2-user@ip-172-31-46-11 ~]$     podman exec -i web2 curl -s http://localhost/test2.txt
Test File 2
[ec2-user@ip-172-31-46-11 ~]$     podman exec -i web3 curl -s http://localhost/test3.txt
Test File 3
[ec2-user@ip-172-31-46-11 ~]$     podman stop -a
61c1837d575da66f01342d29b780a671354aeb62511fe0967a0e73234ec072c1
59969daa442658a76ca75b92181688c8c926084e2d4ba3267c1d6cae5819521f
775a7939711ed1e26acc7f36ab594488d843a9bc73c29aa90d26dbd807220aab
[ec2-user@ip-172-31-46-11 ~]$     podman rm -a
775a7939711ed1e26acc7f36ab594488d843a9bc73c29aa90d26dbd807220aab
61c1837d575da66f01342d29b780a671354aeb62511fe0967a0e73234ec072c1
59969daa442658a76ca75b92181688c8c926084e2d4ba3267c1d6cae5819521f
[ec2-user@ip-172-31-46-11 ~]$     podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
[ec2-user@ip-172-31-46-11 ~]$
```

#### Container Storage Using a Container Volume

1. Create a new container storage volume named `webvol`:
    
    ```
    podman volume create webvol
    ```
    
2. Run the following command to list your volumes:
    
    ```
    podman volume ls
    ```
    
    You should see the `webvol` volume is successfully created.
    
3. Start 3 `nginx` containers. Name them `web1`, `web2`, and `web3`. Attach the `webvol` volume to the container at `/usr/share/nginx/html/`. Run the following command to start the first container:
    
    ```
    podman run -dt --name web1 --volume webvol:/usr/share/nginx/html:z nginx
    ```
    
4. Start the second container:
    
    ```
    podman run -dt --name web2 --volume webvol:/usr/share/nginx/html:z nginx
    ```
    
5. Start the third container:
    
    ```
    podman run -dt --name web3 --volume webvol:/usr/share/nginx/html:z nginx
    ```
    
6. Verify the containers were created:
    
    ```
    podman ps -a
    ```
    
    The 3 containers should be listed.
    
7. Copy the 3 text files from `~/html` into the `web1` container to the `/usr/share/nginx/html/` directory. Run the following command to copy the first file:
    
    ```
    podman cp ~/html/test1.txt web1:/usr/share/nginx/html
    ```
    
8. Copy the second file:
    
    ```
    podman cp ~/html/test2.txt web1:/usr/share/nginx/html
    ```
    
9. Copy the third file:
    
    ```
    podman cp ~/html/test3.txt web1:/usr/share/nginx/html
    ```
    
10. Run an `ls` command inside of each container to list the contents of the `/usr/share/nginx/html/` directory. Run the following command on the first container:
    
    ```
    podman exec web1 ls -la /usr/share/nginx/html/
    ```
    
11. Repeat the command on the second container:
    
    ```
    podman exec web2 ls -la /usr/share/nginx/html/
    ```
    
12. Repeat the command on the third container:
    
    ```
    podman exec web3 ls -la /usr/share/nginx/html/
    ```
    
    All 3 files should yield the same results. All containers see the same data.
    
13. Run a `curl` command inside each container to test. Try pulling each of your text files. Run the following command to pull the first file:
    
    ```
    podman exec web1 curl -s http://localhost/test1.txt
    ```
    
14. Pull the second file:
    
    ```
    podman exec web2 curl -s http://localhost/test2.txt
    ```
    
15. Pull the third file:
    
    ```
    podman exec web3 curl -s http://localhost/test3.txt
    ```
    
    You should see all 3 files displayed.
    
16. Stop the 3 containers:
    
    ```
    podman stop -a
    ```
    
17. Remove the 3 containers:
    
    ```
    podman rm -a
    ```
    
18. Check to ensure you have no containers:
    
    ```
    podman ps -a
    ```
    

my output
```
[ec2-user@ip-172-31-46-11 ~]$     podman volume create webvol
webvol
[ec2-user@ip-172-31-46-11 ~]$     podman volume ls
DRIVER      VOLUME NAME
local       webvol
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web1 --volume webvol:/usr/share/nginx/html:z nginx
65fdc0f08b101c2e275bfd04358f04735a71055cb7dd1915b795f203db107fd4
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web2 --volume webvol:/usr/share/nginx/html:z nginx
253f126755b0af5d65227b4a7ed9d6afb4500438fdc7cf35520068941c05a54b
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web3 --volume webvol:/usr/share/nginx/html:z nginx
93323d95d7d0c4cbfbd61ab9d0bd80c169b200f0cc934dad3e30ca4de0040753
[ec2-user@ip-172-31-46-11 ~]$     podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS         PORTS       NAMES
65fdc0f08b10  docker.io/library/nginx:latest  nginx -g daemon o...  24 seconds ago  Up 24 seconds  80/tcp      web1
253f126755b0  docker.io/library/nginx:latest  nginx -g daemon o...  15 seconds ago  Up 16 seconds  80/tcp      web2
93323d95d7d0  docker.io/library/nginx:latest  nginx -g daemon o...  7 seconds ago   Up 8 seconds   80/tcp      web3
[ec2-user@ip-172-31-46-11 ~]$     podman cp ~/html/test1.txt web1:/usr/share/nginx/html
[ec2-user@ip-172-31-46-11 ~]$     podman cp ~/html/test2.txt web1:/usr/share/nginx/html
[ec2-user@ip-172-31-46-11 ~]$     podman cp ~/html/test3.txt web1:/usr/share/nginx/html
[ec2-user@ip-172-31-46-11 ~]$     podman exec web1 ls -la /usr/share/nginx/html/
total 20
drwxr-xr-x. 2 root root  91 May 14 15:40 .
drwxr-xr-x. 3 root root  18 May 13 19:05 ..
-rw-r--r--. 1 root root 497 May 13 12:43 50x.html
-rw-r--r--. 1 root root 896 May 13 12:43 index.html
-rw-r--r--. 1 root root  12 May 14 15:34 test1.txt
-rw-r--r--. 1 root root  12 May 14 15:34 test2.txt
-rw-r--r--. 1 root root  12 May 14 15:35 test3.txt
[ec2-user@ip-172-31-46-11 ~]$     podman exec web2 ls -la /usr/share/nginx/html/
total 20
drwxr-xr-x. 2 root root  91 May 14 15:40 .
drwxr-xr-x. 3 root root  18 May 13 19:05 ..
-rw-r--r--. 1 root root 497 May 13 12:43 50x.html
-rw-r--r--. 1 root root 896 May 13 12:43 index.html
-rw-r--r--. 1 root root  12 May 14 15:34 test1.txt
-rw-r--r--. 1 root root  12 May 14 15:34 test2.txt
-rw-r--r--. 1 root root  12 May 14 15:35 test3.txt
[ec2-user@ip-172-31-46-11 ~]$     podman exec web3 ls -la /usr/share/nginx/html/
total 20
drwxr-xr-x. 2 root root  91 May 14 15:40 .
drwxr-xr-x. 3 root root  18 May 13 19:05 ..
-rw-r--r--. 1 root root 497 May 13 12:43 50x.html
-rw-r--r--. 1 root root 896 May 13 12:43 index.html
-rw-r--r--. 1 root root  12 May 14 15:34 test1.txt
-rw-r--r--. 1 root root  12 May 14 15:34 test2.txt
-rw-r--r--. 1 root root  12 May 14 15:35 test3.txt
[ec2-user@ip-172-31-46-11 ~]$     podman exec web1 curl -s http://localhost/test1.txt
Test File 1
[ec2-user@ip-172-31-46-11 ~]$     podman exec web2 curl -s http://localhost/test2.txt
Test File 2
[ec2-user@ip-172-31-46-11 ~]$     podman exec web3 curl -s http://localhost/test3.txt
Test File 3
[ec2-user@ip-172-31-46-11 ~]$     podman stop -a
253f126755b0af5d65227b4a7ed9d6afb4500438fdc7cf35520068941c05a54b
93323d95d7d0c4cbfbd61ab9d0bd80c169b200f0cc934dad3e30ca4de0040753
65fdc0f08b101c2e275bfd04358f04735a71055cb7dd1915b795f203db107fd4
[ec2-user@ip-172-31-46-11 ~]$     podman rm -a
93323d95d7d0c4cbfbd61ab9d0bd80c169b200f0cc934dad3e30ca4de0040753
253f126755b0af5d65227b4a7ed9d6afb4500438fdc7cf35520068941c05a54b
65fdc0f08b101c2e275bfd04358f04735a71055cb7dd1915b795f203db107fd4
[ec2-user@ip-172-31-46-11 ~]$     podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
[ec2-user@ip-172-31-46-11 ~]$
```
### Configuring Container Networking

For this objective, you will make your `nginx` web servers available outside of their containers.

#### Publish Your Web Servers' Ports

1. Start 3 `nginx` containers. Name them `web1`, `web2`, and `web3`. Attach the `webvol` volume to the container at `/usr/share/nginx/html/`. Publish the containers' port `80` to `8081` (`web1`), `8082` (`web2`), and `8083` (`web3`). Run the following command for the first container:
    
    ```
    podman run -dt --name web1 --publish 8081:80 --volume webvol:/usr/share/nginx/html:z nginx
    ```
    
2. Repeat the command for the second container:
    
    ```
    podman run -dt --name web2 --publish 8082:80 --volume webvol:/usr/share/nginx/html:z nginx
    ```
    
3. Repeat the command for the third container:
    
    ```
    podman run -dt --name web3 --publish 8083:80 --volume webvol:/usr/share/nginx/html:z nginx
    ```
    
4. Verify the containers were created:
    
    ```
    podman ps -a
    ```
    
    You should see all 3 containers.
    
5. Check the published container ports:
    
    ```
    podman port -a
    ```
    
    You should see all 3 ports.
    
6. Use a `curl` command to pull the first test text file from the first web server container:
    
    ```
    curl -s http://localhost:8081/test1.txt
    ```
    
7. Pull the second test text file:
    
    ```
    curl -s http://localhost:8082/test2.txt
    ```
    
8. Pull the third test text file:
    
    ```
    curl -s http://localhost:8083/test3.txt
    ```
    
    You should see your test text files via the 3 `nginx` containers.
    
9. Stop the 3 containers:
    
    ```
    podman stop -a
    ```
    
10. Remove the 3 containers:
    
    ```
    podman rm -a
    ```
    
11. Check to ensure you have no containers:
    
    ```
    podman ps -a
    ```
    

You are now ready to set up some container healthchecks.

my output
```
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web1 --publish 8081:80 --volume webvol:/usr/share/nginx/html:z nginx
1bd9514338fdaa484595842119413ac7beff0c9dbffe2d71c8623e682de600d3
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web2 --publish 8082:80 --volume webvol:/usr/share/nginx/html:z nginx
c349ed716ddfc6ee19794fe05926be3fb8c8a468b861df4a414acd72dfca27f0
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web3 --publish 8083:80 --volume webvol:/usr/share/nginx/html:z nginx
272314006cf5a94f61aec62ed07fb5267fafe3d3eb520d50f54eb3beb22d0dba
[ec2-user@ip-172-31-46-11 ~]$     podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS         PORTS                 NAMES
1bd9514338fd  docker.io/library/nginx:latest  nginx -g daemon o...  20 seconds ago  Up 20 seconds  0.0.0.0:8081->80/tcp  web1
c349ed716ddf  docker.io/library/nginx:latest  nginx -g daemon o...  14 seconds ago  Up 14 seconds  0.0.0.0:8082->80/tcp  web2
272314006cf5  docker.io/library/nginx:latest  nginx -g daemon o...  7 seconds ago   Up 7 seconds   0.0.0.0:8083->80/tcp  web3
[ec2-user@ip-172-31-46-11 ~]$     podman port -a
1bd9514338fd    80/tcp -> 0.0.0.0:8081
c349ed716ddf    80/tcp -> 0.0.0.0:8082
272314006cf5    80/tcp -> 0.0.0.0:8083
[ec2-user@ip-172-31-46-11 ~]$     curl -s http://localhost:8081/test1.txt
Test File 1
[ec2-user@ip-172-31-46-11 ~]$     curl -s http://localhost:8082/test2.txt
Test File 2
[ec2-user@ip-172-31-46-11 ~]$     curl -s http://localhost:8083/test3.txt
Test File 3
[ec2-user@ip-172-31-46-11 ~]$     podman stop -a
1bd9514338fdaa484595842119413ac7beff0c9dbffe2d71c8623e682de600d3
c349ed716ddfc6ee19794fe05926be3fb8c8a468b861df4a414acd72dfca27f0
272314006cf5a94f61aec62ed07fb5267fafe3d3eb520d50f54eb3beb22d0dba
[ec2-user@ip-172-31-46-11 ~]$     podman rm -a
272314006cf5a94f61aec62ed07fb5267fafe3d3eb520d50f54eb3beb22d0dba
1bd9514338fdaa484595842119413ac7beff0c9dbffe2d71c8623e682de600d3
c349ed716ddfc6ee19794fe05926be3fb8c8a468b861df4a414acd72dfca27f0
[ec2-user@ip-172-31-46-11 ~]$     podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
[ec2-user@ip-172-31-46-11 ~]$

```
### Using Container Healthchecks

Now that the web server containers are up and running with shared storage and networking, you will add a check that validates `nginx` is available.

#### Configure Healthchecks For Your Web Servers

1. Start 3 `nginx` containers. Name them `web1`, `web2`, and `web3`. Attach the `webvol` volume to the container at `/usr/share/nginx/html/`. Publish the containers' port `80` to `8081` (`web1`), `8082` (`web2`), and `8083` (`web3`). Add a healthcheck for each server: 'curl [http://localhost](http://localhost/) || exit 1'. Run the following command to start the first container:
    
    ```
    podman run -dt --name web1 --publish 8081:80 --volume webvol:/usr/share/nginx/html:z --health-cmd 'curl http://localhost || exit 1' --health-interval=0 nginx
    ```
    
2. Repeat the command for the second container:
    
    ```
    podman run -dt --name web2 --publish 8082:80 --volume webvol:/usr/share/nginx/html:z --health-cmd 'curl http://localhost || exit 1' --health-interval=0 nginx
    ```
    
3. Repeat the command for the third container:
    
    ```
    podman run -dt --name web3 \
      --publish 8083:80 \
      --volume webvol:/usr/share/nginx/html:z \
      --health-cmd 'curl http://localhost || exit 1' --health-interval=0 nginx
    ```
    
4. Verify the containers were started:
    
    ```
    podman ps -a
    ```
    
    You should see all 3 containers.
    
5. Use a `curl` command to pull the first test text file from the web server container:
    
    ```
    curl -s http://localhost:8081/test1.txt
    ```
    
6. Pull the second test text file:
    
    ```
    curl -s http://localhost:8082/test2.txt
    ```
    
7. Pull the third test text file:
    
    ```
    curl -s http://localhost:8083/test3.txt
    ```
    
    You should see your test text files via the 3 `nginx` containers.
    
8. Run the healthchecks for all 3 servers. Validate that the `nginx` service is `healthy` on all 3 servers. Run the following command for the first server:
    
    ```
    podman healthcheck run web1
    ```
     
1. Run the following command for the second server:
    
    ```
    podman healthcheck run web2
    ```
    
2. Run the following command for the third server:
    
    ```
    podman healthcheck run web3
    ```
    

You should see a `healthy` status for all 3 containers!
says it should be healthy but how do i know it is healhy? what is the expected output

```
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web1 --publish 8081:80 --volume webvol:/usr/share/nginx/html:z --health-cmd 'curl http://localhost || exit 1' --health-interval=0 nginx
216af5b5b26eba79dbd5ff06ddeb504499120d67999c5162e40dacedaeedb7a8
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web2 --publish 8082:80 --volume webvol:/usr/share/nginx/html:z --health-cmd 'curl http://localhost || exit 1' --health-interval=0 nginx
cb8fdb778bb0ddd645f39fa0cb7a2a7024694f46355970483473266f85ebcf08
[ec2-user@ip-172-31-46-11 ~]$     podman run -dt --name web3 \
      --publish 8083:80 \
      --volume webvol:/usr/share/nginx/html:z \
      --health-cmd 'curl http://localhost || exit 1' --health-interval=0 nginx
ca8349cf4b4f69e73ab19ef041e6fed0b6cae24f819aa6ece5b898492962fdcf
[ec2-user@ip-172-31-46-11 ~]$     podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED             STATUS                        PORTS                 NAMES
216af5b5b26e  docker.io/library/nginx:latest  nginx -g daemon o...  About a minute ago  Up About a minute (starting)  0.0.0.0:8081->80/tcp  web1
cb8fdb778bb0  docker.io/library/nginx:latest  nginx -g daemon o...  About a minute ago  Up About a minute (starting)  0.0.0.0:8082->80/tcp  web2
ca8349cf4b4f  docker.io/library/nginx:latest  nginx -g daemon o...  18 seconds ago      Up 19 seconds (starting)      0.0.0.0:8083->80/tcp  web3
[ec2-user@ip-172-31-46-11 ~]$     curl -s http://localhost:8081/test1.txt
Test File 1
[ec2-user@ip-172-31-46-11 ~]$     curl -s http://localhost:8082/test2.txt
Test File 2
[ec2-user@ip-172-31-46-11 ~]$     curl -s http://localhost:8083/test3.txt
Test File 3
[ec2-user@ip-172-31-46-11 ~]$     podman healthcheck run web1
[ec2-user@ip-172-31-46-11 ~]$ echo $?
0
[ec2-user@ip-172-31-46-11 ~]$     podman healthcheck run web2
[ec2-user@ip-172-31-46-11 ~]$ echo $?
0
[ec2-user@ip-172-31-46-11 ~]$     podman healthcheck run web3
[ec2-user@ip-172-31-46-11 ~]$ echo $?
0
[ec2-user@ip-172-31-46-11 ~]$
```

### Managing Your Podman Environment

With healthy containers, you can check out the host.

#### Managing Your Podman System

1. Display system information for the Podman host:
    
    ```
    podman system info | more
    ```
    
    You can see a lot of information about the host displayed.
    
2. Display Podman's disk space utilization. First, use the regular mode. Note that this may take a minute to display:
    
    ```
    podman system df
    ```
    
    A summary of storage utilization is displayed. Note that you currently have no reclaimable storage.
    
3. Try pulling an image:
    
    ```
    podman pull httpd
    ```
    docker.io/library/httpd:latest
4. Rerun the disk usage command:
    
    ```
    podman system df
    ```
    
    You should now have some reclaimable storage.
    
5. Check Podman's disk space utilization with verbose mode (this may take a minute):
    
    ```
    podman system df -v
    ```
    
    You can see a more detailed version of storage, image utilization, container utilization, and local volume utilization.
    
6. Clean up the reclaimable space using `prune`:
    
    ```
    podman system prune -a
    ```
    
7. Display Podman's disk space utilization again:
    
    ```
    podman system df
    ```
    
    All storage should be reclaimed.
    
my output
```
[ec2-user@ip-172-31-46-11 ~]$     podman system info | more
host:
  arch: amd64
  buildahVersion: 1.41.8
  cgroupControllers:
  - cpu
  - memory
  - pids
  cgroupManager: systemd
  cgroupVersion: v2
  conmon:
    package: conmon-2.1.13-1.el10.x86_64
    path: /usr/bin/conmon
    version: 'conmon version 2.1.13, commit: '
  cpuUtilization:
    idlePercent: 98.67
    systemPercent: 0.37
    userPercent: 0.95
  cpus: 2
  databaseBackend: sqlite
  distribution:
    distribution: rhel
    version: "10.1"
  eventLogger: journald
  freeLocks: 2044
  hostname: ip-172-31-46-11.ec2.internal
  idMappings:
    gidmap:
    - container_id: 0
      host_id: 1000
      size: 1
    - container_id: 1
      host_id: 524288
      size: 65536
    uidmap:
    - container_id: 0
      host_id: 1000
      size: 1
    - container_id: 1
      host_id: 524288
      size: 65536
  kernel: 6.12.0-124.52.1.el10_1.x86_64
  linkmode: dynamic
  logDriver: journald
  memFree: 2989522944
  memTotal: 3827535872
  networkBackend: netavark
  networkBackendInfo:
    backend: netavark
    dns:
      package: aardvark-dns-1.16.0-2.el10.x86_64
      path: /usr/libexec/podman/aardvark-dns
      version: aardvark-dns 1.16.0
    package: netavark-1.16.0-1.el10.x86_64
    path: /usr/libexec/podman/netavark
    version: netavark 1.16.0
  ociRuntime:
    name: crun
    package: crun-1.27-1.el10_1.x86_64
    path: /usr/bin/crun
    version: |-
      crun version 1.27
      commit: a718a92cc9a94955a5a550b6fdec1378c247ec50
      rundir: /run/user/1000/crun
      spec: 1.0.0
      +SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +CRIU +YAJL
  os: linux
  pasta:
    executable: /usr/bin/pasta
    package: passt-0^20250512.g8ec1341-5.el10_1.x86_64
    version: ""
  remoteSocket:
    exists: true
    path: /run/user/1000/podman/podman.sock
  rootlessNetworkCmd: pasta
  security:
    apparmorEnabled: false
    capabilities: CAP_CHOWN,CAP_DAC_OVERRIDE,CAP_FOWNER,CAP_FSETID,CAP_KILL,CAP_NET_BIND_SERVICE,CAP_SETFCAP,CAP_SETGID,CAP_SETPCAP,CAP_SETUID,CAP_SYS_CHROOT
    rootless: true
    seccompEnabled: true
    seccompProfilePath: /usr/share/containers/seccomp.json
    selinuxEnabled: true
  serviceIsRemote: false
  slirp4netns:
    executable: ""
    package: ""
    version: ""
  swapFree: 0
  swapTotal: 0
  uptime: 0h 34m 34.00s
  variant: ""
plugins:
  authorization: null
  log:
  - k8s-file
  - none
  - passthrough
  - journald
  network:
  - bridge
  - macvlan
  - ipvlan
  volume:
  - local
registries:
  search:
  - registry.access.redhat.com
  - registry.redhat.io
  - docker.io
store:
  configFile: /home/ec2-user/.config/containers/storage.conf
  containerStore:
    number: 3
    paused: 0
    running: 3
    stopped: 0
  graphDriverName: overlay
  graphOptions: {}
  graphRoot: /home/ec2-user/.local/share/containers/storage
  graphRootAllocated: 10458476544
  graphRootUsed: 2098462720
  graphStatus:
    Backing Filesystem: xfs
    Native Overlay Diff: "true"
    Supports d_type: "true"
    Supports shifting: "false"
    Supports volatile: "true"
    Using metacopy: "false"
  imageCopyTmpDir: /var/tmp
  imageStore:
    number: 1
  runRoot: /run/user/1000/containers
  transientStore: false
  volumePath: /home/ec2-user/.local/share/containers/storage/volumes
version:
  APIVersion: 5.6.0
  BuildOrigin: Red Hat, Inc. <http://bugzilla.redhat.com/bugzilla>
  Built: 1771545600
  BuiltTime: Fri Feb 20 00:00:00 2026
  GitCommit: b194cd996eb74ecf0ff67d710d4b2aaa90e1c27e
  GoVersion: go1.25.7 (Red Hat 1.25.7-1.el10_1)
  Os: linux
  OsArch: linux/amd64
  Version: 5.6.0

[ec2-user@ip-172-31-46-11 ~]$     podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         1           1           164.9MB     0B (0%)
Containers     3           3           89.46kB     0B (0%)
Local Volumes  1           3           1.429kB     0B (0%)
[ec2-user@ip-172-31-46-11 ~]$     podman pull httpd

Error: ^C
[ec2-user@ip-172-31-46-11 ~]$     podman pull docker.io/library/httpd:latest
Trying to pull docker.io/library/httpd:latest...
Getting image source signatures
Copying blob 4b705a141cea done   |
Copying blob 57fb71246055 skipped: already exists
Copying blob 4b31e7d9a3b8 done   |
Copying blob 4f4fb700ef54 done   |
Copying blob 6e3e6d1d5b56 done   |
Copying blob 777279f5ce9f done   |
Copying config c194ed9b9e done   |
Writing manifest to image destination
c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a
[ec2-user@ip-172-31-46-11 ~]$     podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         2           1           204MB       120.2MB (59%)
Containers     3           3           89.46kB     0B (0%)
Local Volumes  1           3           1.429kB     0B (0%)

[ec2-user@ip-172-31-46-11 ~]$ podman system df -v
Images space usage:

REPOSITORY               TAG         IMAGE ID      CREATED     SIZE        SHARED SIZE  UNIQUE SIZE  CONTAINERS
docker.io/library/nginx  latest      6f8edba05e38  21 hours    164.9MB     81.04MB      83.81MB      3
docker.io/library/httpd  latest      c194ed9b9e8f  5 days      120.2MB     81.04MB      39.18MB      0

Containers space usage:

CONTAINER ID  IMAGE         COMMAND               LOCAL VOLUMES  SIZE        CREATED     STATUS      NAMES
216af5b5b26e  6f8edba05e38  nginx -g daemon off;  1              29.82kB     10 minutes  running     web1
cb8fdb778bb0  6f8edba05e38  nginx -g daemon off;  1              29.82kB     10 minutes  running     web2
ca8349cf4b4f  6f8edba05e38  nginx -g daemon off;  1              29.82kB     9 minutes   running     web3

Local Volumes space usage:

VOLUME NAME  LINKS       SIZE
webvol       3           1.429kB


[ec2-user@ip-172-31-46-11 ~]$     podman system prune -a
WARNING! This command removes:
        - all stopped containers
        - all networks not used by at least one container
        - all images without at least one container associated with them
        - all build cache

Are you sure you want to continue? [y/N] y
Deleted Images
c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a
Total reclaimed space: 120.2MB
[ec2-user@ip-172-31-46-11 ~]$     podman system df
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         1           1           164.9MB     0B (0%)
Containers     3           3           89.46kB     0B (0%)
Local Volumes  1           3           1.429kB     0B (0%)
[ec2-user@ip-172-31-46-11 ~]$
```
#### Monitoring Your Podman System

1. Take a look at events from the last 10 minutes:
    
    ```
    podman events --since 10m
    ```
    
    Press **CTRL+C** to exit.
    
2. Display all `prune` events for the past 15 minutes:
    
    ```
    podman events --since 15m --filter event=prune
    ```
    
    You can see that 1 prune event is listed. Press **CTRL+C** to exit.
    

> [!NOTE]- `prune -a` does not register in log
> The `podman system prune` command is actually a "meta-command." Instead of generating a single event called `prune`, it acts as a wrapper that triggers individual cleanup actions across the system.
> 
> When you run `podman system prune -a`, Podman breaks that down under the hood into distinct sub-operations:
> 
> - It removes stopped containers (generating `remove` events for each container).
>     
> - It deletes unused images (generating `remove` or `unmount` events for images).
>     
> - It cleans up networks and pods (generating `remove` events for those resources).
>     
> 
> Because of this design, **there is actually no specific event type called `prune`** logged by the system daemon. The filter `event=prune` matches nothing, which is why your `podman events` command was sitting there in total silence

my output
```
[ec2-user@ip-172-31-46-11 ~]$ podman events --since 10m
2026-05-14 15:55:18.7418953 +0000 UTC container health_status cb8fdb778bb0ddd645f39fa0cb7a2a7024694f46355970483473266f85ebcf08 (image=docker.io/library/nginx:latest, name=web2, health_status=healthy, health_failing_streak=0, health_log=, maintainer=NGINX Docker Maintainers <docker-maint@nginx.com>)
2026-05-14 15:55:28.378710244 +0000 UTC container health_status ca8349cf4b4f69e73ab19ef041e6fed0b6cae24f819aa6ece5b898492962fdcf (image=docker.io/library/nginx:latest, name=web3, health_status=healthy, health_failing_streak=0, health_log=, maintainer=NGINX Docker Maintainers <docker-maint@nginx.com>)
2026-05-14 16:01:16.74729299 +0000 UTC image pull-error  httpd ^C
2026-05-14 16:02:18.075702522 +0000 UTC image pull c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a docker.io/library/httpd:latest
2026-05-14 16:03:30.347189272 +0000 UTC image untag c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a docker.io/library/httpd:latest
2026-05-14 16:03:30.278492571 +0000 UTC image remove c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a

[ec2-user@ip-172-31-46-11 ~]$     podman events --since 15m --filter event=prune
[ec2-user@ip-172-31-46-11 ~]$ podman events --since 15m --filter event=prune
```

```
[ec2-user@ip-172-31-46-11 ~]$ podman events --since 15m --filter event=remove
2026-05-14 16:03:30.278492571 +0000 UTC image remove c194ed9b9e8f9c5811690935a41438aea4e16b9dc66a4230b84092b4214f0d9a
```
## Conclusion

Congratulations — you've completed this hands-on lab!


# AWS

Software Image (AMI)
Provided by Red Hat, Inc.
ami-0fdfb4d987b63ae72
Virtual server type (instance type)
t2.medium
Firewall (security group)
New security group
Storage (volumes)
1 volume(s) - 10 GiB
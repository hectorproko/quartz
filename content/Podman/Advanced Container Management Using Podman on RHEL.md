---
tags:
  - Podman
  - RHE
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
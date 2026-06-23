---
tags:
  - RHEL
  - Ansible
  - selinux
  - RHEL10
  - security
quartz: "True"
linkedin: "False"
hardlinked: "True"
---
**Role:** Linux Administrator at a financial services provider
**Platform:** Red Hat Enterprise Linux 10  
**Environment:** Two RHEL 10 VMs on an isolated network, Management Node (Ansible control) and Workload Node (Apache web server)

---

## Overview

Unix file permissions answer one question: *who* can access a file. SELinux answers a different question: *what processes* are allowed to access a file, and in what way. This distinction matters a lot in a web server context, an Apache process running as `www-data` may have the right Unix permissions to read a directory, but SELinux can still deny that access if the file's security label doesn't permit it.

In this lab I used that capability to restrict a running Apache server from serving a confidential file, using only an SELinux label change, without touching Unix permissions, firewall rules, or Apache configuration. I then used Ansible with the `rhel-system-roles.selinux` role to codify that SELinux must remain in enforcing mode on the Workload Node.

%%[[Ansible Role#Example]]%%

---

## Environment

| Node | Role | Private IP |
|---|---|---|
| Management Node | Ansible control plane | `10.0.0.64` |
| Workload Node | Apache HTTP server | `10.0.1.110` |

---

## Step 1 - Prepare the Web Application on the Workload Node

I connected to the **Workload Node** and confirmed the baseline state:

```bash
getenforce
rpm -q httpd
rpm -q policycoreutils-python-utils
```

**Output:**
```
Enforcing
httpd-2.4.63-4.el10_1.3.x86_64
policycoreutils-python-utils-3.9-1.el10.noarch
```

SELinux was already in enforcing mode, Apache and the SELinux management tools were installed. I enabled and started the web server:

```bash
sudo systemctl enable --now httpd
sudo systemctl status httpd --no-pager
```

**Output:**
```
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Wed 2026-05-13 14:52:53 UTC; 26s ago
     Status: "Total requests: 0; Idle/Busy workers 100/0;..."
...
May 13 14:52:53 ip-10-0-1-110.ec2.internal httpd[6824]: Server configured, listening on: port 80
May 13 14:52:53 ip-10-0-1-110.ec2.internal systemd[1]: Started httpd.service - The Apache HTTP Server.
```

I then created two demo pages, a public one and a "confidential" one:

```bash
sudo bash -c 'echo "Hello from the public page" > /var/www/html/index.html'
sudo bash -c 'echo "Highly confidential report" > /var/www/html/secret.html'
```

Both were reachable by default:

```bash
curl http://localhost/
curl http://localhost/secret.html
```
```
Hello from the public page
Highly confidential report
```

Inspecting their SELinux contexts showed both files carried the same **label**, `httpd_sys_content_t`, which is the type that allows Apache to read them:
<!--
Exactly right. In SELinux, a type label is essentially a **declaration of what a resource is and how it should behave**.

When a file is labeled `httpd_sys_content_t`, SELinux reads that as: _"this is web content, and the httpd process is allowed to read it."_ That permission isn't granted ad-hoc — it's baked into the SELinux policy that ships with RHEL.
-->
```bash
ls -Z /var/www/html
```
```
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
unconfined_u:object_r:httpd_sys_content_t:s0 secret.html
```

Both files are identical from SELinux's perspective. That's the problem I needed to solve.
<!--
[[SELinux#options]]
-->
---

## Step 2 - Restrict the Secret Page Using Persistent SELinux Labels

SELinux type enforcement works by allowing or denying access based on the ***type* of the process** making the request and the ***type* of the resource** being accessed. Apache runs in the `httpd_t` domain and can read files labeled `httpd_sys_content_t`. It cannot read files labeled `user_home_t`.
<!--
In SELinux, a **domain** is the security type assigned to a _running process_ (as opposed to a type assigned to a file).

When Apache starts, SELinux transitions it into the `httpd_t` domain. Think of it as a security sandbox — it defines exactly what the Apache process is _allowed to do_ on the system: which ports it can bind, which files it can read, which syscalls it can make

so the type of the process comes from the domain that the process was put into?

Exactly right — and they're actually the same thing, just two words for the same field depending on context.

In an SELinux security context like:

```
system_u:system_r:httpd_t:s0
         ───────  ───────
           role     type
```
  user                                   sensitivity    
  
That third field is technically called the **type** in SELinux's data model. But when that type is assigned to a _process_, it gets called a **domain** — it's just the community's way of distinguishing "this type belongs to a process" from "this type belongs to a file."
**Domain** = what you call the type when it's on a process
**Label** is just the informal word for the full SELinux security context string attached to anything — a file, a process, a port, a socket.
-->
The key tool here is `semanage fcontext`, not `chcon`. The difference matters:

- `chcon` changes a label immediately but it **does not persist**, a `restorecon` or system relabel will revert it.
- `semanage fcontext` writes a **persistent rule** into the SELinux policy database. When `restorecon` runs, it applies the rule from that database. This is the correct approach for anything meant to survive reboots or system relabels.

```bash
# Write the persistent rule
sudo semanage fcontext -a -t user_home_t "/var/www/html/secret\.html"

# Apply it to the file
sudo restorecon -v /var/www/html/secret.html
```
<!--
[[semanage (SELinux Management)#Options]]
-->
**Output:**
```
Relabeled /var/www/html/secret.html from unconfined_u:object_r:httpd_sys_content_t:s0 to unconfined_u:object_r:user_home_t:s0
```

Verifying the new label:

```bash
ls -Z /var/www/html/secret.html
```
```
unconfined_u:object_r:user_home_t:s0 /var/www/html/secret.html
```

Now testing access:

```bash
# Secret page - should be denied
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/secret.html

# Public page - should still work
curl http://localhost/
```

**Output:**
```
403

Hello from the public page
```

Apache returned `403 Forbidden` for the secret page. The public page was unaffected. I confirmed SELinux was the reason, not Apache configuration, by checking the audit log:

```bash
sudo grep -i "avc:.*httpd" /var/log/audit/audit.log | tail -n 10
```

**Output:**
```
type=AVC msg=audit(1778684439.243:570): avc:  denied  { read } for  pid=6827 comm="httpd" name="secret.html" dev="nvme0n1p3" ino=33587077 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
```

The audit log entry confirms exactly what happened: the `httpd_t` process tried to `read` `secret.html`, which is now labeled `user_home_t`, and SELinux denied it (`permissive=0` means enforcing mode, real denial). Unix permissions were never changed, only the SELinux label.
<!--
When you say **"the `httpd_t` process"** it's shorthand for **"the process that was transitioned into the `httpd_t` domain"** — in this case, Apache.
-->
---

## Step 3 - Create an Ansible Inventory

Back on the **Management Node**, I set up the working directory and Ansible inventory to automate the compliance piece:

```bash
mkdir -p ~/rhel10-security-lab
cd ~/rhel10-security-lab

cat > inventory.ini << 'EOF'
[workload]
work ansible_host=10.0.1.110

[all:vars]
ansible_user=cloud_user
ansible_become=true
EOF
```

Testing Ansible connectivity to the Workload Node:

```bash
ansible -i inventory.ini workload -m ping -k -K
```

**Output:**
```
work | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

Connectivity confirmed.

---

## Step 4 - Apply the SELinux System Role via Ansible

RHEL System Roles provide **idempotent**, supported Ansible roles for common RHEL subsystems. The `rhel-system-roles.selinux` role handles everything from setting the SELinux mode and policy to managing booleans, file contexts, and port labels. Here I used it for one specific purpose: ensure the Workload Node is in enforcing mode and stays that way, regardless of what someone might change manually.
<!--
The `.el10` in the package name means it came from Red Hat's RHEL 10 repositories. It was installed with `dnf` just like any other system package — not pulled from Ansible Galaxy.
[[#^c92677]]
-->

```bash
cat > selinux_enforce.yml << 'EOF'
- name: Ensure SELinux is enforcing on workload node
  hosts: workload
  become: true

  roles:
    - rhel-system-roles.selinux

  vars:
    selinux_policy: targeted
    selinux_state: enforcing
EOF

ansible-playbook -i inventory.ini selinux_enforce.yml -k -K
```

**Output (final recap):**
```
PLAY RECAP ****************************************************
work   : ok=16   changed=0    unreachable=0    failed=0    skipped=23   rescued=0    ignored=0
```

`failed=0` and `unreachable=0` are the key indicators. `changed=0` tells me SELinux was already in the correct state, the role confirmed it and made no unnecessary modifications. This is the correct behavior of an idempotent role.

Finally, I verified enforcing mode directly from the Management Node:

```bash
ssh cloud_user@10.0.1.110 'getenforce'
```
```
Enforcing
```

---

## Key Takeaways

- SELinux type enforcement restricts access based on *what the process is*, not just *who owns the file*. A web server running as root can still be blocked by SELinux if the file's type doesn't permit it.
- Use `semanage fcontext` + `restorecon` (not `chcon`) for persistent label changes. `chcon` changes are lost on relabel; `semanage` writes to the policy database so they survive.
- The audit log at `/var/log/audit/audit.log` is the **authoritative source** for SELinux denials. The `avc: denied` entries tell you exactly which process, which file, and which operation was blocked.
- `rhel-system-roles.selinux` is the supported, Red Hat-maintained way to manage SELinux state in an Ansible-automated fleet. It's idempotent, handles edge cases (like ostree systems), and keeps enforcement codified in version-controlled playbooks rather than relying on manual configuration. ^c92677
- An `ok=16, changed=0` Ansible recap on an already-compliant system is success, it means the system is in the desired state and nothing had to be changed to get it there.

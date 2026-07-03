---
tags:
  - selinux
  - draft
linkedin: "False"
quartz: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
## Introduction

By default, SELinux enforces strict rules about which ports a service is allowed to use. When you configure a service to listen on a non-standard port, SELinux will block it unless you explicitly update the policy to allow it. In addition, SELinux controls service behavior through **booleans**, toggleable settings that enable or disable specific access rules without requiring a full policy rewrite.

In this lab, MariaDB is configured to run on port `3333` instead of the default `3306`. The goal is to update the SELinux policy to allow this, and then enable the `mysql_connect_any` boolean so the service can make outbound connections freely.

**Environment:**

- Cloud Server running MariaDB
- Access via SSH as `cloud_user`

---

## Step 1 - Update the MariaDB Configuration

The first step is to tell MariaDB itself to listen on the new port by editing its configuration file.

Open the server config file:

```bash
sudo vim /etc/my.cnf.d/server.cnf
```

Under the `[server]` section, set the port to `3333`:

```ini
[server]
port 3333
```

> [!NOTE]- server.cnf
> ```bash
> [root@database ~]# cat /etc/my.cnf.d/server.cnf
> #
> # These groups are read by MariaDB server.
> # Use it for options that only the server (but not clients) should see
> #
> # See the examples of server my.cnf files in /usr/share/mysql/
> #
> 
> # this is read by the standalone daemon and embedded servers
> [server]
> port 3333
> 
> # this is only for the mysqld standalone daemon
> [mysqld]
> 
> # this is only for embedded server
> [embedded]
> 
> # This group is only read by MariaDB-5.5 servers.
> # If you use the same .cnf file for MariaDB of different versions,
> # use this group for options that older servers don't understand
> [mysqld-5.5]
> 
> # These two groups are only read by MariaDB servers, not by MySQL.
> # If you use the same .cnf file for MySQL and MariaDB,
> # you can put MariaDB-only options here
> [mariadb]
> 
> [mariadb-5.5]
> 
> [root@database ~]#
> ```

At this point, MariaDB is configured to use port `3333`, but SELinux still does not know this port is allowed for the MySQL service. Attempting to start MariaDB now would result in a denial.

---

## Step 2 - Add the Port to SELinux

SELinux maintains a list of allowed ports for every service type. The MySQL service type is `mysqld_port_t`. Before adding the new port, it helps to review what ports are currently allowed.

Check the current allowed ports for MySQL:

```bash
sudo semanage port -l | grep mysql
```

**Expected output:**

```
mysqld_port_t                  tcp      1186, 3306, 63132-63164
mysqlmanagerd_port_t           tcp      2273
```

Port `3333` is not listed. SELinux would block MariaDB from binding to it. Add `3333` to the allowed list under the `mysqld_port_t` type:

```bash
sudo semanage port -a -t mysqld_port_t -p tcp 3333
```

Confirm the port was added:

```bash
sudo semanage port -l | grep mysql
```

**Expected output:**

```
mysqld_port_t                  tcp      3333, 1186, 3306, 63132-63164
mysqlmanagerd_port_t           tcp      2273
```

Port `3333` now appears in the list. SELinux will permit MariaDB to bind to this port.

---

## Step 3 - Check and Enable the SELinux Boolean

SELinux booleans are named on/off switches that control optional behaviors within a policy. The `mysql_connect_any` boolean controls whether MySQL processes are allowed to make outbound network connections on any port. It is off by default, which limits where the service can connect to.

Check the current state of the boolean:

```bash
sudo semanage boolean -l | grep mysql_connect_any
```

**Expected output:**

```
mysql_connect_any              (off  ,  off)  Allow mysql to connect any
```

The two values shown represent the current runtime state and the persistent (default) state. Both are `off`.

Verify with `getsebool` for a simpler view:

```bash
getsebool mysql_connect_any
```

**Expected output:**

```
mysql_connect_any --> off
```

Enable the boolean for the current session:

```bash
sudo setsebool mysql_connect_any 1
```

Make the change permanent so it survives a reboot:

```bash
sudo setsebool mysql_connect_any 1 -P
```

Verify it is now on:

```bash
getsebool mysql_connect_any
```

**Expected output:**

```
mysql_connect_any --> on
```

Using `-P` writes the change to the SELinux policy on disk. Without it, the setting would reset after a reboot.

---

## Conclusion

This lab demonstrated how SELinux enforces port restrictions and how to extend its policy without modifying or rewriting it entirely. The key tools used were:

- `semanage port` - to register a non-standard port for a specific service type
- `semanage boolean -l` and `getsebool` - to inspect boolean states
- `setsebool -P` - to enable a boolean persistently

This approach follows the principle of least privilege: rather than disabling SELinux to get a service working, the policy is extended in a targeted and auditable way.

<!--
Cloud Server MariaDB Server

Username

cloud_user

Password

2q(2F[fs

MariaDB Server Private IP

10.0.1.101

MariaDB Server Public IP

98.93.209.55

https://app.pluralsight.com/hands-on/labs/196361d0-dbb2-4b83-840c-93cd1c095990?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F

# Working with SELinux Booleans and Ports

## Introduction

Beyond managing the context for each file and process on a server, SELinux also has expected configurations and port settings for most services we use. In this lab, we'll attempt to configure MariaDB on a non-standard port, then use SELinux tools to update our policy without rewriting it.

## Solution

Log in to the server using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP>
```

### Update MariaDB Configuration

1. Open the MariaDB configuration at `/etc/my.cnf.d/server.cnf`. When prompted, use the password that you used to log in:
    
    ```
    sudo vim /etc/my.cnf.d/server.cnf
    ```
    
2. Look for the `[server]` section. Press **i** to enter Insert mode, and set the port to `port 3333`:
    
    ```
    ...
    [server]
    port 3333
    ```
    
3. Press **ESC** followed by `wq` to save and quit the file.
    

```
[cloud_user@database ~]$ sudo -i
[sudo] password for cloud_user:
[root@database ~]# vim /etc/my.cnf.d/server.cnf
[root@database ~]# cat /etc/my.cnf.d/server.cnf
#
# These groups are read by MariaDB server.
# Use it for options that only the server (but not clients) should see
#
# See the examples of server my.cnf files in /usr/share/mysql/
#

# this is read by the standalone daemon and embedded servers
[server]
port 3333

# this is only for the mysqld standalone daemon
[mysqld]

# this is only for embedded server
[embedded]

# This group is only read by MariaDB-5.5 servers.
# If you use the same .cnf file for MariaDB of different versions,
# use this group for options that older servers don't understand
[mysqld-5.5]

# These two groups are only read by MariaDB servers, not by MySQL.
# If you use the same .cnf file for MySQL and MariaDB,
# you can put MariaDB-only options here
[mariadb]

[mariadb-5.5]

[root@database ~]#
```

### Add Port to SELinux

1. Review your current ports:
    
    ```
    sudo semanage port -l | grep mysql
    ```
    
    Observe that you have a couple of default ports, but none of them are `3333`.
    
2. Set the `mysql` port to accept port `3333`:
    
    ```
    sudo semanage port -a -t mysqld_port_t -p tcp 3333
    ```
    
3. Re-run the command from step 1 to ensure `3333` was added to the list of ports.
    

```
[root@database ~]# sudo semanage port -l | grep mysql
mysqld_port_t                  tcp      1186, 3306, 63132-63164
mysqlmanagerd_port_t           tcp      2273
[root@database ~]# sudo semanage port -a -t mysqld_port_t -p tcp 3333
[root@database ~]# sudo semanage port -l | grep mysql
mysqld_port_t                  tcp      3333, 1186, 3306, 63132-63164
mysqlmanagerd_port_t           tcp      2273
[root@database ~]#

```

### Check and Potentially Change SELinux Boolean

1. Check the `mysql_connect_any` boolean:
    
    ```
    sudo semanage boolean -l | grep mysql_connect_any
    ```
    
    It is currently set to `off`.
    
2. Verify the current setting:
    
    ```
    getsebool mysql_connect_any
    ```
    
3. Update the setting to `on`:
    
    ```
    sudo setsebool mysql_connect_any 1
    ```
    
4. Re-run the command from step 2 to ensure it is on.
    
5. Make the setting change permanent:
    
    ```
    sudo setsebool mysql_connect_any 1 -P
    ```
    
```
[root@database ~]# sudo semanage boolean -l | grep mysql_connect_any
mysql_connect_any              (off  ,  off)  Allow mysql to connect any
[root@database ~]# getsebool mysql_connect_any
mysql_connect_any --> off
[root@database ~]# sudo setsebool mysql_connect_any 1
[root@database ~]# sudo setsebool mysql_connect_any 1 -P
[root@database ~]#
```
## Conclusion

Congratulations — you've completed this hands-on lab!
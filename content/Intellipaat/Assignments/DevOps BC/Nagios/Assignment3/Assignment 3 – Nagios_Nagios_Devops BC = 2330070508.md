---
tags:
  - nagios
---


> [!info] Nagios Assignment - 3
> **To Be Performed:** 
> 1. Use the previous deployment cluster 
> 2. Check whether the FTP service is up or not (use the configuration files for this)

Continuing from [[Assignment 2 – Nagios_Nagios_Devops BC = 2330070508|Assignment 2 – Nagios]]:

> [!tip] EC2s used and Private IPs
> nagios `10.0.1.146`
> my-linux-host1 `10.0.1.164`
> my-linux-host2 `10.0.1.228`
> 

^1ead0f

Before defining the service to monitor FTP, I checked the `commands.cfg` file to see if there were any pre-existing commands I could use. There, I found the `check_ftp` command.
<br>![[Pasted image 20231122140213.png]]

On both `host1.cfg` and `host2.cfg`, I added the following block of code, making sure to adjust the `host_name` appropriately for each host.
```bash
define service {
    use                     local-service 
    host_name               my-linux-host1
    service_description     FTP Check
    check_command           check_ftp
    notifications_enabled   0
}
```

I install `ftpd` in `my-linux-host1`
```
sudo apt-get install ftpd
```

I ensured that the Security Group attached to the [[Assignment 3 – Nagios_Nagios_Devops BC = 2330070508#^1ead0f|EC2 instances]], which I'm using as part of my infrastructure, has ports 20-21 (TCP) opened to facilitate the necessary network communications.

> [!success]
> <br>![[Pasted image 20231122143305.png]]
> As expected, the FTP monitoring on `host2` fails because I did not install `ftpd` on this host. This outcome confirms the dependency of the service monitoring on the actual installation of the required software.

%%
This is how it looks when I ftp from nagios, for record
<br>![[Pasted image 20231122143350.png]]
%%
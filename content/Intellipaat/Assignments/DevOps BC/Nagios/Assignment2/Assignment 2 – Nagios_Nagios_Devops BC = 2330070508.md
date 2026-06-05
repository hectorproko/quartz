---
tags:
  - nagios
---


> [!info] Nagios Assignment - 2
> **To Be Performed:** 
> 1. Create a monitoring service which will check whether Apache2 service is up or not


I'll continue where I left off in [[Assignment 1 – Nagios_Nagios_Devops BC = 2330070508|Assignment 1 – Nagios]]
with:
> [!tip] EC2s used and Private IPs
> nagios `10.0.1.146`
> my-linux-host1 `10.0.1.164`
> my-linux-host2 `10.0.1.228`
> 


I install Apache in `my-linux-host1` and `my-linux-host2` and confirm service is running
```bash
sudo apt-get install apache2
sudo systemctl status apache2
```

In my Nagios setup, the `Current Status > Services` section currently only displays information about the localhost. To expand this, I aim to monitor the Apache service on both of our hosts, `host1` and `host2`.
<br>![[Pasted image 20231122113226.png]]


### Creating A Monitoring Service
%%might not be that file%%
In the `localhost.cfg` file, I located the following block of code for reference:
```
# Define a service to check HTTP on the local machine.
# Disable notifications for this service by default, as not all users may have HTTP enabled.
define service {

    use                     local-service           ; Name of service template to use
    host_name               localhost
    service_description     HTTP
    check_command           check_http
    notifications_enabled   0
}
```


Using the code block found in `nagios.cfg` as a reference, I appended the following configuration to both `host1.cfg` and `host2.cfg`. In the `host2.cfg` file, the only change I made was to the `host_name` parameter, as shown in this sample from `host1.cfg`:
```bash
define service {
    use                     local-service
    host_name               my-linux-host1
    service_description     Apache Check
    check_command           check_http
    notifications_enabled   0
```

Restarting Nagios
```bash
sudo systemctl restart nagios
```


Now, if I [[Pasted image 20231122113226.png|return]] to Nagios and navigate to `Current Status > Services`, it now displays two new services being monitored, each on their respective host, `my-linux-host1` and `my-linux-host2`.

> [!success]
> <br>![[Pasted image 20231122123057.png]]
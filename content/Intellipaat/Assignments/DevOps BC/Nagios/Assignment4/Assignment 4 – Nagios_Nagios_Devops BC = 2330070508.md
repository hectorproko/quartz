---
tags:
  - nagios
---


> [!info] Nagios Assignment - 4
> To Be Performed: 
> 1. On both the slaves run a CPU checking service 
> 2. Reduce the check interval on slave2 to 1 minute since it’s a critical server



To create a service that checks CPU load, I copied the definition from `localhost.cfg` and modified it accordingly for my specific needs.
```bash
define service {

    use                     local-service  
    host_name               localhost
    service_description     Current Load
    check_command           check_local_load!5.0,4.0,3.0!10.0,6.0,4.0
}
```


For `host1.cfg`, I changed the `host_name`
```bash
define service {

    use                     local-service  
    host_name               my-linux-host1
    service_description     Current Load
    check_command           check_local_load!5.0,4.0,3.0!10.0,6.0,4.0
}
```
For `host2.cfg`, in addition to changing the `host_name`, I also added a `check_interval` parameter with a value of 1 to overwrite the default values.
```bash
define service {

    use                     local-service  
    host_name               my-linux-host2
    check_interval          1
    service_description     Current Load
    check_command           check_local_load!5.0,4.0,3.0!10.0,6.0,4.0
}
```

Restarted Nagios service
```bash
sudo systemctl restart nagios
```

Now, the 'Current Load' service appears as a monitored service for both `my-linux-host1` and `my-linux-host2` in the Nagios interface.
<br>![[Pasted image 20231122152746.png]]

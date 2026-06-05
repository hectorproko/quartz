---
tags:
  - azure
---

[[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]]

> [!attention]
> [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104]] might have better steps

> [!attention]
> Might need a explanation of how you ge a number of IPs and Interfaces that we attach to them so if you VM doesnt Public IP
> 

> [!tip]
> There is an option a check box that says to delete interface and IP once the VM is deleted


> [!Azure CLI]
> 
> ```bash
> az vm create \
>   --resource-group rg-1 \
>   --name myLinuxVM \
>   --image Canonical:UbuntuServer:18.04-LTS:latest \
>   --admin-username azureuser \
>   --admin-password YourPassword12345 \
>   --authentication-type password \
>   --location eastus
> ```
> 

--- 

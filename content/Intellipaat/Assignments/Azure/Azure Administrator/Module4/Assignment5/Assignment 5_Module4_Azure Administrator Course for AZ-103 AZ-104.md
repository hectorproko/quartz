---
tags:
  - azure
title: Deploying a VM from a Custom Image and Configuring Apache2 Web Service on Azure
---
<!--
 **Project Completion: Deploying and Managing a VM with Azure!** I'm excited to share my experience with the latest assignment in my Azure Administrator course, where I successfully deployed a virtual machine from a previously created image. The project involved setting up the VM, opening port 80 in the network security group (NSG), starting the Apache2 service, and ensuring the website was accessible via the VM's public IP. This assignment was a practical exercise in Azure VM deployment, network configuration, and service management, enhancing my skills in setting up and managing web services in Azure's cloud environment.

#Azure #VirtualMachine #NetworkSecurity #CloudComputing #AzureAdministrator #WebServices #ProfessionalDevelopment
-->

> [!info] Module 4: Assignment - 5
> **Tasks To Be Performed:** 
> 1. Deploy a VM from the previously created image 
> 2. Open port 80 in NSG 
> 3. Start the Apache2 service in the VM 
> 4. Verify if you are able to access the website 

To create the VM, I referred to [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]], ensuring that I selected the image I created in [[Assignment 4_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 4: Module 4]] and that I opened Port 80.

<br>![[Pasted image 20231207182310.png]]

<br>![[Pasted image 20231207182459.png]]


I obtain the Public IP from the newly create VM

<br>![[Pasted image 20231207182848.png]]

> [!success] I test the service using the public IP.
> <br>![[Pasted image 20231207183016.png]]
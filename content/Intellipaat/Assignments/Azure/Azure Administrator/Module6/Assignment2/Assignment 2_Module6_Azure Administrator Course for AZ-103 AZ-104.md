---
tags:
  - azure
title: Configuring a VM with a Static IP Address in Azure
---
<!--
**Skill Enhancement: Assigning Static IP to Azure VMs!** In a recent assignment for my Azure Administrator course, I focused on the crucial task of assigning a static IP address to a virtual machine in Azure. The process involved creating a VM in the West US region, navigating to its network settings, and configuring the network interface to use a static IP. This task sharpened my understanding of Azure networking features and the importance of IP address management in cloud environments. Successfully completing this assignment reinforced my abilities in configuring and managing virtual network settings, a fundamental skill for any cloud administrator.

#Azure #VirtualMachine #Networking #StaticIP #AzureAdministrator #CloudComputing #ProfessionalDevelopment
-->

> [!info] Module 6: Assignment - 2
> **Tasks To Be Performed:** 
> 1. Create a VM in West US 
> 2. Assign a Static IP address to the VM


### Step 1: create VM
To create the VM, I follow the steps outlined in [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]]. I name it "StaticIPVM" and ensure that its location is set to "West US.
<br>![[staticipvm overview.png]]
### Step 2: Assign a Static IP Address to the VM

  - I navigate to the virtual machine I just created.
  - I select "Networking" in the VM's settings.
  - I click on the network interface associated with the VM.
    <br>![[Pasted image 20231208115551.png]]
  - In the network interface settings, I click on "IP configurations", then click on the IP configuration for the VM.
    <br>![[Pasted image 20231208115704.png]]
  - In "Private IP address settings" I switch "Allocation" to "Static".
    <br>![[Pasted image 20231208115817.png]]
  - I save the changes.

> [!success] Now we see Private IP Address show "Static"
> <br>![[Pasted image 20231208120147.png]]

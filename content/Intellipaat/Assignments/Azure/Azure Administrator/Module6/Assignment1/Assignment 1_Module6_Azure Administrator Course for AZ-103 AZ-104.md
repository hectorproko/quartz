---
tags:
  - azure
title: Establishing Cross-Region Virtual Network Peering in Azure
---
<!--
**Project Accomplishment: Virtual Network Creation and VM Deployment in Azure!** In the latest assignment of my Azure Administrator course, I successfully established a virtual network infrastructure across different regions. The project involved creating virtual networks in the West US and South India regions, deploying VMs in each, and then setting up VNet-to-VNet peering to ensure connectivity. This task allowed me to apply practical skills in Azure networking, demonstrating the ability to manage and connect resources across multiple regions effectively. It was a valuable exercise in building and configuring a geographically distributed network in Azure, enhancing my expertise in cloud network management.

#Azure #VirtualNetworking #VNetPeering #CloudComputing #AzureAdministrator #NetworkManagement #ProfessionalDevelopment
-->

> [!info] Module 6: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Create a virtual network in West US 
> 2. Create another virtual network in South India 
> 3. Deploy virtual machine in West US with the virtual network in West US 
> 4. Deploy virtual machine in South India inside virtual network in South India 
> 5. Create VNet-VNet peering to connect West US and South India VM 
> 6. Check this by pinging VM1 to VM2 via ping command using private IP address

### Step 1: Create a Virtual Network in West US

1. **I Log into the Azure Portal**:
    
    - I open a web browser and navigate to the [Azure Portal](https://portal.azure.com/). I sign in with my credentials.
2. **I Create the West US Virtual Network**:
    
    - In the Azure Portal, I search "Virtual Network" click "+ Create".
    - I fill in the details, setting the "Name" and choosing "West US" as the "Region".
    - I specify the address space and subnet details.
    - I review and create the virtual network.
      <br>![[Pasted image 20231208102601.png]]
    

### Step 2: Create Another Virtual Network in South India

1. **I Repeat the Creation Process for South India**:
    - Similarly, I create another virtual network, this time choosing "South India" as the region.
    - I ensure that the address space does not overlap with the West US virtual network.
      <br>![[Pasted image 20231207203055.png]]
      <br>![[Pasted image 20231208102917.png]]


### Step 3: Deploy a VM in West US within the West US Virtual Network

1. **I Create a VM in the West US Virtual Network**:
    - Using [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]] as reference.
    - I make sure to pick "West US" region and the virtual network I just created.

### Step 4: Deploy a VM in South India within the South India Virtual Network

1. **I Create a VM in the South India Virtual Network**:
    - I follow the same steps to create another VM, this time in the South India region and associated with the South India virtual network.

<br>![[Pasted image 20231208105100.png]]

I'll SSH into the SouthIndiaVM and attempt to ping WestUSVM using its private IP, but as expected, it fails.
<br>![[Pasted image 20231208104659.png]]

### Step 5: Create VNet-to-VNet Peering

1. **I Set Up Peering for West US Virtual Network**:
    
    - In the Azure Portal, I go to the West US virtual network and select "Peerings".
    - I click "Add" and fill in the details to create a peering connection to the South India virtual network.
      <br>![[Vnetpeering add.png]]
      <br>![[Pasted image 20231208112613.png]]
2. **I Set Up Peering for South India Virtual Network**:
    
    - Similarly, I set up peering from the South India virtual network to the West US virtual network.
      <br>![[Pasted image 20231208112530.png]]


### Step 6: Verify Connectivity with Ping Using Private IP Addresses

1. **Ping Each VM from the Other**:
- From the West US VM, I successfully ping the South India VM.
- From the South India VM, I successfully ping the West US VM.

> [!success]
> <br>![[Pasted image 20231208110909.png]]



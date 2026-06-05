---
tags:
  - azure
title: Adding and Attaching a Network Interface Card (NIC) to an Existing Azure VM
---
<!--
**Advancing Azure Skills: Network Interface Card Management and VM Configuration!** In a recent assignment from my Azure Administrator course, I focused on enhancing my networking skills within Azure. The task involved using an existing VM, creating a new Network Interface Card (NIC), and then attaching this NIC to the VM. This exercise was critical in deepening my understanding of Azure's networking components and their integration with virtual machines. Successfully completing this assignment showcased my ability to manage network interfaces in Azure, reinforcing my skills in configuring and optimizing cloud-based network infrastructure for better performance and connectivity.

#Azure #Networking #AzureAdministrator #VirtualMachine #NIC #CloudComputing #ProfessionalDevelopment
-->

> [!info] Module 6: Assignment - 3
>  **Tasks To Be Performed:** 
> 1. Use the previously created VM 
> 2. Created a NIC 
> 3. Attach NIC to the previously created VM


### Step 1: Previous VM
Using `StaticIPVM` from [[Assignment 2_Module6_Azure Administrator Course for AZ-103 AZ-104|Assignment 2_Module 6]]
<br>![[staticipvm overview.png]]

### Step 2: Create a Network Interface Card (NIC)

2. **I Navigate to 'Network Interfaces'**:
    
    - In the Azure Portal, I search for "Network Interface", and select it.
3. **I Create the NIC**:
    
    - I click "Create" to start the process of creating a new NIC.
    - I fill in the necessary information like the name for the NIC, subscription, resource group (using the same one as the VM), and Region (which should match the VM's location).
    - I select the Virtual Network and Subnet where my existing VM resides.
    - I review all the settings and click "Create" to provision the new NIC.
      <br>![[Pasted image 20231208121618.png]]

### Step 3: Attach NIC to the Previously Created VM

1. **I Stop the VM**:
    
    - Before attaching the NIC, I ensure that the VM is stopped. I go to the VM's overview page in the Azure Portal and click "Stop".
2. **I Attach the NIC to the VM**:
    
    - Once the VM is stopped, I navigate to the VM's settings and select "Networking" - "Network settings".
      
      <br>![[Pasted image 20231208115551.png]]
    - I click "Attach network interface" and select the NIC I just created.
      <br>![[Pasted image 20231208123149.png]]
      
    - I click "Ok" to attach the NIC to the VM.
3. **Verify the NIC Attachment**.

> [!success]
>    <br>![[Pasted image 20231208123339.png]]




---
tags:
  - azure
title: Deploying a Linux VM with Apache2 and Configuring VM Backup in Azure
---
<!--
**Enhancing Azure Recovery Skills: VM Backup and Recovery Services Vault!** I've successfully completed an assignment in my Azure Administrator course that focused on critical data protection strategies. The task involved creating a Linux VM with Apache2, setting up an Azure Recovery Services Vault, and configuring the backup of the VM. This project was crucial in understanding Azure's backup and recovery capabilities, providing me with hands-on experience in safeguarding cloud-based virtual machines. Successfully configuring and managing these vital recovery services has strengthened my proficiency in Azure's data protection and disaster recovery solutions.

#Azure #DataProtection #RecoveryServices #VMBackup #AzureAdministrator #CloudComputing #ProfessionalDevelopment
-->
> [!info] Module 9: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Create a Linux VM and install Apache2 on it 
> 2. Create Recovery Services vault 
> 3. Take backup of the VM deployed


### Step 1: Linux VM wiht Apache2

To create the VM, I follow the steps outlined in [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]] and run the following:

<br>![[Install Apache]]



### Step 2: Create Recovery Services Vault

2. **I Create the Recovery Services Vault**:
    
    - I search for "Recovery Services vault".
    - I click "Create".
    - I fill in the details for the Recovery Services vault, such as the name, subscription, resource group (I use the same resource group as my VM for convenience), and region (the same region as my VM).
    - I review the settings and click "Create" to provision the vault.
      <br>![[Pasted image 20231212152614.png]]

### Step 3: Take Backup of the VM Deployed

1. **I Navigate to the Newly Created Recovery Services Vault**:
    
    - After the vault is created, I go to the "Recovery Services vaults" section in the portal and open the vault I just created.
      <br>![[Pasted image 20231212153111.png]]
2. **I Set Up Backup**:
    
    - Inside the Recovery Services vault, I click "+Backup".
    - I select "Azure Virtual Machine" as the workload running in Azure.
    - I click "Backup" to configure and enable backup for the VM.
      <br>![[Pasted image 20231212153227.png]]
3. **Configure Backup**:
    
    - "Policy sub type" I select "Enhanced" as it covers VMs
    - I then add the VM I want to backup
      <br>![[Pasted image 20231212153757.png]]
1. **I Enable Backup**:
    
    - Once I've configured the backup settings, I click "Enable Backup".
    - Azure will schedule the backup job based on the policy I've defined.


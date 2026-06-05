---
tags:
  - azure
title: Setting Up a Storage Account and Integrating with Azure Storage Explorer
---
<!--
**Progressing in Azure Administration: Mastering Storage Account Management!** As part of Module 3 in the Azure Administrator course, I completed an assignment that further developed my skills in Azure Storage. This involved creating a new storage account and connecting it with Azure Storage Explorer. The process required navigating through Azure's interface to set up the storage account and integrating it with Storage Explorer, enhancing my proficiency in Azure storage solutions. This practical task was an excellent exercise in managing cloud storage resources, providing valuable hands-on experience with Azure's storage services.
-->

> [!info] Module 3: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Create a storage account 
> 2. Connect storage explorer to this storage account

### Step 1: Create a Storage Account 
 - In the Azure Portal, I navigate to "Storage accounts" and select  `hectorstorage12345` (used in [[Assignment 3_Module2_Azure Administrator Course for AZ-103 AZ-104|Module 2: Assignment 3]]) from the list of available storage accounts.

### Step 2: Connect Storage Explorer to the Storage Account

1. **I Install Azure Storage Explorer**:
    - I download and install "Azure Storage Explorer" from the [official website](https://azure.microsoft.com/en-us/features/storage-explorer/).
      
2. **I get Access Keys**:
    - I select the Storage Account. I navigate "Security + networking" > "Access keys" and copy the "Connection string"
      <br>![[Pasted image 20231206200142.png]]
      

3. **I Connect to My Azure Account**:
    - I open "Storage Explorer"
    - I click the connect icon
      <br>![[Pasted image 20231206200332.png]]
    - I select to connect to storage account
      <br>![[Pasted image 20231206200529.png]]
    - I select "Connection string (Key or SAS)"
      <br>![[Pasted image 20231206200647.png]]<br>![[Pasted image 20231206200901.png]]
      <br>![[Pasted image 20231206200924.png]]
      I click "Connect"

> [!success]
> I can now see the container and the files I uploaded
> <br>![[Pasted image 20231206201128.png]]
> 
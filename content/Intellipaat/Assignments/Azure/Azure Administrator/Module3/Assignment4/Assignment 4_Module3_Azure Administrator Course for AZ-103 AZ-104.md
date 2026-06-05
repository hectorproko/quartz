---
tags:
  - azure
title: Employing AzCopy for Efficient Data Transfer Between Azure Storage Containers
---
<!--
**Enhancing Cloud Skills: Data Transfer with Azure's AzCopy Utility!** For my latest Azure Administrator course assignment, I delved into using Azure's AzCopy utility to efficiently transfer data between storage containers. This involved generating SAS tokens for source and destination storage accounts and executing the AzCopy command to copy data. The task provided hands-on experience with Azure's data transfer tools and deepened my understanding of managing cloud storage and data accessibility. Successfully completing this assignment enhanced my capability in Azure storage management and data transfer, key skills for any cloud professional.
-->

> [!info] Module 3: Assignment - 4
> **Tasks To Be Performed:** 
> 1. Use the same storage accounts from previous assignment 
> 2. Use AzCopy utility to copy data from one storage container to another



Continuing from [[Assignment 3_Module3_Azure Administrator Course for AZ-103 AZ-104|Assignment 3]]

### Step 1: Install AzCopy

I install it following the instructions on the [AzCopy v10 page](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10).


### Step 2: Generate SAS Tokens for Both Storage Accounts

1. **I Generate a SAS Token for the Source Storage Account**:
    
    - I log into the Azure Portal and navigate to the first storage account (the source).
    - I go to the "Shared access signature" section and generate a SAS token.
    - I copy this SAS token for later use.
      <br>![[Pasted image 20231206224910.png]]
      I make in "Allowed services" only "Blob" is selected and
      in "Allowed resource types" I select all of them
      
   - I click the button Generate
      <br>![[Pasted image 20231206225038.png]]
      
      <br>![[Pasted image 20231206225138.png]]

      I'll copy ''Blob service SAS URL''
      `https://hectorstorage12345.blob.core.windows.net/?sv=2022-11-02&ss=b&srt=sco&sp=rwdlaciytfx&se=2023-12-07T11:43:22Z&st=2023-12-07T03:43:22Z&spr=https&sig=65jplHCQ0zkeBkaZrnNIDOY9pLAWSx1vkEzjYii90dU%3D`
       
2. **I Generate a SAS Token for the Destination Storage Account**:
    
    - I repeat the process for the second storage account (the destination).
    - I ensure to copy this SAS token as well.
      `https://team1storageaccount54321.blob.core.windows.net/?sv=2022-11-02&ss=b&srt=sco&sp=rwdlaciytfx&se=2023-12-07T11:55:19Z&st=2023-12-07T03:55:19Z&spr=https&sig=5vLf%2Bf717PW%2Bhf%2BMwSExJ6fdDjWaZpgB%2FMs0u5pU5zw%3D`



The whole command looks like this:
```
.\azcopy copy 'https://hectorstorage12345.blob.core.windows.net/mycontainer?sv=2022-11-02&ss=b&srt=sco&sp=rwdlaciytfx&se=2023-12-07T11:43:22Z&st=2023-12-07T03:43:22Z&spr=https&sig=65jplHCQ0zkeBkaZrnNIDOY9pLAWSx1vkEzjYii90dU%3D' 'https://team1storageaccount54321.blob.core.windows.net/mycontainer2?sv=2022-11-02&ss=b&srt=sco&sp=rwdlaciytfx&se=2023-12-07T11:55:19Z&st=2023-12-07T03:55:19Z&spr=https&sig=5vLf%2Bf717PW%2Bhf%2BMwSExJ6fdDjWaZpgB%2FMs0u5pU5zw%3D' --recursive
```

I opened the command prompt (CMD) where the .exe file was located and run the command.
<br>![[Pasted image 20231206232538.png]]

> [!success] I confirm the files from `mycontainer` in [[Assignment 2_Module3_Azure Administrator Course for AZ-103 AZ-104#^326ee1|Assignment 2]] were copied to `mycontainer2`
> <br>![[Pasted image 20231206232746.png]]

> [!attention] The transfer of `chest.png` failed because we changed the tier to 'archive' in [[Assignment 5_Module2_Azure Administrator Course for AZ-103 AZ-104|Assignment 5: Module 2]].
> <br>![[Pasted image 20231206191126.png]]


---
tags:
  - azure
title: Efficient Deletion of Resource Groups with Azure CLI and PowerShell
---
<!--
**Gaining Proficiency in Azure: Mastering Resource Group Management!** In my latest assignment for the Azure Administrator course, I focused on enhancing my Azure CLI and PowerShell skills, particularly in managing resource groups. The task involved using Azure CLI to delete all resource groups in the West US region with a single command. This exercise not only tested my command-line proficiency but also deepened my understanding of efficient resource management in Azure. Successfully executing this task has further solidified my capabilities in Azure's environment, proving essential for any aspiring Azure Administrator.
-->

> [!info] Module 1: Assignment - 4
> **Tasks To Be Performed:** 
> Use Azure CLI and Azure PowerShell to: 
> 1. Delete all the resource groups in West US region using one command

Continuing from [[Assignment 3_Module1_Azure Administrator Course for AZ-103 AZ-104|Assignment 3: Module 1]]

To delete all resource groups in the West US region I use the following command:
```bash
az group list --query "[?location=='westus'].name" -o tsv | xargs -I {} az group delete --name {} --yes --no-wait
```

> `az group list --query "[?location=='westus'].name" -o tsv` lists all resource groups and filters them to only include those in the "West US" region, outputting only their names in a tab-separated values format.
>    <br>![[Pasted image 20231206095622.png]]
> 
> `xargs -I {} az group delete --name {} --yes --no-wait` takes each name and uses it to run the `az group delete` command, which deletes the resource group. The `--yes` flag bypasses the confirmation prompt, and the `--no-wait` flag makes the command return immediately without waiting for the deletion to complete.

To verify deletion:
```bash
az group list --query "[?location=='westus']" --output table
```

> [!success]
> <br>![[Pasted image 20231206100730.png]]

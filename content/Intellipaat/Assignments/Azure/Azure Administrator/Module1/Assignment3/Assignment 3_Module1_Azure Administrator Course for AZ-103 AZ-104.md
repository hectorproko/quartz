---
tags:
  - azure
title: Resource Group Management with Azure CLI and PowerShell
---
<!--
**Advancing Azure Skills: Azure CLI for Resource Group Management!** I recently completed an insightful assignment in the Azure Administrator course, where I used Azure CLI to create and manage resource groups. The task involved creating three new resource groups in the "West US" region and then listing all resource groups in that region using specific Azure CLI commands. This exercise not only bolstered my command-line skills but also deepened my understanding of Azure's resource management capabilities. Successfully executing these tasks has furthered my proficiency in Azure's diverse cloud services, a crucial step in my journey as an Azure Administrator.
-->

> [!info] Module 1: Assignment - 3
> **Tasks To Be Performed:**
> Use Azure CLI and Azure PowerShell to: 
> 1. Create three more resource groups in a specific region. For example: “West US” 
> 2. List all resource groups in West US

Continuing from [[Assignment 2_Module1_Azure Administrator Course for AZ-103 AZ-104|Assignment  2:_Module1]]

In order to use "Azure CLI" I make sure the terminal is set to "Bash"
<br>![[azure cloud shell powershell bash.png]]

<br>![[Pasted image 20231206094413.png|400]]

To create three new resource groups in the "West US" region, In the command-line interface I enter the following commands:
```bash
az group create --name ResourceGroup1 --location westus
az group create --name ResourceGroup2 --location westus
az group create --name ResourceGroup3 --location westus
```

To list all resource groups in the "West US" region, I use the following command:
```bash
az group list --query "[?location=='westus']" --output table
```

> [!success]
> <br>![[Pasted image 20231206091725.png]]

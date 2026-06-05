---
tags:
  - azure
title: Creating a Resource Group Using Azure Cloud Shell
---
<!--
**Exploring Azure Cloud: Azure Administrator Module 1 Assignment!** I've just completed an engaging assignment as part of the Azure Administrator course. The task involved connecting to Azure Cloud Shell and creating a resource group named "new-rg" in the South Central US region. This exercise not only familiarized me with Azure's interface and Cloud Shell but also deepened my understanding of resource group management in Azure. Successfully executing these tasks has bolstered my confidence in navigating and managing resources in the Azure cloud environment. This assignment marks an important step in my journey towards becoming an Azure Administrator.
-->

> [!info] Module 1: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Connect to Azure Cloud Shell 
> 2. Create a resource group “new-rg” in South Central US region 
> 
> Send the commands along with the screenshots for the completion of this assignment. 


In the Azure Portal I navigate to the "Cloud Shell"  Icon in the upper right corner
<br>![[Pasted image 20231206090119.png]]

It prompts me to create ''Storage account", I click "Create" which results in a new account
<br>![[azure storage account module1 assig1.png]]


Afterwards "Cloud Shell" opens a bash terminal by default (Azure CLI)

In the terminal I run the following command to create resource group:
```bash
az group create --name new-rg --location southcentralus
```
<br>![[Pasted image 20231206092831.png]]

To verify I list the resource groups in that region
```bash
az group list --query "[?location=='southcentralus']" --output table
```

> [!success]
> <br>![[Pasted image 20231206093026.png]]




---
tags:
  - azure
title: Managing Resource Groups with Azure PowerShell
---
<!--
**Deepening Azure Expertise: Azure PowerShell and Resource Management!** In the latest assignment of my Azure Administrator journey, I've successfully connected Azure PowerShell to my Azure account and created a new resource group named “rg-powershell” in the South Central US region. This task enhanced my understanding of Azure PowerShell commands and resource group management, key skills for any Azure Administrator. The hands-on experience of executing these commands in a live environment was invaluable, strengthening my command-line proficiency and reinforcing my ability to manage Azure resources effectively. It's another step forward in mastering Azure's powerful cloud capabilities.
-->

> [!info] Module 1: Assignment - 2
> **Tasks To Be Performed:** 
> 1. Connect Azure PowerShell to your Azure account 
> 2. Create a new resource group in South Central US with the name “rg-powershell” 
> 
> Please submit the screenshot along with the commands.

Continuing from [[Assignment 1_Module1_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 1]]

In the Azure Portal I navigate to the "Cloud Shell"  Icon in the upper right corner
<br>![[Pasted image 20231206090119.png]]
I make sure to change the terminal to "PowerShell"
<br>![[azure cloud shell powershell bash.png]]

To create the resource group I run:
```powershell
New-AzResourceGroup -Name rg-powershell -Location "South Central US"
```
<br>![[Pasted image 20231206093535.png]]

I verify the resource group creation with:
```powershell
Get-AzResourceGroup -Name rg-powershell
```
*The output is the same as above*
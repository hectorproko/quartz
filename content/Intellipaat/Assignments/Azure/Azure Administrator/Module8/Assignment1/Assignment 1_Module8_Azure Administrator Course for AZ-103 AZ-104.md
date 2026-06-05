---
tags:
  - azure
title: Creating a Custom Azure Role for Restricted VM Management
---
<!--
**Expanding Azure Skillset: Custom Role Creation for VM Management!** In a recent assignment for my Azure Administrator course, I was tasked with creating a custom role in Azure. This role was designed specifically to view, start, and stop VMs but not perform any other actions. The process involved defining the role in a JSON file with precise permissions, using Azure Cloud Shell for role creation, and verifying the role's functionality. This assignment was a valuable exercise in understanding and implementing Azure's role-based access control (RBAC), enhancing my skills in customizing security and access within the Azure environment.

#Azure #RBAC #VirtualMachineManagement #AzureAdministrator #CloudSecurity #ProfessionalDevelopment
-->

> [!info] Module 8: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Create a custom role which can view, start and stop VMs 
> 2. But should not be able to do anything else




### Step 1: Define the Custom Role

1. **I Open Azure Cloud Shell**:

2. **I Create a JSON Definition for the Custom Role**:
    
    - I define the role in a JSON file with the necessary permissions. Here’s a sample JSON that allows viewing, starting, and stopping VMs:
		```json
		{
		    "Name": "VM Operator",
		    "Id": null,
		    "IsCustom": true,
		    "Description": "Can view, start, and stop virtual machines.",
		    "Actions": [
		        "Microsoft.Compute/virtualMachines/start/action",
		        "Microsoft.Compute/virtualMachines/deallocate/action",
		        "Microsoft.Compute/virtualMachines/read",
		        "Microsoft.Compute/virtualMachines/instanceView/read"
		    ],
		    "NotActions": [],
		    "AssignableScopes": ["/subscriptions/SUBSCRIPTION_ID"]
		}
		```
    - I replace `SUBSCRIPTION_ID` with my actual subscription ID. 

3. **I Save the JSON Definition**:

  - I save the JSON content to a file named `vm-operator-role.json`.


### Step 2: Create the Custom Role Using Azure Cloud Shell

1. **I Run the az Command to Create the Role**:
    
    - I run the following command to create the new role:
		```bash
		az role definition create --role-definition ./vm-operator-role.json
		```

### Verify
Go to Subscriptions > Access control (IAM) > Roles

> [!success]
> <br>![[Pasted image 20231210174432.png]]



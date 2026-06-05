---
tags:
  - azure
title: Setting Up an Alert for VM Deletion in Azure
---
<!--
**Proficiency in Azure Monitoring: Setting Up VM Deletion Alerts!** I've completed an assignment in the Azure Administrator course, which involved using the previously deployed VM to create an alert rule for its deletion. This task allowed me to explore Azure's alerting mechanisms, focusing on detecting and responding to specific activities like VM deletion. The process involved configuring the alert condition, creating an action group with email notifications, and testing the alert by deleting the VM. Successfully implementing this alert system demonstrated my growing skills in Azure's monitoring and notification capabilities, an essential part of managing cloud resources.

#Azure #CloudMonitoring #AzureAdministrator #VMManagement #AlertSystem #ProfessionalDevelopment
-->
> [!info] Module 9: Assignment - 3
> **Tasks To Be Performed:** 
> 1. Use the previous VM deployment 
> 2. Create an alert for the deletion of the above VM

### Task 1: Use the Previous VM Deployment
Using VM from [[Assignment 1_Module9_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 9]]

### Task 2: Create an Alert for the Deletion of the VM

I search "Alerts"
Click "+ Create" and select "Alert Rule"

I selected my VM as the scope.
<br>![[Pasted image 20231214103540.png]]

As for the condition, I configured it to detect a signal for the deletion of the VM.
<br>![[Pasted image 20231214104216.png]]

For the 'Actions' section, I created an action group with 'Notifications' set to an email address.
<br>![[Pasted image 20231214104119.png]]

I reviewed the settings and clicked on 'Create.'
<br>![[Pasted image 20231214104404.png]]
To test the alert, I deleted the VM by clicking the 'Delete' button.
<br>![[Pasted image 20231214111937.png]]

Upon checking the alerts, I observed the notifications, and in my mailbox, I received corresponding emails. I received multiple notifications because I clicked 'Delete' several times.

> [!success]
> <br>![[Pasted image 20231214113513.png]]
> <br>![[Pasted image 20231214113933.png]]
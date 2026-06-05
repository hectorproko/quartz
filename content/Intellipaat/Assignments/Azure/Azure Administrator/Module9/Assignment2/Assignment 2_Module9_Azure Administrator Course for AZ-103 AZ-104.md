---
tags:
  - azure
title: Integrating Azure Log Analytics with Previously Configured VMs
---
<!--
**Project Achievement: Mastering Azure Log Analytics Integration!** I've completed an insightful assignment from my Azure Administrator course where I connected Azure Log Analytics to a VM. This project involved setting up a Log Analytics workspace, linking VMs for data monitoring, and configuring optimal data collection settings. It was an enriching experience to delve into Azure's monitoring tools, gaining practical skills in operational analytics and diagnostics. This task has significantly enhanced my understanding of Azure's capabilities in cloud analytics and performance monitoring.

#Azure #LogAnalytics #CloudMonitoring #AzureAdministrator #DataAnalytics #ProfessionalDevelopment
-->


> [!info] Module 9: Assignment - 2
> **Tasks To Be Performed:** 
> 1. Use the [[Assignment 1_Module9_Azure Administrator Course for AZ-103 AZ-104|previously connected VMs]] 
> 2. Connect Log Analytics to this virtual machine


**Step 1: Create a Log Analytics Workspace**

1. Log into the Azure portal: [https://portal.azure.com](https://portal.azure.com/).
2. Go to All services and search for Log Analytics.
3. Select Log Analytics workspaces.
4. In the Log Analytics workspaces blade, click Create.
5. Enter the necessary information: Subscription, Resource Group, Name, and Region.
6. Click Next to select a Pricing tier, and then choose the appropriate pricing tier.
7. Review your settings and click Create.
8. Refresh the Log Analytics workspaces blade to display the new workspace.

<br>![[Pasted image 20231212174955.png]]

**Step 3: Connect Virtual Machines to the Log Analytics Workspace**

1. In the Azure portal, go to the workspace you created in Step 1.
2. Click on "Connect a data source" and choose "Azure Virtual machine (VMs)".
   <br>![[Pasted image 20231212180709.png]]
   <br>![[Pasted image 20231212180838.png]]
3. Search for the virtual machine you want to connect.
   <br>![[Pasted image 20231212180949.png]]
   <br>![[Pasted image 20231212181223.png]]
4. Click the entry of your virtual machine and then click Connect.
5. Wait for the extension to be installed on the virtual machine.


---
VIDEO

create the workspace
goes to resource
Click connect a data source, azure virtual
selects the VM, just to look

goes to the "agent" in the left side of workspae "legacy actiivty log connector"

create diagnostics settings, creates one, all logs all metrics
<br>![[Pasted image 20231214095625.png]]

<br>![[Pasted image 20231214095647.png]]
creates diagnostic settings

then goes to logs to run some queries


----

create the works pace
<br>![[Pasted image 20231212174955.png]]

To connect the VM
<br>![[Pasted image 20231214100315.png]]
<br>![[Pasted image 20231214100519.png]]

We select allLogs, AllMetrics and select "Send to Log Analytics workspace"
<br>![[Pasted image 20231214100729.png]]

back in workspace we click Logs and try a query
<br>![[Pasted image 20231214100947.png]]

<br>![[Pasted image 20231214101322.png]]
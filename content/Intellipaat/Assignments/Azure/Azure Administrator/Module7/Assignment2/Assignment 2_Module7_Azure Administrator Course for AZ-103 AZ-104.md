---
tags:
  - azure
title: Implementing an Application Gateway with Path-Based Routing in Azure
---
<!--
**Project Mastery: Implementing Azure Application Gateway for URL-Based Routing!** In this advanced assignment from my Azure Administrator course, I successfully set up and configured an Azure Application Gateway. The task involved creating an application gateway with specific configurations to route traffic to different VMs based on URL paths. This required meticulous setup of backend pools, listeners, and routing rules, providing a hands-on experience in managing traffic distribution in Azure. Successfully implementing URL path-based routing using the Application Gateway was a significant achievement that showcased my ability to optimize network traffic management in complex cloud environments.

#Azure #ApplicationGateway #TrafficManagement #URLRouting #AzureAdministrator #CloudNetworking #ProfessionalDevelopment
-->

> [!info] Module 7: Assignment - 2
> **Tasks To Be Performed:** 
> Create an application gateway with the following configuration: 
> `a.` /vm1 should point to VM1 
> `b.` /vm2 should point to VM2


Using VMs from [[Assignment 1_Module7_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 7]]


I need to create the 'VM1' and 'VM2' directories on the servers.

<br>![[Pasted image 20231209131711.png]]
<br>![[Pasted image 20231209131658.png]]


### Step 2: Create an Application Gateway

1. **I Create an Application Gateway**:
    
    - I search "Application Gateway" and click "+ Create".
    - I start the creation process and fill in the basic information such as name, region, and the VNet where my VMs are located.
    - Under "Tier", I can select "Standard".
    - Will need to create a subnet just for the gateway
      <br>![[Pasted image 20231209123719.png]]
      Click "Manage subnet configuration" and "+ Subnet"
      <br>![[Pasted image 20231209123632.png]]

2. **Frontends**
I'll reuse the same Public IP address as [[Assignment 1_Module7_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 7]]

<br>![[Pasted image 20231209123917.png]]

### Step 3: Configure Backend Pools

1. **I Add Backend Pools**:
    - In the Application Gateway creation wizard, I add two backend pools:
        - One for VM1: I name it `vm1Pool` and add VM1 to this pool.
          <br>![[Pasted image 20231209124107.png]]
        - One for VM2: I name it `vm2Pool` and add VM2 to this pool.
        
        - This is wrong the listener is attached to the pool, so the pool has to have both VMs
        - <br>![[Pasted image 20231209124254.png]]
### Step 4: Configure Settings

1. **I Set Up Listeners**:
    - In "Routing rules" I click "+ Add a routing rule".
    - I create a basic listener for incoming traffic. If HTTPS is required, certificates will need to be configured here.
      <br>![[Pasted image 20231209124834.png]]
2. **I Configure Routing Rules**:
    
    - I add path-based routing rules:
        - For `/vm1`, I create a rule that routes traffic to `vm1Pool`.
          added new "Backend settings" left everything default
          <br>![[Pasted image 20231209125248.png]]
          after clicking "Add multiple targets to create a path-base rule"
          <br>![[Pasted image 20231209125332.png]]
        - For `/vm2`, I create a rule that routes traffic to `vm2Pool`.
          <br>![[Pasted image 20231209125805.png]]
          <br>![[Pasted image 20231209125717.png]]
<br>![[Pasted image 20231209130825.png]]
### Step 5: Review and Create

1. **I Review All Settings**:
    <br>![[Pasted image 20231209131827.png]]
    - I ensure all configurations are correct, including backend pools, listeners, and routing rules.
2. **I Create the Application Gateway**:
    
    - I finalize the setup and create the Application Gateway.

### Step 6: Test the Application Gateway

1. **I Find the Public IP of the Application Gateway**:
    
    - Once the Application Gateway is deployed, I note its public IP address or DNS name.
2. **I Test the URL Path-Based Routing**:
    
    - In a web browser, I test the URL paths:
        - `http://<AppGateway-IP-or-DNS>/vm1` should route to VM1.
        - `http://<AppGateway-IP-or-DNS>/vm2` should route to VM2.

By completing these steps, you have successfully created an Azure Application Gateway that routes traffic based on URL paths to different VMs, according to your assignment's requirements.

---

1 rule has 1 listener, once listener port 80, is taken can be used in another rule, so 1 rule needs to access both vm, which means both VM need to go into one Backend Pool, cuse 1 listener 1 pool
<br>![[Pasted image 20231209155445.png]]


<br>![[Pasted image 20231209155720.png]]
Click on add multiple

<br>![[Pasted image 20231209155819.png]]
<br>![[Pasted image 20231209155857.png]]

<br>![[Pasted image 20231209160015.png]]

<br>![[Pasted image 20231209160206.png]]

<br>![[Pasted image 20231209160245.png]]

<br>![[Pasted image 20231209161239.png]]

<br>![[Pasted image 20231209161521.png]]

Issue:
<br>![[Pasted image 20231209184313.png]]

caused by Backend Setting
<br>![[Pasted image 20231209184356.png]]
<br>![[Pasted image 20231209184521.png]]

So the issue is when the gateway recived a request to acccess the page, it gets header with gateway IP, and that IP is passed to the VMs but the VMs do not have that IP so they dont recognize it. Need to put override so gateway translate to what the vm is expexting
<br>![[gateway_issue_header_overwrite.png]]


---

Need three pools 
1 fort vm1 "VM1_path"
2 for vm2 "VM2_path"
<br>![[Pasted image 20231209194657.png]]

3 for both "VM_Pool"
<br>![[Pasted image 20231209194709.png]]

Initially I had 1 pool ''VM_pool' with both and that was put in 
<br>![[Pasted image 20231209194916.png]]

This cause, when i access IP/vm1 to work then when i refresh it will look for the other server and fail, refresh work, resfresh look for the other server fail, cuse the other servers doesnt have path


For notes
<br>![[Pasted image 20231209195044.png]]
<br>![[Pasted image 20231209195055.png]]


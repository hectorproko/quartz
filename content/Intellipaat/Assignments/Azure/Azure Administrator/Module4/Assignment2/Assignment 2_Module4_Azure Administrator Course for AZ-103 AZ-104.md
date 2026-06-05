---
tags:
  - azure
title: Deploying and Connecting to a Windows VM in Azure via Remote Desktop
---
<!--
**Expanding Azure Expertise: Deploying and Connecting to a Windows VM!** I've just completed an enriching assignment in my Azure Administrator course, focusing on deploying a Windows virtual machine in Azure. The task involved creating the VM in the West US region, configuring it with the necessary specifications, and setting up Remote Desktop Protocol (RDP) access. Successfully deploying and connecting to the VM using Windows Remote Desktop was a practical demonstration of my growing skills in Azure VM management. This assignment provided valuable hands-on experience in Azure’s virtualization capabilities, enhancing my ability to deploy and manage Windows-based environments in the cloud.
-->

> [!info] Module 4: Assignment - 2
> **Tasks To Be Performed:** 
> 1. Create a Windows VM in west US region 
> 2. Open the RDP port 
> 3. Connect to it using Windows Remote Desktop 



### Step 1: Create a Windows VM in the West US Region

1. **I Log into the Azure Portal**:
    
    - I open a web browser and navigate to the [Azure Portal](https://portal.azure.com/). I sign in with my credentials.
2. **I Navigate to Virtual Machines**:
    
    - In the Azure Portal, I select "Virtual machines" from the menu or search for it in the search bar.
3. **I Create the VM**:
    
    - I click on "Create" to start the process of creating a new virtual machine.
    - In the "Basics" tab, I select the appropriate subscription and resource group (or create a new resource group if necessary).
    - I enter a name for my VM.
    - Under "Region", I select "West US".
    - In the "Image" dropdown, I choose a Windows-based image (such as Windows 10 Pro or Windows Server 2019).

### Step 2: Configure the VM

1. **I Choose the VM Size**:
    
    - I select the size for the VM. A size like B1s.
2. **I Set Up Authentication**:
    
    - Under the "Administrator account" section, I choose a username and a strong password that I will use to log in to the VM.
      
3. **I Configure Ports**:
    
    - In the "Public inbound ports" section, I choose to allow selected ports and ensure that "RDP (3389)" is checked to allow Remote Desktop connections to the VM.


### Step 3: Review and Create the VM

1. **I Review the Configuration**:
    - I go through the remaining settings and adjust them as necessary (such as networking, management, advanced settings, etc.).
    - Once I'm satisfied with the configuration, I click "Review + create", and then "Create" to deploy the VM.
      <br>![[crate windows vm azure.png]]

### Step 3: Review and Create the VM

1. **I Review the Configuration**:
    - I go through the remaining settings and adjust them as necessary (such as networking, management, advanced settings, etc.).
    - Once I'm satisfied with the configuration, I click "Review + create", and then "Create" to deploy the VM.

### Step 4: Connect to the VM Using Windows Remote Desktop

1. **I Obtain the VM's IP Address**:
    
    - Once the VM is deployed, I go to the VM's overview page in the Azure Portal.
    - I find the public IP address of the VM displayed on this page.
      <br>![[Pasted image 20231207112437.png]]
2. **I Use Remote Desktop to Connect**:
    
    - I open the Remote Desktop Connection application on my Windows computer.
    - In the "Computer" field, I enter the public IP address of the VM and click "Connect".
      <br>![[Pasted image 20231207112648.png]]
    - When prompted, I enter the username and password I set up for the VM.
      <br>![[Pasted image 20231207112727.png]]
3. **I Start the Remote Session**:
    
    - After entering my credentials, I may receive a warning about the certificate of the VM. As this is expected, I click "Yes" or "Continue" to proceed.
    - The Remote Desktop window opens, showing the desktop of the Windows VM.
      <br>![[Pasted image 20231207112902.png]]


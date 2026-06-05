---
tags:
  - azure
title: Setting Up and Accessing an Ubuntu Virtual Machine in Azure
---
<!--
**Project Insights: Deploying and Managing a Linux VM on Azure!** I've successfully completed an assignment as part of the Azure Administrator course, where I focused on deploying a Linux virtual machine in Azure. The task involved creating a VM in the West US region using an Ubuntu image, configuring the VM settings, and establishing SSH connectivity for remote management. This exercise was crucial in enhancing my skills in Azure VM deployment and management, providing hands-on experience with Azure's virtualization capabilities. It was a valuable opportunity to apply Azure best practices in a real-world scenario, furthering my expertise in cloud-based infrastructure solutions.
-->

> [!info] Module 4: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Create a VM in the west US region 
> 2. Select the Ubuntu image for creating the VM 
> 3. Open the SSH port 4. Connect to the Linux VM using the terminal


### Step 1: Create a VM in the West US Region

1. **I Log into the Azure Portal**:
    
    - I open a web browser and navigate to the [Azure Portal](https://portal.azure.com/). I sign in with my credentials.
2. **I Navigate to Virtual Machines**:
    
    - In the Azure Portal, I select "Virtual machines" from the menu or search for it in the search bar.
3. **I Create the VM**:
    
    - I click on "Create" to start the process of creating a new virtual machine.
    - In the "Basics" tab, I select the appropriate subscription and resource group.
    - I enter a name for my VM.
    - Under "Region", I select "West US".
    - In the "Image" dropdown, I choose an Ubuntu Server image (such as Ubuntu 20.04 LTS).

### Step 2: Configure the VM

1. **I Choose the VM Size**:
    
    - I select the size for the VM. I can use a standard size like B1s or B2s for testing purposes.
2. **I Set Up Authentication**:
    
    - Under the "Administrator account" section, I choose "SSH public key" as the authentication type.
    - I generate a new SSH key pair
3. **I Configure Ports**:
    
    - In the "Public inbound ports" section, I choose to allow selected ports and ensure that "SSH (22)" is checked to allow SSH connections to the VM.

<br>![[azure linux vm creation.png]]

### Step 3: Review and Create the VM

1. **I Review the Configuration**:
    
    - I go through the remaining settings and adjust them as necessary (such as networking, management, advanced settings, etc.).
    - Once I'm satisfied with the configuration, I click "Review + create" to check my settings.
    - I make sure to download the keys when prompted
      <br>![[generate new  key pair download.png]]


### Step 4: Connect to the Linux VM Using the Terminal

1. **I Obtain the VM's IP Address**:
    
    - Once the VM is deployed, I go to the VM's overview page in the Azure Portal.
    - I find the public IP address of the VM displayed on this page.
      <br>![[Pasted image 20231207101724.png]]
2. **I Connect via SSH**:
    
    - I open a terminal on my computer.
    - I use the SSH command to connect to the VM: `ssh <username>@<VM-public-IP>`, replacing `<username>` with the username I set up and `<VM-public-IP>` with the VM's public IP address.
    - If prompted, I accept the host key to continue with the connection.
      <br>![[Pasted image 20231207102427.png]]

2. **I Verify the Connection**:
    
    - Once connected, I should be logged into the VM's shell prompt, ready to execute commands on the VM.
      <br>![[Pasted image 20231207102447.png]]



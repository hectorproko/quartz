---
tags:
  - azure
---
> [!info] Module 7: Assignment - 5
> **Tasks To Be Performed:** 
> 1. Create a VM without public IP address 
> 2. Connect to this VM using bastion host


### Step 1: VM without Public IP
To create VM I follow steps in [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]]
 and sure "Public IP" is set to "None"
<br>![[azure public ip none.png]]

### Step 2: Set Up Azure Bastion

1. **I Navigate to Azure Bastion**:
    
    - In the Azure Portal, I search for "Bastion" and select it.
2. **I Create a Bastion Host**:
    
    - I click "+Crreate" to create a new Bastion host.
    - I select the same virtual network as my VM and fill in the details required for the Bastion service, including a name and the region.
      
      
      <br>![[Pasted image 20231210142256.png]]
    - For Azure Bastion, I do need to create a new public IP address because Bastion will serve as my secure access point.
    - I review and create the Bastion host.
      <br>![[Pasted image 20231210151356.png]]


### Step 3: Connect to the VM using Bastion Host

1. **I Navigate to My VM**:
    
    - Once the Bastion host is provisioned, I go to the VM I want to connect to.
    - In the overview page of the VM, I click "Connect" and choose "Connect via Bastion" from the options.
2. **I Enter Credentials**:
    
    - In the Bastion connection page, I enter the username and password for the VM.
3. **I Establish the Connection**:
    - I click "Connect" to initiate the Bastion service.
      <br>![[Pasted image 20231210151824.png]]
    - A new browser window opens with a secure SSH session to the VM.
      <br>![[Pasted image 20231210152737.png]]




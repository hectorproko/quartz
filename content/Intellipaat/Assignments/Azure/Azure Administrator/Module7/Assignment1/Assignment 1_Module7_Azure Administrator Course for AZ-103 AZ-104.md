---
tags:
  - azure
title: Configuring Azure Load Balancer for Two Apache2 Ubuntu VMs
---
<!--
**Project Achievement: Implementing a Load-Balanced Web Service on Azure!** I've just completed a challenging assignment as part of the Azure Administrator course, where I successfully created and managed a load-balanced web service. The project involved deploying two VMs with Ubuntu and Apache2, customizing their `index.html` to display unique messages, and then configuring an Azure Load Balancer to distribute incoming traffic between them. This hands-on experience was invaluable in understanding the setup and management of load balancers in Azure, ensuring high availability and efficiency of web services. It was a great opportunity to apply Azure networking concepts in a practical scenario.
-->

> [!info] Module 7: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Deploy 2 VMs with Ubuntu and Apache2 installed 
> 2. Change index.html to include the following text 
>    `a.` “This is VM1” on VM1 
>    `b.` “This is VM2” on VM2 
> 3. Create a load balancer which will balance the traffic between these two VMs



### Step 1: Deploy 2 VMs with Ubuntu and Apache2 Installed

- Use [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]] for reference
- Make sure to put the following in "user data"

```bash
#!/bin/bash

# Update the package list
sudo apt-get update

# Install Apache2
sudo apt-get install apache2 -y

# Ensure Apache2 is running and enabled
sudo systemctl start apache2
sudo systemctl enable apache2

# Change the contents of the default Apache page
echo "This is VM1" | sudo tee /var/www/html/index.html
```

<br>![[Pasted image 20231208222410.png]]
### Step 2: Change index.html on Both VMs

1. **I SSH into VM1**:
    
    - I connect to VM1 using SSH.
2. **I Modify index.html on VM1**:
    
    - I run `sudo nano /var/www/html/index.html` (or use any other text editor).
    - I add the text “This is VM1” to the file.
    - I save and close the file.
3. **I Repeat for VM2**:
    
    - I SSH into VM2.
    - I follow the same steps to modify the `index.html`, this time adding “This is VM2”.

### Step 3: Create a Load Balancer

1. **I Navigate to Load Balancers in Azure Portal**:
    
    - I log into the Azure Portal.
    - I search for "Load Balancer" and click "+ Create"
    - Made sure "Type" was "Public"
    - "Frontend IP configuration" picked a Public IP I created in "Public IP Addresses"
      <br>![[Pasted image 20231208223457.png]]
    - "Backend pools" I add my 2 VMs
      <br>![[Pasted image 20231208224628.png]]
2. **I Create a New Load Balancer**:
    
    - I fill in the details for the load balancer, like name, region (should be the same as VMs), and choose Public.
    - I create a new public IP address for the load balancer if it's public.
3. **I Configure the Backend Pool**:
    
    - After creating the load balancer, I navigate to its settings.
    - I add a backend pool and include both VM1 and VM2 in this pool.
      <br>![[Pasted image 20231209095331.png]]
5. **I Configure Load Balancing Rules**:
  - The probe can be set to check the HTTP availability on port 80.
    <br>![[Pasted image 20231209095638.png]]

   
    <br>![[Pasted image 20231209095801.png]]
    - I create a load balancing rule that directs traffic to the backend pool.
    - I set it to listen on port 80 and direct traffic to port 80 on the VMs.
      <br>![[Pasted image 20231209100056.png]]
      <br>![[loadbalancer.png]]

7. **I Test the Load Balancer**:
    
    - I navigate to the public IP address of the load balancer in a web browser.
    - The page should display “This is VM1” or “This is VM2”, depending on which VM is serving the request.
      
      <br>![[Pasted image 20231209100935.png]]
      After refreshing the page 100 times..
      <br>![[Pasted image 20231209100947.png]]


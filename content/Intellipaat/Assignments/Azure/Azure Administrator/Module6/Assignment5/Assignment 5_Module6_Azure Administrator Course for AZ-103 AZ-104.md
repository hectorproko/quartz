---
tags:
  - azure
title: Integrating Azure DNS with a Domain for an Apache2 VM
---
<!--
**Project Breakthrough: Linking Azure VM with a Custom Domain!** In the latest assignment of my Azure Administrator course, I successfully linked an Azure Virtual Machine running Apache2 with a custom domain name. The process involved obtaining a free domain from Freenom, creating an Apache2 VM, and using Azure DNS to point the domain to the VM's IP address. This task not only provided hands-on experience with Azure DNS but also showcased the practical application of linking a custom domain to a cloud-hosted service. It was an enlightening experience in understanding the nuances of domain management and DNS configuration in Azure.

#Azure #DNS #VirtualMachine #CustomDomain #Apache2 #AzureAdministrator #CloudComputing #ProfessionalDevelopment
-->

> [!info] Module 6: Assignment - 5
> **Tasks To Be Performed:** 
> 1. Use the previously created Apache2 VM 
> 2. Get a free domain from freenom.com 
> 3. Use Azure DNS to point this free domain to your VMs IP

### Step 1: Use the Previously Created Apache2 VM
- I use "StaticIPVM" from [[Assignment 4_Module6_Azure Administrator Course for AZ-103 AZ-104|Assignment 4: Module 6]] that has Apache2
- I note down the VM's public IP address for future reference.
  <br>![[staticipvm overview.png]]
### Step 2: Obtain a Free Domain from Freenom.com

Will reuse the subdomain from [[Assignment 3 – Route 53_Module4_AWS Weekday BC = 2301080808#Created a Hosted Zone in Route 53|Assignment 3 – Route 53]] above `temp.hectorproko.com`

### Step 3: Use Azure DNS to Point the Free Domain to the VM's IP

1. **I Create a DNS Zone in Azure**:
    
    - I search for "DNS zones".
    - I create a new DNS zone, entering the domain name I acquired from IONOS.
    - I fill out the necessary information and create the zone.
      <br>![[Pasted image 20231208171957.png]]
2. **I Set Up DNS Records**:
    
    - In the newly created DNS zone, I add an "A" record.
    - For the "A" record, I use "@" as the name (for the root domain) and enter the public IP address of my Apache2 VM as the value.
    - I save the record.
      <br>![[Pasted image 20231208182346.png|350]]
      <br>![[Pasted image 20231208184635.png]]
1. **I Update the Domain's Nameservers**:
    
    - Azure provides me with a set of nameservers for my DNS zone. I note these down.
    - I go back to IONOS (or my domain provider) and update the domain's nameserver settings to the ones provided by Azure.
     - Just as I did in [[Assignment 3 – Route 53_Module4_AWS Weekday BC = 2301080808#^c3e5c2|Assignment 3 – Route 53]] 
4. **I Verify the Domain Setup**:
    
    - I wait for the DNS changes to propagate, which can take some time.
    - Once propagated, I navigate to my new domain in a web browser to see if it loads the Apache2 default page hosted on my Azure VM.
      <br>![[Pasted image 20231208184406.png]]



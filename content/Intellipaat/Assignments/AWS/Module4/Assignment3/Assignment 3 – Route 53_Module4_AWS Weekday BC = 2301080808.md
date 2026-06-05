---
tags:
  - AWS
title: Configuring Traffic Management with Route 53 in AWS
---
<!--
**Navigating DNS with AWS Route 53 for Robust Web Solutions!** I've completed an engaging hands-on assignment focusing on AWS Route 53 for domain name system management. I set up an EC2 instance running Apache web server and created a hosted zone in Route 53, successfully routing traffic to my instance using a domain name. This task strengthened my grasp on managing DNS records and integrating AWS services for a scalable and reliable web solution.
-->

> [!info] Module 4: Route 53 Assignment
> **Problem Statement:** 
> You work for XYZ Corporation that uses on premise solutions and some limited number of systems. With the increase in requests in their application, the load also increases. So, to handle the load the corporation has to buy more systems almost on a regular basis. Realizing the need to cut down the expenses on systems, they decided to move their infrastructure to AWS. 
> 
> **Tasks To Be Performed:**
> 1. Use the Route 53 hosted zone created in the assignment. 
> 2. Route the traffic to an EC2 instance with an Apache web server running in it using its IP address.


### Created an EC2 instance with a Web Server:


1. Launched the EC2 Instance Noted the Public IP.
   <br>![[Pasted image 20231004142226.png]]
2. Installed the Web Server 
```bash
#!/bin/bash
# Update the package manager
sudo yum update
# Install the Apache web server
sudo yum install httpd -y
# Start the Apache web server
sudo systemctl start httpd
# Create a web page
echo "Hello from Web Server!" | sudo tee /var/www/html/index.html
# Restart the Apache web server
sudo systemctl restart httpd
```
3. Testing.
   <br>![[Pasted image 20231004142258.png]]

### Created a Hosted Zone in Route 53:

1. Opened the **Route 53 console** in the AWS Management Console.
2. In the left navigation pane, clicked on "Hosted zones".
3. Clicked the "Create hosted zone" button.
4. In the "Create Hosted Zone" form:
    - **Domain Name**: Entered my domain name `temp.hectorproko.com`.
    - **Type**: Selected "Public Hosted Zone".
5. Clicked "Create".
   <br>![[Pasted image 20231004143005.png]]

After creating the hosted zone, Route 53 provides me with a set of **NS (Name Server) records**. We'll need to update out domain registrar's nameserver settings with these NS records to delegate the domain's DNS to Route 53.

> [!NOTE] Updating Registrar
> 
> <br>![[Pasted image 20231004143128.png]]
> *I've mapped all four Name Servers to my domain 'temp.hectorproko.com' within the [[IONOS]] platform, inputting them one by one.*
> 
> <br>![[Pasted image 20231004153525.png]]
> 






2. Clicked on "Create Record" to point `temp.hectorproko.com` to Web Server.
3. In the record creation form:
    - **Name**: Entered the domain I wish to route. Left it blank to route to the root domain.
    - **Type**: Chose "A - IPv4 address".
    - **Alias**: Select "No".
    - **Value/Route traffic to**: Entered the public IPv4 address of your EC2 instance.
    - **TTL (Time to Live)**: A common value is 300 seconds, but you can adjust based on your needs.
    - **Record type**: Chose "Simple routing".
      <br>![[Pasted image 20231004151545.png]]
4. Clicked "Create records".
   <br>![[Pasted image 20231004151639.png]]


### Verifying the Route:


Opened a browser and enter my domain name. I should see the Apache web server page I created.
   
> [!success]
>    <br>![[Pasted image 20231004153026.png]]


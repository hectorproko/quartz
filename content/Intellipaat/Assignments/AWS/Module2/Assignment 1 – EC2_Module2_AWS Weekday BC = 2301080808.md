---
tags:
  - AWS
  - Nginx
title: Launching and Configuring an Ubuntu EC2 Instance with Nginx on AWS
---
<!--
As part of my continuous learning and hands-on practice in cloud computing, I successfully completed a #miniproject where I demonstrated the ability to launch and configure an Ubuntu #EC2 instance on #AWS and set up an #Nginx web server.
-->
 

> [!NOTE]
> Problem Statement: 
> You work for XYZ Corporation. Your wants to launch a new web-based application using AWS Virtual Machines. Configure the resources accMdingly for the tasks. Tasks To Be Performed: 
> 1. Create an instance in the US-East-l (N. Virginia) region with an Ubuntu OS and install Nginx for making them web servers. 
> 2. Change the default website with a page displaying the message: "Hello World"


I'll log in to my AWS account and go to **EC2 Dashboard** and click **Launch instance**. 
<br>![[LaunchinstanceButton.png]]
*I happen to be in the correct Region (N. Virginia)*

I'll name my instance **Web_Server**
Pick Ubuntu:
<br>![[Pastedimage20230822213106.png]]  

Instance type:
<br>![[Pastedimage20230822213124.png]]  

I'll make sure to **Allow** HTTP, HTTPS 
<br>![[Pastedimage20230822213324.png]]  

In **Advanced details** I'll run some commands in **User data** to make some changes to the page
<br>![[Pastedimage20230822214313.png]]  

The full script:
```bash
#!/bin/bash

# Update package lists
apt-get update -y

# Install Nginx
apt-get install nginx -y

# Enable and start Nginx service
systemctl enable nginx
systemctl start nginx

# Change the default website content
echo '<!DOCTYPE html>
<html>
<head>
    <title>Hello World</title>
</head>
<body>
    <h1>Hello World</h1>
</body>
</html>' > /var/www/html/index.html

# Reload Nginx to pick up the new configuration
systemctl reload nginx
```


I'll wait for instance to be ready  
<br>![[Pasted image 20230822213658.png]]  

I'll get the instance's **Public IP**    
<br> ![[Pastedimage20230822213826.png]]

When I check in the browser the expected page appears:  
<br> ![[Pastedimage20230822213731.png]]

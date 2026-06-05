---
tags:
  - AWS
title: Implementing a Scalable, Multi-Tier Web Architecture on AWS with EC2 and RDS
---
<!--
**Major Project: Deploying a Multi-Tier Website on AWS EC2!** I'm thrilled to share my experience with a significant project where I deployed a multi-tier website using Amazon EC2. The project involved setting up an EC2 instance, configuring auto-scaling for high availability, creating an RDS instance for database management, and ensuring seamless connectivity between the EC2 instance and the RDS. My tasks included launching the instance, enabling auto-scaling, establishing database connections, and transferring the website's files using FileZilla. This project was an excellent opportunity to apply and deepen my knowledge in AWS services, demonstrating the practicality and efficiency of AWS for scalable web hosting solutions.
-->
 

> [!info] Project - 1: Deploying a Multi-Tier Website Using AWS EC2
> **Description:** 
> Amazon Elastic Compute Cloud (Amazon EC2) provides scalable computing capacity in the Amazon Web Services (AWS) cloud. Using Amazon EC2 eliminates your need to invest in hardware up front so you can develop and deploy applications faster. You can use Amazon EC2 to launch as many or as few virtual servers as you need, configure security and networking, and manage storage. Amazon EC2 enables you to scale up or down to handle changes in requirements or spikes in popularity, reducing your need to forecast traffic. 
> 
> **Problem Statement:** 
> Company ABC wants to move their product to AWS. They have the following things set up right now: 
> 1. MySQL DB 
> 2. Website (PHP) 
> 
> The company wants high availability on this product, therefore wants Auto Scaling to be enabled on this website. 
> 
> **Steps To Solve:** 
> 1. Launch an EC2 Instance 
> 2. Enable Auto Scaling on these instances (minimum 2) 
> 3. Create an RDS Instance 
> 4. Create Database & Table in RDS instance: 
>    a. Database name: intel 
>    b. Table name: data 
>    c. Database password: intel123 
> 5. Change hostname in website 
> 6. Allow traffic from EC2 to RDS instance 
> 7. Allow all-traffic to EC2 instance

# EC2

Referring to [[Assignment 1 – EC2_Module2_AWS Weekday BC = 2301080808|Assignment 1 – EC2]], I have configured an EC2 instance with the following specifications:

- Selected key pair: daro.io
- Operating System: Ubuntu
- Chosen VPC: MyVPC
- Deployed in a public subnet
- Ports 22 and 80 are open.

---
Convert the private key 'daro.io.pem' using PuTTYgen to `.ppk` format
<br>![[Pasted image 20231030160515.png|450]]

I get the Public IP of the instance
<br>![[ec2 public ip.png]]


I use PuTTY to connect by loading the newly converted 'daro.io.ppk' key into the PuTTY configuration.
<br>![[PuTTY .ppk key.png]]

Once the login prompt appears, I use 'ubuntu' as the username.
<br>![[Pasted image 20231030161441.png]]

I proceed with the following commands
```bash
#!/bin/bash
# Update all system packages to their latest versions
sudo apt-get update -y
# Install the Apache2 web server
sudo apt-get install apache2 -y
# Add the 'ondrej/php' Personal Package Archive (PPA) to get PHP versions not provided in standard Ubuntu repositories
sudo add-apt-repository -y ppa:ondrej/php -y
# Install PHP version 5.6 along with mysql-client and the mysqli extension for PHP
sudo apt install php5.6 mysql-client php5.6-mysqli -y
```


# Create RDS
Using [[Assignment 3 – AWS Migration_Module11_AWS Weekday BC = 2301080808|Assignment 3 – AWS Migration]] as a reference, I create an RDS database instance with the following specifications:

- Engine: MySQL
- Tier: Free tier
- DB Instance Identifier: MySQLDatabase
- Template: Free tier
- Master Username: admin
- Password: `[SecuredPassword]`
- Chosen VPC: MyVPC
- Public Access: Checked
- Instance Configuration: db.t2.micro.

I ensure that the initial database name 'Intel' is set up.
<br>![[Pasted image 20231030164833.png|400]]


<br>![[Pasted image 20231030165859.png]]

Clicked on "View connection details"
<br>![[Pasted image 20231101145437.png|500]]

```
mysqldatabase.cnk8gecauqro.us-east-1.rds.amazonaws.com
```

---
I copy the Security Group from the EC2
<br>![[Pasted image 20231101145537.png]]
```
launch-wizard-73
```

I click on the security group associated with the RDS to edit its settings.
<br>![[Pasted image 20231101145729.png]]

Once I open the security group for the RDS, I edit its inbound rules to restrict access solely to my EC2 instance through the EC2's assigned security group `launch-wizard-73`.
<br>![[Pasted image 20231101145941.png]]

I establish a connection to the RDS from the EC2 instance.
<br>![[Pasted image 20231101150809.png]]

I create a table named 'data' in the 'Intel' database.
<br>![[Pasted image 20231101154523.png]]

---

I will transfer my website files to my EC2 instance for hosting using FileZilla.

I have installed the FileZilla client, which is available for download at the [FileZilla project website](https://filezilla-project.org/download.php#close).

I will establish a connection to the EC2 instance using the FileZilla client by following these steps: Navigate to `File > Site Manager`, select the 'SFTP' protocol, enter the [[ec2 public ip.png|EC2 public IP]], and use the same .`ppk` key for authentication that we [[PuTTY .ppk key.png|used with PuTTY]].

<br>![[Pasted image 20231101151811.png]]

In the FileZilla client, I select the 'index.php' file and 'images' folder from the 'Local site' panel, which is my source directory. In the 'Remote site' panel, I target the '/var/www/html' directory as my destination on the EC2 instance. By right-clicking the selected items and clicking 'Upload', I initiate the process to transfer the files from my local machine to the remote server.

> [!attention]
> Before I can transfer files using FileZilla with the 'ubuntu' user, I must set the proper permissions for the 'html' directory by executing the command

<br>![[Pasted image 20231101152041.png]]

I verify 'index.php' for database connection setup.
<br>![[Pasted image 20231101154040.png]]




`sudo nano /etc/apache2/sites-available/000-default.conf`
<br>![[Pasted image 20231101153224.png]]
Always ensure to check the syntax of the Apache configuration files before restarting the service:
`sudo apache2ctl configtest`
If you get `Syntax OK`, then it's safe to proceed with the restart. Otherwise, check for any errors reported by the command and correct them before restarting Apache.

After you've modified the file, restart Apache to apply the changes:
`sudo systemctl restart apache2`

---

> [!NOTE]
> To ensure Apache serves 'index.php' by default, I take the following step:
> 
> - I edit the Apache default configuration file by executing:
> ```bash
> sudo nano /etc/apache2/sites-available/000-default.conf
> ```
> <br>![[Pasted image 20231101153224.png]]
> Within this file, I prioritize 'index.php' by setting it as the first value in the 'DirectoryIndex' directive to ensure it's served as the default page.
> 
> Before applying changes, it's crucial to verify the configuration's syntax. I run the following command:
> ```bash
> sudo apache2ctl configtest
> ```
> 
> If I receive a `Syntax OK` result, it's safe for me to restart Apache with:
> ```bash
> sudo systemctl restart apache2
> ```
> 

To verify that the webpage is active and to test data input functionality, I navigate to my [[ec2 public ip.png|EC2 instance's public IP address]] in a web browser.
<br>![[Pasted image 20231101154222.png]]
Click "Submit" 


I verify the data input by executing a SELECT query on the 'data' table within the 'intel' database. The query results confirm that the input has been successfully recorded, as shown by the retrieved data entries.

> [!success]
> <br>![[Pasted image 20231101154650.png]]



---


Now that my EC2 instance is fully equipped for hosting, I ensure it is also capable of auto-scaling, following the procedures outlined in [[Assignment 2 – Auto Scaling_Module4_AWS Weekday BC = 2301080808|Assignment 2 – Auto Scaling]]. My first step is to create an Amazon Machine Image (AMI) from the existing EC2 instance. Next, I craft a Launch Template incorporating this AMI. Finally, I establish an Auto Scaling group utilizing this Launch Template and configure it to maintain a desired capacity, with the parameters set to a minimum of 2 instance and a maximum of 3 instances. This setup guarantees that the hosting infrastructure adjusts automatically to the incoming traffic, maintaining efficiency and cost-effectiveness.
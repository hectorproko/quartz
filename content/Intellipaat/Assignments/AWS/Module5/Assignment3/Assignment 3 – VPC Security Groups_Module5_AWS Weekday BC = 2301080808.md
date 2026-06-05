---
tags:
  - AWS
title: Establishing Secure EC2 Communications with VPC Security Groups in AWS
---
<!--
**Mini-Project: Implementing Robust Security with AWS VPC Security Groups!** I recently completed a hands-on project where I set up and configured security groups within a VPC to establish a secure network environment. The project involved creating two EC2 instances, named Master and Client, in a public subnet and configuring security groups to ensure that the Client instance could only be accessed via SSH through the Master instance. This task underscored the importance of precise security configurations in cloud environments and enhanced my skills in crafting secure, efficient AWS networks.
--> 
> [!info] Module 5: VPC Security Groups Assignment
>  **Problem Statement:** 
>  Working for an organization, you are required to provide them a safe and secure environment for the deployment of their resources. They might require different types of connectivity. Implement the following to fulfill the requirements of the company. 
>  
>  **Tasks To Be Performed:** 
>  1. Create 2 EC2 instances in any public subnet of any VPC and name them Master and Client. 
>  2. Using security groups, make sure that the Client instance can only be accessed (SSH) through the Master instance.


### **1. Creating Two EC2 Instances:**

**Step 1:** I began by navigating to the AWS Management Console. From there, I accessed the EC2 dashboard.

**Step 2:** On the EC2 dashboard, I clicked on the “Launch Instance” button to create a new EC2 instance.

**Step 3:** I went through the instance configuration process
- I ensured the instance was being launched within a public subnet

**Step 4:** For the "Add Tags" step, I added a tag with the key as “Name” and value as “Master” for the first instance.

**Step 5:** I repeated the process for the second EC2 instance, but this time naming it "Client".
<br>![[Pasted image 20230925222840.png]]
### **2. Configuring Security Groups for Secure Access:**

**Step 1:** Once both instances were running, I navigated to the "Security Groups" section under the EC2 dashboard.

**Step 2:** I created a new security group named “MasterSG” for the Master instance. I allowed inbound SSH traffic from any IP address so I could directly SSH into it when needed.

**Step 3:** I associated the "MasterSG" with the Master instance
<br>![[Pasted image 20230925221534.png]]

**Step 4:** I then created another security group named “ClientSG” for the Client instance. Instead of allowing direct SSH access from any IP, I limited the inbound SSH traffic to only come from the private IP address of the Master instance. This ensures that the Client can only be accessed (SSH) through the Master instance.
<br>![[Pasted image 20230925221022.png]]
*I specify the instance Master through the security group that is attached to it.*

I associated the "ClientSG" with the Client instance.
<br>![[Pasted image 20230925221717.png]]
*Security Groups are applied to interfaces*


---

### **Validation:**

To validate the setup, I did the following:

#### Step 1: 
I SSH-ed into the Master instance using its public IP address.

#### Step 2: 
From the Master instance, I then tried to SSH into the Client instance using its private IP. The connection was successful, which validated our security group configuration.


**Preparing for SSH into Client Instance**:
   - To SSH into the private instance from the public instance, I needed the .pem key of the Client instance.
   - I securely transferred and stored this key temporarily on the Master instance as `client_private_key`.

```bash
[ec2-user@ip-172-31-81-207 ~]$ nano client_private_key
[ec2-user@ip-172-31-81-207 ~]$ chmod 400 client_private_key
```
   
  **SSH into the Client Instance**:
   
   - Using the transferred key, I initiated an SSH connection from the Master instance to the Client instance:

> [!success]
> ```
> [ec2-user@ip-172-31-81-207 ~]$ ssh -i client_private_key ec2-user@172.31.95.129
> The authenticity of host '172.31.95.129 (172.31.95.129)' can't be established.
> ED25519 key fingerprint is SHA256:zKm3eFUmNWwJSWTgcOpAoDvv3nmNxX1HC3KGxdZvFes.
> This key is not known by any other names
> Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
> Warning: Permanently added '172.31.95.129' (ED25519) to the list of known hosts.
>    ,     #_
>    ~\_  ####_        Amazon Linux 2023
>   ~~  \_#####\
>   ~~     \###|
>   ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
>    ~~       V~' '->
>     ~~~         /
>       ~~._.   _/
>          _/ _/
>        _/m/'
> [ec2-user@ip-172-31-95-129 ~]$ 
> ```
> 

#### Step 3:
To further ensure the setup's security, I attempted to SSH directly to the Client instance using "EC2 Instance Connect" as expected connections only work from the Master instance. 

> [!Fail]
> <br>![[Pasted image 20230925224234.png]]





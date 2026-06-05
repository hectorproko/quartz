---
tags:
  - AWS
---
==PENDING CLEANUP==
 

> [!note]
> **Problem Statement:**
> You work for XYZ Corporation and the current requirement in the organization is for faster file sharing, which can also help in data replication from on-premises infrastructure.
> 
> Tasks To Be Performed:
> 1. Create an FSx file system for a Windows file server:
>    a. Make sure you have AWS Managed Active Directory with a valid domain name
>    b. Connect it to your Windows EC2 server
> 2. Create an FSx file system for Lustre and attach it to an Amazon Linux 2 instance.


<br>![[Pasted image 20230823133404.png]]

The DNS we need is the DNS of Directory Service
Make it member of the Directory with the domain service you cireated

When you run the command in cmd
<br>![[Pasted image 20230823150829.png]]

#### Part 1: Setting up Directory Service

1. Navigate to AWS Directory Service and choose to **Set up a directory**.
2. Select [[AWS Managed Microsoft AD]].
3. Click on "Next".
4. Choose the standard edition and provide a DNS name. You can use an existing DNS name or create a new one.
   <br>![[Pasted image 20230823161731.png]]
   Directory DNS name: `corp.example.com`
5. Set an admin password. Ensure it includes at least three out of these four categories: lowercase, uppercase, numeric, and special characters.
6. Configure the VPC settings:
    - Left Defaults
7. Review your settings and click "Create Directory". 
   <!--The creation might take up to 30 minutes.-->
<br>![[Pasted image 20230823162031.png]]

Once Status is Active I click on the Directory ID and navigate to **Networking & security** tab to extract DNS address for later use
<br>![[Pasted image 20230823165858.png]]
```bash
#DNS address
172.31.30.171
172.31.7.154
```

#### Part 2: Creating Amazon FSx for Windows File Server

1. Navigate to Amazon FSx.
2. Click "**Create File System**".
3. Choose Amazon FSx for Windows File Server.
4. Provide a File System name.
   `FSx File System`
5. Choose the deployment type: Multi-AZ or Single-AZ.
6. Set the storage capacity to a minimum of 32 GiB.
7. **Network & security** left defaults
8. **Windows authentication** selected previously created directory. `corp.example.com`
   <br>![[Pasted image 20230823162630.png]]
9. Review settings and click "Create File System". 
   <!--Creation can take up to 30 minutes.-->
<br>![[Pasted image 20230823165010.png]]
Once ready we click on it and **Attach** to get Attach instructions
```bash
net use Z: \\fs-045731302b7827386.corp.example.com\share
```


#### Part 3.0: Create EC2 Instance with Windows OS
**EC2 Dashboard** and click **Launch instance**. 
<br>![[LaunchinstanceButton.png]]
I'll name my instance **Windows** and pick:
<br>![[Pasted image 20230823170454.png]]
*Left the rest as defaults*

#### Part 3.1: Connecting to the Windows Instance

1. Go to the AWS EC2 dashboard and choose your Windows machine.
2. Click "Connect" and download the RDP file.
   <br>![[Pasted image 20230823170921.png]]
3. Use your private key file to decrypt the password.
4. Use the decrypted password to connect through RDP to the instance.
   <br>![[Pasted image 20230823171155.png|400]]

#### Part 4: Configuring Network Settings

1. Open "Control Panel" on your Windows instance.
2. Navigate to "Network and Internet" -> "Network and Sharing Center" -> "Change Adapter Settings".
3. Go to "Ethernet" -> "Properties" -> "Internet Protocol Version 4 (TCP/IPv4)".
4. Use the following DNS server addresses and input DNS from your AWS directory.
   <br>![[Pasted image 20230823171727.png]]

#### Part 5: Joining the Domain

1. Navigate to "System" -> "Advanced System Settings" -> "Computer Name" -> "Change".
2. Choose "Member of Domain" and input your domain name.
   <br>![[Pasted image 20230823172125.png]]
3. Provide the admin username and password.
   User: `Admin`
   <br>![[Pasted image 20230823172658.png]]

#### Part 6: Mounting the File System

1. Open Command Prompt.
2. Run a command to map the FSx file system to a drive letter
   `net use Z: \\fs-045731302b7827386.corp.example.com\share`
   
3. When prompted, provide your directory credentials.

You should now see the Amazon FSx mapped to a drive on your Windows instance.

[[FSx for Lustre#FSx For Lustre Creation]]


# Issue

<br>![[Pasted image 20230824081624.png]]

```bash
Microsoft Windows [Version 10.0.20348.1906]
(c) Microsoft Corporation. All rights reserved.

C:\Users\Administrator>net use Z: \\amznfsxobtpwo8x.corp.example.com\share
System error 53 has occurred.

The network path was not found.


C:\Users\Administrator>
```

Join the domain manually and through EC2

Cant ping but DNS works
<br>![[Pasted image 20230824081937.png]]
Was joined to domain
<br>![[Pasted image 20230824083240.png]]

# Possible Solutions
Follow this tutorial:
[Step 1: Create your file system - Amazon FSx for Windows File Server](https://docs.aws.amazon.com/fsx/latest/WindowsGuide/getting-started-step1.html)



# 2. Create an FSx file system for Lustre and attach it to an Amazon Linux 2 instance.

### Step 1: Launch FSx for Lustre

1. Navigate to the Amazon FSx service in the AWS Console **Create file System**
2. Select "Amazon FSx for Lustre" and click on it.
    Name it Lustre and picked minimum storage of 1.2TiB
Click Create file system

> [!NOTE]
> VPC Security GroupsInfo
> Specify VPC Security Groups to associate with your file system’s network interface.
> Choose VPC security group(s)
> - sg-e7a6a0dd (default)
>   The VPC Security Groups associated with your file system’s network interfaces must allow inbound Lustre traffic (TCP ports 988, 1018-1023).

> [!attention]
> Might want to harden this

<mark style="background: #FFF3A3A6;">Fsx  File systems</mark>

<br>![[Pasted image 20230824145425.png]]
Once available click on it and click Attach to get mount commands

```bash
#Create a new directory
sudo mkdir /fsx
#Mount
sudo mount -t lustre -o noatime,flock fs-0e0819e73ccdda5f1.fsx.us-east-1.amazonaws.com@tcp:/3lm7tbev /fsx
```

---

### Step 3: Network and Security

1. Select the default VPC.
2. Open your machine's security group to edit inbound rules.
    - Add a custom TCP rule to allow inbound traffic on TCP port 988.
    - Add another custom TCP rule to allow port range 1021-1023.
3. Save the changes.


from default VPC's CIDR `172.31.0.0/16`. This is aimed at enabling the instance to communicate with the EFS **mount target**.
Name Lustre
<br>![[lustre security group.png]]

---

### Step 4: EC2 Setup

1. Ensure you have an EC2 instance running Amazon Linux.
   
   attach lustre Security Group
   <br>![[Pasted image 20230824145106.png]]

1. Connect to your EC2 instance.
2. Update the instance by running `sudo yum update`.
3. Install dependencies needed for FSx for Lustre with the appropriate commands. 
```bash
sudo amazon-linux-extras install -y lustre2.10
```

### Step 5: Attach FSx to EC2

1. Navigate back to the FSx console and click "Attach" on your file system.
2. Copy the provided command to create a directory on your EC2 instance. Run this command.
3. Copy and run the provided command to mount the FSx file system to the directory you created.
<br>![[Pasted image 20230824145753.png]]

<br>![[Pasted image 20230824150021.png]]

https://docs.aws.amazon.com/fsx/latest/FileCacheGuide/install-lustre-client.html
sudo yum update -y
sudo yum install -y lustre-client



---



---

### Step 6: Verify

1. Run verification commands to ensure the FSx filesystem is successfully mounted.
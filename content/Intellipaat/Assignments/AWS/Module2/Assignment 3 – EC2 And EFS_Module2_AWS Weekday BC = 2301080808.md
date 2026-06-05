---
tags:
  - AWS
title: Integrating Multi-OS EC2 Instances with Elastic File System (EFS) on AWS
---
<!--
Just completed a mini-project on AWS, focusing on integrating multi-OS EC2 instances with Elastic File System (EFS). I successfully connected an EFS to three different EC2 instances running Ubuntu, Red Hat Linux, and Amazon Linux 2, demonstrating the versatility and scalability of AWS services. This project was a valuable exercise in managing complex cloud environments and deepening my understanding of AWS storage solutions.
-->

> [!NOTE]
> **Tasks To Be Performed:**
> 1. Create an EFS and connect it to 3 different EC2 instances. Make sure that all instances have different operating systems. For instance, Ubuntu, Red Hat Linux and Amazon Linux 2.
> 

First I'll create 2 **Security Groups**
<br>![[Pastedimage20230823102350.png]]
**SSH** - Allows SSH connection from anywhere
**EFS/NFS** - Allows NFS traffic (typically on port 2049) from default VPC's CIDR `172.31.0.0/16`. This is aimed at enabling the instance to communicate with the EFS **mount target**.
<!-- 
Since this is only for allowing NFS traffic, it should have an outbound rule that permits traffic to the EFS mount target on port 2049.

So in EC2 allow outboud but in mount target allow inboud?
-->
<br>![[Pastedimage20230823083807.png]]

I'll create an **EFS** filesystem called **EFS_for3Instance** at `Amazon EFS > Filesystems > Create file system`
<br>![[Pastedimage20230823103312.png]]

I'll click on **EFS_for3Instance** and navigate to **Network** tab to replace the **Security Group** associated with **Mount Targets** with the one I create **EFS/NFS**
<br>![[Pastedimage20230823111350.png]]

I click the button **Attach** to get the **EFS**'s mount command. In this case is the following:
```
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-035a12e0097bf3b55.efs.us-east-1.amazonaws.com:/ efs
```


I will now create 3 EC2 Instances Ubuntu, Red Hat and Amazon Linux. 
They will all have the same **Security Groups** that I created attached
<br>![[Pastedimage20230823112024.png]]

For the EC2 instance to work I need to install **Amazon EFS client** in them. Sense all 3 instances have different OS the commands are different

> Followed AWS Doc: 
> https://docs.aws.amazon.com/efs/latest/ug/installing-amazon-efs-utils.html#installing-other-distro


I'll do this installation at the time I launch the Instance through Advance details - User data
<br>![[Pastedimage20230823112643.png]]

Ubuntu's - *User data*
```bash
#!/bin/bash
sudo apt-get update
sudo apt-get -y install git binutils
git clone https://github.com/aws/efs-utils
cd /efs-utils
./build-deb.sh
sudo apt-get -y install ./build/amazon-efs-utils*deb

# Create the directory where the EFS will be mounted
sudo mkdir /efs

# Mount the EFS filesystem
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-035a12e0097bf3b55.efs.us-east-1.amazonaws.com:/ /efs
```

Red Hat's - *User data*
```bash
#!/bin/bash
sudo yum -y install git
git clone https://github.com/aws/efs-utils
cd /efs-utils
sudo yum -y install make
sudo yum -y install rpm-build
sudo make rpm
sudo yum -y install ./build/amazon-efs-utils*rpm

# Create the directory where the EFS will be mounted
sudo mkdir /efs

# Mount the EFS filesystem
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-035a12e0097bf3b55.efs.us-east-1.amazonaws.com:/ /efs
```

Amazon Linux 2's - *User data*
```bash
#!/bin/bash

# Install amazon-efs-utils
sudo yum install -y amazon-efs-utils

# Create the directory where the EFS will be mounted
sudo mkdir /efs

# Mount the EFS filesystem
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-035a12e0097bf3b55.efs.us-east-1.amazonaws.com:/ /efs
```

<br>![[Pastedimage20230823114001.png]]

Now I'll connect to each instance to check:

**Ubuntu:**
<br>![[Pastedimage20230823113731.png]]
I see the mounted EFS and create a test file `fromUbuntu` inside

**Amazon Linux:**
<br>![[Pastedimage20230823113844.png]]

**Red Hat:**
<br>![[Pastedimage20230823114135.png]]


Everything works as expected


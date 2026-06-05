---
tags:
  - AWS
title: Launching a Linux EC2 Instance with Attached and Resized EBS Storage
---
<!--
**Recently completed a mini-project on AWS where I launched a Linux EC2 instance and managed Elastic Block Store (EBS) volumes. I successfully attached and resized an EBS volume, enhancing my practical skills in cloud storage solutions. This task was a great way to deepen my understanding of resource management in AWS.**
-->

> [!NOTE]
> **Problem Statement:** 
> You work for XYZ Corporation. Your tnrporation wants to launch a new web-based appliætion using AWS Virtual Machines. Configure the resources accMdingly with appropriate storage for the tasks. 
> 
> **Tasks To Be Performed:** 
> 1. Launch a Linux EC2 instance. 
> 2. Create an EBS volume with 20 GB of storage and attach it to the created EC2 instance. 
> 3. Resize the attached volume and make sure it reflects in the connected instance.


I'll log in to my AWS account and go to **EC2 Dashboard** and click **Launch instance**. 
<br>![[LaunchinstanceButton.png]]

I'll named the instance **Assig2** and keep the defaults
<br>![[Pastedimage20230822221059.png]]

To create an **EBS** I'll go to `Elastic Block Store > Volumes > Create volume`
I'll keep the default except for **Size** and click **Create Volume**
<br>![[Pastedimage20230822220932.png]]

The Volume is ready
<br>![[Pastedimage20230822221356.png]]

To attach the volume I select it and in `Actions > Attach volume`

I make sure I select the right instance
<br>![[Pastedimage20230822221702.png]]

I'll connect to the instance using **EC2 Instance Connect** to check the newly attached **Volume** by running `lsblk`
<br>![[Pastedimage20230822222320.png]]
*Shows `xvdf` with expected size of 20GB*

Now we resize the Volume by going back to `Elastic Block Store > Volumes > Actions > Modify volume`.
<br>![[Pastedimage20230822222807.png]]
*We change it to 30Gib*

We wait for the changes to complete. Progress shown in column Volume State.

Then I check again in the EC2 instance
<br>![[Pastedimage20230822223508.png]]

We confirmed the new size is reflected

<!-- This is a comment and won't appear in the reading view 
To reflect the new size in the instance, you may need to resize the filesystem. For an ext4 filesystem, you can use:
sudo resize2fs /dev/xvdf
-->








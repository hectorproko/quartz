---
tags:
  - AWS
title: Transferring and Deploying a Local Ubuntu Server to AWS EC2
---
<!--
**Mini-Project: Advanced AWS Migration and VM Import/Export!** In a step further into AWS migration, I tackled the challenge of transferring a virtual machine to AWS, converting it into an Amazon Machine Image (AMI) for scalability and flexibility. Starting with exporting an Ubuntu server VM, I uploaded it to an S3 bucket, then used the AWS VM Import/Export service to create an AMI. Subsequently, I launched an EC2 instance using the new AMI, demonstrating the seamless transition from on-premise to cloud. This project provided practical experience in cloud migration strategies, enhancing my ability to manage and execute complex migrations efficiently.
-->

> [!info] Module 11: Migration Assignment - 2
> **Problem Statement:**
> You work for XYZ Corporation. Your corporation wants to move their infrastructure to the cloud to improve the performance and availability of the application hosted. You are given the opportunity to accomplish the tasks for successful migration.
> 
> **Tasks To Be Performed:** 
> 1. Export the Ubuntu server VM created in [[Assignment 1 – AWS Migration_Module11_AWS Weekday BC = 2301080808|task 1]]. 
> 2. Upload it to an S3 bucket. 
> 3. Use that VM to create an AMI and create an EC2 instance with it. 


Continuing from [[Assignment 1 – AWS Migration_Module11_AWS Weekday BC = 2301080808|Assignment 1 – AWS Migration]]

### Exporting the Ubuntu Server VM:

1. I open the Oracle VM VirtualBox Manager where I can see my "UbuntuDesktop for Migration" VM.
2. I right-click on the VM and select 'Export to OCI'.
   <br>![[Pasted image 20231015223909.png]]
   
3. I follow the prompts and export it in the Open Virtualization Format (OVF).

**Uploading the .OVA to an S3 Bucket:**

4. I log into my AWS Management Console.
5. I navigate to S3 and select existing bucket  `my-test-bucket-unique-name`.
   <br>![[Pasted image 20231015175057.png|400]]
6. I click on 'Upload', select the exported .ova file, and start the upload.
   <br>![[Pasted image 20231015224028.png]]
   <br>![[Pasted image 20231015224130.png]]
   <br>![[Pasted image 20231015230609.png]]
   
### Creating the "MigrationRole" in AWS IAM

1. **Role Creation**:
    - I navigate to the AWS IAM console and select the option to create a new role.
    - Named the role "MigrationRole".
2. **Trust Relationship**:
    - I choose the "Custom trust policy" option.
    - Insert the trust policy with the following details:
	```json
	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
	        "Service": "vmie.amazonaws.com"
	      },
	      "Action": "sts:AssumeRole"
	    }
	  ]
	}
	```
   - This trust policy allows the "vmie.amazonaws.com" service (Virtual Machine Import/Export) to assume this role. This service is used for tasks like converting `.ova` files to Amazon Machine Images (AMIs).
3. **Permission Policies**:
  - Attach the following AWS managed policies to the role:
      - `AmazonEC2FullAccess`: This provides full access to all EC2 actions and resources.
      - `AmazonS3FullAccess`: This grants full access to S3 actions and resources, allowing for storage operations which might be necessary during migrations.

[[migration-role-review.png|Role Review]]


### Using AWS CLI:

1. **I've already installed the AWS CLI.**
   [Installing instructions](https://aws.amazon.com/cli/)
2. **Now, I'll generate an access key for AWS CLI use:**
    - I navigate to the AWS Management Console.
    - I click on my account (e.g., `hrodGmail`) at the top right.
    - In the dropdown, I select "Security credentials."
      <br>![[Pasted image 20231019142304.png]]
    - Under "Access keys," I click on "Create access key."
      <br>![[Pasted image 20231019142209.png]]

3. **After receiving the keys, I configure the AWS CLI:**
    - I run the command: `aws configure`
    - I enter the provided AWS Access Key ID.
    - I enter the AWS Secret Access Key.
    - I set the default region to `us-east-1`.
    - I leave the default output format as it is.
      
4. **To verify the connection, I list the S3 buckets:**
    - I run the command: `aws s3 ls`
      
<br>![[Pasted image 20231019142746.png]]


5. I create a new file named `container.json` with the following content:
	```json
	[
	  {
	    "Description": "Ubuntu Server OVA to AMI",
	    "Format": "ova",
	    "UserBucket": {
	      "S3Bucket": "my-test-bucket-unique-name",
	      "S3Key": "UbuntuServer.ova"
	    }
	  }
	]
	```

### Creating an AMI from the OVA:

1. To initiate the import of the OVA image into an Amazon Machine Image (AMI), I run the following command:

	```
	aws ec2 import-image --disk-containers "file://C:\Users\Hecti\Downloads\container.json" --role MigrationRole
	```
	
	<br>![[Pasted image 20231019150718.png]]

2. To monitor the progress of the image import task and obtain its status, I use:
	```
	aws ec2 describe-import-image-tasks --import-task-ids "import-ami-0faf4b221ede0a163"
	```
	
	<br>![[Pasted image 20231019150823.png]]
	*The progress was at `39%` when I last checked.*


After the process completes, I receive a snapshot.
<br>![[Pasted image 20231019151924.png]]

The AWS VM Import/Export service processed my disk image, took a snapshot, and subsequently generated an Amazon Machine Image (AMI) from that snapshot.
<br>![[Pasted image 20231019161606.png]]

The name of the AMI is `import-ami-0faf4b221ede0a163`.


**Launching an EC2 Instance using the AMI:**

Using this AMI, I launch an EC2 instance by clicking the button "Launch instance from AMI". When setting it up, I ensure to use the .pem file for authentication.

To access the EC2 instance, I use SSH with the username 'hector'. This username was established when I initially installed the operating system in VirtualBox.

The command I use is:
```bash
ssh hector@ec2-54-80-238-211.compute-1.amazonaws.com
```

> [!success]
> <br>![[Pasted image 20231023104729.png]]





---
tags:
  - AWS
  - CloudFormation
  - CloudWatch
  - IaC
title: Implementing Secure Amazon SNS Messaging in a Private AWS Environment
---
<!--
**Mini-Project: Implementing Secure Amazon SNS Messaging in a Private AWS Environment!** I've just completed a fascinating project that involved setting up a private environment within AWS to securely publish and deliver messages via Amazon SNS. The project required creating a VPC and an EC2 instance without internet access, establishing a VPC endpoint for Amazon SNS, and then using AWS CLI to publish messages to an SNS topic securely. This task not only honed my skills in AWS networking and security but also demonstrated the practical use of SNS in private networks, enhancing message security and delivery. It was a great exercise in integrating various AWS services for a secure and efficient cloud infrastructure.
-->
 
> [!info] Project - 3: Publishing Amazon SNS Messages Privately
> **Industry:** Healthcare 
> 
> **Problem Statement:** 
> How to secure patient records online and send it privately to the intended party 
> 
> **Topics:** 
> In this project, you will be working on a hospital project to send reports online and develop a platform so the patients can access the reports via mobile and push notifications. You will publish the report to an Amazon SNS keeping it secure and private. Your message will be hosted on an EC2 instance within your Amazon VPC. By publishing the messages privately, you can improve the message delivery and receipt through Amazon SNS. 
> 
> **Highlights:** 
> 1. AWS CloudFormation to create a VPC 
> 2. Connect VPC with AWS SNS 
> 3. Publish message privately with SNS



To clone the repository that contains the CloudFormation templates, I use the following command and link:
```bash
git clone https://github.com/aws-samples/aws-sns-samples.git
```

### Step 1: EC2 Key Pair
I have will make usre of a private key I already have
<br>![[Pasted image 20231103143217.png]]




### Step 2: Create the AWS Resource

> [!NOTE] Template
> [SNS-VPCE-Tutorial-CloudFormation.template](https://github.com/aws-samples/aws-sns-samples/blob/master/templates/SNS-VPCE-Tutorial-CloudFormation.template)
> 
> The provided CloudFormation template defines a set of AWS resources for setting up a VPC and related components for an SNS VPC Endpoints tutorial. Here's a summary of the resources it creates and their purposes:
> 
> 


I navigate to CloudFormation Dashboard and click "Create stack"

- **Step 1**: Create stack
  <br>![[Step 1Create stack.png|400]]
  *We upload the template*

- **Step 2**: Specify stack details
  <br>![[Step 2Specify stack details.png|400]]
  *After naming the stack, I fill in the 'KeyName' parameter with the name of an existing EC2 KeyPair to enable SSH access. I set the 'SSHLocation' parameter to '0.0.0.0/0', which allows SSH connections from any IP address.*

- **Step 3**: Configure stack options
  *Left all options at their default settings.*  

- **Step 4**: Review



### Step 3: Confirming the EC2 Instance lacks internet access

First, I'll need to establish an SSH connection, it is necessary to obtain the EC2 instance's public IP address.
<br>![[Pasted image 20231103145918.png]]

Using the Public IP I `ssh` into the EC2
<br>![[Pasted image 20231103150221.png]]

Using ping verifies instance's lack of internet connectivity:
<br>![[Pasted image 20231103150323.png]]

To verify that the instance lacks connectivity to Amazon SNS:
<br>![[Pasted image 20231103150832.png|450]]

Using AWS CLI I attempt to publish a message SNS topic `VPCE-Tutorial-Topic`
```bash
aws sns publish --region us-east-1 --topic-arn arn:aws:sns:us-east-1:838427752759:VPCE-Tutorial-Topic --message "Hello"
```
<br>![[Pasted image 20231103151127.png]]
*No message is published*


### Step 4: VPC Endpoint for Amazon SNS

I begin by assigning the endpoint a Name Tag: 'VPCE-Tutorial'. Under **Services**, I select the AWS service that the endpoint will connect to, which is the Simple Notification Service (SNS) in the US East (N. Virginia) region, identified by `com.amazonaws.us-east-1.sns`.
<br>![[Pasted image 20231103153253.png]]
I ensure that I select the VPC that was created from the CloudFormation template.
<br>![[Pasted image 20231103153311.png]]
<br>![[Pasted image 20231103153517.png]]
I select the 'Tutorial Security Group' as the Security Group, which was previously established by the CloudFormation template.
<br>![[Pasted image 20231103153433.png]]

<br>![[endpoint vpce-tutorial.png]]



### Step 5: Publish a message to SNS topic

[[Pasted image 20231103151127.png|Once again]], I use the AWS CLI to publish a message to the SNS topic `VPCE-Tutorial-Topic`.
```bash
aws sns publish --region us-east-1 --topic-arn arn:aws:sns:us-east-1:838427752759:VPCE-Tutorial-Topic --message "Hello"
```
<br>![[Pasted image 20231103153954.png]]



### Step 6: Message deliveries verification

To verify that the Lambda functions were invoked

<br>![[Pasted image 20231103154701.png|300]]

To verify that the CloudWatch logs were updated:

> [!NOTE] VPCE-Tutorial-Lambda-1
> <br>![[VPCE-Tutorial-Lambda-1 cloudwatch.png]]
> 
> > [!success]
> > <br>![[Pasted image 20231103155309.png]]
> 

> [!NOTE] VPCE-Tutorial-Lambda-2
> 
> <br>![[Pasted image 20231103155506.png]]
> 
> 
> > [!success]
> > <br>![[Pasted image 20231103155525.png]]
> 
> 



### Step 7: Clean Up

To delete the VPC endpoint: 
I go to `Actions > Delete VPC endpoints`
<br>![[endpoint vpce-tutorial.png]]


To delete the AWS CloudFormation stack:
<br>![[Pasted image 20231103160118.png|500]]


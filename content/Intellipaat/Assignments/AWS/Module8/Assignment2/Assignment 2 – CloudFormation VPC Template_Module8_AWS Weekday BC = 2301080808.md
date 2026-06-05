---
tags:
  - AWS
  - CloudFormation
  - IaC
title: Streamlining VPC and EC2 Deployment with CloudFormation in AWS
---
<!--
**Mini-Project: Streamlining Network Deployment with AWS CloudFormation!** In this project, I leveraged AWS CloudFormation to automate the creation of a robust VPC and a public subnet, culminating in the deployment of an Amazon Linux EC2 instance. The entire infrastructure setup was defined in a CloudFormation template, ensuring a repeatable and efficient deployment process. By completing this project, I've reinforced my skills in using infrastructure as code to create secure and scalable network architectures, a vital component in efficient cloud management.
-->
> [!info] Module 8: CloudFormation Assignment - 2 
> **Problem Statement:** 
> You work for XYZ Corporation. Your team is asked to deploy similar architecture multiple times for testing, development, and production purposes. Implement CloudFormation for the tasks assigned to you below. 
> 
> **Tasks To Be Performed:** 
> 1. Create a template with 1 VPC and 1 public subnet. 
> 2. Launch an Amazon Linux EC2 instance in the public subnet and tag the instance as “CFinstance” 

We followed the methodology from [[Assignment 1 – CloudFormation S3 Template_Module8_AWS Weekday BC = 2301080808|Assignment 1 – CloudFormation S3 Template]] to establish our CloudFormation stack.

**Stack Name:** `EC2VPC`

The applied template performs the following tasks:
1. Creates a VPC.
2. Creates a public subnet within the VPC.
3. Launches an Amazon Linux EC2 instance within the public subnet and tags it as "CFinstance".



```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudFormation Assignment: VPC, Public Subnet, and EC2 Instance'

Resources:
  MyVPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      Tags:
        - Key: 'Name'
          Value: 'AssignmentVPC'

  PublicSubnet:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: '10.0.1.0/24'
      MapPublicIpOnLaunch: 'true'
      Tags:
        - Key: 'Name'
          Value: 'AssignmentPublicSubnet'

  InstanceSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: 'Allow SSH and HTTP'
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: 'tcp'
          FromPort: '22'
          ToPort: '22'
          CidrIp: '0.0.0.0/0'

  EC2Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: 't2.micro'
      ImageId: 'ami-0c55b159cbfafe1f0'  # Amazon Linux 2 LTS AMI
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      Tags:
        - Key: 'Name'
          Value: 'CFinstance'

Outputs:
  VPCID:
    Description: 'VPC ID'
    Value: !Ref MyVPC

  PublicSubnetID:
    Description: 'Public Subnet ID'
    Value: !Ref PublicSubnet

  EC2InstanceID:
    Description: 'EC2 Instance ID'
    Value: !Ref EC2Instance

```

Events Leading to the Stack Creation:
<br>![[Pasted image 20231007192923.png]]
The stack `EC2VPC` was successfully created as indicated by the `CREATE_COMPLETE` status.
<br>![[Pasted image 20231007194538.png]]

The VPC defined in the template (`AssignmentVPC`) appears to be available.
<br>![[Pasted image 20231007193250.png]]

Instances `CFintnanace` is up and running
<br>![[Pasted image 20231007193535.png]]
This instance is associated with the `AssignmentVPC` and `AssignmentPublicSubnet` that were recently created.
<br>![[Pasted image 20231007193611.png]]


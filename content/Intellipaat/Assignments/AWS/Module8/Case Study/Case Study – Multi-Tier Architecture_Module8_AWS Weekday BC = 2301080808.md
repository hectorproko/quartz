---
tags:
  - AWS
  - CloudFormation
  - IaC
title: Designing a Scalable Multi-Tier Architecture on AWS
---
<!--
**Mini-Project: Designing a Scalable Multi-Tier Architecture on AWS!** I've completed an extensive project using AWS CloudFormation to deploy a scalable and secure multi-tier web application. This case study involved setting up a VPC with distinct public and private subnets, launching an EC2 instance as a web tier with Apache2, an application tier with MySQL in a private subnet, and an RDS MySQL instance as a database tier. Additionally, I integrated Route 53 for domain name management. This project allowed me to practically apply concepts of network segmentation, security, and infrastructure automation, providing a comprehensive understanding of deploying resilient and efficient cloud architectures.
-->
 

> [!info] Module 8: Case Study - 1
> **Problem Statement:** 
> You work for XYZ Corporation. Your corporation wants to launch a new web-based application. The development team has prepared the code but it is not tested yet. The development team needs the system admins to build a web server to test the code but the system admins are not available. 
> 
> **Tasks To Be Performed:** 
> 1. Web tier: Launch an instance in a public subnet and that instance should allow HTTP and SSH from the internet. 
> 2. Application tier: Launch an instance in a private subnet of the web tier and it should allow only SSH from the public subnet of Web Tier-3. 
> 3. DB tier: Launch an RDS MYSQL instance in a private subnet and it should allow connection on port 3306 only from the private subnet of Application Tier-4. 
> 4. Setup a Route 53 hosted zone and direct traffic to the EC2 instance. 
> 
> **You have been also asked to propose a solution so that:** 
>    1. Development team can test their code without having to involve the system admins and can invest their time in testing the code rather than provisioning, configuring and updating the resources needed to test the code. 
>    2. Make sure when the development team deletes the stack, RDS DB instances should not be deleted.


### Template 1: VPC Configuration

This template will set up a VPC, subnets, an internet gateway, and all the foundational components:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC and foundational network resources

Resources:

  # VPC
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      Tags:
        - Key: Name
          Value: MyVPC

  # Public Subnet
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: 'true'
      AvailabilityZone: !Select [ 0, !GetAZs '' ] # Choose the first AZ in the region

  # Private Subnet
  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [ 1, !GetAZs '' ] # Choose the second AZ in the region

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [ 2, !GetAZs '' ] # Choose the third AZ in the region

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  # Route table for the public subnet
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

  # NEW ADDITION: Elastic IP for NAT Gateway
  NatEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc

  # NEW ADDITION: NAT Gateway in the public subnet
  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEIP.AllocationId
      SubnetId: !Ref PublicSubnet

  # NEW ADDITION: Update the private subnet's route table to route traffic through the NAT Gateway
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

  PrivateSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet
      RouteTableId: !Ref PrivateRouteTable

  PrivateRoute:
    Type: AWS::EC2::Route
    DependsOn: NatGateway
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

Outputs:
  VPC:
    Description: A reference to the created VPC
    Value: !Ref MyVPC
  PublicSubnet:
    Description: A reference to the public subnet
    Value: !Ref PublicSubnet
  PrivateSubnet:
    Description: A reference to the private subnet
    Value: !Ref PrivateSubnet
  PrivateSubnet2:
    Description: A reference to the private2 subnet
    Value: !Ref PrivateSubnet2

```

This template establishes a VPC with two distinct subnets:

- A **Public Subnet** for the web tier, granting direct internet access via an **Internet Gateway**.
- A **Private Subnet** for the app tier, which utilizes a **NAT Gateway** in the public subnet to securely update instances and access the internet.

The infrastructure ensures that the app tier remains isolated while the web tier can be directly accessed from the internet.

---
### Template 2: Application Infrastructure

This template assumes the VPC and foundational resources are already set up:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Web, App, and DB tiers with Route53 setup

Parameters:
  VPCId:
    Description: The VPC ID
    Type: AWS::EC2::VPC::Id
  PublicSubnetId:
    Description: The public subnet ID
    Type: AWS::EC2::Subnet::Id
  PrivateSubnetId:
    Description: The private subnet ID
    Type: AWS::EC2::Subnet::Id
  PrivateSubnetId2:
    Description: The private subnet ID 2
    Type: AWS::EC2::Subnet::Id
  MasterUsername:
    Description: RDS Master Username
    Type: String
    Default: admin
  MasterUserPassword:
    Description: RDS Master Password
    Type: String
    Default: Passw0rd!
    NoEcho: true
  DomainName:
    Description: The domain name for the Route53 hosted zone (e.g., example.com.)
    Type: String
    Default: temp.hectorproko.com
  ImageId:
    Description: The AMI ID
    Type: String
    Default: ami-053b0d53c279acc90 #redhat ami-067d1e60475437da2
  KeyName:
    Description: SSH Key Name
    Type: String
    Default: daro.io

Resources:
  # Web Tier
  WebTierInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: !Ref ImageId
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnetId
      SecurityGroupIds:
        - !Ref WebTierSecurityGroup
      UserData: 
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          sudo apt-get update -y
          sudo apt-get install apache2 -y

  WebTierSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VPCId
      GroupDescription: "Security group for web tier"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '80'
          ToPort: '80'
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: 0.0.0.0/0

  # Application Tier
  AppTierInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: !Ref ImageId
      KeyName: !Ref KeyName
      SubnetId: !Ref PrivateSubnetId
      SecurityGroupIds:
        - !Ref AppTierSecurityGroup
      UserData: 
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          sudo apt-get update -y
          sudo apt-get install mysql-server -y

  AppTierSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VPCId
      GroupDescription: "Security group for app tier"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          SourceSecurityGroupId: !Ref WebTierSecurityGroup

  # RDS MySQL (simplified)
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: "Subnet group for RDS"
      SubnetIds:
        - !Ref PrivateSubnetId
        - !Ref PrivateSubnetId2

  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Retain
    Properties:
      AllocatedStorage: '5'
      DBInstanceClass: db.t2.micro
      Engine: MySQL
      MasterUsername: !Ref MasterUsername
      MasterUserPassword: !Ref MasterUserPassword
      VPCSecurityGroups:
        - !Ref DBSecurityGroup
      DBSubnetGroupName: !Ref DBSubnetGroup

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VPCId
      GroupDescription: "Security group for RDS"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '3306'
          ToPort: '3306'
          SourceSecurityGroupId: !Ref AppTierSecurityGroup

  # Route53
  MyHostedZone:
    Type: "AWS::Route53::HostedZone"
    Properties:
      Name: !Ref DomainName
      HostedZoneConfig:
        Comment: "Hosted zone for the web application"

  WebDNSRecord:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneId: !Ref MyHostedZone
      Name: !Sub "${DomainName}."
      Type: A
      TTL: '300'
      ResourceRecords:
        - !GetAtt [WebTierInstance, PublicIp]
```

This CloudFormation template sets up a multi-tier architecture with clear layers of security and accessibility for a web application. The architecture comprises three main components: a web tier, an application tier, and a database tier.

The **Web Tier** creates an EC2 instance within a specified public subnet. This instance is equipped with a security group that permits incoming HTTP traffic, making it the public-facing component of the architecture. As part of its initialization, this instance installs the Apache2 server, allowing it to host a webpage.

The **Application Tier** establishes another EC2 instance, but this time in a private subnet, reinforcing its isolated nature. It's set up to install a MySQL server upon initialization. The security group associated with this tier only allows SSH traffic, and uniquely, only from the web tier instance. This design ensures that direct external access is restricted, providing an additional layer of security.

Lastly, the **Database Tier** introduces an RDS instance configured for MySQL. It resides within a database subnet group spanning two private subnets. The security group for this RDS instance is designed to only accept connections on the MySQL port from the application tier, thereby ensuring that only the app tier can communicate with the database directly. Importantly, a safeguard has been put in place to ensure that even if the CloudFormation stack is deleted, the RDS DB instance will not be deleted, providing data resilience.

To complete the setup, a Route53 hosted zone and DNS record are created to associate the domain name with the public IP address of the web tier instance, ensuring users can access the hosted page via a friendly domain name.

By structuring the infrastructure in this way, the template promotes a layered security model, ensuring each component is shielded appropriately based on its function and exposure.

### Verifying Web Page Accessibility

The hosted zone in Route 53 has been correctly configured with a record set pointing to the `WebTier` instance - `54.242.228.183`.
<br>![[Pasted image 20231013105213.png]]
When accessing the public IP address `54.242.228.183` directly, the default Apache2 welcome page from an Ubuntu server is displayed, confirming the web server's correct operation.
<br>![[Pasted image 20231013105012.png|460]]
After updating the domain's nameservers in the registrar settings to match those provided by AWS Route 53, navigating to `temp.hectorproko.com` successfully resolves to the Apache2 default page. This confirms that the domain is correctly pointing to the intended web server.

> [!success]
> <br>![[Pasted image 20231013104938.png|460]]

### Testing RDS Connection

From the Web Tier instance with IP ending in `.247`, I initiated an SSH connection to the App Instance:
<br>![[Pasted image 20231012210418.png]]

Upon establishing a successful connection to the App Instance ending in `.67`, I then connected to the RDS instance using its provided endpoint:

> [!success]
> 
> <br>![[Pasted image 20231012210158.png]]
> The MySQL prompt was displayed, confirming a successful connection to the RDS instance and indicating that all components are functioning as expected.

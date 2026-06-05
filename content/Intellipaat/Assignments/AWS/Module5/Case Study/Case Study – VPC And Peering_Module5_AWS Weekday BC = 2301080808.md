---
tags:
  - AWS
title: Streamlining Connectivity with VPC and Peering
---
<!--
**Mini-Project: Streamlining Connectivity with VPC and Peering!** In a comprehensive hands-on project, I designed and established a secure and efficient network architecture using AWS VPC and Peering. The project involved creating two distinct VPC environments for production and development, each with tailored subnets and security configurations. I successfully implemented VPC peering between these networks, facilitating seamless and secure intercommunication, especially for database connectivity. This endeavor enhanced my skills in advanced AWS networking, highlighting the importance of thoughtful design in cloud-based infrastructures.
-->

> [!info] Module 5: Case Study - 1
> **Problem Statement:** 
> You work for XYZ Corporation and based on the expansion requirements of your corporation you have been asked to create and set up a distinct Amazon VPC for the production and development team. You are expected to perform the following tasks for the respective VPCs. 
> 
> **Production Network:** 
> 1. Design and build a 4-tier architecture. 
> 2. Create 5 subnets out of which 4 should be private named app1, app2, dbcache and db and one should be public, named web. 
> 3. Launch instances in all subnets and name them as per the subnet that they have been launched in. 
> 4. Allow dbcache instance and app1 subnet to send internet requests. 
> 5. Manage security groups and NACLs. 
>  
> **Development Network:** 
> 1. Design and build 2-tier architecture with two subnets named web and db and launch instances in both subnets and name them as per the subnet names. 
> 2. Make sure only the web subnet can send internet requests. 
> 3. Create peering connection between production network and development network. 
> 4. Setup connection between db subnets of both production network and development network respectively.



# Production

### Step 1: Initiating the VPC Configuration

- **Action**: Created a new VPC.
  - **Name**: ProductionVPC
  - **CIDR block**: Configured with `10.0.0.0/16`.

### Step 2: Carving out the Subnets

1. **Public Subnet for Web Tier**:
  - **Name**: Web
  - **CIDR**: Designated `10.0.1.0/24`. Selected an availability zone (AZ) for this subnet.
   
2. **Private Subnets**:
  - **App1 Subnet**:
    - **Name**: App1
    - **CIDR**: Chose `10.0.2.0/24`. Positioned this in the same AZ as Web to reduce latency.
     
  - **App2 Subnet**:
    - **Name**: App2
    - **CIDR**: Allocated `10.0.3.0/24`. Kept it in the same AZ as Web.
     
  - **DBCache Subnet**:
    - **Name**: DBCache
    - **CIDR**: Assigned `10.0.4.0/24` and placed it in a separate AZ for HA.
  - **DB Subnet**:
    - **Name**: DB
    - **CIDR**: Went with `10.0.5.0/24` in the same AZ as DBCache.  

<br>![[Pasted image 20230926182412.png]]

### Step 3: Setting Up Internet Connectivity

1. **IGW**:
  - **Action**: Established an Internet Gateway (IGW) `ProductionVPC-igw` and linked it to my ProductionVPC.
   
2. **For the Web Tier**:
  - **Action**: Created a route table `Web Tier` with a default route (`0.0.0.0/0`) pointed towards the IGW `ProductionVPC-igw`.
   
3. **For DBCache and App1 Subnets**:
  - **Action**: To maintain security while still permitting outbound internet connectivity, I initiated a NAT Gateway *(Requires Elastic IP)* in the Web subnet *(a public subnet)*. 
  - **New Route Table Creation**:
    - **Action**: Created a new route table, let's call it "PrivateSubnetsRouteTable," specifically for the DBCache and App1 subnets.
    - Added a route to this table, directing `0.0.0.0/0` (all internet-bound traffic) to the NAT Gateway.
  - **Route Table Adjustment**: Set the default route (`0.0.0.0/0`) for both DBCache and App1 to route through the NAT Gateway.


### Step 4: Implementing the Security Groups 



1. **Web Tier**:
  
  - **Action**: Constructed a security group allowing:
    - **Inbound Rules**: HTTP/HTTPS on ports 80 and 443 for potential web traffic.
    - **Outbound Rules**: Allowed ICMP (ping) to App1 and App2 subnets to test connectivity.
     
2. **For App1 and App2**:
  
  - **Action**: Established a security group that:
    - **Inbound Rules**: Allows ICMP (ping) from the Web tier to test connectivity.
    - **Outbound Rules**: Allows ICMP (ping) to DBCache and DB subnets for inter-tier connectivity testing.
     
3. **For DBCache**:
  
  - **Action**: Introduced a security group that:
    - **Inbound Rules**: Allows ICMP (ping) from App1 and App2 to test connectivity.
    - **Outbound Rules**: Allows ICMP (ping) to the DB tier for inter-tier connectivity testing.
     
4. **For the DB Tier**:
  
  - **Action**: Securely configured a group, which:
    - **Inbound Rules**: Allows ICMP (ping) only from DBCache to test connectivity.

### Step 5: Launching the EC2 Instances

- **Action**: For each subnet, I spun up an EC2 instance, naming them in alignment with their subnet names.



|Source/Target|**Web** 10.0.1.94|**App1** 10.0.2.188|**App2** 10.0.3.70|**DBCache** 10.0.4.209|**DB** 10.0.5.93|
|---|---|---|---|---|---|
|**Web**  <br>10.0.1.94  |-|✅|✅|❌|❌|
|**App1**  <br>10.0.2.188  |❌|-|❌|✅|❌|
|**App2**  <br>10.0.3.70  |❌|❌|-|✅|❌|
|**DBCache**  <br>10.0.4.209  |❌|❌|❌|-|✅|
|**DB**  <br>10.0.5.93  |❌|❌|❌|❌|-|

✅ = Successful Ping ❌ = Failed Ping

---
# Development

### Step 1: Establishing the Development VPC

- **Action**: Initiated a new VPC.
  - **Name**: DevelopmentVPC
  - **CIDR block**: Opted for `10.1.0.0/16` to avoid any overlap with ProductionVPC.

### Step 2: Structuring the Subnets

1. **Public Subnet for Web Tier**:
  - **Name**: WebDev
  - **CIDR**: Chose `10.1.1.0/24`. Positioned this subnet in my preferred AZ for optimal performance.
2. **Private Subnet for DB Tier**:
  - **Name**: DBDev
  - **CIDR**: Went with `10.1.2.0/24`. Positioned this in the same AZ as WebDev to maintain low latency.

### Step 3: Spinning up the EC2 Instances

- **Action**: For each of the subnets, I instantiated EC2 instances, naming them following their respective subnet names.

### Step 4: Setting Up Internet Connectivity for WebDev Subnet

1. **IGW for Development VPC**:

  - **Action**: Crafted an Internet Gateway and attached it to DevelopmentVPC.
   
2. **For the WebDev Subnet**:
  
  - **Action**: Formulated a route table and pointed the default route (`0.0.0.0/0`) towards the IGW, granting the WebDev subnet internet access.

### Step 5: Building the VPC Peering Connection

- **Action**: Initiated a VPC peering connection between ProductionVPC (`10.0.0.0/16`) and DevelopmentVPC (`10.1.0.0/16`) called `Production-Development`.
  - Modified route tables of both VPCs to ensure they can communicate across the peering connection.
   
   1. **For `ProductionVPC` Route Table**:
     - Destination: `10.1.0.0/16` (CIDR block of `DevelopmentVPC`)
     - Target: `Production-Development` (Peering connection)
   2. **For `DevelopmentVPC` Route Table**:
     - Destination: `10.0.0.0/16` (CIDR block of `ProductionVPC`)
     - Target: `Production-Development` (Peering connection)

### Step 6: Setting Up DB Connectivity Between Networks

1. **DB in Production Network**:
  - **Action**: Updated the security group of the DB instance in ProductionVPC to allow traffic from the DBDev subnet of the DevelopmentVPC.
   
2. **DBDev in Development Network**:
  - **Action**: Adjusted the security group of the DBDev instance to receive traffic from the DB subnet of the ProductionVPC.

### Testing:

The ping test shows that the `DBDev` instance with IP `10.1.2.73` in the `DevelopmentVPC` can successfully communicate with the **DB** instance with IP `10.0.5.93` in the `ProductionVPC`.

```bash
[ec2-user@ip-10-1-2-73 ~]$ ping 10.0.5.93
PING 10.0.5.93 (10.0.5.93) 56(84) bytes of data.
64 bytes from 10.0.5.93: icmp_seq=1 ttl=127 time=1.02 ms
64 bytes from 10.0.5.93: icmp_seq=2 ttl=127 time=2.61 ms
64 bytes from 10.0.5.93: icmp_seq=3 ttl=127 time=0.920 ms
```

This indicates:

1. **VPC Peering**: The VPC peering connection between `ProductionVPC` and `DevelopmentVPC` is set up correctly.
  
2. **Route Tables**: The route tables in both VPCs have been correctly configured to route traffic across the VPC peering connection.
  
3. **Security Groups**: The security group of the **DB** instance in `ProductionVPC` is correctly configured to allow ICMP traffic (ping) from the `DBDev` subnet of the `DevelopmentVPC`.
---
tags:
  - AWS
  - CloudWatch
title: Integrating ELB, ASG, and Route 53 for Scalable AWS Solutions
---
<!--
**Integrating ELB, ASG, and Route 53 for Scalable AWS Solutions!** In an in-depth case study for Module 4, I harnessed the power of AWS Elastic Load Balancing (ELB), Auto Scaling Groups (ASG), and Route 53 to build a robust, scalable web application infrastructure. The challenge involved managing scaling requirements, distributing loads effectively, and routing traffic to the domain—all while ensuring optimal performance and cost-effectiveness. This hands-on learning experience was a pivotal moment in understanding how these AWS services interconnect to provide a seamless, efficient user experience.
-->

> [!NOTE] Module 4: Case Study - 1
> **Problem Statement:** 
> You work for XYZ Corporation that uses on premise solutions and a limited number of systems. With the increase in requests in their application, the load also increases. So, to handle the load the corporation has to buy more systems almost on a regular basis. Realizing the need to cut down the expenses on systems, they decided to move their infrastructure to AWS. 
> 
> **Tasks To Be Performed:** 
> 1. Manage the scaling requirements of the company by: 
>    a. Deploying multiple compute resources on the cloud as soon as the load increases and the CPU utilization exceeds 80% 
>    b. Removing the resources when the CPU utilization goes under 60% 
> 2. Create a load balancer to distribute the load between compute resources. 3. Route the traffic to the company’s domain.

Based on the steps from [[Assignment 2 – Auto Scaling_Module4_AWS Weekday BC = 2301080808|Assignment 2 – Auto Scaling]], the following prerequisites were completed:
- [[Assignment 2 – Auto Scaling_Module4_AWS Weekday BC = 2301080808#Task 1 Create a Web Server AMI|Task 1: Create a Web Server AMI]]
- [[Assignment 2 – Auto Scaling_Module4_AWS Weekday BC = 2301080808#Task 2: Create a Launch Template|Task 2: Create a Launch Template]]

**Created the Target Group**: No Targets were added.
<br>![[Pasted image 20231005191645.png]]

**Created an Application Load Balancer (ALB)**:
- Similar to [[Assignment 1 – ELB_Module4_AWS Weekday BC = 2301080808|Assignment 1 – ELB]]
- In the listeners section, added the previously created Target Group `WebServers`.
  <br>![[Pasted image 20231005155913.png]]

**Create the Auto Scaling Group (ASG)**:
- Selected the "Launch template" previously created `WebServer`

- When configuring my ASG, I was presented with options for load balancing. I opted to attach it to an existing Application Load Balancer through the target group named 'WebServers | HTTP'. By making this selection, I ensured that any instances initiated within the ASG would be automatically registered with this target group. As a result, the Application Load Balancer will effectively distribute incoming traffic among these registered instances.
  <br>![[Pasted image 20231005160410.png]]

- To keep things simple, I configured the Auto Scaling group to have a minimum of 1 instance and a maximum of 2.
  <br>![[autoscaling group  size mini1 max2.png]]

- For the [[scaling strategy]], I've chosen the 'Target Tracking Scaling Policy'. I've set the scaling policies to trigger a scale-up once the CPU usage surpasses 80%
  <br>![[Pasted image 20231005164918.png]]
  *While there's an option to 'Disable scale in to create only a scale-out policy,' I've left it unchecked. This means the Auto Scaling group is configured to both scale out and scale in, offering flexibility based on the demand* ^b2587c

- In the EC2 Auto Scaling groups panel, it's evident that the 'WebServers' ASG currently contains the desired count of 1 instance
  <br>![[Pasted image 20231005162611.png]]


- To validate the setup, I tested the Load Balancer using the DNS link: `LoadBalancer-2130920093.us-east-1.elb.amazonaws.com`


> [!success]
> <br>![[Pasted image 20231005170540.png]]

**Testing Autoscaling group by making EC2 go over 80% utilization** 


To simulate high CPU usage in the EC2 instances, I mirrored the approach from [[Assignment 4 – CloudWatch Dashboard_Module3_AWS Weekday BC = 2301080808|Assignment 4 – CloudWatch Dashboard]] . Specifically, I executed the command `while true; do openssl dgst -sha256 /dev/zero; done`.

Subsequently, I observed multiple instances of the `openssl` process in action, confirming the induced CPU stress:
<br>![[Pasted image 20231005173245.png]]

As a direct result of the heightened CPU utilization, the Auto Scaling group triggered the scaling policies. I noticed that the number of running instances increased to 2:
<br>![[Pasted image 20231005173802.png]]

The ASG Activity Tab provided further insight. It showed that the new instance was initiated in response to the breach of the set CPU threshold:
<br>![[Pasted image 20231005173904.png]]

After terminating the high CPU processes, I noticed a change in the ASG's Activity History. The system detected the reduced CPU load and decided to scale in, terminating the instance `i-05ded3b3629d3a48` to adjust the capacity from 2 back to 1.
<br>![[Pasted image 20231005182105.png]]



**Route Domain Traffic to the ALB using Route 53**:
    
I'll used the hosted zone created in [[Assignment 3 – Route 53_Module4_AWS Weekday BC = 2301080808#Created a Hosted Zone in Route 53|Assignment 3 – Route 53]] and added a record to point to the load balancer (after deleting previous record since I'll be needing the same domain `temp.hectorproko.com`).

<br>![[Pasted image 20231005184859.png]]<br>![[Pasted image 20231005184943.png]]

**Domain Test Results**

Upon accessing `temp.hectorproko.com`, it's evident that the domain successfully routes to the Load Balancer. The displayed message, "This is a Web Server!", confirms the correct server response.

> [!success]
> <br>![[Pasted image 20231005185157.png]]




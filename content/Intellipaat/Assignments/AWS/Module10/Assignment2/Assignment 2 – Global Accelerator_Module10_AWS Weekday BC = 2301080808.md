---
tags:
  - AWS
title: Enhancing Performance with AWS Global Accelerator Endpoints
---
<!--
**Mini-Project: Enhancing Network Performance with AWS Global Accelerator!** In this project, I delved into AWS Global Accelerator to optimize network performance. My task involved creating a Global Accelerator with two endpoints: an EC2 instance in the "us-east-1" region and a Load Balancer in the "us-west-2" region. I configured the accelerator to route traffic based on geographic proximity and health checks, ensuring efficient and reliable access to my test page hosted on the EC2 instance. This project was a valuable exercise in improving application availability and performance across global AWS regions.
-->
 
> [!info] Module 10: Global Accelerator Assignment
> **Tasks To Be Performed:** 
> 1. Create a Global Accelerator: 
>    a. Make an EC2 instance as an endpoint 
>    b. Make a load balancer as an endpoint and should be in a different region than the EC2 instance 


I create an EC2 instance in the "us-east-1" region. This instance will host my test page, similar to "[[Assignment 1 – EC2_Module2_AWS Weekday BC = 2301080808|Assignment 1 – EC2]]."
<br>![[Pasted image 20231014191329.png]]

To ensure the page is successfully hosted, I access it using the provided public IPv4 address.
<br>![[Pasted image 20231014184913.png]]


I set up a Load Balancer in the "us-west-2" region that directs traffic to my EC2 instance hosting the test page.
<br>![[Pasted image 20231014185210.png]]

To verify the setup, I access the test page through the Load Balancer's link. Successfully, the message "Hello World from us-west-2" appears, indicating the page is correctly routed through the Load Balancer.
<br>![[Pasted image 20231014184944.png]]

I head to the AWS dashboard and click on the "Create accelerator" button. I then name the accelerator "myAccelerator".

Next, I move on to configure the listeners. I set up a listener for port 80 using the TCP protocol, no client affinity set.
<br>![[Pasted image 20231013180530.png|400]]

After setting up the listener, I proceed to add endpoint groups for both the "us-east-1" and "us-west-2" regions.
<br>![[Pasted image 20231013200915.png|400]]

I add an endpoint for each endpoint group: the EC2 in "us-east-1" and the Load Balancer in "us-west-2". Then, I click the "Create accelerator" button.
<br>![[Pasted image 20231014185439.png]]
The newly created Accelerator
<br>![[Pasted image 20231014191202.png]]

Now, I test the link of the accelerator, and it connects me to the EC2 in `us-east-1`. AWS Global Accelerator directs traffic to the optimal AWS endpoint based on health, geographic proximity, and routing policies configured. Being in Florida, I'm geographically closer to the US East 1 (Northern Virginia) AWS region than to US West 2 (Oregon).
<br>![[Pasted image 20231014190331.png]]



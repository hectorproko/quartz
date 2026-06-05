---
tags:
  - AWS
  - CloudWatch
title: Creating a CloudWatch Dashboard for Monitoring EC2 Instances in AWS
---
<!--
**Leveraging AWS CloudWatch for Enhanced Monitoring!** In my latest endeavor, I focused on creating a custom CloudWatch dashboard to monitor EC2 instances. The task involved tracking CPU utilization and networking metrics for specific instances. By simulating load on one instance and observing the real-time metrics, I honed my skills in performance monitoring and operational management within AWS. This assignment not only enhanced my technical acumen but also underlined the importance of proactive monitoring in cloud infrastructure.
-->
 
> [!Info] Module 3: CloudWatch Dashboard Assignment
> **Problem Statement:** 
> You work for XYZ Corporation. To maintain the security of the AWS account and the resources you have been asked to implement a solution that can help easily recognize and monitor the different users. Also, you will be monitoring the machines created by these users for any errors or misconfigurations. 
> 
> **Tasks To Be Performed:** 
> 1. Create a dashboard which lets you check the CPU utilization and networking for a particular EC2 instance.

1. **Preparation**:
    - I first ensured that I had two EC2 instances running: `instance1` and `instance2`.
      <br>![[Pasted image 20230921163553.png]]

2. **Building My Dashboard**:
    - I logged into AWS and went straight to CloudWatch.
    - On the left side, I clicked on `Dashboards` and then hit `Create Dashboard`.
    - I gave my dashboard the name `EC2s` and clicked `Create dashboard`
      
3. **Populating the Dashboard with Widgets**:
    - Inside my new dashboard I get prompted to `Add widget`.
    - I chose the `Line` widget because it visually made the most sense for what I wanted.
    - Under `Metrics`, I navigated to `EC2 -> Per-Instance Metrics`.
    - I selected `CPUUtilization` and then filtered by my EC2 instance's ID or name for accuracy.
    - For networking, I picked metrics like `NetworkPacketsIn` and `NetworkPacketsOut`. Again, I made sure to filter by my EC2 instance.
      <br>![[Pasted image 20230921164405.png]]
      *Here I picked `Instance1` will repeat steps for `Instance2`*
   - Once I added all my metrics, I clicked `Create widget`.
   - The result is a dashboard with 2 widgets for `Instance1` and `Instance2`
     <br>![[Pasted image 20230921165156.png]]
      
4. **Saving My Work**:
    - I was happy with how everything looked, so I clicked `Save dashboard`. 

5. **Simulating Load on `Instance1`**: ^877754
    - To compare the metrics side-by-side, I decided to simulate some load on `instance1` while leaving `instance2` idle.
    - I SSH-ed into `instance1`.
    - To increase CPU usage, I ran: `while true; do openssl dgst -sha256 /dev/zero; done &`
    - To generate some network traffic, I used the `curl` command to download a large file multiple times: 
      `while true; do curl -O https://dumps.wikimedia.org/enwiki/latest/enwiki-latest-pages-articles.xml.bz2; done &`
    - This step ensured that `instance1` was now actively using CPU and network resources, while `instance2` remained idle.

6. **Results from CloudWatch Metrics**:
   After initiating the CPU-intensive task on `instance1`
   
   NetworkPacketsIn:
   <br>![[Pasted image 20230921172021.png]]
   `Instance1` showcased heightened incoming packets due to the network tasks.

   CPUUtilization:
   <br>![[Pasted image 20230921171959.png]]
   High CPU usage was evident on `Instance1` from the hashing operation.
 
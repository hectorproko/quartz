---
tags:
  - AWS
  - CloudWatch
title: Implementing CloudWatch Alarms for Cost Management and EC2 Monitoring in AWS
---
<!--
**Hands-On Learning with AWS CloudWatch Alarms!** In my latest hands-on learning exercise, I navigated AWS CloudWatch to set up two alarms: one for monitoring unexpected billing charges and another for high CPU utilization on an EC2 instance. While configuring these alarms and linking them to SNS notifications, I gained practical experience in setting up proactive monitoring and alerts within a cloud environment. This assignment was an enriching opportunity to deepen my understanding of AWS's monitoring tools and enhance my cloud management skills.
-->

> [!info] Module 3: CloudWatch Alarms Assignment
> **Problem Statement:** 
> You work for XYZ Corporation. To maintain the security of the AWS account and the resources you have been asked to implement a solution that can help easily recognize and monitor the different users. Also, you will be monitoring the machines created by these users for any errors or misconfigurations. 
> 
> **Tasks To Be Performed:** 
> 1. Create a CloudWatch billing alarm which goes off when the estimated charges go above $500. 
> 2. Create a CloudWatch alarm which goes off to an Alarm state when the CPU utilization of an EC2 instance goes above 65%. Also add an SNS topic so that it notifies the person when the threshold is crossed.

### 1. CloudWatch Billing Alarm:

**Steps**:

1. Opened the CloudWatch console.

2. In the navigation pane, clicked on `Alarms` and then `Create Alarm`.

3. In the `Create Alarm` screen, clicked on `Select metric`.

4. Chose the `Billing` namespace.

5. Chose `Total Estimated Charge`, then select `EstimatedCharges`.


6. Defined the conditions. Set `Threshold type` to `Static` and whenever the metric is... `Greater` than... `$500`.
<br>![[Pasted image 20230921200826.png]]

7. Selected `In alarm` configured action to send a notification. Chose to create a new SNS topic `demoSNS`

8. Added an email address to receive the notification.
<br>![[Pasted image 20230921201338.png]]
9. Named the alarm `$500_alarm`

10. Click `Create Alarm`.

---

### 2. EC2 CPU Utilization Alarm:

1. Opened the CloudWatch console.

2. In the navigation pane, clicked on `Alarms` and then clicked `Create Alarm`.

3. In the `Create Alarm` screen, clicked on `Select metric`.

4. Chose the `EC2` namespace and then `Per-Instance Metrics`.

5. Chose `CPUUtilization` for my EC2 instance `Instance2`, then clicked `select metric`.
<br>![[Pasted image 20230921204216.png]]
6. Defined the conditions. Set `Threshold type` to `Static` and whenever the metric is... `Greater` than... `65`.
 <br>![[Pasted image 20230921204337.png]]

7. Configure actions. For ALARM state, chose to send a notification. Select an existing SNS topic `demoSNS`.

8. Added the email address to receive the notification.

9. Named the alarm `CPU_Over60`

10. Clicked `Create alarm`.

> [!success] Now we can see the 2 newly created alarms
> <br>![[Pasted image 20230921204943.png]]


**Testing and Verification**:

1. To verify the CPU utilization alarm, I SSHed into our EC2 instance and ran `while true; do openssl dgst -sha256 /dev/zero; done &` to artificially spike the CPU usage and trigger the alarm.
  
2. I keenly observed and waited for the email notification, confirming that the setup was functioning as expected.
   <br>![[Pasted image 20230921210841.png]]
   
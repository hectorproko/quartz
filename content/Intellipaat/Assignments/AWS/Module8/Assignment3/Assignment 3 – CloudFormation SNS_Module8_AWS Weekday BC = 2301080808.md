---
tags:
  - AWS
  - CloudFormation
  - IaC
title: Enhancing CloudFormation Deployments with SNS Notifications in AWS
---
<!--
**Mini-Project: Enhancing AWS Deployments with CloudFormation and SNS!** In this project, I deepened my AWS expertise by integrating CloudFormation with Simple Notification Service (SNS) to automate infrastructure deployment and receive real-time notifications. I created a standard SNS topic, subscribed to it via email, and linked it to my CloudFormation stack for ongoing updates. This allowed me to efficiently monitor the stack creation process and receive immediate alerts on every step, ensuring a smooth and informed deployment workflow. This practical project was a great exercise in combining various AWS services for enhanced operational efficiency.
-->
 
> [!info] Module 8: CloudFormation Assignment - 3 
> **Problem Statement:** 
> You work for XYZ Corporation. Your team is asked to deploy similar architecture multiple times for testing, development, and production purposes. Implement CloudFormation for the tasks assigned to you below. 
> 
> **Tasks To Be Performed:** 
> 1. Use the template from CloudFormation task 1. 
> 2. Add Notification to the CloudFormation stack using SNS so that you get a notification via mail for every step of the stack creation process.


1. **I'll Create the SNS Topic in the AWS Console**:
    - I'll log into my **AWS Management Console**.
    - I'll search for "SNS" or "Simple Notification Service" in the Services dropdown and select it.
    - I'll click on "Topics" in the left sidebar and then on "Create topic".
    - I'll make sure to select "Standard" as the type because FIFO topics don't offer email subscription
    - I'll provide a name for my topic `CloudFormationNotifications`, and then click on "Create topic".
    - Once my topic is created, I'll click on it to see its details. I need to remember to copy the **Topic ARN** because I'll use it later.
      <br>![[Pasted image 20231008105455.png]]
      ARN: `arn:aws:sns:us-east-1:838427752759:CloudFormationNotifications`
      
2. **I'll Subscribe My Email to the SNS Topic**:
    - While I'm in the topic details in the SNS console, I'll click on "Create subscription".
    - For the "Protocol", I'll choose `Email`.
    - In the "Endpoint", I'll type in my email address where I want to receive the notifications.
    - I'll click on "Create subscription".
      <br>![[Pasted image 20231008110311.png]]
    - I'll then check my email. I should find a message from AWS asking me to confirm the subscription. I'll click on the link in that email to confirm.
      <br>![[Pasted image 20231008110338.png]]
3. **I'll Deploy the CloudFormation Stack**:
    
    - I'll head over to the CloudFormation service in the AWS Console.
    - I'll click on "Create stack".
    - I'll choose the "Upload a template file" option and upload the CloudFormation template used in [[Assignment 2 – CloudFormation VPC Template_Module8_AWS Weekday BC = 2301080808|Assignment 2 – CloudFormation VPC Template]].
      <br>![[Pasted image 20231007210826.png]]
    - I'm prompted to provide an email address for receiving CloudFormation notifications.
      <br>![[Pasted image 20231007210926.png]]
    - After clicking "Next", I'll give a unique name for my stack `EC2VPC`.
    - I'll go through the prompts until I reach the "Configure stack options" page.
    - In the "Advanced options" section, I'll find the "Amazon SNS topics" heading. Here, I'll paste the **Topic ARN** I copied earlier.
      <br>![[Pasted image 20231008111754.png]]
    - I'll continue with the stack creation process, reviewing everything, and then start the creation by clicking "Create stack".
      
4. **I'll Monitor the Stack Creation and Check My Notifications**:
    
    - The CloudFormation Dashboard will show me the progress of my stack. As the stack gets created, I'll also start receiving email notifications because of the SNS topic I set up.
      <br>![[Pasted image 20231008111928.png]]
      Lets open one email
      <br>![[Pasted image 20231008112858.png]]
      
5. **Clean Up When I'm Done**:
    
    - If I'm done testing or just want to clean things up, I'll head to the CloudFormation dashboard. I'll select my stack and then choose "Delete". I'll confirm that I want to delete it.
    - If I decide I don't want the SNS topic anymore, I'll head back to the SNS dashboard and delete it. This will stop any further notifications and ensure I don't get any unexpected charges.




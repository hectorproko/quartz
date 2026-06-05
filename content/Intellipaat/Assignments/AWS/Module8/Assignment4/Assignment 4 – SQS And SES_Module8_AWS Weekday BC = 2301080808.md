---
tags:
  - AWS
  - CloudFormation
  - IaC
title: Integrating SQS and SES for Messaging and Email Services in AWS
---
<!--
**Mini-Project: Streamlining Communication and Queue Management with AWS SQS and SES!** In a focused hands-on project, I ventured into automating message queuing and email services using AWS Simple Queue Service (SQS) and Simple Email Service (SES). I utilized CloudFormation to deploy a FIFO SQS queue, ensuring ordered message handling and no duplicates. Following this, I registered and verified my email with SES to send out a test email, marking a successful implementation. This project allowed me to practically implement these services, enhancing operational communication
-->
 

> [!info] Module 8: SQS and SES Assignment
> **Problem Statement:** 
> You work for XYZ Corporation. Your team is asked to deploy similar architecture multiple times for testing, development, and production purposes. Implement CloudFormation for the tasks assigned to you below. 
> 
> **Tasks To Be Performed:** 
> 1. Create a FIFO [[SQS (Simple Queue Service)_aws|SQS]] queue and test by sending messages. 
> 2. Register your mail in [[SES (Simple Email Service)_aws|SES]] and send a test mail to yourself. 



### Deploying and Testing the FIFO SQS Queue:

1. **Create a FIFO SQS Queue with CloudFormation**: 
- I've created a CloudFormation template named `sqs_fifo.yaml` with the necessary configuration for a FIFO SQS queue.
- In the AWS Management Console, I'll open the **CloudFormation** service.
- I'll click on **Create stack**.
- I'll choose **Upload a template file** and then upload my `sqs_fifo.yaml` file.
- After providing a name for the stack, like "FIFOSQSStack", I'll follow through the prompts, leaving all default settings, and initiate the stack creation.

2. **Test the FIFO SQS Queue**:
- Once my CloudFormation stack shows a status of `CREATE_COMPLETE`, I'll go to the **SQS** service in the AWS Console.
<br>![[Pasted image 20231009163149.png]]
- I'll find the `my-test-fifo-queue.fifo` in the list of queues.
<br>![[Pasted image 20231009163132.png]]
- To send a test message, I'll select the queue, click on **Send and receive messages**, input a test message, and click **Send message**.
<br>![[Pasted image 20231009164124.png]]
- I can then click on **Poll for messages** to see if my message appears, confirming the queue is working correctly.
<br>![[Pasted image 20231009164214.png]]
<br>![[Pasted image 20231009164242.png]]


```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
MyFIFOSQSQueue:
Type: "AWS::SQS::Queue"
Properties:
QueueName: "my-test-fifo-queue.fifo"
FifoQueue: true
ContentBasedDeduplication: true
```

### Setting Up and Testing SES:

1. **Verify My Email Address with SES**:

- In the AWS Management Console, I'll navigate to the **SES** service.
- Clicked **Create identity**, I'll choose **Email Addresses**.
- I'll click on **Create identity** button
- Received a verification email in my inbox, I'll click the verification link to confirm.
<br>![[Pasted image 20231009165046.png]]

2. **Send a Test Email**:

In the SES console, displaying my verified email identity (`nakamo5309@mugadget.com`):

- I'll click on the **Send test email** button, found at the top right corner. 
- In the **Send test email** window that appears: 
- For the **From** and **To** sections, I'll use my verified email, `nakamo5309@mugadget.com`.
- I'll fill in the **Subject** with "Test Email from SES".
- For the **Message body**, I'll write "This is a test email sent from Amazon SES."
- After inputting the details, I'll click on **Send test email**. 
- I'll then check my email inbox for `nakamo5309@mugadget.com` to confirm that I've received the test email from SES.
  <br>![[Pasted image 20231009172335.png]]

I've now successfully deployed a FIFO SQS queue and set up SES to send a test email to my verified address.




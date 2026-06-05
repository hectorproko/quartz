---
tags:
  - AWS
title: Configuring IAM Roles for Targeted VPC and DynamoDB Access in AWS
---
<!--
**Enhancing Security with AWS IAM Roles!** In a recent assignment, I delved into IAM roles within AWS, creating a custom role that provided specific users complete access to VPCs and DynamoDB. I crafted a custom policy, attached it to a new role, and modified trust relationships to ensure secure, exclusive access for selected users. This task reinforced my understanding of role-based access control and the importance of precise permission management in maintaining a secure cloud environment.
-->  
 
> [!info] Module 3: IAM Roles Assignment
> **Problem Statement:** 
> You work for XYZ Corporation. To maintain the security of the AWS account and the resources you have been asked to implement a solution that can help easily recognize and monitor the different users. 
> 
> **Tasks To Be Performed:** 
> 1. Create a role which only lets  `User1` and  `User2` from task 1 to have complete access to VPCs and DynamoDB. 
> 2. Login into  `User1` and shift to the role to test out the feature.



> [!attention] Created accounts as pre-requisite for assignment.
> Created  IAM users `User1` and  `User2` made sure to "Provide user access to the AWS Management Console"
> <br>![[Pasted image 20230922100128.png]]
>
>    


### Task 1: Create a Role for VPC and DynamoDB Access

#### Create Policy for Role

First, I started by creating a custom policy that grants complete access to VPCs and DynamoDB. In the IAM dashboard, I clicked on `Policies` and then `Create policy`. I pasted the following JSON in the JSON tab:

```json
{
  "Version": "2012-10-17",
  "Statement": [
 {
   "Effect": "Allow",
   "Action": [
  "ec2:*",
  "dynamodb:*"
   ],
   "Resource": "*"
 }
  ]
}
```

<br>![[VPCAndDynamoDBFullAccess.png]]  

I reviewed the policy, named it `VPCAndDynamoDBFullAccess`, and then clicked `Create policy`.

#### Create the Role and Attach Policy

Next, I navigated back to the IAM dashboard and clicked on `Roles` followed by `Create role`. 
I selected `AWS account` and checked `This account(XXXXXX)`

It then asks for `Add permissions`, I attached the previously created `VPCAndDynamoDBFullAccess` policy to the role. I reviewed all the settings, gave the role the name `VPC_DynamoDB_Access` and then clicked `Create role`.
<br>![[add permissions.png]]


**Modifying the Trust Relationship**:
After the role is created, I need to modify its trust relationship to ensure only `User1` and `User2` can assume this role:
- I'll head back to the `Roles` section and click on the role I've just created.
- In the `Trust relationships tab`, I'll click `Edit trust policy`. 
- I'll adjust the policy document to the following:

Original: `"AWS": "arn:aws:iam::838427752759:root"`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::838427752759:user/User1",
                    "arn:aws:iam::838427752759:user/User2"
                ]
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

### Task 2: Login into  `User1` and Assume the Role
<br>![[Pasted image 20230922104310.png]]  

To test the role, I logged in as  `User1`. 

I tried to create a VPC and a DynamoDB table. However, as **expected**, I could not.

<br>![[Pasted image 20230922104642.png]]

<br>![[Pasted image 20230922104731.png]]

I navigated to the `Switch Role` option in the AWS Management Console. 

<br>![[Pasted image 20230922104516.png]]

I entered the account ID where the role was created and provided the name of the role `VPC_DynamoDB_Access`.
<br>![[Pasted image 20230922104843.png]]


Upon successfully switching roles, I verified my access to VPC and DynamoDB. As expected, I had full permissions to work with both services.

<br>![[Pasted image 20230922104937.png]]

<br>![[Pasted image 20230922105103.png]]


**Extra Testing**:
- I created a new IAM user named `User3`.
- After logging out as `User1`, I signed in as `User3`.
- I navigated to the `Switch Role` option, but I couldn't proceed further. This confirmed that the [[Trust Policy_aws|trust policy]] was working correctly and restricting role assumption to only `User1` and `User2`.
  <br>![[Pasted image 20230922104843.png|400]]


<!--Took 1h 20m to finish, not countering proof reading -->


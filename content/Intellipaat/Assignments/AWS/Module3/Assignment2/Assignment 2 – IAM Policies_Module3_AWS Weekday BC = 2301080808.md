---
tags:
  - AWS
  - CloudWatch
title: Crafting and Managing IAM Policies for User Access Control in AWS
---
<!--
**Diving Deeper into AWS IAM Policies!** In my latest assignment, I crafted and managed IAM policies to bolster security and efficiency within AWS. I created two distinct policies; the first providing comprehensive access to S3, permission to create EC2 instances, and full RDS access. The second policy focused on complete CloudWatch and billing access, along with limited listing capabilities for EC2 and S3. Successfully attaching these policies to respective user groups, I've enhanced my grasp on fine-tuning access controls and security measures in cloud environments.
-->  
> [!info] Module 3: IAM Policies Assignment
> **Problem Statement:** 
> You work for XYZ Corporation. To maintain the security of the AWS account and the resources you have been asked to implement a solution that can help easily recognize and monitor the different users. 
> 
> **Tasks To Be Performed:** 
> 1. Create policy number 1 which lets the users to: 
>    a. Access S3 completely 
>    b. Only create EC2 instances 
>    c. Full access to RDS 
> 2. Create a policy number 2 which allows the users to: 
>    a. Access CloudWatch and billing completely 
>    b. Can only list EC2 and S3 resources 
> 3. Attach policy number 1 to the `DevTeam` from task 1 
> 4. Attach policy number 2 to `OpsTeam` from task 1
> 

### Task 1: Creating Policy Number 1

First, I logged into the AWS Management Console and navigated to the IAM dashboard. From there, I clicked on the `Policies` tab on the left sidebar to begin creating a new policy.

I clicked the `Create policy` button and then switched to the JSON tab. Here, I pasted in the JSON policy that allows for full S3 access, EC2 instance creation only, and full RDS access. 
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ec2:RunInstances",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "rds:*",
            "Resource": "*"
        }
    ]
}
```
After reviewing the policy to make sure it met the requirements, I named the policy `PolicyNumber1` and clicked `Create policy`.
<br>![[PolicyNumber1.png]]
### Task 2: Creating Policy Number 2
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "cloudwatch:*",
                "aws-portal:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "s3:List*"
            ],
            "Resource": "*"
        }
    ]
}
```
Continuing in the IAM dashboard, I clicked on the `Policies` tab again and then clicked the `Create policy` button. I switched over to the JSON tab, just like before, and pasted in the JSON policy for allowing full access to CloudWatch and billing as well as the ability to only list EC2 and S3 resources. After reviewing this second policy, I named it `PolicyNumber2` and clicked `Create policy`.
<br>![[PolicyNumber1-2.png]]
### Task 3: Attaching Policy Number 1 to the `DevTeam`

Next, I moved on to attaching `PolicyNumber1` to the `DevTeam`. I clicked on the `User groups` tab on the left sidebar and selected the `D evTeam` group by clicking its name. 

Within the `DevTeam` group interface, I clicked the `Permissions` tab, `Add permissions` button and `Attach policies`. 
<br>![[attach policies.png]]

A list of available policies appeared, and I searched for `PolicyNumber1`. Upon finding it, I selected it and then clicked the `Attach policies` button.
<br>![[attach permission.png]]
### Task 4: Attaching Policy Number 2 to the `OpsTeam`

Finally, to attach `PolicyNumber2` to the `OpsTeam`, I went back to the `User groups` tab and clicked on the `OpsTeam` group. 

Just like with the `DevTeam`, while inside the `OpsTeam` group interface I clicked the `Permissions` tab, `Add permissions` button and `Attach policies`. I searched for `PolicyNumber2` selected it from the list, and then clicked the `Attach policies` button.
<br>![[OpsTeam permissions PolicyNumber2.png]]
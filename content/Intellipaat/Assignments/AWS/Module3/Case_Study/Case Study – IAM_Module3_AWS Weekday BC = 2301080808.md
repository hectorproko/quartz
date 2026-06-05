---
tags:
  - AWS
title: Transitioning to AWS for Scalable Infrastructure Management
---
<!--
**Exploring IAM in Depth: AWS Case Study!** I completed a hands-on case study on AWS Identity and Access Management (IAM), designed to enhance security and efficiency for a hypothetical XYZ Corporation moving to AWS. My tasks included creating a user and group with specific EC2 instance access, granting permissions for VPC, Subnets, NACLs, RDS instance creation, and exploring security options like MFA and fine-grained policies. This exercise was an excellent opportunity to apply IAM best practices and deepen my understanding of managing secure access in cloud environments.
-->
> [!info] Module 3: Case Study - 1
> Problem Statement: 
> You work for XYZ Corporation that uses on premise solutions and a limited number of systems. With the increase in requests in their application, the load also increases. So, to handle the load the corporation has to buy more systems almost on a regular basis. Realizing the need to cut down the expenses on systems, they decided to move their infrastructure to AWS. 
> Tasks To Be Performed: 
> 1. 
>    a. Create a user account that can login to the console 
>    b. Create a group and make sure that the group can only launch and stop EC2 instances using that previously created account 
> 2. a. Provide permission to let the user of a previously created account to create VPCs, Subnets, NACL and security groups 
>    b. Further add the permission so that the user can create RDS instance 
>    c. Explore security options to protect the AWS resources and secure the permissions provided to the group

### Part 1-A
**Create a user account named `User4` that can login to the console:**

1. I opened the AWS Management Console.
2. I navigated to the `IAM` (Identity and Access Management) dashboard.
3. On the navigation pane, I clicked on `Users` and then selected `Create user`.
4. I entered the username as `User4`.
5. I selected `Provide user access to the AWS Management Console` and set a console password.
6. In Set Permissions `Add user to group` was selected so I clicked on `Create Group`

### Part 1-B
**Create a group for `User4` to launch and stop EC2 instances:**

1. I provided a suitable group name `User4Group`.
2. I attached permissions by choosing `Create policy`. In the JSON editor, I input the policy to allow launching and stopping EC2 instances.
   <br>![[Pasted image 20230922142242.png]]
   
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",  // Allows launching instances
                "ec2:StopInstances"  // Allows stopping instances
            ],
            "Resource": "*"
        }
    ]
}
```

3. I reviewed and created the policy `RunStopEC2`
4. I attached this policy to the group as I created it.
   <br>![[Pasted image 20230922142537.png]]

> [!tip] Summary
> I created a new user named `User4`. During this process, I was prompted to attach `User4` to a group. Instead of choosing an existing group, I created `User4Group`. While setting up `User4Group`, I was given the option to attach a policy. Instead of selecting a pre-existing one, I created a new policy named `RunStopEC2`. Finally, the `User4Group` was created with `User4` in it and the `RunStopEC2` policy attached to it.


### Part 2-A
**I provided permissions for `User4` to create VPCs, Subnets, NACL, and security groups:**

1. On the IAM dashboard, I clicked on `Policies` from the left sidebar.
2. I then clicked the `Create policy` button.
3. On the "Create Policy" page, I switched to the "JSON" tab.
4. I pasted the following permissions to allow `User4` to create VPCs, Subnets, NACLs, and security groups:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateVpc",
                "ec2:CreateSubnet",
                "ec2:CreateNetworkAcl",
                "ec2:CreateSecurityGroup"
            ],
            "Resource": "*"
        }
    ]
}
```

5. Named the policy `VPC_Sub_NACL_SG`
6. After reviewing, I created the policy.

7. I navigated to `User groups` and selected `User4Group` the group  associated with `User4`.
8. In `Permissions` tab I clicked `Add permissions`.
9. I attached  `VPC_Sub_NACL_SG` policy to the group.
   <br>![[Pasted image 20230922152608.png]]

### Part 2-B
**I added the permission so that `User4` can create an RDS instance:**

1. I repeated the steps above to create and attach a policy named `Create_RDS` which has the following

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "rds:CreateDBInstance",
            "Resource": "*"
        }
    ]
}
```

### Part 2-C
**I explored security options to protect the AWS resources and secure the permissions for the group:**

1. **MFA**: I enabled MFA for `User4`, adding an additional layer of security for console logins.
   <br>![[Pasted image 20230922155152.png|300]]
   <br>![[Pasted image 20230922155023.png|300]]
2. **Fine-grained Policies**: I ensured that the policies I created granted only necessary permissions to avoid excessive privileges.


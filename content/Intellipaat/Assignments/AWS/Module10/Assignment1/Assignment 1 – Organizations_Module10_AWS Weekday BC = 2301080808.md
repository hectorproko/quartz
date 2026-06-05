---
tags:
  - AWS
title: Optimizing Resource Management with AWS Organizations
---
<!--
**Mini-Project: Optimizing Resource Management with AWS Organizations!** I've taken a significant step in cloud resource management by diving into AWS Organizations for this project. I created an AWS Organization and structured it into three organizational units—OU1, OU2, and OU3—to manage resources more efficiently and implement centralized governance. Furthermore, I attached a service control policy to restrict access to only EC2 services across these units, enhancing security and specificity of roles and permissions. This project was instrumental in understanding and utilizing AWS Organizations for improved control and operational efficiency.
-->

> [!info] Module 10: Organizations Assignment
> **Tasks To Be Performed:** 
> 1. Create an AWS Organization. 
> 2. Create 3 organization units – OU1, OU2 and OU3. 
> 3. Attach a service control policy which only allows access to EC2.



### Task 1: Create an AWS Organization

In this instance, the account in use is not new and an AWS Organization has already been established.

### Task 2: Create 3 Organization Units (OUs)

**1. Navigate to AWS Organizations:**

- I make sure I'm in the "Organize accounts" view, which shows me my AWS Organization and its structure.

**2. Create Organization Units:**

1. I navigate to the AWS Organizations page.
2. In the "Organizational structure" section, I click on the "Root" organizational unit (OU).
3. On the right side of the page, I click on the "Actions" dropdown menu and select "Create new".
4. A pop-up dialogue appears, and I enter a name for my organizational unit, such as "OU1".
5. I click on "Create organization unit".
6. I see that the new organizational unit "OU1" has appeared under the "Root" OU in the organizational structure.
7. I repeat the above steps if I want to create additional organizational units.
<br>![[Pasted image 20231013124855.png]]


### Task 3: Restricting AWS Access to EC2 Only

1. I log into my AWS account and navigate to the AWS Organizations dashboard.
2. On the left side of the dashboard, I click on "Policies".
3. Within the "Policies" section, I click on "Service control policies".
   <br>![[Pasted image 20231013143604.png]]
4. To enable service control policies, I click on the "Enable service control policies" button.
   <br>![[Pasted image 20231013143618.png]]
5. Once enabled, I proceed to create a new policy that allows EC2 access. To do this: a. I click on the "Create policy" button (or a similar action button). b. In the policy editor, I provide the necessary JSON policy document that allows EC2 access. c. I provide a name for the policy, for example, "Access to EC2", and any other required details. d. I click "Save" or "Create" to finalize the policy creation.
   <br>![[policy.png]]
6. After the policy "Access to EC2" is created, I select it from the list of available policies.
7. With the policy selected, I click on the "Actions" dropdown menu.
8. From the dropdown, I select "Attach policy".
   <br>![[Pasted image 20231013150316.png]]
   
9. A dialogue or new page appears, listing all the organizational units (OUs). I select the OUs I recently created to which I want to attach this policy.
10. Once the desired OUs are selected, I confirm the action, attaching the "Access to EC2" policy to the chosen OUs.
    <br>![[Pasted image 20231013150448.png]]
  

Now, the "Access to EC2" policy is attached to the selected organizational units, granting them the permissions defined in the policy.
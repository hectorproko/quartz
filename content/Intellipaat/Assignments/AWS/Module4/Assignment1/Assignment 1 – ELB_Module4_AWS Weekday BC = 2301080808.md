---
tags:
  - AWS
title: Setting Up and Migrating Load Balancers for Efficient Traffic Management on AWS
---
<!--
**Optimizing Web Traffic with AWS Elastic Load Balancer!** In a hands-on learning assignment, I mastered using AWS ELB by creating three EC2 instances, each hosting a simple web page. I initially set up a Classic Load Balancer to manage traffic among these instances and then successfully migrated it to an Application Load Balancer for more efficient, feature-rich handling. This experience has significantly improved my ability to implement scalable and reliable traffic management solutions in AWS.
-->
 
> [!info] Assignment 1 – ELB Assignment
> **Problem Statement:** 
> You work for XYZ Corporation that uses on premise solutions and some limited number of systems. With the increase in requests in their application, the load also increases. So, to handle the load the corporation has to buy more systems almost on a regular basis. Realizing the need to cut down the expenses on systems, they decided to move their infrastructure to AWS. 
> 
> **Tasks To Be Performed:** 
> 1. Create a Classic Load Balancer and register 3 EC2 instances with different web pages running in them. 
> 2. Migrate the Classic Load Balancer into an Application Load Balancer.


### Creation of the Three EC2 Instances:
Created 3 EC2 instances with the following:
<br>![[Pasted image 20230922172403.png]]
- **Security Group Attached**: 
  I attached a security group to each of the three instances that allows HTTP, HTTPS, and SSH from all sources.
  <br>![[Pasted image 20230922162116.png|500]]

- **Commands Executed on Each EC2 Instance**:
  
```bash
# Update the package manager
sudo yum update
# Install the Apache web server
sudo yum install httpd -y
# Start the Apache web server
sudo systemctl start httpd
# Create a web page
echo "Hello from Instance1!" | sudo tee /var/www/html/index.html
# Restart the Apache web server
sudo systemctl restart httpd
```

Changed: `"Hello from Instance[1,2,3]!"` for each instance


### Creating a Classic Load Balancer:

1. **Open AWS Management Console**:
   - I'll log into my AWS account and go to the EC2 dashboard.

2. **Start Creating the Classic Load Balancer**:
   - Under the `Load Balancers` section, I'll click on `Create Load Balancer`.
  - I'll choose `Classic Load Balancer` and hit `Create`.
    <br>![[Pasted image 20230922172146.png|400]]

3. **Set Up the Load Balancer**:
 - I'll give it a unique name.
 - I'll ensure it's in my desired VPC and Mappings `us-east-1a (use1-az6)` is selected since my EC2s are in Availability Zone `us-east-1a`.
 - Under `Listeners and routing`, I'll ensure HTTP is set up.

4. **Assigning Security Groups**: 
 - I'll click "Next: Assign Security Groups".
 - I'll select the security group I previously created that allows HTTP, HTTPS, and SSH.

5. **Configure Health Checks**: 
 - I'll set the Ping Protocol to HTTP, Ping Port to 80, and Ping Path to `/index.html`.

6. **Adding My EC2 Instances**:
 - I'll click on `Add instances`.
 - I'll select the three instances I set up manually.
   <br>![[Pasted image 20230922173204.png]]


7. **Finalize Creation**:
 - I'll review everything and click `Create load balancer`.

<br>![[Pasted image 20230922172900.png]]

When I open the browser using the DNS name of the load balancer, I see that it hits all three servers.
<br>![[Pasted image 20230922173742.png]]
<br>![[Pasted image 20230922173758.png]]
<br>![[Pasted image 20230922174124.png]]
### Migrating to an Application Load Balancer:

To migrate a Classic Load Balancer to an ALB using the migration wizard, follow these steps:

1. Open the AWS Elastic Load Balancing console.
2. In the left pane, click **Load Balancers**.
3. Select the Classic Load Balancer that you want to migrate.
4. Click the **Migration** tab.
5. Click the **Launch ALB Migration Wizard** button.
   <br>![[Pasted image 20230922215643.png|500]]
6. Follow the instructions in the wizard to migrate your Classic Load Balancer to an ALB.
   <br>![[Pasted image 20230922220124.png]]

Once the migration is complete, you will be able to use the ALB to load balance your traffic.
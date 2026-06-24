---


tags:
  - draft
  - ECS
  - SecretsManagement
linkedin: "False"
quartz: "False"
refactored: "False"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
## Overview

In this hands-on lab, I deployed a WordPress application on AWS using a fully containerized, serverless architecture. The goal was to practice tying together several core AWS services that are commonly used in production environments:

- **[[Amazon RDS (Relational Database Service)|Amazon RDS]]** for a managed MySQL database
- **[[AWS Systems Manager (SSM)|AWS Systems Manager Parameter Store]]** for storing non-sensitive configuration values 
- **[[AWS Secrets Manager]]** for storing and rotating database credentials securely
- **[[AWS Elastic Container Registry (ECR)|Amazon ECR]]** for hosting the WordPress container image 
- **[[AWS ECS (Elastic Container Service)|Amazon ECS]] on [[AWS Fargate|Fargate]]** for running the application without managing servers
- **Application Load Balancer (ALB)** for routing public traffic to the containerized app

**Why this matters:** This pattern (RDS + Parameter Store + Secrets Manager + ECS Fargate) is a realistic, security-conscious way to run a web application in AWS. It avoids hardcoding credentials in the container, keeps the database in private subnets, and uses Fargate so there's no EC2 infrastructure to patch or scale manually.

## Prerequisites

The the following resources were already created:

| Type                      | Name                       |
| ------------------------- | -------------------------- |
| Application Load Balancer | OurApplicationLoadBalancer |
| Security Group            | ALBAllowHttp               |
| Cloud9 Environment        | OurCloud9Environment       |
| IAM Role                  | OurEcsTaskExecutionRole    |
| IAM Role                  | OurEcsTaskRole             |

![[Pasted image 20260618172932.png]]

![[Pasted image 20260618173014.png]]

![[Pasted image 20260618173103.png]]

![[Pasted image 20260618173235.png]]

![[Pasted image 20260618173235.png]]
## Step 1: Create a Database Subnet Group

Before creating the RDS instance, it needs a custom subnet group so it can be placed across multiple Availability Zones in a private network.

1. Navigated to **Amazon RDS** > **Subnet groups** > **Create DB subnet group**.
2. Named it `database-subnet-group`.
3. Selected **Your Custom VPC** as the VPC.
4. Under Availability Zones, selected **us-east-1a**, **us-east-1b**, and **us-east-1c**.
5. Selected the subnets with CIDRs `10.0.20.0/24`, `10.0.21.0/24`, and `10.0.22.0/24`.
6. Clicked **Create**.

> [!NOTE]- Subnet Group Details
> ![[Pasted image 20260618173559.png]]

![[Pasted image 20260618173754.png]]

## Step 2: Create the Amazon RDS Instance

With the subnet group ready, I created the MySQL database that WordPress will use.

1. Navigated to **Amazon RDS** > **Databases** > **Create database**. 
   ![[Pasted image 20260618173917.png|260]]
2. Selected **Full configuration**.
3. Chose **MySQL** as the engine type, leaving the default edition (**MySQL Community**) and the latest engine version.
4. Under **Templates**, the only options available were _Production_, _Dev/Test_, and _Sandbox_ (no "Free Tier" option as the original instructions suggested) — I selected **Sandbox**.
5. Set the **DB cluster identifier** to `wordpress`.
6. Left the **Master username** as `admin`.
7. Enabled **Manage master credentials in AWS Secrets Manager**, keeping the default encryption key.
8. Under **Instance configuration**, selected **db.t4g.micro** (Burstable classes).
9. Set **Storage type** to **General Purpose SSD (gp3)** with **20 GiB** allocated.
10. Under **Connectivity**, selected **Don't connect to an EC2 compute resource**.
11. Placed the database in **Your Custom VPC** (not the default VPC).
12. Selected the `database-subnet-group` created in Step 1.
13. Set **Public access** to **No**.
14. For the VPC security group, selected **Create new**, named it `database-sg`, and left **Availability Zone** as **No preference**.
15. Under **Additional configuration**, set the **Initial database name** to `wordpress`.
16. Left all remaining settings as defaults and clicked **Create database**.

> [!NOTE]- Database Creation Details
> ![[Pasted image 20260618174900.png]]

While the database was provisioning, I updated the security group to allow MySQL traffic from within the VPC:

1. Opened **Amazon EC2** > **Security Groups** > `database-sg`.
    
2. Edited **Inbound rules**, removing the existing rule and adding:
    
    |Type|Protocol|Port range|Source|Value|Description|
    |---|---|---|---|---|---|
    |MYSQL/Aurora|TCP|3306|Custom|`10.0.0.0/16`|Allow MySQL access from VPC|
    
3. Saved the rules and waited until the database status showed **Available** before continuing.
    

![[Pasted image 20260618175213.png]]

## Step 3: Create Parameter Store Parameters and Verify the Secrets Manager Secret

WordPress needs the database host and database name. Rather than hardcoding these, I stored them in Parameter Store.

1. Navigated to **Parameter Store** > **Create Parameter**.
    
2. Created the following two parameters, pulling the RDS endpoint from the RDS instance's _Connectivity & security_ tab:
    
    |Name|Description|Tier|Type|Data type|Value|
    |---|---|---|---|---|---|
    |`/dev/WORDPRESS_DB_HOST`|WordPress RDS endpoint|Standard|String|text|`<YOUR_RDS_ENDPOINT>:3306` (e.g. `wordpress.coruwcaic84r.us-east-1.rds.amazonaws.com:3306`)|
    |`/dev/WORDPRESS_DB_NAME`|WordPress RDS Database Name|Standard|SecureString (default encryption)|text|`wordpress`|
    

> [!NOTE]- Pulling RDS endpoint
> ![[Pasted image 20260618175722.png]]

> [!NOTE]- Parameter Details
> > [!info]- DB_HOST
> > ![[Pasted image 20260618175809.png]]
> 
> > [!info]- DB_NAME
> > ![[Pasted image 20260618180103.png]]
> 

![[Pasted image 20260618180158.png]]

3. Opened **AWS Secrets Manager** and confirmed a new secret existed with a naming convention similar to `rds!db-1380b131-f0b2-4ef3-833f-4ab7a78f29fd`.
    
4. Selected the secret and clicked **Retrieve secret value** to view the admin credentials.
    

> [!NOTE]- Snapshot
> ![[Pasted image 20260618180316.png]]

    
5. Checked the **Versions** tab to confirm there was only one version tagged **AWSCURRENT** (the staging label shown was `AWSCURRENT`).
    
6. Left both tabs (Parameter Store and Secrets Manager) open for reference when configuring the ECS container later.
    

## Step 4: Create the ECR Repository and Push the WordPress Image

Next, I built and pushed the WordPress container image to a private ECR repository.

1. Navigated to **Elastic Container Registry** > **Repositories** > **Create repository**.
    
2. Set **Visibility settings** to **Private**, and **Repository name** to `wordpress`.
    
3. Left **Tag immutability** disabled, **enabled** _Scan on push_, and left **KMS encryption** disabled.
    
4. Clicked **Create repository**.
    

> [!NOTE]- Snapshot
> ![[Pasted image 20260618180637.png]]

    
5. Selected the new `wordpress` repository.
    
6. Opened **AWS Cloud9** > `OurCloud9Environment` > **Open in Cloud9**, then expanded the terminal session.
    

### Configuring CLI Access

To push the image, the Cloud9 environment needed AWS CLI credentials tied to the `cloud_user` IAM user:

1. In **IAM** > **Users** > `cloud_user` > **Security Credentials**, created a new access key under **Access keys**, selecting **Command Line Interface (CLI)** and confirming the recommendation acknowledgment.
    
2. Tagged the key with the description `cli access key`.
    
3. Copied the access key and secret access key values.
   ![[Pasted image 20260618181117.png|500]]
    
    
4. In Cloud9, ran:
    
    ```shell
    aws configure --profile cloud_user
    ```
    
5. Pasted in the access key, then the secret access key, set the default region to `us-east-1`, and the default output format to `json`.
   ![[Pasted image 20260618181506.png]]
    
6. If prompted about _AWS managed temporary credentials_, selected **Cancel**, then **Re-enable after refresh**.
    

#### Troubleshooting: Credential Validation Check

To confirm the `cloud_user` profile was configured correctly, I ran a quick credential validation test:

```shell
aws s3 ls --profile cloud_user
```

This command isn't really about S3, it's a low-stakes way to confirm the CLI can authenticate with the configured credentials and reach AWS. If the credentials were invalid, the command would fail with an authentication error. In this case, it returned an empty result with no error, which confirmed the credentials were valid (the account simply had no S3 buckets to list). 

### Authenticating Docker and Pushing the Image

1. Left the Cloud9 tab running and returned to the ECR tab, then opened **View push commands**.
    ![[Pasted image 20260618181801.png]]
    ![[Pasted image 20260618182137.png|500]]
    
2. Copied **[[#Step 1 Create a Database Subnet Group|Step 1]]** of the push commands into the Cloud9 terminal, adding `--profile cloud_user` before the pipe:
    
    ```shell
    aws ecr get-login-password --region us-east-1 --profile cloud_user | docker login --username AWS --password-stdin 864572276231.dkr.ecr.us-east-1.amazonaws.com
    ```
    
    Output:
    
    ```
    cloud_user:~/environment $ aws ecr get-login-password --region us-east-1 --profile cloud_user | docker login --username AWS --password-stdin 864572276231.dkr.ecr.us-east-1.amazonaws.com
    WARNING! Your password will be stored unencrypted in /home/ec2-user/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store
    
    Login Succeeded
    cloud_user:~/environment $
    ```
    
3. Pulled the latest WordPress image:
    
    ```shell
    docker pull wordpress:latest
    ```
    
4. Tagged the image using **[[Pasted image 20260618182137.png|Step 3 from the ECR push commands]]** *(Step 2, the build step, wasn't needed since the image was already built)*:
    
    ```shell
    docker tag wordpress:latest 864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest
    ```
    
5. Verified the image existed locally:
    
    ```shell
    docker image ls
    ```
    
    Output:
    
    ```
    cloud_user:~/environment $ docker tag wordpress:latest 864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest
    cloud_user:~/environment $ docker image ls
    REPOSITORY                                               TAG       IMAGE ID       CREATED      SIZE
    864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress   latest    f6104f9a84b4   7 days ago   762MB
    wordpress                                                latest    f6104f9a84b4   7 days ago   762MB
    cloud_user:~/environment $
    ```
    
6. Pushed the image using **[[Pasted image 20260618182137.png|Step 4 of the push commands]]**:
    
    ```shell
    docker push 864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest
    ```
    
7. Verified the image appeared in the ECR console, then left the tab open for the next step.
    

![[Pasted image 20260618182453.png]]

## Step 5: Create the Amazon ECS Task Definition

With the image in ECR, I defined how the container should run.

1. Navigated to **Amazon ECS** > **Task definitions** > **Create new task definition**.
    
2. Set the **Task definition family** to `wordpress-td`.
    
3. Under **Infrastructure requirements**, set **Launch type** to **AWS Fargate**.
    
4. Set the **Task role** to **OurEcsTaskRole** and the **Task execution role** to **OurEcsTaskExecutionRole**.
    
5. Configured **Container - 1**:
    
    |Name|Image URI|Essential container|
    |---|---|---|
    |`wordpress`|`864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest`|Yes|
    
6. Added four environment variables, all sourced via `ValueFrom` so no secrets are hardcoded in the task definition:
    
    |Key|Value type|Value|
    |---|---|---|
    |`WORDPRESS_DB_HOST`|ValueFrom|ARN of the `/dev/WORDPRESS_DB_HOST` Parameter Store parameter|
    |`WORDPRESS_DB_NAME`|ValueFrom|ARN of the `/dev/WORDPRESS_DB_NAME` Parameter Store parameter|
    |`WORDPRESS_DB_USER`|ValueFrom|ARN of the Secrets Manager RDS secret, with `:username::` appended (e.g. `arn:aws:secretsmanager:us-east-1:864572276231:secret:rds!db-7874795c-0509-460b-a932-a761a7dbd3a4-07y1yB:username::`)|
    |`WORDPRESS_DB_PASSWORD`|ValueFrom|ARN of the Secrets Manager RDS secret, with `:password::` appended (e.g. `arn:aws:secretsmanager:us-east-1:864572276231:secret:rds!db-7874795c-0509-460b-a932-a761a7dbd3a4-07y1yB:password::`)|
    
7. Clicked **Create**.
    

`Pasted image 20260618184655.png`

## Step 6: Create the ECS Cluster and Service

1. In **Amazon ECS** > **Clusters** > **Create cluster**, named it `Wordpress-Cluster` and selected **AWS Fargate (serverless)** infrastructure, then clicked **Create**.
    
    `Pasted image 20260618185335.png`
    

#### Troubleshooting: Cluster Creation Errors

If the cluster creation produces service-related errors, AWS generates a CloudFormation stack behind the scenes. Navigating to that CloudFormation template and retrying the deployment from there resolves most transient creation failures.

`Pasted image 20260618185638.png`

2. Selected the new `Wordpress-Cluster`, then under the **Services** tab clicked **Create**.
    
3. Under **Compute options**, selected **Launch type**, set to **FARGATE**, with **Platform version** set to **LATEST**.
    
4. Under **Deployment configuration**, the **Application type** field described in the original instructions (select "Service") wasn't present in this version of the console, so I proceeded without setting it.
    
5. For **Family**, selected the `wordpress-td` task definition (`LATEST` revision).
    
6. Named the service `wordpress-service` with a desired task count of `1`.
    
7. Under **Networking**, selected **Your Custom VPC**, cleared the default subnet selection, and selected only the **Private Subnet** entries.
    
8. Created a new security group named `app-sg` with the following inbound rule:
    
    |Type|Protocol|Port range|Source|Value|
    |---|---|---|---|---|
    |HTTP|TCP|80|Source group|Security Group of `ALBAllowHttp`|
    
    (Referenced from `Pasted image 20260618173014.png`)
    
9. Set **Public IP** to **off**.
    
10. Under **Load balancing**, selected **Application Load Balancer** > **Use an existing load balancer** > `OurApplicationLoadBalancer`.
    

#### Troubleshooting: Missing Health Check Grace Period Field

The instructions called for setting the **Health check grace period** to `30 seconds`, but this field wasn't visible anywhere in the service creation flow for this console version. I proceeded without setting it explicitly and the service still came up successfully, but it's worth checking the **Service auto scaling** or **Deployment configuration** sections in your own console version if you need this set.

11. Left the **Listener** values as default.
12. For **Target group**, named it `wordpress-tg`.
13. Set the **Health check path** to `/wp-admin/images/wordpress-logo.svg` — this is necessary because WordPress isn't fully set up yet at this point, and the default health check path would otherwise fail.
14. Left the remaining values as defaults and clicked **Create**.
15. Waited for the service to reach a running state.

`Pasted image 20260618191035.png` `Pasted image 20260618191334.png` `Pasted image 20260618191507.png`

## Step 7: Connect to the Application

1. Once the service was running, confirmed a task was listed under the cluster's **Tasks** tab.
    
    `Pasted image 20260618191507.png`
    
2. Navigated to **Amazon EC2** > **Load balancers** > `OurApplicationLoadBalancer`.
    
3. Copied the ALB's DNS name into a new browser tab over **HTTP**.
    
4. Was greeted with the WordPress setup page, confirming the application was successfully running on ECS Fargate and connected to the RDS database.
    

`Pasted image 20260618191711.png` `Pasted image 20260618191757.png`

## Conclusion

This lab tied together networking, managed database services, secrets handling, container registries, and serverless compute into a single working deployment. The most valuable part wasn't just getting WordPress running, it was seeing how Parameter Store and Secrets Manager keep configuration and credentials out of the application code and task definitions entirely, which is a pattern I'll carry into future containerized projects on AWS.


<!--
[Hosting a Wordpress Application on ECS Fargate with RDS, Parameter Store, and Secrets Manager](https://app.pluralsight.com/hands-on/labs/0c585b09-7459-4280-88c0-31d3cc02f297?originUrl=https%3A%2F%2Fapp.pluralsight.com%2Fsearch%2F)

## Lab Overview

During this hands-on lab you will explore how to deploy a WordPress application using several AWS services, including: Amazon RDS, AWS Systems Manager, Parameter Store, Amazon ECS, Amazon ECR, and Application Load Balancers. Let's get started!


# Hosting a Wordpress Application on ECS Fargate with RDS, Parameter Store, and Secrets Manager

## Introduction

During this hands-on lab you will explore how to deploy a WordPress application using several AWS services, including: Amazon RDS, AWS Systems Manager, Parameter Store, Amazon ECS, Amazon ECR, and Application Load Balancers.

## Solution

Log in to the AWS Management Console using the credentials provided on the lab instructions page. Make sure you're using the `us-east-1` Region.

### Verify Existing Infrastructure

Before you begin, please verify the below resources exist and if any of these do not exist, please exit the lab and restart.

```
| Type                      | Name                       |
|---------------------------|----------------------------|
| Application Load Balancer | OurApplicationLoadBalancer |
| Security Group            | ALBAllowHttp               |
| Cloud9 Environment        | OurCloud9Environment       |
| IAM Role                  | OurEcsTaskExecutionRole    |
| IAM Role                  | OurEcsTaskRole             |
```

![[Pasted image 20260618172932.png]]
![[Pasted image 20260618173014.png]]
![[Pasted image 20260618173103.png]]
![[Pasted image 20260618173235.png]]
### Create Database Subnet Group

First, we need to create our custom subnet group to host our DB instance.

1. Navigate to the **Amazon RDS** service.
2. Find and select **Subnet groups** from the menu.
3. click **Create DB subnet group**.
4. Name it `database-subnet-group`.
5. Enter the description of your choosing.
6. Choose the **Your Custom VPC** for the _VPC_ options.
7. Move to the _Add subnets_ section.
8. Under _Availability Zones_, select **us-east-1a**, **us-east-1b**, and **us-east-1c**.
9. For _Subnets_, select the subnets with the CIDRs of `10.0.20.0/24`, `10.0.21.0/24`, and `10.0.22.0/24`.
10. Click on **Create**.

![[Pasted image 20260618173559.png]]

![[Pasted image 20260618173754.png]]
### Create the Amazon RDS Instance

Now, we can create our RDS instance.

1. Navigate to the **Amazon RDS** service.
    
2. Under _Databases_ find and select **Create database**.
    ![[Pasted image 20260618173917.png]]
    selected Full configuration
3. Select **MySQL** from the list of engine types.
    
4. Leave the default _Edition_ selected, which should be **MySQL Community**.
    
5. For _Engine version_ the latest version should populate by default.
    
6. Under _Templates_ select **Free Tier**.
    the only templates sections had Production, Dev/Test and Sandbox
    i picked Sandbox
7. Move to the _Settings_ section. For _DB cluster identifier_, change the name to `wordpress`.
    
8. Leave the _Master username_ set to `admin`.
    
9. Check the box for the option to **Manage master credentials in AWS Secrets Manager**.
    
10. Leave the default encryption key for the Secrets Manager credentials.
    
11. Move to the _Instance configuration_ section. For _DB instance class_, under _Burstable classes_ select **db.t4g.micro**.
    
12. For the _Storage_ section, set the _Storage type_ to **General Purpose SSD (gp3)**.
    
13. Set the allocate storage to `20 GiB`.
    
14. Under the _Connectivity_ section, select **Don’t connect to an EC2 compute resource**.
    
15. Place the database in the VPC titled **Your Custom VPC** (_Do not use the default VPC!_)
    
16. Choose the **database subnet group** from the _DB subnet group_ dropdown.
    
17. Set _Public access_ to **No**.
    
18. For _VPC security group (firewall)_ select **Create new**, name it `database-sg`, and then select **No preference** for _Availability Zone_.
    
19. Skip down to the _Additional configuration_ dropdown menu near the bottom of the page.
    
20. Within the _Database options_ set the _Initial database name_ to `wordpress`.
    
21. Leave the rest of the options set to the defaults.
    
22. Find and select **Create database**. Creation of the DB cluster could take several minutes. While you wait for this to become available, you need to edit the Security Group.
    ![[Pasted image 20260618174900.png]]
23. Navigate to the **Amazon EC2** console in a new tab.
    
24. Find and select the **Security Groups** menu.
    
25. Find and select the `database-sg` Security Group.
    
26. Select _Inbound rules_ and then click **Edit inbound rules**.
    
27. Delete the existing rule and add the following rule:
    
    |Type|Protocol|Port range|Source|Value|Description - optional|
    |---|---|---|---|---|---|
    |MYSQL/Aurora|TCP|3306|Custom|`10.0.0.0/16`|`Allow MySQL access from VPC`|
    
28. Click **Save rules**.
    
29. Move on only after the database is **available**!

![[Pasted image 20260618175213.png]]

### Create the Parameter Store Parameters and Verify Secrets Manager Secret

Let's create our hidden values and plaintext parameters.

1. Navigate to **Parameter Store** in a new tab.
    
2. Click **Create Parameter**.
    
3. Create the following **2** parameters from the table below. You can get your `wordpress` database endpoint from the _Connectivity & security_ section of the RDS page.
    
    |Name|Description|Tier|Type|Data type|Value|
    |---|---|---|---|---|---|
    |/dev/WORDPRESS_DB_HOST|Wordpress RDS endpoint|Standard|String|text|**YOUR_RDS_ENDPOINT**`:3306` (Example: _wordpress.coruwcaic84r.us-east-1.rds.amazonaws.com:3306_|
    ![[Pasted image 20260618175722.png]]
![[Pasted image 20260618175809.png]]
    |/dev/WORDPRESS_DB_NAME|Wordpress RDS Database Name|Standard|SecureString (_default encryption_)|text|`wordpress`|
![[Pasted image 20260618180103.png]]

![[Pasted image 20260618180158.png]]

1. Once you have created the parameters above, open **AWS Secrets Manager** in a new tab.
    
2. Verify there is a new secret there with a naming convention similar to _rds!db-1380b131-f0b2-4ef3-833f-4ab7a78f29fd_.
    
3. Select the secret, and then find and click on **Retrieve secret value** to view your admin credentials.
![[Pasted image 20260618180316.png]]
4. To ensure this is the latest version, click on the **Versions** tab and make sure there is only one _AWSCURRENT_ tag.
    staging labels says awscurrent
5. Leave these two tabs open, as we will reference them later on for our containers.
    

### Create the ECR Repository and Image

1. Navigate to **Elastic Container Registry** in a new tab.
    
2. Under _Private registry_ on the left-hand side menu, select **Repositories**.
    
3. Click on **Create repository**.
    
4. For _Visibility settings_ select **Private**.
    
5. For _Repository name_ enter `wordpress`.
    
6. Leave _Tag immutability_ **disabled**.
    
7. **Enable** the _Scan on push_ option.
    
8. Leave _KMS encryption_ **disabled**.
    
9. Click on **Create repository**.
    ![[Pasted image 20260618180637.png]]
10. Select your newly created `wordpress` repository.
    
11. Open a new tab for the **AWS Cloud9** service.
    
12. Find and select the `OurCloud9Environment` environment.
    
13. Click on **Open in Cloud9** (_Dismiss any message that may popup initially_).
    
14. Once loaded, select and expand the bottom terminal session.
    
15. Now, navigate to **AWS IAM** in a new tab.
    
16. Find and select **Users** from the menu.
    
17. Find your `cloud_user` and select it.
    
18. Click on **Security Credentials**.
    
19. Under _Access keys_, click on **Create access key**.
    
20. Select the **Command Line Interface (CLI)** radio button and click the box to confirm you understand the recommendations.
    
21. Click **Next**.
    
22. Enter your own description tag value and click **Create access key**.
    i put cli access key
23. Under the _Retrieve access keys_ page, copy the **Access key** value, then navigate back to your Cloud9 tab.
    
    
	
![[Pasted image 20260618181117.png]]
	
1. In Cloud9, run `aws configure --profile cloud_user`.
    
2. Paste in your **Access key** and hit enter.
    
3. Go back to your IAM access key tab and then copy the **Secret access key** value.
    
4. Navigate back to Cloud9 and paste in the value, then hit enter.
    
5. Set the default region to `us-east-1`.
    
6. Set default output to `json`.
    ![[Pasted image 20260618181506.png]]
7. If you get a popup about _AWS managed temporary credentials_, select **Cancel** and then **Re-enable after refresh**.
    
8. Test that you can perform an AWS CLIv2 command (Example: `aws s3 ls --profile cloud_user`).
    i run this command and nothing appears
    
9. Leave your Cloud9 tab running, and Navigate back to the **ECR tab**.
    
10. Click on **View push commands**.
    ![[Pasted image 20260618181801.png]]
    ![[Pasted image 20260618182137.png]]
11. Copy and paste **Step 1** into your Cloud9 terminal. Before entering, add the `--profile cloud_user` to the portion before the pipe. Example below:
    
```shell
aws ecr get-login-password --region us-east-1 --profile cloud_user | docker login --username AWS --password-stdin 864572276231.dkr.ecr.us-east-1.amazonaws.com
```

this is what was there
Retrieve an authentication token and authenticate your Docker client to your registry. Use the AWS CLI:
`aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 864572276231.dkr.ecr.us-east-1.amazonaws.com`

this was the output
```
cloud_user:~/environment $ aws ecr get-login-password --region us-east-1 --profile cloud_user | docker login --username AWS --password-stdin 864572276231.dkr.ecr.us-east-1.amazonaws.com
WARNING! Your password will be stored unencrypted in /home/ec2-user/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
cloud_user:~/environment $
```


1. You should see a **Login Succeeded** message.
    
2. Pull the latest Docker image for WordPress running this command:
    
    ```shell
       docker pull wordpress:latest
    ```
    
3. Once complete, tag the image we want to push by copying and pasting **Step 3** from the ECR push commands prompt. (_We don't need to run Step 2 because the image is already built_).

- After the build completes, tag your image so you can push the image to this repository:
`docker tag wordpress:latest 864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest`


4. Verify the new image exists locally by running the following command:
    
    ```shell
    docker image ls
    ```

```
cloud_user:~/environment $ docker tag wordpress:latest 864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest
cloud_user:~/environment $ docker image ls
REPOSITORY                                               TAG       IMAGE ID       CREATED      SIZE
864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress   latest    f6104f9a84b4   7 days ago   762MB
wordpress                                                latest    f6104f9a84b4   7 days ago   762MB
cloud_user:~/environment $ 
```


4. Now run **Step 4** from the ECR push commands.

- Run the following command to push this image to your newly created AWS repository:
    
    `docker push 864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest`


5. It should push our new image to ECR, which you can verify in the ECR console after completion.
    
6. After verifying the image is in place, leave the ECR tab open and then move on to the next section!
    
![[Pasted image 20260618182453.png]]
### Create the Amazon ECS Task Definition

1. Navigate to the **Amazon ECS** service in a new tab.
    
2. Once there, find and select **Task definitions** from the left-hand menu.
    
3. Click on **Create new task definition**.
    
4. Enter `wordpress-td` for the _Task definition family_.
    
5. Under _Infrastructure requirements_, for _Launch type_, select **AWS Fargate**.
    
6. Leave the other defaults as-is, and then select **OurEcsTaskRole** for the _Task role_.
    
7. For the _Task execution role_, find and select the already created role called `OurEcsTaskExecutionRole`.
    
8. Under _Container - 1_, enter the following settings for _Container details_. You can get your `wordpress` image URI from the ECR page.
    
    URI: 864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest
    
    |Name|Image URI|Essential container|
    |---|---|---|
    |`wordpress`|Your ECR Image URI from the custom image you pushed (Example: _864572276231.dkr.ecr.us-east-1.amazonaws.com/wordpress:latest_)|Yes|
    
9. Leave the other defaults and find the _Environment variables_ section.
    
10. Click on **Add environment variable** and then fill in the information for each of the below **4** variables. **PLEASE NOTE THE ARN SYNTAX OF THE SECRETS MANAGER SECRET.**
    
    |Key|Value type|Value|
    |---|---|---|
    |WORDPRESS_DB_HOST|ValueFrom|ARN of the respective Parameter Store parameter (_[arn:aws:ssm:us-east-1:552898056824:parameter/dev/WORDPRESS_DB_HOST](arn:aws:ssm:us-east-1:864572276231:parameter/dev/WORDPRESS_DB_HOST)_|
    |WORDPRESS_DB_NAME|ValueFrom|ARN of the respective Parameter Store parameter (arn:aws:ssm:us-east-1:864572276231:parameter/dev/WORDPRESS_DB_NAME|
    |WORDPRESS_DB_USER|ValueFrom|ARN of the respective Secrets Manager RDS secret, specifying the specific key value by adding `:username::` at the end (`arn:aws:secretsmanager:us-east-1:864572276231:secret:rds!db-7874795c-0509-460b-a932-a761a7dbd3a4-07y1yB:username::`|
    |WORDPRESS_DB_PASSWORD|ValueFrom|ARN of the respective Secrets Manager RDS secret, specifying the specific key value by adding `:password::` at the end (`arn:aws:secretsmanager:us-east-1:864572276231:secret:rds!db-7874795c-0509-460b-a932-a761a7dbd3a4-07y1yB:password::` |
    
11. Click on **Create** at the bottom.
    
![[Pasted image 20260618184655.png]]
### Create the ECS Cluster and Service

1. Within the ECS service, find and select **Clusters**.
    
2. Click **Create cluster**.
    
3. For _Cluster name_ enter `Wordpress-Cluster`.
    
4. Under **Infrastructure**, select `AWS Fargate (serverless)`.
    
5. Click on **Create**.
    ![[Pasted image 20260618185335.png]]
6. Wait for your cluster to be created before moving on. If you get any service related errors, please navigate to the CloudFormation template that is created for you by the service and retry the deployment.
    ![[Pasted image 20260618185638.png]]
7. Select your **Wordpress-Cluster**.
    
8. Under the _Services_ tab, click on **Create**.
    
9. For _Compute options_ select **Launch type**.
    
10. Make sure the _Launch type_ is set to **FARGATE**.
    
11. Make sure _Platform version_ is set to **LATEST**.
    
12. Move to _Deployment configuration_.
    
13. For _Application type_ select **Service**. did not see this option
    
14. For _Family_, under _Task definition_, choose your `wordpress-td` task definition from the dropdown and use the `LATEST` version.
    
15. Name your service `wordpress-service`.
    
16. Set desired tasks to `1`.
    
17. Skip to the _Networking_ section and select **Your Custom VPC**.
    
18. For _Subnets_, click **Clear current selection** and only select the ones titled **Private Subnet**.
    
19. Choose **Create a new security group** for _Security group_.
    
20. Name it `app-sg` and give it your description of choice.
    
21. For _Inbound rules for security groups_, enter the following information:
    
    |Type|Protocl|Port range|Source|Values|
    |---|---|---|---|---|
    |HTTP|TCP|80|Source group|Security Group of the `ALBAllowHttp`|
    from [[Pasted image 20260618173014.png]]
1. Set _Public IP_ to be turned **off**.
    
2. Move to _Load balancing_, for _Load balancer type_, select **Application Load Balancer**.
    
3. Choose **Use an existing load balancer**.
    
4. Find the load balancer named `OurApplicationLoadBalancer`.
    
5. Set the _Health check grace period_ to `30 seconds`. didnt see where to put this
    
6. Leave the _Listener_ values as default.
    
7. For _Target group_, name it `wordpress-tg`.
    
8. For the _Health check path_, enter `/wp-admin/images/wordpress-logo.svg`. (_We do this because we are needing to set up WP still. Otherwise, it will fail the initial health check_).
    
9. Leave the rest of the values as the defaults.
    
10. Skip to the bottom and select **Create**.
    
11. Wait until your service is up and running, then you can move on!
    
![[Pasted image 20260618191035.png]]

![[Pasted image 20260618191334.png]]

![[Pasted image 20260618191507.png]]
### Connect to the Application

1. Once you have a service up and running, you should see a task listed under the _Tasks_ tab within the cluster.
   ![[Pasted image 20260618191507.png]]
2. Navigate to the **Amazon EC2** console.
3. Find and select **Load balancer**.
4. Find the `OurApplicationLoadBalancer` and choose it.
5. Now, copy and paste your ALB DNS name into a new tab, ensuring you use **HTTP**.
6. You should be greeted by the Wordpress setup page!

![[Pasted image 20260618191711.png]]

![[Pasted image 20260618191757.png]]
## Conclusion

Congratulations — you've completed this hands-on lab!
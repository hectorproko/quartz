---
tags:
  - "#Terraform"
  - IaC
hardlinked: "True"
linkedin: "False"
quartz: "False"
refactored: "True"
darey.io: "True"
hands-on: "True"
completed: "True"
title: "Automating AWS Infrastructure with Terraform - Part 3: Remote Backends & Modules"
---

> **Part:** 3 of 4  
> **Topics:** Remote State · S3 Backend · DynamoDB Locking · State Migration · Outputs · Environment Isolation · Modules

---

## **Overview**

By the end of [[PART2_PROJECT_17|Part 2: Building the Full Stack]] the full AWS stack was defined in code, but with two structural problems: the Terraform **state** lived only on my local machine, and every resource sat in one long flat list of `.tf` files.

This article fixes both. First I move state to a remote **S3 backend** with **DynamoDB** locking, so the state is durable, shareable across a team, and protected against concurrent writes. Then I refactor the entire configuration into reusable **modules**, grouping resources by domain (VPC, ALB, Autoscaling, EFS, RDS, Security).

---

## Why a Remote Backend?

Each Terraform configuration can declare a **backend**, which defines where state snapshots are stored and how operations run. The state file is just where Terraform records the real-world state of the infrastructure in JSON.

So far I've used the **default local backend** — no configuration, state stored on disk. That has two problems:

1. **Durability** — a local file is fragile; losing it means losing Terraform's view of the infrastructure.
2. **Collaboration** — other engineers can't reach a state file sitting on my laptop.

The fix is a shared, remote backend. Since the project already runs on AWS, an [S3 bucket as a backend](https://www.terraform.io/docs/language/settings/backends/s3.html) is the natural choice. S3 also supports optional [**state locking**](https://www.terraform.io/docs/language/state/locking.html), which prevents two people from writing state at the same time and corrupting it. Locking on S3 requires one more service: [DynamoDB](https://aws.amazon.com/dynamodb/).

**The migration follows six steps:**

1. Add S3 and DynamoDB resource blocks (before deleting the local state file)
2. Update the `terraform` block to introduce the backend and locking
3. Re-initialize Terraform
4. Delete the local tfstate file and confirm the one in S3
5. Add outputs
6. `terraform apply`

---

## Step 1 - Create the S3 Backend Bucket

I create `backend.tf`. The S3 bucket from [[PART1_PROJECT_16|Part 1: Getting Started]] is reused here, with **versioning** enabled (so I keep the full revision history of state files) and **server-side encryption** enabled by default.

```bash
# must give it a unique name globally
resource "aws_s3_bucket" "terraform_state" {
  bucket = "hector-dev-terraform-bucket"
  # Enable versioning so we can see the full revision history of our state files
  versioning {
    enabled = true
  }

  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
```

> Terraform stores **secret data** inside state — passwords, secret keys, and other sensitive values processed by resources all land there. Always enable encryption on the backend, which is what `server_side_encryption_configuration` handles.

---

## Step 2 - Create the DynamoDB Lock Table

Next, the DynamoDB table that handles locks and consistency checks. Previously locking was a local `terraform.tfstate.lock.info` file; with DynamoDB the lock lives in a central place, so anyone running Terraform against the same infrastructure is coordinated.

```bash
# DynamoDB resource for locking and consistency checking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

---

## Step 3 - Provision the Backend Resources

Terraform expects both the **S3 bucket** and the **DynamoDB table** to exist *before* the backend is configured. So I run `terraform apply` once to create them while still using the local backend.

---

## Step 4 - Configure the S3 Backend Block

Now I add the backend configuration to the `terraform` block, pointing it at the bucket, the state key, the region, and the DynamoDB lock table.

```bash
terraform {
  backend "s3" {
    bucket         = "hector-dev-terraform-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

## Step 5 - Re-initialize and Migrate State

With the backend block in place, Terraform needs to re-initialize.

### 👀 Issue: `terraform plan` fails after changing the backend

Running `terraform plan` first throws an error, because changing the backend requires re-initialization:

<details><summary>terraform plan output</summary>

```bash
hector@hector-Laptop:~/Project16-17/PBL$ terraform plan
╷
│ Error: Backend initialization required, please run "terraform init"
│
│ Reason: Initial configuration of the requested backend "s3"
│
│ The "backend" is the interface that Terraform uses to store state,
│ perform operations, etc. If this message is showing up, it means that the
│ Terraform configuration you're using is using a custom configuration for
│ the Terraform backend.
│
│ Changes to backend configurations require reinitialization. This allows
│ Terraform to set up the new configuration, copy existing state, etc. Please run
│ "terraform init" with either the "-reconfigure" or "-migrate-state" flags to
│ use the current configuration.
╵
hector@hector-Laptop:~/Project16-17/PBL$
```

</details>

**Resolution.** Run `terraform init` again. Terraform detects the existing local state and offers to copy it to the new S3 backend. Typing `yes` migrates the state automatically.

<details><summary>terraform init output</summary>

```bash
hector@hector-Laptop:~/Project16-17/PBL$ terraform init

Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "s3" backend. No existing state was found in the newly
  configured "s3" backend. Do you want to copy this state to the new "s3"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes

Releasing state lock. This may take a few moments...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Reusing previous version of hashicorp/random from the dependency lock file
- Using previously-installed hashicorp/aws v4.14.0
- Using previously-installed hashicorp/random v3.1.3

Terraform has been successfully initialized!
hector@hector-Laptop:~/Project16-17/PBL$
```

</details>

### Confirm the state in S3

After migration, the local `.tfstate` can be removed and the state file is now visible inside the **S3 bucket**.

![Example Image](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/S3Bucket_tfstate.gif)

---

## Step 6 - Add Outputs

Before the next `apply`, I add outputs so the S3 bucket ARN and DynamoDB table name print to screen. I put these in `output.tf`:

```bash
output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
  description = "The ARN of the S3 bucket"
}
output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "The name of the DynamoDB table"
}
```

Then run `terraform apply`.

---

## Step 7 - Isolating Environments

Real projects need separate resources for `dev`, `sit`, `uat`, `preprod`, `prod`, and so on. There are two common ways to separate environments:

1. **Terraform Workspaces**
2. **Directory-based separation** using `terraform.tfvars`

*([[PART4_PROJECT19_TerraformCloud|Part 4: Terraform Cloud]] explores environment separation further using Terraform Cloud and VCS branches.)*

---

## Step 8 - Refactor into Modules

A single long `.tf` file is hard to navigate. The goal is reusable, comprehensible IaC, and Terraform's built-in tool for that is [**Modules**](https://www.terraform.io/docs/language/modules/index.html).

A module is just a collection of `.tf` (or `.tf.json`) files in a directory. They let me group related resources by domain. One **root module** calls **child modules** and inserts their configuration at apply time. This keeps the structure neat and lets different team members work on different parts simultaneously.

I combine resources of a similar type into directories under a `modules` directory:

```
├── modules
    ├── ALB          # Application Load Balancer and similar resources
    ├── Autoscaling  # Autoscaling and launch template resources
    ├── EFS          # Elastic File System resources
    ├── RDS          # Database resources
    ├── Security     # Security group resources
    └── VPC          # VPC and networking resources (subnets, roles, etc.)
```

Each module typically contains:

- `<resource_name>.tf` — files with resource blocks
- `outputs.tf` — optional, when the root module needs to refer to a module's outputs
- `variables.tf` — declarations, so values aren't hard-coded

> `providers` and `backends` are configured in separate files but placed in the **root module**, not inside child modules.

### Full configuration structure

<details><summary>Configuration structure (tree)</summary>

```bash
hector@hector-Laptop:~/Project16-17/PBL$ tree
.
├── backend.tf
├── main.tf
├── modules
│   ├── ALB
│   │   ├── alb.tf
│   │   ├── cert.tf
│   │   ├── output.tf
│   │   └── variables.tf
│   ├── Autoscaling
│   │   ├── asg-bastion-nginx.tf
│   │   ├── asg-wordpress-tooling.tf
│   │   ├── bastion.sh
│   │   ├── lt-bastion-nginx.tf
│   │   ├── lt-tooling-wp.tf
│   │   ├── nginx.sh
│   │   ├── tooling.sh
│   │   ├── variables.tf
│   │   └── wordpress.sh
│   ├── EFS
│   │   ├── efs.tf
│   │   ├── output.tf
│   │   └── variables.tf
│   ├── RDS
│   │   ├── output.tf
│   │   ├── rds.tf
│   │   └── variables.tf
│   ├── Security
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   ├── security.tf
│   │   ├── sg-rule.tf
│   │   └── variables.tf
│   └── VPC
│       ├── internet_gateway.tf
│       ├── main.tf
│       ├── natgateway.tf
│       ├── outputs.tf
│       ├── roles.tf
│       ├── route_tables.tf
│       └── variables.tf
├── providers.tf
├── terraform.tfstate
├── terraform.tfstate.backup
├── terraform.tfvars
└── variables.tf

7 directories, 38 files
```

</details>

### The pattern in action: the VPC module

To see how the pieces connect, take the VPC. In the root `main.tf`, I **call** the module with no hard-coded values — everything is either generated (subnets) or passed in with `var`:

```bash
# creating VPC
module "VPC" {
  source                              = "./modules/VPC"
  region                              = var.region
  vpc_cidr                            = var.vpc_cidr
  enable_dns_support                  = var.enable_dns_support
  enable_dns_hostnames                = var.enable_dns_hostnames
  enable_classiclink                  = var.enable_classiclink
  preferred_number_of_public_subnets  = var.preferred_number_of_public_subnets
  preferred_number_of_private_subnets = var.preferred_number_of_private_subnets
  private_subnets                     = [for i in range(1, 8, 2) : cidrsubnet(var.vpc_cidr, 8, i)]
  public_subnets                      = [for i in range(2, 5, 2) : cidrsubnet(var.vpc_cidr, 8, i)]
}
```

The module's `source = "./modules/VPC"` points at `modules/VPC/main.tf`, where the actual VPC resource lives:

```bash
# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support
  enable_dns_hostnames           = var.enable_dns_hostnames
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink_dns_support
  tags = merge(
    var.tags,
    {
      Name = format("%s-VPC", var.name)
    },
  )
}
```

The chain of variables is consistent end to end: variables **used** inside the VPC module are **declared** in `modules/VPC/variables.tf`, and their **values** originate from the root `terraform.tfvars`.

> **Pro-tips**
> - Run `terraform validate` before `terraform plan` to check that the code is syntactically valid and internally consistent.
> - Run `terraform fmt` to apply Terraform's canonical formatting style to your `.tf` files automatically.

---

## Summary

Part 3 turned a working-but-messy project into a maintainable one:

- ✅ State moved to a remote **S3 backend** with versioning and encryption
- ✅ **DynamoDB** table added for state locking and consistency checks
- ✅ Local state successfully **migrated** to S3 via `terraform init`
- ✅ Outputs added for the bucket ARN and lock table name
- ✅ Two environment-isolation strategies identified (workspaces vs. directories)
- ✅ The entire configuration **refactored into modules** (ALB, Autoscaling, EFS, RDS, Security, VPC)

[[PART4_PROJECT19_TerraformCloud|Part 4: Terraform Cloud]] takes the automation one step further, running Terraform from **Terraform Cloud** with VCS-driven runs and per-branch environments.

---

_GitHub Repository: [AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4)_

<!--
ORIGINAL RAW NOTES (Project 18) - preserved for reference

Reference repo: https://github.com/darey-devops/PBL-project-18.git (some .sh files are empty)

Now we will explore alternative Terraform backends.
Introducing Backend on S3 - each Terraform config can specify a backend (where/how operations run, where state snapshots are stored).
Default = local backend (no config, state stored locally) - not robust, not shareable.
Choose S3 bucket as backend. State Locking (optional) requires DynamoDB.

Steps to re-initialize Terraform to use S3 backend:
1. Add S3 and DynamoDB resource blocks before deleting the local state file
2. Update terraform block to introduce backend and locking
3. Re-initialize terraform
4. Delete the local tfstate file and check the one in S3 bucket
5. Add outputs
6. terraform apply

- backend.tf: aws_s3_bucket "terraform_state" (versioning enabled, SSE AES256)
- aws_dynamodb_table "terraform_locks" (PAY_PER_REQUEST, hash_key LockID)
- terraform { backend "s3" { bucket/key/region/dynamodb_table/encrypt } }
- terraform plan -> error (backend init required) -> terraform init -> yes to copy state
- .tfstate now in S3 bucket   [media/Markdown_Logo-3.gif]
- output.tf: s3_bucket_arn, dynamodb_table_name -> terraform apply

Isolation Of Environments: dev/sit/uat/preprod/prod via (1) Terraform Workspaces or (2) Directory-based separation using terraform.tfvars

REFACTOR YOUR PROJECT USING MODULES
modules/: ALB, Autoscaling, EFS, RDS, Security, VPC
each module: <resource_name>.tf, outputs.tf (optional), variables.tf
providers + backends configured in separate files in the ROOT module
Example: root main.tf module "VPC" { source = "./modules/VPC" ... } -> modules/VPC/main.tf resource aws_vpc
Pro-tips: terraform validate, terraform fmt



For reference
https://github.com/darey-devops/PBL-project-18.git
There are some .sh file that are empty

# AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-3/4
**Project 18 Terraform** 
*==Now we will explore alternative **Terraform** [backends](https://www.terraform.io/language/settings/backends/configuration).*  ==
 
### AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 3 – REFACTORING  

So far we have developed **AWS** Infrastructure code using Terraform and tried to run it from our local workstation.  
Now we will explore alternative **Terraform** [backends](https://www.terraform.io/language/settings/backends/configuration).  

**Introducing Backend on [S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)**   
Each **Terraform** configuration can specify a **backend**, which defines where and how operations are performed *(where state snapshots are stored, etc)*  

States file is basically where terraform stores all the state of the infrastructure in `json` format.  

So far, we have been using the default **backend**, which is the `local backend` – it requires no configuration, and the states file is stored locally. This mode it is not a robust solution, so it is better to store it in some more reliable and durable storage.  

The second problem with storing this file locally is that other engineers will not have access to a state file stored locally on your computer.  

To solve this, we will need to configure a backend where the state file can be accessed remotely by other DevOps team members. There are plenty of different standard **backends** supported by **Terraform** that we can choose from. Since we are already using **AWS** – we can choose an [S3 bucket as a backend](https://www.terraform.io/docs/language/settings/backends/s3.html).  

Another useful option that is supported by **S3** backend is [**State Locking**](https://www.terraform.io/docs/language/state/locking.html) *(used to lock your state for all operations that could write state)*. This prevents others from acquiring the lock and potentially corrupting your state. State Locking feature for S3 backend is optional and requires another AWS service – [DynamoDB](https://aws.amazon.com/dynamodb/).  


Steps to **Re-initialize** Terraform to use **S3 backend**: *(init terraform)*  
1. ###### Add S3 and DynamoDB resource blocks before deleting the local state file
2. ###### Update terraform block to introduce backend and locking
3. ###### Re-initialize terraform
4. ###### Delete the local tfstate file and check the one in S3 bucket
5. ###### Add outputs
6. ###### terraform apply  

#### Add S3 and DynamoDB resource blocks before deleting the local state file  
<!--
To get to know how lock in DynamoDB works, read the following article
https://angelo-malatacca83.medium.com/aws-terraform-s3-and-dynamodb-backend-3b28431a76c1
-->
1. Create a file and name it `backend.tf`.  
*(S3 Bucket should already exist from [Project 16](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PART1_PROJECT_16.md))*

``` bash
#must give it a unique name globally
resource "aws_s3_bucket" "terraform_state" {
  bucket = "hector-dev-terraform-bucket"
  # Enable versioning so we can see the full revision history of our state files
  versioning {
    enabled = true
  }

  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
```
Since **Terraform** stores secret data inside the state files. **Passwords**, and **secret keys** processed by resources are always stored in there. Hence, you must consider to always enable encryption. You can see how we achieved that with [`server_side_encryption_configuration`](https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html).


#### Update terraform block to introduce backend and locking

2. Next, we will create a **DynamoDB** table to handle locks and perform consistency checks. Previously, locks were handled with a local file `terraform.tfstate.lock.info`. Therefore, with a cloud storage database like **DynamoDB**, anyone running Terraform against the same infrastructure can use a central location to control a situation where Terraform is running at the same time from multiple different people.
   
``` bash
#Dynamo DB resource for locking and consistency checking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```
**Terraform** expects that both **S3 bucket** and **DynamoDB** resources are already created **before** we configure the **backend**. So, let us run `terraform apply` to provision resources.

3. Configure **S3 Backend**
``` bash
terraform {
  backend "s3" {
    bucket         = "hector-dev-terraform-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```
Now its time to **re-initialize** the **backend**. We run `terraform init` and confirm to change the backend by typing `yes`  

#### Re-initialize terraform
We run the `terraform plan` command again to get an updated list of resources, we will receive an error because the backend has changed and **Terraform** needs to start again  

<details close>
<summary>Terraform plan Output</summary>

``` bash
hector@hector-Laptop:~/Project16-17/PBL$ terraform plan
╷
│ Error: Backend initialization required, please run "terraform init"
│
│ Reason: Initial configuration of the requested backend "s3"
│
│ The "backend" is the interface that Terraform uses to store state,
│ perform operations, etc. If this message is showing up, it means that the
│ Terraform configuration you're using is using a custom configuration for
│ the Terraform backend.
│
│ Changes to backend configurations require reinitialization. This allows
│ Terraform to set up the new configuration, copy existing state, etc. Please run
│ "terraform init" with either the "-reconfigure" or "-migrate-state" flags to
│ use the current configuration.
│
│ If the change reason above is incorrect, please verify your configuration
│ hasn't changed and try again. At this point, no changes to your existing
│ configuration or state have been made.
╵
hector@hector-Laptop:~/Project16-17/PBL$
```
</details>


Let’s run the `terraform init` command again, Terraform will automatically detect that you already have a state file locally and prompt you to copy it to the **S3** previously created, writing `yes`, the file will be automatically copied  

<details close>
<summary>Terraform init Output</summary>

``` bash
hector@hector-Laptop:~/Project16-17/PBL$ terraform init

Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "s3" backend. No existing state was found in the newly
  configured "s3" backend. Do you want to copy this state to the new "s3"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value:
  Enter a value: yes

Releasing state lock. This may take a few moments...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Reusing previous version of hashicorp/random from the dependency lock file
- Using previously-installed hashicorp/aws v4.14.0
- Using previously-installed hashicorp/random v3.1.3

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
hector@hector-Laptop:~/Project16-17/PBL$d
```
</details>


#### Delete the local tfstate file and check the one in S3 bucket  

Now we can see the `.tfsate` file inside the **S3 Bucket**   
![Markdown Logo](media/Markdown_Logo-3.gif) 


Before we run `terraform apply` let us add an output so that the **S3 bucket** Amazon Resource Names (**ARN**) and **DynamoDB** table name can be displayed.  

Create a new file `output.tf` with the following code.
``` bash
output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
  description = "The ARN of the S3 bucket"
}
output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "The name of the DynamoDB table"
}
```

Now we can run `terraform apply`  


**Isolation Of Environments:**  
Most likely we will need to create resources for different environments, such as: dev, sit, uat, preprod, prod, etc.  

This **separation** of environments can be achieved using one of two methods:  
1. Terraform Workspaces  
2. **Directory** based separation using `terraform.tfvars`  

<!-- ### WHEN TO USE WORKSPACES OR DIRECTORY? -->


### REFACTOR YOUR PROJECT USING MODULES

It is difficult to navigate through all the Terraform blocks if they are all written in a single long `.tf `file. We must strive to produce reusable and comprehensive **IaC** code structure, and one of the tool that Terraform provides out of the box is [**Modules**](https://www.terraform.io/docs/language/modules/index.html).  

Modules serve as containers that allow to logically group Terraform code for similar resources in the same domain *(e.g., Compute, Networking, AMI, etc)*. One **root module** can call other **child modules** and insert their configurations when applying Terraform config. This concept makes your code structure neater, and it allows different team members to work on different parts of configuration at the same time. We can also create and publish out modules to [Terraform Registry](https://registry.terraform.io/browse/modules) for others to use and use someone’s modules in your projects.

Module is just a collection of `.tf` and/or` .tf.json` files in a directory.  

So far in [Project 17](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PART2_PROJECT_17.md) we have a single list of long files for creating all of our resources  

We are going combine resources of a similar type into **directories** within a `modules` directory  

├── `modules`  
    ├── `ALB` *for Application Load balancer and similar resources*  
    ├── `Autoscaling` *for Autoscaling and launch template resources*    
    ├── `EFS` *for Elastic file system resources*      
    ├── `RDS` *for Databases resources*      
    ├── `Security` *for creating security group resources*      
    └── `VPC` *for VPC and networking resources such as subnets, roles, etc..*      


Each module should contain the following files:  
* `<resource_name>.tf` *file(s) with resources blocks*
* `outputs.tf` *optional, when we need to refer outputs from any of these resources in our root module*
* `variables.tf` *as we learned before - it is a good practice not to hard code the values and use variables*  


It is also recommended to configure `providers` and `backends` sections in separate files but should be placed in the root module.  
├── `backend.tf`  
├── `providers.tf`  
├── main.tf  
├── modules  
    ├── ALB  
    ├── Autoscaling  
    ├── EFS  
    ├── RDS  
    ├── Security  
    └── VPC  


### COMPLETE THE TERRAFORM CONFIGURATION


Complete the rest of the codes yourself, the resulting configuration structure in your working directory may look like this:


<details close>
<summary>Configuration Structure</summary>

`hector@hector-Laptop:~/Project16-17/PBL$ tree`  
.   
├── [backend.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/backend.tf)  
├── [main.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/main.tf)  
├── [modules](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules)  
│   ├── [ALB](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/tree/main/PBL/modules/ALB)  
│   │   ├── [alb.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/ALB/alb.tf)  
│   │   ├── [cert.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/ALB/cert.tf)  
│   │   ├── [output.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/ALB/output.tf)  
│   │   └── [variables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/ALB/variables.tf)  
│   ├── [Autoscaling](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/tree/main/PBL/modules/Autoscaling)  
│   │   ├── [asg-bastion-nginx.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/asg-bastion-nginx.tf)  
│   │   ├── [asg-wordpress-tooling.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/asg-wordpress-tooling.tf)  
│   │   ├── [bastion.sh](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/bastion.sh)  
│   │   ├── [lt-bastion-nginx.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/lt-bastion-nginx.tf)  
│   │   ├── [lt-tooling-wp.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/lt-tooling-wp.tf)  
│   │   ├── [nginx.sh](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/nginx.sh)  
│   │   ├── [tooling.sh](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/tooling.sh)  
│   │   ├── [variables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/variables.tf)  
│   │   └── [wordpress.sh](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Autoscaling/wordpress.sh)  
│   ├── [EFS](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/tree/main/PBL/modules/EFS)  
│   │   ├── [efs.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/EFS/efs.tf)  
│   │   ├── [output.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/EFS/output.tf)  
│   │   └── [variables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/EFS/variables.tf)  
│   ├── [RDS](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/tree/main/PBL/modules/RDS)  
│   │   ├── [output.tf]()  
│   │   ├── [rds.tf]()  
│   │   └── [variables.tf]()  
│   ├── [Security](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/tree/main/PBL/modules/Security)  
│   │   ├── [main.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Security/main.tf)  
│   │   ├── [outputs.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Security/outputs.tf)  
│   │   ├── [security.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Security/security.tf)  
│   │   ├── [sg-rule.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Security/sg-rule.tf)  
│   │   └── [variables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/Security/variables.tf)  
│   └── [VPC](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/tree/main/PBL/modules/VPC)  
│       ├── [internet_gateway.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/internet_gateway.tf)  
│       ├── [main.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/main.tf)  
│       ├── [natgateway.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/natgateway.tf)  
│       ├── [outputs.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/outputs.tf)  
│       ├── [roles.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/roles.tf)  
│       ├── [route_tables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/route_tables.tf)  
│       └── [variables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/variables.tf)  
├── [providers.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/providers.tf)  
├── terraform.tfstate    
├── terraform.tfstate.backup  
├── [terraform.tfvars](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/terraform.tfvars)  
└── [variables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/variables.tf)  

`7 directories, 38 files`  
</details>

**Above Principles in action:**  
I'm going to pick the creation of the VPC as an example  

In our root directory `PBL` we have `main.tf` where we create the **module**
``` bash
# creating VPC
module "VPC" {
  source                              = "./modules/VPC"
  region                              = var.region
  vpc_cidr                            = var.vpc_cidr
  enable_dns_support                  = var.enable_dns_support
  enable_dns_hostnames                = var.enable_dns_hostnames
  enable_classiclink                  = var.enable_classiclink
  preferred_number_of_public_subnets  = var.preferred_number_of_public_subnets
  preferred_number_of_private_subnets = var.preferred_number_of_private_subnets
  private_subnets                     = [for i in range(1, 8, 2) : cidrsubnet(var.vpc_cidr, 8, i)]
  public_subnets                      = [for i in range(2, 5, 2) : cidrsubnet(var.vpc_cidr, 8, i)]
}
```
We have no coded values. We either generate it (subnets) or pass variables using `var` keyword. The **values** of these variables are defined in [PBL/`terraform.tfvars`](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/terraform.tfvars)

The **VPC module** defines a `source = "./modules/VPC"`  

There we find [`PBL/modules/VPC/main.tf`](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/main.tf) where we create the resource **VPC**  
``` bash
# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support
  enable_dns_hostnames           = var.enable_dns_hostnames
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink_dns_support
  tags = merge(
    var.tags,
    {
      Name = format("%s-VPC", var.name)
    },
  )
}
```
Once again we used **variables**, variables used within this **module** *(VPC)* need to be defined in its [PBL/modules/VPC/`variables.tf`](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/variables.tf)  

Once again the values of these variables originate from [PBL/`terraform.tfvars`](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/terraform.tfvars)  

**Pro-tips:**  
1. You can validate your codes before running `terraform plan` with `terraform validate` command. It will check if your code is syntactically valid and internally consistent.  
2. In order to make your configuration files more readable and follow canonical format and style – use `terraform fmt` command. It will apply Terraform language style conventions and format your `.tf `files in accordance to them  


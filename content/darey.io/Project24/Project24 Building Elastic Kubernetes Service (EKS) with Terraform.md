---
tags:
  - draft
  - Kubernetes
  - helm
  - EKS
linkedin: "False"
quartz: "False"
refactored: "False"
darey.io: "True"
hands-on: "True"
completed: "True"
title: Building Elastic Kubernetes Service (EKS) with Terraform
hardlinked: "True"
---
Original [Project 24](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM)
## Overview

In this project I provisioned a production-grade **Elastic Kubernetes Service (EKS)** cluster on AWS using Terraform. Rather than clicking through the AWS console, the goal was to define the entire infrastructure as code, networking, state management, and the EKS cluster itself, so it is repeatable, version-controlled, and easy to tear down and rebuild.

This lab is self-contained with one exception: the remote state backend pattern (S3 bucket + DynamoDB lock table) is borrowed from **[Project 18](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PART3_PROJECT_18.md)**, where that pattern was first introduced. All networking, EKS configuration, and variable management is built fresh here.

The project is split into two parts:
- **Part 1** - Bootstrap remote state storage (S3 + DynamoDB) and configure the VPC/networking layer.
- **Part 2** - Define and deploy the EKS cluster using the official Terraform EKS module.
<!--**GitHub Repository:** [BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM)
-->

---

## Prerequisites

- AWS CLI installed and configured with appropriate IAM permissions
- Terraform installed (v1.x recommended)
- An AWS account with access to create EKS, VPC, S3, and DynamoDB resources
- Familiarity with Terraform remote state, the S3 bucket and DynamoDB table naming conventions used here (`hector-dev-terraform-bucket`, `terraform-locks`) follow the pattern established in [Project 18 (Automate Infrastructure with IaC Using Terraform)](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PART3_PROJECT_18.md). That is the only carry-over from prior projects; everything else in this lab is built from scratch.
- General familiarity with Kubernetes concepts (pods, nodes, clusters) is helpful but this project does not depend on any prior Kubernetes hands-on work

---

## Part 1 - Remote State Backend & Networking

### Why Use a Remote Backend?

By default Terraform stores state locally in a `terraform.tfstate` file. This becomes a problem in team environments or when working across multiple machines, there is no locking mechanism to prevent two applies from running simultaneously, and state can easily go out of sync.

The solution is to store state remotely in **S3** with state locking provided by **DynamoDB**. Before the backend can be configured in Terraform, the S3 bucket and DynamoDB table must already exist. This creates a bootstrapping challenge: Terraform needs to create resources that Terraform itself will later depend on.

The approach is to run Terraform **twice**:
1. First, with the backend configuration excluded, provision the S3 bucket and DynamoDB table using local state.
2. Then, enable the backend configuration and re-run `terraform init` to migrate state to S3.

---

### Step 1 - Bootstrap the S3 Bucket and DynamoDB Table

A new working directory called `eks/` was created. The backend configuration was placed in a file named `backend.tfX` (the `.tfX` extension tells Terraform to ignore it) so it would not interfere during the first apply.

**Initial directory structure:**

```
hector@hector-Laptop:~/Project24/eks$ tree
.
├── backend.tfX  ← temporarily excluded from Terraform
├── main.tf
└── providers.tf

0 directories, 3 files
```

Running `terraform init` at this point initializes the project with a **local** backend and downloads the AWS provider:

```bash
hector@hector-Laptop:~/Project24/eks$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 4.0"...
- Installing hashicorp/aws v4.26.0...
- Installed hashicorp/aws v4.26.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

Running `terraform apply` provisions the four required resources, S3 bucket, bucket versioning, server-side encryption, and the DynamoDB table:

```
Plan: 4 to add, 0 to change, 0 to destroy.

  Enter a value: yes

aws_dynamodb_table.terraform_locks: Creating...
aws_s3_bucket.terraform-state: Creating...
aws_s3_bucket.terraform-state: Creation complete after 2s [id=hector-dev-terraform-bucket]
aws_s3_bucket_versioning.version: Creating...
aws_s3_bucket_server_side_encryption_configuration.first: Creating...
aws_s3_bucket_server_side_encryption_configuration.first: Creation complete after 0s [id=hector-dev-terraform-bucket]
aws_s3_bucket_versioning.version: Creation complete after 1s [id=hector-dev-terraform-bucket]
aws_dynamodb_table.terraform_locks: Still creating... [10s elapsed]
aws_dynamodb_table.terraform_locks: Creation complete after 14s [id=terraform-locks]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

---

### Step 2 - Enable the S3 Backend

With the S3 bucket and DynamoDB table in place, the backend configuration was activated by renaming `backend.tfX` back to `backend.tf`:

```yaml
# backend.tf
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
<!--The `key` is simply the **file path inside the S3 bucket** where Terraform will store the state file. Creates the paths-->
```bash
hector@hector-Laptop:~/Project24/eks$ mv backend.tfX backend.tf
```

Running `terraform init` again detects the new backend and offers to **migrate the existing local state** to S3:

```bash
hector@hector-Laptop:~/Project24/eks$ terraform init

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

Terraform has been successfully initialized!
```

State is now stored remotely and all subsequent applies benefit from **S3 versioning** and **DynamoDB locking**.

#### Troubleshooting - Backend Initialization Required Error

After renaming `backend.tfX` to `backend.tf`, running `terraform plan` before re-running `terraform init` produces the following error:

```
│ Error: Backend initialization required, please run "terraform init"
│
│ Reason: Initial configuration of the requested backend "s3"
│
│ Changes to backend configurations require reinitialization. This allows
│ Terraform to set up the new configuration, copy existing state, etc. Please run
│ "terraform init" with either the "-reconfigure" or "-migrate-state" flags to
│ use the current configuration.
```

**Resolution:** Any time the backend block is added or changed, `terraform init` must be re-run before any other Terraform commands. It is not optional even if everything else is unchanged.

---

### Step 3 - Network Configuration

A `network.tf` file was created to define the VPC and subnets. This includes:
- A **VPC** with a configurable CIDR block
- **Private and public subnets** spread across availability zones
- An **Elastic IP** and **NAT Gateway** so private subnet workloads can reach the internet

This networking layer is a prerequisite for the EKS cluster, nodes in private subnets need outbound internet access to pull container images and communicate with AWS services.

---

## Part 2 - EKS Cluster Provisioning

### Configuration Files

The following files were created to define the EKS cluster:
<!--
https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/main/eks/eks.tf
-->

| File               | Purpose                                                                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `eks.tf`           | Calls the [terraform-aws-modules/eks](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest) module to provision the cluster and managed node groups |
| `locals.tf`        | Defines reusable local values shared across configurations                                                                                                                |
| `variables.tf`     | Declares all input variables (cluster name, instance types, scaling limits, etc.)                                                                                         |
| `terraform.tfvars` | Supplies concrete values for all declared variables (auto-loaded by Terraform)                                                                                            |
| `data.tf`          | Data sources, queries available AZs and the current AWS account identity                                                                                                  |

The additional input variables added to `variables.tf` cover cluster scaling, instance types, and user access:

```yaml
# variables.tf (additions)
variable "admin_users" {
  type        = list(string)
  description = "List of Kubernetes admins."
}
variable "developer_users" {
  type        = list(string)
  description = "List of Kubernetes developers."
}
variable "asg_instance_types" {
  type        = list(string)
  description = "List of EC2 instance machine types to be used in EKS."
}
variable "autoscaling_minimum_size_by_az" {
  type        = number
  description = "Minimum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_maximum_size_by_az" {
  type        = number
  description = "Maximum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_average_cpu" {
  type        = number
  description = "Average CPU threshold to autoscale EKS EC2 instances."
}
```

Variable values were customized for this environment. Note the `name_prefix` and `admin_users` were updated from the lab template to reflect personal AWS IAM users:

```yaml
# terraform.tfvars (customized values)
cluster_name                   = "tooling-app-eks"
iac_environment_tag            = "development"
name_prefix                    = "hector-eks"          
main_network_block             = "10.0.0.0/16"
subnet_prefix_extension        = 4
zone_offset                    = 8
admin_users                    = ["hector", "solomon"]
developer_users                = ["leke", "david"]
asg_instance_types             = ["t3.small", "t2.small"]
autoscaling_minimum_size_by_az = 1
autoscaling_maximum_size_by_az = 10
autoscaling_average_cpu        = 30
```

> **Note:** The `admin_users` and `developer_users` values must correspond to **existing IAM users** in your AWS account. An alternative approach is to manage users in a dedicated `iam.tf` file and reference them via data source ARN interpolation.

---

### Step 4 - Initialize Modules and Deploy

Because `eks.tf` and `network.tf` reference external Terraform registry modules, `terraform init` needed to be run again to download them:

```bash
hector@hector-Laptop:~/Project24/eks$ terraform init

Initializing modules...
Downloading registry.terraform.io/terraform-aws-modules/eks/aws 18.27.1 for eks_cluster...
- eks_cluster in .terraform/modules/eks_cluster
- eks_cluster.eks_managed_node_group in .terraform/modules/eks_cluster/modules/eks-managed-node-group
Downloading registry.terraform.io/terraform-aws-modules/vpc/aws 3.14.2 for vpc...
- vpc in .terraform/modules/vpc

Initializing the backend...

Initializing provider plugins...
- Installing hashicorp/cloudinit v2.2.0...
- Installing hashicorp/kubernetes v2.12.1...
- Installing hashicorp/tls v3.4.0...

Terraform has been successfully initialized!
```

#### Troubleshooting - Module Not Installed Error

Before running `terraform init` a second time, attempting `terraform plan` produced this error:

```
│ Error: Module not installed
│
│   on eks.tf line 1:
│    1: module "eks_cluster" {
│
│ This module is not yet installed. Run "terraform init" to install all modules
│ required by this configuration.

│ Error: Module not installed
│
│   on network.tf line 11:
│   11: module "vpc" {
│
│ This module is not yet installed. Run "terraform init" to install all modules
│ required by this configuration.
```

**Resolution:** Whenever new `module` blocks are added to any `.tf` file, `terraform init` must be run to download them before `terraform plan` or `terraform apply` will work.

---

#### Troubleshooting - Terraform Prompting for Variable Values Interactively

After the first `terraform plan`, Terraform stopped and asked for every variable to be typed in by hand:

```bash
hector@hector-Laptop:~/Project24/eks$ terraform plan
var.admin_users
  List of Kubernetes admins.
  Enter a value:

var.asg_instance_types
  List of EC2 instance machine types to be used in EKS.
  Enter a value:

var.cluster_name
  EKS cluster name.
  Enter a value:
# ... and so on for every variable
```

**Root Cause:** Variable values were stored in a file named `variables.tfvars`. Terraform does **not** automatically load `.tfvars` files with **custom names**, it only auto-loads files named exactly `terraform.tfvars` or matching the pattern `*.auto.tfvars`.

**Resolution:** The full contents of `variables.tfvars` were moved into `terraform.tfvars` (which Terraform loads automatically), and the original file was disabled by renaming it:

```yaml
# contents moved from variables.tfvars → terraform.tfvars
cluster_name                   = "tooling-app-eks"
iac_environment_tag            = "development"
name_prefix                    = "hector-eks"
main_network_block             = "10.0.0.0/16"
subnet_prefix_extension        = 4
zone_offset                    = 8
admin_users                    = ["hector", "solomon"]
developer_users                = ["leke", "david"]
asg_instance_types             = ["t3.small", "t2.small"]
autoscaling_minimum_size_by_az = 1
autoscaling_maximum_size_by_az = 10
autoscaling_average_cpu        = 30
```

```bash
hector@hector-Laptop:~/Project24/eks$ mv variables.tfvars variables.tfvarsX
```

After that, `terraform plan` ran cleanly without prompting for any values.
<!--
seems like
initially there was a variables.tfvars file aim to suppliesvalues to he var definitions in variables.tf, but we wanted to not get prompeted for variables input so we need a way to upload the input automatically there fore renaming it to terraform.tfvars?
-->
---

#### Troubleshooting - EKS Cluster Fails Due to Availability Zone Capacity

When attempting to deploy the cluster, `terraform apply` failed with the following error:

```
│ Error: error creating EKS Cluster (tooling-app-eks):
│ UnsupportedAvailabilityZoneException: Cannot create cluster 'tooling-app-eks'
│ because us-east-1e, the targeted availability zone, does not currently have
│ sufficient capacity to support the cluster.
│ Retry and choose from these availability zones:
│ us-east-1a, us-east-1b, us-east-1c, us-east-1d, us-east-1f
```

**Root Cause:** The `data "aws_availability_zones"` data source in `data.tf` was returning all AZs including `us-east-1e`, which had insufficient EKS capacity at the time. The VPC subnets were being created across all returned AZs, and EKS tried to deploy into `us-east-1e`.

**Attempts to fix this in `data.tf`:**

*Attempt 1 - `skip_names` attribute (not a valid argument):*
```yaml
data "aws_availability_zones" "available_azs" {
  state      = "available"
  skip_names = [us-east-1e]  # ← Error: Unsupported argument
}
```

*Attempt 2 - `names` attribute (read-only, cannot be set):*
```yaml
data "aws_availability_zones" "available_azs" {
  state = "available"
  names = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
  # ← Error: Value for unconfigurable attribute
}
```

*Attempt 3 - `filter` with `name = "names"` (not a valid EC2 filter):*
```yaml
filter {
  name   = "names"
  values = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
}
# ← Error: The filter 'name' is invalid
```

**Resolution:** Using the correct EC2 filter key `zone-name` as documented in the [AWS EC2 API Reference](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeAvailabilityZones.html) and the [Terraform `aws_availability_zones` filter block docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones#filter-configuration-block):

```yaml
# data.tf
data "aws_availability_zones" "available_azs" {
  state = "available"
  filter {
    name   = "zone-name"
    values = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
  }
}

data "aws_caller_identity" "current" {} # used for accessing Account ID and ARN
```

The error message itself was the key clue, it said `The filter 'name' is invalid`, which indicated that `name` inside the filter block refers to an EC2 API filter key (like `zone-name`), not a Terraform argument. Looking up the [Terraform `filter` Configuration Block](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones#filter-configuration-block) pointed directly to the AWS API docs listing all valid filter keys.

---

## Summary

| Phase | What Was Accomplished |
|---|---|
| Bootstrap | S3 bucket + DynamoDB table provisioned for remote state |
| Backend Migration | Local state migrated to S3; locking enabled via DynamoDB |
| Networking | VPC, private/public subnets, NAT Gateway defined in `network.tf` |
| EKS Modules | `eks.tf`, `locals.tf`, `variables.tf`, `terraform.tfvars` configured |
| Cluster Deploy | EKS cluster and managed node groups successfully provisioned |

The biggest learning in this project was understanding Terraform's bootstrapping order, you cannot configure a backend in the same apply that creates it. The AZ filtering issue also reinforced that Terraform data source `filter` blocks map directly to AWS API filter parameters, so the Terraform docs and AWS API docs must be read together to use them correctly.



<!--
# BUILDING ELASTIC KUBERNETES SERVICE (EKS) WITH TERRAFORM

[https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PART3_PROJECT_18.md#add-s3-and-dynamodb-resource-blocks-before-deleting-the-local-state-file-1](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PART3_PROJECT_18.md#add-s3-and-dynamodb-resource-blocks-before-deleting-the-local-state-file-1)

1. Open up a new directory on your laptop, and name it eks
2. Use AWS
3.  CLI to create an S3 bucket

4. Create a file – backend.tf Task for you, ensure the backend is configured for remote state in S3

```
#Terraform expects that both S3 bucket and DynamoDB resources are already created before we configure the backend. So, let us run terraform apply to provision resources.

#Configure S3 Backend

terraform {

  backend "s3" {

    bucket         = "hector-dev-terraform-bucket"

    key            = "global/s3/terraform.tfstate"

    region         = "us-east-1"

    dynamodb_table = "terraform-locks"

    encrypt        = true

  }

}
```

So first Im going to do a first terraform init without the backend so I can have everything ready for the s3, dynamo for the backned

```
hector@hector-Laptop:~/Project24/eks$ tree
.
├── backend.tfX  renamed to exclude
├── main.tf
└── providers.tf
0 directories, 3 files
```

```
hector@hector-Laptop:~/Project24/eks$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 4.0"...
- Installing hashicorp/aws v4.26.0...
- Installed hashicorp/aws v4.26.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
hector@hector-Laptop:~/Project24/eks$ 
```


==terraform apply (we needed a variables fiel with region variable)==

Part of the Output:
```
…
…
…
Plan: 4 to add, 0 to change, 0 to destroy.
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_dynamodb_table.terraform_locks: Creating...
aws_s3_bucket.terraform-state: Creating...
aws_s3_bucket.terraform-state: Creation complete after 2s [id=hector-dev-terraform-bucket]
aws_s3_bucket_versioning.version: Creating...
aws_s3_bucket_server_side_encryption_configuration.first: Creating...
aws_s3_bucket_server_side_encryption_configuration.first: Creation complete after 0s [id=hector-dev-terraform-bucket]
aws_s3_bucket_versioning.version: Creation complete after 1s [id=hector-dev-terraform-bucket]
aws_dynamodb_table.terraform_locks: Still creating... [10s elapsed]
aws_dynamodb_table.terraform_locks: Creation complete after 14s [id=terraform-locks]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
hector@hector-Laptop:~/Project24/eks$

```

<mark style="background: #ABF7F7A6;">Doing the backend</mark>

```
hector@hector-Laptop:~/Project24/eks$ mv backend.tfX backend.tf
hector@hector-Laptop:~/Project24/eks$ terraform plan
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
```

```
hector@hector-Laptop:~/Project24/eks$ terraform init

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
- Using previously-installed hashicorp/aws v4.26.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
hector@hector-Laptop:~/Project24/eks$
```

Create a file – [network.tf](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/main/eks/network.tf) and provision Elastic IP for Nat Gateway, VPC, Private and public subnets.

# BUILDING ELASTIC KUBERNETES SERVICE (EKS) WITH TERRAFORM – PART 2

Create a file – eks.tf and provision EKS cluster

Create a file – locals.tf to create local variables.

[[Add more variables to the variables.tf file]]

[[Create a file – variables.tfvars to set values for variables.]]

```bash
word replace:
name_prefix             = "darey-io-eks"
admin_users                    = ["darey", "solomon"]

name_prefix             = "hector-eks"
admin_users                    = ["hector", "solomon"]
```

[variables.tfvars1](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/7ffac4396d11ef1b42f8ed8bc3db7bee6e8cfa42/eks/variables.tfvars)

~~Create file – provider.tf~~

We already created this file to do the first  provisioning of s3


[variables.tfvars2](https://www.darey.io/docs/building-elastic-kubernetes-service-eks-with-terraform/#:~:text=COPY-,Create%20a%20file%20%E2%80%93,asg_instance_types%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%3D%20%5B%22t3.small%22%2C%20%22t2.small%22%5D%0Aautoscaling_minimum_size_by_az%20%20%20%20%20%20%20%20%20%20%20%3D%201%0Aautoscaling_maximum_size_by_az%20%20%20%20%20%20%20%20%20%20%20%3D%2010%0Aautoscaling_average_cpu%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%3D%2030,-Run)
Link to Darey.io


Run terraform init
~~We are already did this before~~
We had to do it again because we got a message about not having a module

```bash
hector@hector-Laptop:~/Project24/eks$ terraform plan
Releasing state lock. This may take a few moments...
╷
│ Error: Module not installed
│
│   on eks.tf line 1:
│    1: module "eks_cluster" {
│
│ This module is not yet installed. Run "terraform init" to install all modules required by this configuration.
╵
╷
│ Error: Module not installed
│
│   on network.tf line 11:
│   11: module "vpc" {
│
│ This module is not yet installed. Run "terraform init" to install all modules required by this configuration.
```

```bash
hector@hector-Laptop:~/Project24/eks$ terraform init
Initializing modules...
Downloading registry.terraform.io/terraform-aws-modules/eks/aws 18.27.1 for eks_cluster...
- eks_cluster in .terraform/modules/eks_cluster
- eks_cluster.eks_managed_node_group in .terraform/modules/eks_cluster/modules/eks-managed-node-group
- eks_cluster.eks_managed_node_group.user_data in .terraform/modules/eks_cluster/modules/_user_data
- eks_cluster.fargate_profile in .terraform/modules/eks_cluster/modules/fargate-profile
Downloading registry.terraform.io/terraform-aws-modules/kms/aws 1.0.2 for eks_cluster.kms...
- eks_cluster.kms in .terraform/modules/eks_cluster.kms
- eks_cluster.self_managed_node_group in .terraform/modules/eks_cluster/modules/self-managed-node-group
- eks_cluster.self_managed_node_group.user_data in .terraform/modules/eks_cluster/modules/_user_data
Downloading registry.terraform.io/terraform-aws-modules/vpc/aws 3.14.2 for vpc...
- vpc in .terraform/modules/vpc

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/cloudinit versions matching ">= 2.0.0"...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Finding hashicorp/kubernetes versions matching ">= 2.10.0"...
- Finding hashicorp/tls versions matching "~> 3.0"...
- Installing hashicorp/cloudinit v2.2.0...
- Installed hashicorp/cloudinit v2.2.0 (signed by HashiCorp)
- Using previously-installed hashicorp/aws v4.26.0
- Installing hashicorp/kubernetes v2.12.1...
- Installed hashicorp/kubernetes v2.12.1 (signed by HashiCorp)
- Installing hashicorp/tls v3.4.0...
- Installed hashicorp/tls v3.4.0 (signed by HashiCorp)

Terraform has made some changes to the provider dependency selections recorded
in the .terraform.lock.hcl file. Review those changes and commit them to your
version control system if they represent changes you intended to make.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
hector@hector-Laptop:~/Project24/eks$

```

So up until [this check in "First commit"](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/commit/7ffac4396d11ef1b42f8ed8bc3db7bee6e8cfa42) we get asked to input variables ^30924d

```bash
hector@hector-Laptop:~/Project24/eks$ terraform plan
var.admin_users
  List of Kubernetes admins.
  Enter a value:
var.asg_instance_types
  List of EC2 instance machine types to be used in EKS.
  Enter a value:
var.autoscaling_maximum_size_by_az
  Maximum number of EC2 instances to autoscale our EKS cluster on each AZ.
  Enter a value:
var.autoscaling_minimum_size_by_az
  Minimum number of EC2 instances to autoscale our EKS cluster on each AZ.
  Enter a value:
var.cluster_name
  EKS cluster name.
  Enter a value:
var.developer_users
  List of Kubernetes developers.
  Enter a value:
var.iac_environment_tag
  AWS tag to indicate environment name of each infrastructure object.
  Enter a value:
var.main_network_block
  Base CIDR block to be used in our VPC.
  Enter a value:
var.name_prefix
  Prefix to be used on each infrastructure object Name created in AWS.
  Enter a value:
var.subnet_prefix_extension
  CIDR block bits extension to calculate CIDR blocks of each subnetwork.
  Enter a value:
var.zone_offset
  CIDR block bits extension offset to calculate Public subnets, avoiding collisions with Private subnets.
  Enter a value:
```


> [[Project24#^30924d|Solution]]: (to us getting asked to input information manually)
> 
> All these values are in [variables.tfvars1](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/7ffac4396d11ef1b42f8ed8bc3db7bee6e8cfa42/eks/variables.tfvars) as per instructions I think we have to put them in [terraform.tfvars](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/7ffac4396d11ef1b42f8ed8bc3db7bee6e8cfa42/eks/terraform.tfvars)
> 
> So I'm moving everything from [variables.tfvars1](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/7ffac4396d11ef1b42f8ed8bc3db7bee6e8cfa42/eks/variables.tfvars) to [terraform.tfvars](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/7ffac4396d11ef1b42f8ed8bc3db7bee6e8cfa42/eks/terraform.tfvars) and disabling  [variables.tfvars1](https://github.com/hectorproko/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/blob/7ffac4396d11ef1b42f8ed8bc3db7bee6e8cfa42/eks/variables.tfvars) by renaming it
> 
> ```
> hector@hector-Laptop:~/Project24/eks$ mv variables.tfvars variables.tfvarsX
> ```
> <mark style="background: #BBFABBA6;">WORKED</mark>
> We tested with `terraform plan`


**Issue:**

```
	│ Error: error creating EKS Cluster (tooling-app-eks): UnsupportedAvailabilityZoneException: Cannot create cluster 'tooling-app-eks' because us-east-1e, the targeted availability zone, does not currently have sufficient capacity to support the cluster. Retry and choose from these availability zones: us-east-1a, us-east-1b, us-east-1c, us-east-1d, us-east-1f
	│ {
	│   RespMetadata: {
	│     StatusCode: 400,
	│     RequestID: "84bb50b9-9184-456b-bf10-aba13fe5369a"
	│   },
	│   ClusterName: "tooling-app-eks",
	│   Message_: "Cannot create cluster 'tooling-app-eks' because us-east-1e, the targeted availability zone, does not currently have sufficient capacity to support the cluster. Retry and choose from these availability zones: us-east-1a, us-east-1b, us-east-1c, us-east-1d, us-east-1f",
	│   ValidZones: [
	│     "us-east-1a",
	│     "us-east-1b",
	│     "us-east-1c",
	│     "us-east-1d",
	│     "us-east-1f"
	│   ]
	│ }
	│
	│   with module.eks_cluster.aws_eks_cluster.this[0],
	│   on .terraform/modules/eks_cluster/main.tf line 14, in resource "aws_eks_cluster" "this":
	│   14: resource "aws_eks_cluster" "this" {
	│
	╵
	Releasing state lock. This may take a few moments...
	hector@hector-Laptop:~/Project24/eks$
```

**Solution:**
It seems like something in the creation of the VPC

Try:
```
hector@hector-Laptop:~/Project24/eks$ bat data.tf
───────┬─────────────────────────────────────────────────────────────────────────────────────────────
       │ File: data.tf
───────┼─────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # get all available AZs in our region
   2   │ data "aws_availability_zones" "available_azs" {
   3   │     state = "available"
   4 + │     skip_names = [us-east-1e]
   5   │ }
   6   │ data "aws_caller_identity" "current" {} # used for accesing Account ID and ARN
───────┴─────────────────────────────────────────────────────────────────────────────────────────────
```

> [!error]
>  Error: Unsupported argument

Try:
```
hector@hector-Laptop:~/Project24/eks$ bat data.tf
───────┬─────────────────────────────────────────────────────────────────────────────────────────────
       │ File: data.tf
───────┼─────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # get all available AZs in our region
   2   │ data "aws_availability_zones" "available_azs" {
   3   │     state = "available"
   4 + │     names = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
   5   │ }
   6 ~ │ data "aws_caller_identity" "current" {} # used for accesing Account ID and ARN
   7 ~ │
───────┴─────────────────────────────────────────────────────────────────────────────────────────────
hector@hector-Laptop:~/Project24/eks$
```

> [!error]
> Error: Value for unconfigurable attribute
> Can't configure a value for "names": its value will be decided automatically based on the result of applying this configuration.

> [!error]
> Error: Error fetching Availability Zones: InvalidParameterValue: The filter 'name' is invalid  (or names)

```
filter {
  name   = "names"
  values = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
}
```


> [!done] Solution
> ```
> filter {
>   name   = "zone-name"
>   values = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
> }
> ```


> So [this error] let me know the keyword next to name is a parameter. Then in Terraform Documentation I found name in the [filter Configuration Block](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones#filter-configuration-block) of [aws_availability_zones](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones#filter-configuration-block) which led me to aws documenation where all parameter are listed and I found which one I needed to use [zone-name](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeAvailabilityZones.html#:~:text=local%2Dzone.-,zone%2Dname,-%2D%20The%20name%20of)




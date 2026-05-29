---
tags:
  - "#Terraform"
  - IaC
hardlinked: "True"
linkedin: "False"
quartz: "False"
refactored: "False"
darey.io: "True"
hands-on: "True"
completed: "True"
title: "Automating AWS Infrastructure with Terraform - Part 1: Getting Started"
---

> **Part:** 1 of 3  
> **Topics:** IAM Setup · AWS CLI · S3 · Terraform Installation · VPC · Subnets · Variables · Refactoring

---

## **Overview**

In previous projects [[Project15 AWS CLOUD SOLUTION FOR 2 COMPANY WEBSITES USING A REVERSE PROXY TECHNOLOGY|AWS Solution for 2 Company Websites using a Reverse Proxy]] I built AWS infrastructure for two websites manually through the console. The goal of this series is to automate that exact same setup using **Terraform**, an Infrastructure as Code (IaC) tool that lets you define, provision, and manage cloud resources through code instead of clicking through a UI.

This first article covers the foundational setup: creating a dedicated IAM user for Terraform, configuring the AWS CLI, creating an S3 bucket that will later be used as a remote backend in **Part 3**, installing Terraform, and writing the first configuration files to provision a VPC and public subnets. Along the way I refactor the code to eliminate hard-coded values and introduce variables, a critical best practice for maintainable infrastructure code.

---

## Prerequisites

- An AWS account
- `python3` and `pip` installed locally
- Basic familiarity with the Linux terminal

---

## Step 1 - Create a Dedicated IAM User for Terraform

Rather than using a root account or a personal IAM user, I create a dedicated `terraform` IAM user with **programmatic-only access**. This keeps Terraform operations isolated and auditable.

**Navigate to:** IAM → Users → Add Users

|Step|Setting|Value|
|---|---|---|
|1|User name|`terraform`|
|1|Credential type|Access key – Programmatic access|
|2|Group name|`terraform`|
|2|Policy|`AdministratorAccess`|
|3|Tag|Name = `terraform`|

> ⚠️ **Important:** At the final step of the IAM wizard (**Review & Download**), save the **Access Key ID** and **Secret Access Key**, these are only shown once.

The result is two users visible in IAM: my personal user and the new `terraform` service user.

![[Markdown_Logo-3.png]]



---

## Step 2 - Install and Verify Dependencies
<!--
Removed this is only used as verification of credentials down the line when we could just do aws cli s3 list or something

### boto3 (AWS Python SDK)

I need `boto3` to programmatically verify access to AWS resources from Python.

```bash
pip install boto3
```

Verifying it's installed:

```bash
hector@hector-Laptop:~$ pip list | grep boto3
boto3                        1.17.112
```

-->

### AWS CLI

I'll use the AWS CLI to authenticate the `terraform` IAM user locally.

```bash
hector@hector-Laptop:~$ aws --version
aws-cli/1.22.71 Python/3.8.10 Linux/5.4.0-109-generic botocore/1.24.16
```
<!--
**`botocore`** is the low-level foundation library that handles all the raw AWS API communication — building HTTP requests, signing them with AWS credentials, parsing responses, and handling errors.

**`boto3`** is built on top of `botocore`. It's the high-level, developer-friendly Python SDK. When you write `boto3.resource('s3')`, boto3 is translating that into botocore calls, which then hit the actual AWS API.

**`awscli`** is also built on top of `botocore` — independently from boto3. So both the AWS CLI and boto3 share the same underlying engine.
-->
---

## Step 3 - Configure AWS CLI Authentication

With the Access Key ID and Secret Access Key from [[#Step 1 - Create a Dedicated IAM User for Terraform|Step 1]], I configure the AWS CLI to authenticate as the `terraform` user.

```bash
hector@hector-Laptop:~$ aws configure
AWS Access Key ID [****************HQXB]: <your-access-key>
AWS Secret Access Key [****************equ5]: <your-secret-key>
Default region name [us-east-1]:
Default output format [None]:
```

> This writes credentials to `~/.aws/credentials`, which both the AWS CLI and `boto3` will use automatically.

---

## Step 4 - Create an S3 Bucket

I create an S3 bucket now as a prerequisite, it will be configured as a remote Terraform backend with state locking in Part 3. For now it just needs to exist.
**Navigate to:** Amazon S3 → Buckets → Create Bucket

|Setting|Value|
|---|---|
|Bucket name|`hector-dev-terraform-bucket`|
|AWS Region|`us-east-1`|
|Tag|Name = `hector-dev-terraform-bucket`|

![[Markdown_Logo-2.png]]

### Verify AWS CLI Authentication

With credentials configured, I run a quick check to confirm the AWS CLI can authenticate and reach AWS:

```bash
hector@hector-Laptop:~$ aws s3 ls
2022-05-11 16:27:09 hector-dev-terraform-bucket
```

The bucket we just created comes back, credentials are working correctly.
<!--
replaced it with above

### Verify Programmatic Access with boto3

With the bucket created and credentials configured, I verify that Python can reach it:

```bash
hector@hector-Laptop:~$ python3
Python 3.8.10 ...
>>> import boto3
>>> s3 = boto3.resource('s3')
>>> for bucket in s3.buckets.all():
...     print(bucket.name)
...
hector-dev-terraform-bucket   # ✅ Our bucket is returned
```

This confirms that both the credentials and the bucket are set up correctly.
<!--
boto3 looks for the aws cli credentials in the file it saved it in ~/.aws/credentials
so we just using  boto3  to test that the credentials we configured thru aws cli are working
-->
---

## Step 5 - [[Installing Terraform#Linux|Install Terraform]]

I download the Terraform binary, extract it, and move it into the system `PATH` so it's available globally.

```bash
# Download the Terraform zip
hector@hector-Laptop:~/Project16-17$ sudo wget https://releases.hashicorp.com/terraform/1.1.9/terraform_1.1.9_linux_amd64.zip

# Verify the download
hector@hector-Laptop:~/Project16-17$ ls
PBL  README.md  terraform_1.1.9_linux_amd64.zip

# Extract the binary
hector@hector-Laptop:~/Project16-17$ sudo unzip terraform_1.1.9_linux_amd64.zip
Archive:  terraform_1.1.9_linux_amd64.zip
  inflating: terraform

# Move it to the system bin directory
hector@hector-Laptop:~/Project16-17$ sudo mv terraform /usr/local/bin/

# Confirm the installation
hector@hector-Laptop:~/Project16-17$ terraform -v
Terraform v1.1.9
on linux_amd64
```

---

## Step 6 - Write the First Terraform Configuration

The working directory is `PBL/`, which starts with a single file: `main.tf`.

### Define the AWS Provider and VPC

The `provider` block tells Terraform which cloud platform to use. The `resource` block defines the actual infrastructure, in this case a VPC.

```yaml
# Provider block - instructs Terraform to build in AWS
provider "aws" {
  region = "us-east-1"
}

# Resource block - creates a VPC
resource "aws_vpc" "main" {
  cidr_block                     = "10.0.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"
}
```

### Initialize Terraform

Before running any commands, Terraform needs to download the AWS **provider plugin**. This is done once per working directory with `terraform init`.

```bash
hector@hector-Laptop:~/Project16-17/PBL$ terraform init
```

> [!NOTE]- Output
> ```bash
> Initializing the backend...
> 
> Initializing provider plugins...
> - Finding latest version of hashicorp/aws...
> - Installing hashicorp/aws v4.13.0...
> - Installed hashicorp/aws v4.13.0 (signed by HashiCorp)
> 
> Terraform has created a lock file .terraform.lock.hcl to record the provider
> selections it made above. Include this file in your version control repository
> so that Terraform can guarantee to make the same selections by default when
> you run "terraform init" in the future.
> 
> Terraform has been successfully initialized!
> ```
> 

A `.terraform/` directory is created to store the downloaded plugins. It's safe to delete and recreate by running `terraform init` again.

```bash
hector@hector-Laptop:~/Project16-17/PBL$ ls -a
.  ..  main.tf  .terraform  .terraform.lock.hcl
```

> The `.terraform.lock.hcl` [Dependency Lock File](https://www.terraform.io/language/files/dependency-lock) pins the exact provider version used, commit this to source control.

### Plan and Apply

Before creating anything, I preview what Terraform intends to do:

```bash
terraform plan   # Shows what will be created/modified/destroyed
terraform apply  # Executes the plan
```

After a successful `apply`, two new files appear:

```bash
hector@hector-Laptop:~/Project16-17/PBL$ ls
main.tf  terraform.tfstate  terraform.tfstate.backup
```

| File                          | Purpose                                                                           |
| ----------------------------- | --------------------------------------------------------------------------------- |
| `terraform.tfstate`           | Tracks the exact current state of all provisioned resources                       |
| `terraform.tfstate.backup`    | Previous state, used for rollback                                                 |
| `terraform.tfstate.lock.info` | Temporary lock file while Terraform is running, prevents concurrent modifications |
<!--
### `terraform.tfstate.backup`

Terraform automatically creates this file every time `terraform apply` runs — it's a snapshot of the **previous** state, saved before the new state is written. Think of it as a one-step undo.

---

#### Scenario: You accidentally destroy a resource

Say your `main.tf` had 2 public subnets and someone on your team edits it down to 1, then runs `terraform apply`. Terraform sees only 1 subnet declared and **destroys** the second one. The `terraform.tfstate` now reflects that — only 1 subnet exists.

You catch the mistake. The resource is gone. Here's how you recover:

bash

```bash
# 1. Look at what the backup contains
cat terraform.tfstate.backup

# 2. Restore it by overwriting the current state
cp terraform.tfstate.backup terraform.tfstate

# 3. Fix the code back to 2 subnets in main.tf, then re-apply
terraform apply
```

Terraform reads the restored state, sees 2 subnets should exist but only 1 does, and recreates the missing one.
-->
---

## Step 7 - Add Public Subnets

With the VPC in place, I add two public subnets to `main.tf`:

```yaml
# Create public subnet 1
resource "aws_subnet" "public1" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"
}

# Create public subnet 2
resource "aws_subnet" "public2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.3.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1b"
}
```

Running `terraform plan` and `terraform apply` provisions both subnets. But this code has two problems that need to be addressed.

---

## Step 8 - Refactoring: Removing Hard-Coded Values

### Problem 1: Hard-Coded Values

The `availability_zone` and `cidr_block` arguments are hard-coded strings. Any infrastructure change requires editing the code directly, this is fragile and not reusable.

**Fix - Introduce Variables**

I replace all hard-coded values with `variable` declarations. The provider block becomes:

```yaml
variable "region" {
  default = "us-east-1"
}

provider "aws" {
  region = var.region
}
```

And the VPC block follows the same pattern for every argument:

```yaml
variable "vpc_cidr"                    { default = "10.0.0.0/16" }
variable "enable_dns_support"          { default = "true" }
variable "enable_dns_hostnames"        { default = "true" }
variable "enable_classiclink"          { default = "false" }
variable "enable_classiclink_dns_support" { default = "false" }

resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support
  enable_dns_hostnames           = var.enable_dns_hostnames
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink_dns_support
}
```

### Problem 2: Multiple Resource Blocks for Subnets

Having a separate `resource` block for every subnet doesn't scale. If I need 6 subnets I'd need 6 blocks. The fix is to use a **loop** and a **data source**.

**Step 1 - Fetch Availability Zones Dynamically**

Instead of hard-coding `us-east-1a`, I pull the list of available AZs directly from AWS:

```yaml
data "aws_availability_zones" "available" {
  state = "available"
}
```

**Step 2 - Use `count` to Loop**

The `count` argument tells Terraform to create multiple copies of a resource. Combined with `count.index`, each iteration picks a different AZ from the list:

```yaml
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"    # ⚠️ Still hard-coded, will fail on second loop
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

`data.aws_availability_zones.available.names` returns something like:

```
["us-east-1a", "us-east-1b", "us-east-1c", ...]
```

So `names[0]` = `us-east-1a`, `names[1]` = `us-east-1b`, and so on.

**Step 3 - Make `cidr_block` Dynamic with `cidrsubnet()`**

The same CIDR block can't be assigned to two subnets in the same VPC. The built-in `cidrsubnet()` function solves this by computing a unique CIDR for each loop iteration.

```
cidrsubnet(prefix, newbits, netnum)
```

|Parameter|Meaning|
|---|---|
|`prefix`|Base CIDR in notation (e.g. `10.0.0.0/16`)|
|`newbits`|How many bits to extend the prefix by|
|`netnum`|Which subnet number to generate|

Testing in the Terraform console:

```bash
hector@hector-Laptop:~$ terraform console
> cidrsubnet("172.16.0.0/16", 4, 0)
"172.16.0.0/20"
```

The updated subnet block with dynamic CIDR:

```yaml
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

**Step 4 - Remove the Hard-Coded `count` Value**

The final hard-coded value is the `count = 2`. I replace it with a variable and a conditional using the `length()` function:

```yaml
variable "preferred_number_of_public_subnets" {
  default = 2
}
```

```yaml
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

Breaking down the condition:

|Part|Meaning|
|---|---|
|`var.preferred_number_of_public_subnets == null`|Is the variable unset?|
|`? length(data.aws_availability_zones.available.names)`|If yes, create one subnet per available AZ|
|`: var.preferred_number_of_public_subnets`|If no, use the number we defined|

This means if you don't specify a preference, Terraform will automatically create one subnet per AZ in the region.

---

## Step 9 - Separating Variables into Their Own Files

Having all variable declarations inside `main.tf` becomes messy as the project grows. I split things into dedicated files.

**File structure after refactoring:**

```bash
hector@hector-Laptop:~/Project16-17/PBL$ tree
.
├── main.tf
├── terraform.tfstate
├── terraform.tfstate.backup
├── terraform.tfvars
└── variables.tf

0 directories, 5 files
```

**`main.tf`** - contains only provider and resource blocks (no variable declarations):

<details> <summary>main.tf</summary>

```yaml
data "aws_availability_zones" "available" {
  state = "available"
}

provider "aws" {
  region = var.region
}

resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support
  enable_dns_hostnames           = var.enable_dns_hostnames
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink_dns_support
}

resource "aws_subnet" "public" {
  count                   = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

</details>

**`variables.tf`** - all variable declarations with types and descriptions:

<details> <summary>variables.tf</summary>

```yaml
variable "region"                          { default = "us-east-1" }
variable "vpc_cidr"                        { default = "10.0.0.0/16" }
variable "enable_dns_support"              { default = "true" }
variable "enable_dns_hostnames"            { default = "true" }
variable "enable_classiclink"              { default = "false" }
variable "enable_classiclink_dns_support"  { default = "false" }
variable "preferred_number_of_public_subnets" { default = null }
```

</details>

**`terraform.tfvars`** - actual values for each variable. This is the only file you need to change to adjust the configuration:

<details> <summary>terraform.tfvars</summary>

```yaml
region                              = "us-east-1"
vpc_cidr                            = "10.0.0.0/16"
enable_dns_support                  = "true"
enable_dns_hostnames                = "true"
enable_classiclink                  = "false"
enable_classiclink_dns_support      = "false"
preferred_number_of_public_subnets  = 2
```

</details>

Running `terraform plan` with this structure confirms everything works correctly. The separation of concerns means you can hand `terraform.tfvars` to someone else to customize without them needing to touch the core resource logic.

---

## Summary

By the end of this project the foundation is solid:

- ✅ A dedicated IAM `terraform` user with programmatic-only access
- ✅ AWS CLI configured and verified with `boto3`
- ✅ An S3 bucket created (to be configured as a remote backend in Part 3)
- ✅ Terraform installed and initialized against the `PBL/` directory
- ✅ A VPC and 2 public subnets provisioned via Terraform
- ✅ Code refactored to use variables, data sources, loops, and dynamic CIDR generation
- ✅ Configuration split across `main.tf`, `variables.tf`, and `terraform.tfvars`

**Part 2** picks up from here and builds out the full AWS stack: networking (Internet Gateway, NAT Gateway, Route Tables), IAM roles, Security Groups, Load Balancers, Auto Scaling Groups, EFS, and RDS.

---

_GitHub Repository: [AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4)_


<!--
# AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1/4
Project 16 Terraform

After you have built AWS infrastructure for 2 websites manually, it is time to automate the process using Terraform.
Let us start building the same set up with the power of Infrastructure as Code (IaC)

![[Markdown_Logo-2.png]]

### AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 1
You must have completed Terraform course from the Learning dashboard

Create an IAM user, name it terraform (ensure that the user has only programatic access to your AWS account) and grant this user AdministratorAccess permissions.

IAM > Users > Add users  
* **Step1**:  
  * User name: **terraform**  
  * Select AWS credential type: **Access key - Programmatic access**  
* **Step2**:  
  * Create group  
    * Group name: **terraform**  
    * Policy name:  `AdministratorAccess`  
* **Step3**:  
  * Add tags  
    * Name **terraform**  
* **Step4**:  
    * Review  
* **Step5**:  
    * Store Access key ID and Secret access key  


![[Markdown_Logo-3.png]]

So I have pip installed

pip install boto3

``` bash
hector@hector-Laptop:~$ pip list | grep boto3
boto3                        1.17.112
hector@hector-Laptop:~$
```

I will use AWS CLI to authenticate so I make sure I have installed
``` bash
hector@hector-Laptop:~$ aws --version
aws-cli/1.22.71 Python/3.8.10 Linux/5.4.0-109-generic botocore/1.24.16
hector@hector-Laptop:~$
```

Going to set authentication configuration on AWS CLI for user terraform  
[Guide](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html) to configure the Python SDK properly. 

``` bash
hector@hector-Laptop:~$ aws configure
AWS Access Key ID [****************HQXB]: xxxxxxxxxxxxxxxxxxxx
AWS Secret Access Key [****************equ5]: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Default region name [us-east-1]:
Default output format [None]:
hector@hector-Laptop:~$
```

Create an [S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) to store Terraform state file.   
Amazon S3 > Buckets > Create Bucket  
* General Configuration  
  * Bucket name: `hector-dev-terraform-bucket`  
  * AWS Region: `us-east-1 ` 
* Tags
  * Name `hector-dev-terraform-bucket`  


![[Markdown_Logo-2.png  ]]

When you have configured authentication and installed `boto3`, make sure you can programmatically access your
``` bash
hector@hector-Laptop:~$ python3
Python 3.8.10 (default, Mar 15 2022, 12:22:08)
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import boto3
>>> s3 = boto3.resource('s3')
>>> for bucket in s3.buckets.all():
...     print(bucket.name)
...
hector-dev-terraform-bucket #Returns the name of our bucket`
>>>
```
### VPC | SUBNETS | SECURITY GROUPS

Update the system packages
udo apt update
Install the wget and unzip package to download and extract terraform setup

sudo apt-get install wget unzip -y

Installing **terraform**   
``` bash
hector@hector-Laptop:~/Project16-17$ sudo wget https://releases.hashicorp.com/terraform/0.14.7/terraform_0.14.7_linux_amd64.zip #download terraform setup
hector@hector-Laptop:~/Project16-17$ ls #zip file downloaded
PBL  README.md  terraform_1.1.9_linux_amd64.zip
hector@hector-Laptop:~/Project16-17$ sudo unzip terraform_1.1.9_linux_amd64.zip #Extract the downloaded setup using unzip
Archive:  terraform_1.1.9_linux_amd64.zip
inflating: terraform
hector@hector-Laptop:~/Project16-17$ ls #terraform extracted
PBL  README.md  terraform  terraform_1.1.9_linux_amd64.zip
hector@hector-Laptop:~/Project16-17$ sudo mv terraform /usr/local/bin/ #moving extracted setup to /bin directory
hector@hector-Laptop:~/Project16-17$ terraform -v #checking `version (not latest)
Terraform v1.1.9
on linux_amd64
hector@hector-Laptop:~/Project16-17$
```

Our current directory structure consists of a folder called `PBL` with a file inside `main.tf`   

`main.tf` **AWS** as a provider, and a resource to create a VPC  

``` bash
#Provider block informs Terraform that we intend to build infrastructure within AWS.
provider "aws" { 
  region = "us-east-1"
}

#Resource block will create a VPC.
resource "aws_vpc" "main" {
  cidr_block                     = "10.0.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"
}
```
For Terraform to work we need to download the necessary **plugin**. **Plugins** are used by [providers](https://www.terraform.io/language/providers) and [provisioners](https://www.terraform.io/language/resources/provisioners/syntax). So far we only have a **provider** in our `main.tf` file. So, Terraform will just download **plugin** for **AWS** provider.  

We accomplish this with `terraform init`  

We will use this command to initialize the `PBL` directory  

``` bash
hector@hector-Laptop:~/Project16-17$ cd PBL/
hector@hector-Laptop:~/Project16-17/PBL$ ls
main.tf
hector@hector-Laptop:~/Project16-17/PBL$ terraform init
```
<details close>
<summary>Output</summary>

``` bash
Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v4.13.0...
- Installed hashicorp/aws v4.13.0 (signed by HashiCorp)

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
``` 
</details>




Notice that a new directory has been created: `.terraform` This is where **Terraform** keeps **plugins**. Generally, it is safe to delete this folder. You just have to execute `terraform init` again, to download them.  

``` bash
hector@hector-Laptop:~/Project16-17/PBL$ ls -a
.  ..  main.tf  .terraform  .terraform.lock.hcl
```
*In addition, [.terraform.lock.hcl](https://www.terraform.io/language/files/dependency-lock) Dependency Lock File*


Now we create the only resource we just defined `aws_vpc`. Before creating anything we should check to see what **terraform** intends to provision with `terraform plan` 
If we are happy with changes planned we execute `terraform apply`  


Files generated after running **plan** and **apply**  
``` bash
hector@hector-Laptop:~/Project16-17/PBL$ ls
main.tf  terraform.tfstate  terraform.tfstate.backup
hector@hector-Laptop:~/Project16-17/PBL$
```

`terraform.tfstate`  is how **Terraform** keeps itself up to date with the exact **state** of the infrastructure. It reads this file to know what already exists, what should be added, or destroyed based on the entire terraform code that is being developed.  

`terraform.tfstate.lock.info` *(gets deleted immediately)* This is what **Terraform** uses to track, who is running its code against the infrastructure at any point in time. This is very important for teams working on the same Terraform repository at the same time. The lock prevents a user from executing **Terraform** configuration against the same infrastructure when another user is doing the same – it allows to avoid duplicates and conflicts.  


Lets create the first 2 **public subnets** by adding below configuration to the `main.tf` file:
``` bash
# Declaring 2 resource blocks – one for each of the subnets

# Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id #<< interpolate the value of the VPC id
    cidr_block                 = "10.0.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-east-1"
}
# Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id #<<
    cidr_block                 = "10.0.3.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-east-1"
}
```

Once again we run `terraform plan` and `terraform apply`  

**Observations:**  *(Best Practices)*     
*Hard coded values:*  
`availability_zone` and `cidr_block` arguments are **hard coded** and our goal should always be to make our work **dynamic**  
*Multiple Resource Blocks:*  
We have declared multiple **resource blocks** for each subnet in the code. We need to create a single resource block that can **dynamically** create resources without specifying **multiple blocks**.  

### FIXING THE PROBLEMS BY CODE REFACTORING

**Fixing Hard Coded Values:** We will introduce variables, and remove hard coding.  
Starting with the *provider block*, we declare a **variable** named `region`, give it a default value, and update the provider section by referring to the declared variable.
``` bash
variable "region" {
  default = "us-east-1"
}

provider "aws" {
  region = var.region
}
```

Doing the same to `cidr` value in the vpc block, and all the other arguments.  
``` bash
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "enable_dns_support" {
  default = "true"
}

variable "enable_dns_hostnames" {
 default ="true" 
}

variable "enable_classiclink" {
  default = "false"
}

variable "enable_classiclink_dns_support" {
  default = "false"
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink
}
```
	
**Fixing multiple resource blocks:** We are going to introduce **Loops** & **Data sources**  
	
We will explore the use of Terraform’s **Data Sources** to fetch information outside of Terraform *(in this case, from AWS)*  

Let us fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

**Get list of availability zones**
``` bash
data "aws_availability_zones" "available" {
state = "available"
}
```

To make use of this new data resource, we introduce a `count` argument in the subnet block as follow:   
``` bash
# Create public subnet1
resource "aws_subnet" "public" { 
count                   = 2 #invokes a loop twice
vpc_id                  = aws_vpc.main.id
cidr_block              = "10.0.1.0/24" # << Will make second loop fail
map_public_ip_on_launch = true
availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

The data resource `data.aws_availability_zones.available.names[count.index]` will return a list object that contains a list of AZs. In this case the `counter` is set to 2 so it will return something like this `["us-east-1a", "us-east-1b"]`  

Another way to look at it:  
`us-east-1a` = `data.aws_availability_zones.available.names[0]`  
`us-east-1b` = `data.aws_availability_zones.available.names[1]`  


If we run Terraform with this configuration, it may succeed for the first time, but by the time it goes into the second loop, it will fail because we still have `cidr_block` hard coded. The same `cidr_block` cannot be created twice within the same VPC. So...

**Let’s make `cidr_block` dynamic.** 
We will introduce a function `cidrsubnet()` that works like an algorithm to dynamically create a subnet CIDR per AZ. Regardless of the number of subnets created, it takes care of the cidr value per subnet.  

Its parameters are **cidrsubnet(`prefix`, `newbits`, `netnum`)**  
* The `prefix` parameter must be given in CIDR notation, same as for VPC.
* The `newbits` parameter is the number of additional bits with which to extend the prefix. For example, if given a prefix ending with /16 and a newbits value of 4, the resulting subnet address will have length /20
* The `netnum` parameter is a whole number that can be represented as a binary integer with no more than newbits binary digits, which will be used to populate the additional bits added to the prefix   

Testing **terraform console**  
``` bash
hector@hector-Laptop:~$ terraform console
> cidrsubnet("172.16.0.0/16", 4, 0)
"172.16.0.0/20"
>
```
We update the configuration  
``` bash
# Create public subnet1
resource "aws_subnet" "public" { 
    count                   = 2
    vpc_id                  = aws_vpc.main.id
    cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
    map_public_ip_on_launch = true
    availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

**The final problem to solve is removing hard coded `count` value.**  
*We'll introduce `length()` function, which basically determines the length of a given list, map, or string.*

Declare a variable to store the desired number of public subnets, and set the default value
``` bash	
variable "preferred_number_of_public_subnets" {
  default = 2
}
```
Next, update the count argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the length function. See how that is presented below.

``` bash	
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

Now lets break it down: `count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets`  

* The first part `var.preferred_number_of_public_subnets == null` checks if the value of the variable is set to `null` or has some value defined.
* The second part `?` and `length(data.aws_availability_zones.available.names)` means, if the first part is true, then use this. In other words, if preferred number of public subnets is `null` (Or not known) then set the value to the data returned by `length` function *(number of subnets created will equal amount of AZ)*
* The third part `:` and `var.preferred_number_of_public_subnets` means, if the first condition is false, i.e preferred number of public subnets is `not null` then set the value to whatever is defined in `var.preferred_number_of_public_subnets`  


  
<details close>
<summary>Now the entire configuration looks like this</summary>

``` bash
# Get list of availability zones
data "aws_availability_zones" "available" {
  state = "available"
}
variable "region" {
  default = "us-east-1"
}
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}
variable "enable_dns_support" {
  default = "true"
}
variable "enable_dns_hostnames" {
  default ="true" 
}
variable "enable_classiclink" {
  default = "false"
}
variable "enable_classiclink_dns_support" {
  default = "false"
}
variable "preferred_number_of_public_subnets" {
  default = 2
}
provider "aws" {
  region = var.region
}
# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink
}
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```  
</details>



### INTRODUCING VARIABLES.TF &AMP; TERRAFORM.TFVARS

To make our code a lot more readable and better structured instead of having a long list of variables in `main.tf` file I'll move out some parts of the configuration content to other files.


We will put all **variable declarations** in a separate file and provide non **default values** to each of them  
* Created a new file `variables.tf` where I moved all the **variable declarations**  
* Created another file `terraform.tfvars` where I **set values** for each of the variables  

<details close>
<summary>Maint.tf</summary>

``` bash
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}
provider "aws" {
  region = var.region
}
# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink
}
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```  
</details>

<details close>
<summary>variables.tf</summary>

``` bash
variable "region" {
  default = "us-east-1"
}
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}
variable "enable_dns_support" {
  default = "true"
}
variable "enable_dns_hostnames" {
  default ="true" 
}
variable "enable_classiclink" {
  default = "false"
}
variable "enable_classiclink_dns_support" {
  default = "false"
}
variable "preferred_number_of_public_subnets" {
  default = null
}

```  
</details>

<details close>
<summary>terraform.tfvars</summary>

``` bash
region = "us-east-1"
vpc_cidr = "10.0.0.0/16" 
enable_dns_support = "true" 
enable_dns_hostnames = "true"  
enable_classiclink = "false" 
enable_classiclink_dns_support = "false" 
preferred_number_of_public_subnets = 2
```  
</details>


Now the file structure in the **PBL** folder look like this  
``` bash
hector@hector-Laptop:~/Project16-17/PBL$ tree
.
├── main.tf
├── terraform.tfstate
├── terraform.tfstate.backup
├── terraform.tfvars
└── variables.tf

0 directories, 5 files
hector@hector-Laptop:~/Project16-17/PBL$
```
We run `terraform plan` to ensure everything works  
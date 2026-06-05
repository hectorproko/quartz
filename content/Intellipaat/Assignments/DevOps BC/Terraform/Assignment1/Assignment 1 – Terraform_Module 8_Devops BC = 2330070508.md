---
tags:
  - Terraform
---
==PENDING CLEANUP==
> [!info] Module 8: Terraform Assignment - 1
> **Tasks To Be Performed:** 
> 1. Create an EC2 service in the default subnet in the Ohio region


### Step 1: Terraform Setup

<br><br>![[Installing Terraform#Windows]]
### Step 2:  A way to for Terraform to authenticate with AWS

I will be using the same AWS CLI setup from [[Assignment 2 – AWS Migration_Module11_AWS Weekday BC = 2301080808#**Using AWS CLI **|Assignment 2 – AWS Migration]]

### Step 3:  Crafting main.tf

Need to make a comment about the provider

```yaml
provider "aws" {
  region = "us-east-2" # Ohio region
}


resource "aws_instance" "example" {
  ami           = "ami-06d4b7182ac3480fa" #AmazonLinux
  instance_type = "t2.micro"

  subnet_id = "subnet-01384a93bc22b2937"

  tags = {
    Name = "ExampleEC2"
  }
}
```
The `aws_instance` resource defines your EC2 instance.


```
Hecti@DESKTOP-7SBLCCA MINGW64 ~/Desktop
$ pwd
/c/Users/Hecti/Desktop

Hecti@DESKTOP-7SBLCCA MINGW64 ~/Desktop
$ mkdir Terraform1

Hecti@DESKTOP-7SBLCCA MINGW64 ~/Desktop
$ nano main.tf

Hecti@DESKTOP-7SBLCCA MINGW64 ~/Desktop
```

<br><br>![[Pasted image 20231115132535.png]]

<br><br>![[Pasted image 20231115133155.png]]

Terraform plan

> [!attention]
> Note: You didn't use the -out option to save this plan, so Terraform can't
> guarantee to take exactly these actions if you run "terraform apply" now.
> 

`terraform apply`
<br><br>![[Pasted image 20231115133459.png]]

<br><br>![[Pasted image 20231115140546.png]]
<br><br>![[Pasted image 20231115140913.png]]
<br><br>![[Pasted image 20231115140622.png]]


---
tags:
  - Terraform
---
==PENDING CLEANUP==
> [!info] Module 8: Terraform Assignment - 2
> **Tasks To Be Performed:** 
> 1. Destroy the previous deployment 
> 2. Create a new EC2 instance with an Elastic IP

Step 1:
Destroying infrasctructure (EC2) created in [[Assignment 1 – Terraform_Module 8_Devops BC = 2330070508|Assignment 1 – Terraform]]

<br><br>![[Pasted image 20231115141034.png]]
<br><br>![[Pasted image 20231115141107.png]]

Step 2: New instance with Elastic IP

will switch to us-east-1

so we take the main.tf from [[Assignment 1 – Terraform_Module 8_Devops BC = 2330070508|Assignment 1 – Terraform]]
and modify it

```ymal
provider "aws" {
  region = "us-east-2" # Ohio region
}

resource "aws_instance" "example" {
  ami           = "ami-06d4b7182ac3480fa" # AmazonLinux
  instance_type = "t2.micro"

  subnet_id = "subnet-01384a93bc22b2937"

  tags = {
    Name = "ExampleEC2"
  }
}

resource "aws_eip" "example" {
  instance = aws_instance.example.id
  domain   = "vpc"
}
```
- The `aws_eip` resource creates an Elastic IP.
- The `instance` attribute in the `aws_eip` resource is set to `aws_instance.example.id`, which associates the EIP with your EC2 instance.



``terraform apply``

<br><br>![[Pasted image 20231115142813.png]]

my instance has Elastic IP
<br><br>![[Pasted image 20231115143419.png]]
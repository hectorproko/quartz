---
tags:
  - Terraform
---
==PENDING CLEANUP==
> [!info] Module 8: Terraform Assignment - 3
> **Tasks To Be Performed:** 
> 1. Destroy the previous deployment 
> 2. Create 2 EC2 instances in Ohio and N.Virginia respectively 
> 3. Rename Ohio’s instance to ‘hello-ohio’ and Virginia’s instance to ‘hello-virginia’

### Step1:

Destroyed deployment [[Assignment 2 – Terraform_Module 8_Devops BC = 2330070508|Assignment 2 – Terraform]]

`terraform destroy`
<br><br>![[Pasted image 20231115143815.png]]

### Step 2-3:

```yaml
provider "aws" {
  region = "us-east-2" # Ohio region
  alias  = "ohio"
}

provider "aws" {
  region = "us-east-1" # N. Virginia region
  alias  = "virginia"
}

resource "aws_instance" "hello_ohio" {
  provider = aws.ohio

  ami           = "ami-06d4b7182ac3480fa"
  instance_type = "t2.micro"
  subnet_id     = "subnet-01384a93bc22b2937" 

  tags = {
    Name = "hello-ohio"
  }
}

resource "aws_instance" "hello_virginia" {
  provider = aws.virginia

  ami           = "ami-0230bd60aa48260c6"
  instance_type = "t2.micro"
  subnet_id     = "subnet-05419866677eb6366"

  tags = {
    Name = "hello-virginia"
  }
}
```

In this configuration:

- Two `provider` blocks are defined, each with a different `alias` (`ohio` and `virginia`), targeting the `us-east-2` (Ohio) and `us-east-1` (N. Virginia) regions, respectively.
- Two `aws_instance` resources are created: `hello_ohio` for the Ohio region and `hello_virginia` for the N. Virginia region. Each instance uses its respective provider by specifying `provider = aws.<alias>`.
- The `ami` and `subnet_id` for each instance should be appropriate for their respective regions. You need to replace the placeholders with the actual AMI IDs and subnet IDs for each region.
- The `tags` for each instance are set to `hello-ohio` and `hello-virginia`, respectively.

<br><br>![[Pasted image 20231115153825.png]]


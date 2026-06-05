---
tags:
  - Terraform
---
==PENDING CLEANUP==
> [!info] Module 8: Terraform Assignment - 5
> **Tasks To Be Performed:** 
> 1. Destroy the previous deployments 
> 2. Create a script to install Apache2 
> 3. Run this script on a newly created EC2 instance 
> 4. Print the IP address of the instance in a file on the local once deployed


Step1.
Destroy [[Assignment 4 – Terraform_Module 8_Devops BC = 2330070508|Assignment 4 – Terraform]]
<br><br>![[Pasted image 20231115212716.png]]

Step2:
```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y apache2
```

### Step3:
I'll add the following to 
```yaml
  user_data = <<-EOF
              #!/bin/bash
              sudo apt-get update
              sudo apt-get install -y apache2
              EOF
```


### Step 4: Output the IP Address to a Local File

You can use Terraform's `output` functionality to print the instance's IP address to a file after deployment. Define an output variable in your Terraform configuration:

```yaml
output "instance_ip_address" {
  value = aws_instance.example_instance.public_ip
}
```

After running `terraform apply`, the IP address of the instance will be displayed in the output. To save this to a file, you can redirect Terraform's output:
```
terraform apply | tee apply_output.txt
```

Or, you can just output the specific value after apply:
```
terraform output instance_ip_address > instance_ip.txt
```


---

```yaml
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "example_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "example-vpc"
  }
}

resource "aws_subnet" "example_subnet" {
  vpc_id            = aws_vpc.example_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "example-subnet"
  }
}

resource "aws_internet_gateway" "example_igw" {
  vpc_id = aws_vpc.example_vpc.id

  tags = {
    Name = "example-igw"
  }
}

resource "aws_route_table" "example_rt" {
  vpc_id = aws_vpc.example_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.example_igw.id
  }

  tags = {
    Name = "example-route-table"
  }
}

resource "aws_route_table_association" "example_rta" {
  subnet_id      = aws_subnet.example_subnet.id
  route_table_id = aws_route_table.example_rt.id
}

resource "aws_security_group" "example_sg" {
  name        = "example-sg"
  description = "Example Security Group"
  vpc_id      = aws_vpc.example_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "example-security-group"
  }
}

resource "aws_instance" "example_instance" {
  ami           = "ami-0230bd60aa48260c6"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.example_subnet.id
  associate_public_ip_address = true
  vpc_security_group_ids = [aws_security_group.example_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              sudo yum update -y
              sudo yum install -y httpd
              sudo systemctl start httpd
              sudo systemctl enable httpd
              EOF

  tags = {
    Name = "example-instance"
  }
  
  output "instance_ip_address" {
    value = aws_instance.example_instance.public_ip
  }

}
```

<br><br>![[Pasted image 20231115222703.png]]

<br><br>![[Pasted image 20231115222727.png]]


```
terraform output instance_ip_address > instance_ip.txt
```
<br><br>![[Pasted image 20231115222818.png]]



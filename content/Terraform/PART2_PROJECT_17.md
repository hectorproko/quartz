---


tags:
  - "#Terraform"
  - IaC
hardlinked: "True"
---
[Hector's LearningHub Link](https://hectorproko.github.io/quartz/darey.io/Project15/Project15-AWS-CLOUD-SOLUTION-FOR-2-COMPANY-WEBSITES-USING-A-REVERSE-PROXY-TECHNOLOGY)

# AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-2/4
Project 17 Terraform

Best practices Tagging  

**Tagging** helps you manage your resources much more efficiently:
* Resources are much better organized in ‘virtual’ groups
* They can be easily filtered and searched from console or programmatically
* Billing team can easily generate reports and determine how much each part of infrastructure costs how much (by department, by type, by environment, etc.)
* You can easily determine resources that are not being used and take actions accordingly
* If there are different teams in the organization using the same account, tagging can help differentiate who owns which resources  


Lets add multiple tags as a default set. for example, in out [terraform.tfvars](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/terraform.tfvars) file we can have default tags defined.
``` bash
tags = {
  Enviroment      = "development" 
  Owner-Email     = "hectore@email.com"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
```

Now we can tag all resources using the format below
``` bash
tags = merge(
    var.tags,
    {
      Name = "Name of the resource"
    },
  )
```

We need to to declare the variable `tags` in [variables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/variables.tf) using the following format
``` bash
variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```
Now every time we need to make a change to the **tags**, we can do that in one single place `terraform.tfvars`  


**Internet Gateways & `format()` function**  
Create an **Internet Gateway** in a separate Terraform file [internet_gateway.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/internet_gateway.tf)  

We have can use `format()` function to dynamically generate a unique name for a resource  

``` bash
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.main.id
tags = merge(
    var.tags,
    {
      Name = format("%s-%s!", aws_vpc.main.id,"IG")
    } 
  )
}
```


In the example above the **first** of the `%s` takes the interpolated value of `aws_vpc.main.id` while the **second** `%s` appends a literal string IG and finally an exclamation mark is added in the end.  

This is useful when creating a resource with a `count` function or creating multiple resources using a `loop` which requires the **key-value pair** to be unique  




For example, each of our subnets should have a unique name in the **tag** section. We can accomplish this with `format()` function.

``` bash
tags = merge(
  var.tags,
  {
    Name = format("PrivateSubnet-%s", count.index)
  } 
)
```
The output should look something like this  
`Name = PrvateSubnet-0`  
`Name = PrvateSubnet-1`  
`Name = PrvateSubnet-2`  



NAT Gateways
Create 1 NAT Gateways and 1 Elastic IP (EIP) addresses
Now use similar approach to create the NAT Gateways in a new file called natgateway.tf.

Note: We need to create an Elastic IP for the NAT Gateway, and you can see the use of depends_on to indicate that the Internet Gateway resource must be available before this should be created. Although Terraform does a good job to manage dependencies, but in some cases, it is good to be explicit.
You can read more on dependencies here
resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.ig]
tags = merge(
    var.tags,
    {
      Name = format("%s-EIP", var.name)
    },
  )
}
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]
tags = merge(
    var.tags,
    {
      Name = format("%s-Nat", var.name)
    },
  )
}



**NAT Gateways**  

Creating 1 **NAT Gateway** and 1 **Elastic IP (EIP)** address in a new file called [natgateway.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/natgateway.tf)  

We need to create an **Elastic IP** for the **NAT Gateway**, and we introduce the use of `depends_on` to indicate that the **Internet Gateway** resource must be available before this should be created.  

`depends_on` [Documentation](https://www.terraform.io/language/meta-arguments/depends_on)  

``` bash
resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.ig]
  tags = merge(
    var.tags,
    {
      Name = format("%s-EIP", var.name)
    },
  )
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]
  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat", var.name)
    },
  )
}
```

### AWS ROUTES
To create **routes** for both **public** and **private** subnets we generate [route_tables.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/route_tables.tf) with resources `aws_route_table`, `aws_route`, `aws_route_table_association`  

``` bash	
# create private route table
resource "aws_route_table" "private-rtb" {
  vpc_id = aws_vpc.main.id
  tags = merge(
    var.tags,
    {
      Name = format("%s-private-rtb", var.name)
    },
  )
}

# associate all private subnets to the private route table
resource "aws_route_table_association" "private-subnets-assoc" {
  count          = length(aws_subnet.private[*].id)
  subnet_id      = element(aws_subnet.private[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}

# create route table for the public subnets
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.main.id
  tags = merge(
    var.tags,
    {
      Name = format("%s-public-rtb", var.name)
    },
  )
}

# create route for the public route table and attach the internet gateway
resource "aws_route" "public-rtb-route" {
  route_table_id         = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}

# associate all public subnets to the public route table
resource "aws_route_table_association" "public-subnets-assoc" {
  count          = length(aws_subnet.public[*].id)
  subnet_id      = element(aws_subnet.public[*].id, count.index)
  route_table_id = aws_route_table.public-rtb.id
}
```


Now we run `terraform plan` and `terraform apply` it should add the following resources to **AWS** in **multi-az** set up:
* Our main VPC
* 2 Public subnets
* 4 Private subnets
* 1 Internet Gateway
* 1 NAT Gateway
* 1 EIP
* 2 Route tables  
  
This concludes the **Networking** part of **AWS** set up

Moving on to **Compute and Access Control** configuration automation using Terraform!

**AWS Identity and Access Management**  

[**IAM**](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) and [**Roles**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)  
We want to pass an **IAM role** to our **EC2 instances** to give them **access** to some specific resources, so we need to do the following:  
1. Create [AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)  
   
*Assume Role uses Security Token Service (STS) API that returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access.*  

Adding the following code to a new file named [roles.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PBL/modules/VPC/roles.tf)

``` bash
resource "aws_iam_role" "ec2_instance_role" {
  name = "ec2_instance_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })
  tags = merge(
    var.tags,
    {
      Name = "aws assume role"
    },
  )
}
```
In this code we are creating **AssumeRole** with **AssumeRole policy**. It grants to an entity, in our case it is an **EC2**, permissions to assume the role.  

2. Create **IAM policy** for this role  
   
This is where we need to define a required policy (i.e., permissions) according to our requirements. For example, allowing an **IAM role** to perform action **describe** applied to **EC2** instances:  
``` bash
resource "aws_iam_policy" "policy" {
  name        = "ec2_instance_policy"
  description = "A test policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:Describe*",
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]
})
tags = merge(
    var.tags,
    {
      Name =  "aws assume policy"
    },
  )
}
```

3. Attach the **Policy** to the **IAM Role**
This is where, we will be attaching the policy which we created above, to the role we created in the first step.  
``` bash
resource "aws_iam_role_policy_attachment" "test-attach" {
  role       = aws_iam_role.ec2_instance_role.name
  policy_arn = aws_iam_policy.policy.arn
}
```
4. Create an **Instance Profile** and interpolate the IAM Role  
``` bash
resource "aws_iam_instance_profile" "ip" {
  name = "aws_instance_profile_test"
  role =  aws_iam_role.ec2_instance_role.name
}
```
For now we are done with **Identity and Management**  

### CREATE SECURITY GROUPS

**Terraform Documentation:** [Security Group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) and [Security Group Rule](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule)  


We are going to create all the **security groups** in a single file `security.tf`, then we are going to reference a security group within each resources that needs it  

**NOTE**: We used the `aws_security_group_rule` to reference another **security group** in a **security group**  

<details close>
<summary>security.tf</summary>


``` bash
# security group for alb, to allow acess from any where for HTTP and HTTPS traffic
resource "aws_security_group" "ext-alb-sg" {
  name        = "ext-alb-sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow TLS inbound traffic"
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "HTTPS"
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
  tags = merge(
    var.tags,
    {
      Name = "ext-alb-sg"
    },
  )
}

# security group for bastion, to allow access into the bastion host from you IP
resource "aws_security_group" "bastion_sg" {
  name        = "vpc_web_sg"
  vpc_id = aws_vpc.main.id
  description = "Allow incoming HTTP connections."
  ingress {
    description = "SSH"
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
  tags = merge(
    var.tags,
    {
      Name = "Bastion-SG"
    },
  )
}

#security group for nginx reverse proxy, to allow access only from the extaernal load balancer and bastion instance
resource "aws_security_group" "nginx-sg" {
  name   = "nginx-sg"
  vpc_id = aws_vpc.main.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = merge(
    var.tags,
    {
      Name = "nginx-SG"
    },
  )
}

resource "aws_security_group_rule" "inbound-nginx-http" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.ext-alb-sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}

resource "aws_security_group_rule" "inbound-bastion-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}

# security group for ialb, to have acces only from nginx reverser proxy server
resource "aws_security_group" "int-alb-sg" {
  name   = "my-alb-sg"
  vpc_id = aws_vpc.main.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = merge(
    var.tags,
    {
      Name = "int-alb-sg"
    },
  )
}

resource "aws_security_group_rule" "inbound-ialb-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.nginx-sg.id
  security_group_id        = aws_security_group.int-alb-sg.id
}

# security group for webservers, to have access only from the internal load balancer and bastion instance
resource "aws_security_group" "webserver-sg" {
  name   = "my-asg-sg"
  vpc_id = aws_vpc.main.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = merge(
    var.tags,
    {
      Name = "webserver-sg"
    },
  )
}

resource "aws_security_group_rule" "inbound-web-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.int-alb-sg.id
  security_group_id        = aws_security_group.webserver-sg.id
}

resource "aws_security_group_rule" "inbound-web-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.webserver-sg.id
}

# security group for datalayer to alow traffic from websever on nfs and mysql port and bastiopn host on mysql port
resource "aws_security_group" "datalayer-sg" {
  name   = "datalayer-sg"
  vpc_id = aws_vpc.main.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = merge(
    var.tags,
    {
      Name = "datalayer-sg"
    },
  )
}

resource "aws_security_group_rule" "inbound-nfs-port" {
  type                     = "ingress"
  from_port                = 2049
  to_port                  = 2049
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver-sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-bastion" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-webserver" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver-sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}
```
</details>


### CREATE CERTIFICATE FROM AMAZON CERIFICATE MANAGER

Created [cert.tf](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/edit/main/PBL/modules/ALB/cert.tf) file and add the following code snippets to it.  

**Terraform Documentation**: [AWS Certificate manager](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate) 
<details close>
<summary>cert.tf</summary>

``` bash
# The entire section create a certiface, public zone, and validate the certificate using DNS method
# Create the certificate using a wildcard for all the domains created in hracompany.ga
resource "aws_acm_certificate" "hracompany" {
  domain_name       = "*.hracompany.ga"
  validation_method = "DNS"
  tags = merge(
    var.tags,
    {
      Name = format("%s-Cert", var.name)
    },
  )
}

resource "aws_route53_zone" "hracompany" {
  name = "hracompany.ga"
}

# calling the hosted zone
data "aws_route53_zone" "hracompany" {
  name         = "hracompany.ga"
  private_zone = false
  depends_on    = [aws_route53_zone.hracompany]
}

# selecting validation method
resource "aws_route53_record" "hracompany" {
  for_each = {
    for dvo in aws_acm_certificate.hracompany.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.hracompany.zone_id
}

# validate the certificate through DNS method
resource "aws_acm_certificate_validation" "hracompany" {
  certificate_arn         = aws_acm_certificate.hracompany.arn
  validation_record_fqdns = [for record in aws_route53_record.hracompany : record.fqdn]
}

# create records for tooling
resource "aws_route53_record" "tooling" {
  zone_id = data.aws_route53_zone.hracompany.zone_id
  name    = "tooling.hracompany.ga"
  type    = "A"
  alias {
    name                   = aws_lb.ext-alb.dns_name
    zone_id                = aws_lb.ext-alb.zone_id
    evaluate_target_health = true
  }
}

# create records for wordpress
resource "aws_route53_record" "wordpress" {
  zone_id = data.aws_route53_zone.hracompany.zone_id
  name    = "wordpress.hracompany.ga"
  type    = "A"
  alias {
    name                   = aws_lb.ext-alb.dns_name
    zone_id                = aws_lb.ext-alb.zone_id
    evaluate_target_health = true
  }
}
```
</details>


Creating an **external** *(Internet facing)* **Application Load Balancer (ALB)**  
Create a file called alb.tf  
First we will create the **ALB**, then we create the **target group** and lastly we will create the **listener rule**.  

**Terraform Documentation** of resources:
* [ALB](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb)
* [ALB-target](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group)
* [ALB-listener](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener)
  
We need to create an **ALB** to balance the traffic between the Instances:  
``` bash
resource "aws_lb" "ext-alb" {
  name     = "ext-alb"
  internal = false
  security_groups = [
    aws_security_group.ext-alb-sg.id,
  ]
subnets = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  tags = merge(
    var.tags,
    {
      Name = "ACS-ext-alb"
    },
  )
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```

To inform our **ALB** where to route traffic we need to create a [**Target Group**](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) to point to its targets:  
``` bash
resource "aws_lb_target_group" "nginx-tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }
  name        = "nginx-tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
```

Then we create a **Listener** for this **Target Group**
``` bash
resource "aws_lb_listener" "nginx-listner" {
  load_balancer_arn = aws_lb.ext-alb.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.oyindamola.certificate_arn
default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nginx-tgt.arn
  }
}
```

Adding the following outputs to `output.tf` to print them on screen  
``` bash
output "alb_dns_name" {
  value = aws_lb.ext-alb.dns_name
}
output "alb_target_group_arn" {
  value = aws_lb_target_group.nginx-tgt.arn
}
```



**Create an (Internal) [Application Load Balancer (ALB)](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-internal-load-balancers.html)**   
For the **Internal Load balancer** we will follow the same concepts with the external load balancer.  

Add the code snippets inside the `alb.tf` file  
``` bash
#Internal Load Balancers for webservers
#---------------------------------
resource "aws_lb" "ialb" {
  name     = "ialb"
  internal = true
  security_groups = [
    aws_security_group.int-alb-sg.id,
  ]
  subnets = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  tags = merge(
    var.tags,
    {
      Name = "HRA-int-alb"
    },
  )
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```

To inform our **ALB** to where route the traffic we need to create a [**Target Group**](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) to point to its targets:
``` bash
# --- target group  for wordpress -------
resource "aws_lb_target_group" "wordpress-tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }
name        = "wordpress-tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
# --- target group for tooling -------
resource "aws_lb_target_group" "tooling-tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }
name        = "tooling-tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
```

Then we will need to create a [**Listener**](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html) for this **Target Group** 
``` bash 
# For this aspect a single listener was created for the wordpress which is default,
# A rule was created to route traffic to tooling when the host header changes
resource "aws_lb_listener" "web-listener" {
  load_balancer_arn = aws_lb.ialb.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.hracompany.certificate_arn
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.wordpress-tgt.arn
  }
}
# listener rule for tooling target
resource "aws_lb_listener_rule" "tooling-listener" {
  listener_arn = aws_lb_listener.web-listener.arn
  priority     = 99
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tooling-tgt.arn
  }
  condition {
    host_header {
      values = ["tooling.hracompany.ga"]
    }
  }
}
```

### CREATING AUSTOALING GROUPS


In this Section we will create the **Auto Scaling Group (ASG)** to be able to scale the **EC2s** out and in depending on the application traffic.

Before we start configuring an ASG, we need to create the launch template

Based on our Architecture we need **Auto Scaling Groups** for **bastion**, **nginx**, **wordpress** and **tooling**, so we will create two files     

`asg-bastion-nginx.tf` will contain **Launch Template** and **Auto Scaling Group** for **Bastion** and **Nginx**  
`asg-wordpress-tooling.tf` will contain **Launch Template** and **Auto Scaling group** for **wordpress** and **tooling**  

Useful Terraform Documentation, go through this documentation and understand the arguement needed for each resources:
* [SNS-topic](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic)
* [SNS-notification](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_notification)
* [Austoscaling](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group)
* [Launch-template](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template)

Create `asg-bastion-nginx.tf` and paste all the code snippet below;

<details close>
<summary>asg-bastion-nginx.tf</summary>

``` bash
#Creating SNS Topic for all the Auto Scaling Groups
resource "aws_sns_topic" "hector-sns" {
  name = "Default_CloudWatch_Alarms_Topic"
}


#Creating notification for all the Auto Scaling Groups
resource "aws_autoscaling_notification" "david_notifications" {
  group_names = [
    aws_autoscaling_group.bastion-asg.name,
    aws_autoscaling_group.nginx-asg.name,
    aws_autoscaling_group.wordpress-asg.name,
    aws_autoscaling_group.tooling-asg.name,
  ]
  notifications = [
    "autoscaling:EC2_INSTANCE_LAUNCH",
    "autoscaling:EC2_INSTANCE_TERMINATE",
    "autoscaling:EC2_INSTANCE_LAUNCH_ERROR",
    "autoscaling:EC2_INSTANCE_TERMINATE_ERROR",
  ]
  topic_arn = aws_sns_topic.david-sns.arn
}


#Launch Template for Bastion
resource "random_shuffle" "az_list" {
  input        = data.aws_availability_zones.available.names
}

resource "aws_launch_template" "bastion-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.bastion_sg.id]
  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }
  key_name = var.keypair
  placement {
    availability_zone = "random_shuffle.az_list.result"
  }
  lifecycle {
    create_before_destroy = true
  }
  tag_specifications {
    resource_type = "instance"
    tags = merge(
      var.tags,
      {
        Name = "bastion-launch-template"
      },
    )
  }
  user_data = filebase64("${path.module}/bastion.sh")
}

# ---- Autoscaling for bastion  hosts
resource "aws_autoscaling_group" "bastion-asg" {
  name                      = "bastion-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1
  vpc_zone_identifier = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  launch_template {
    id      = aws_launch_template.bastion-launch-template.id
    version = "$Latest"
  }
  tag {
    key                 = "Name"
    value               = "bastion-launch-template"
    propagate_at_launch = true
  }
}
# launch template for nginx
resource "aws_launch_template" "nginx-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.nginx-sg.id]
  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }
  key_name =  var.keypair
  placement {
    availability_zone = "random_shuffle.az_list.result"
  }
  lifecycle {
    create_before_destroy = true
  }
  tag_specifications {
    resource_type = "instance"
    tags = merge(
      var.tags,
      {
        Name = "nginx-launch-template"
      },
    )
  }
  user_data = filebase64("${path.module}/nginx.sh")
}
# ------ Autoscslaling group for reverse proxy nginx ---------
resource "aws_autoscaling_group" "nginx-asg" {
  name                      = "nginx-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1
  vpc_zone_identifier = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  launch_template {
    id      = aws_launch_template.nginx-launch-template.id
    version = "$Latest"
  }
  tag {
    key                 = "Name"
    value               = "nginx-launch-template"
    propagate_at_launch = true
  }
}
# attaching autoscaling group of nginx to external load balancer
resource "aws_autoscaling_attachment" "asg_attachment_nginx" {
  autoscaling_group_name = aws_autoscaling_group.nginx-asg.id
  alb_target_group_arn   = aws_lb_target_group.nginx-tgt.arn
}
```
</details>


Autoscaling for **wordpress** and **tooling** will be created in a separate file `asg-wordpress-tooling.tf` with the following code  

<details close>
<summary>asg-wordpress-tooling.tf</summary>

``` bash
# launch template for wordpress
resource "aws_launch_template" "wordpress-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.webserver-sg.id]
  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }
  key_name = var.keypair
  placement {
    availability_zone = "random_shuffle.az_list.result"
  }
  lifecycle {
    create_before_destroy = true
  }
  tag_specifications {
    resource_type = "instance"
  tags = merge(
    var.tags,
    {
      Name = "wordpress-launch-template"
    },
  )
}
user_data = filebase64("${path.module}/wordpress.sh")
}
# ---- Autoscaling for wordpress application
resource "aws_autoscaling_group" "wordpress-asg" {
  name                      = "wordpress-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1
  vpc_zone_identifier = [
  aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  launch_template {
    id      = aws_launch_template.wordpress-launch-template.id
    version = "$Latest"
  }
  tag {
    key                 = "Name"
    value               = "wordpress-asg"
    propagate_at_launch = true
  }
}
# attaching autoscaling group of  wordpress application to internal loadbalancer
resource "aws_autoscaling_attachment" "asg_attachment_wordpress" {
  autoscaling_group_name = aws_autoscaling_group.wordpress-asg.id
  alb_target_group_arn   = aws_lb_target_group.wordpress-tgt.arn
}
# launch template for toooling
resource "aws_launch_template" "tooling-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.webserver-sg.id]
  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }
  key_name = var.keypair
  placement {
    availability_zone = "random_shuffle.az_list.result"
  }
  lifecycle {
    create_before_destroy = true
  }
  tag_specifications {
    resource_type = "instance"
    tags = merge(
      var.tags,
      {
        Name = "tooling-launch-template"
      },
    )
  }
  user_data = filebase64("${path.module}/tooling.sh")
}
# ---- Autoscaling for tooling -----
resource "aws_autoscaling_group" "tooling-asg" {
  name                      = "tooling-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1
  vpc_zone_identifier = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  launch_template {
    id      = aws_launch_template.tooling-launch-template.id
    version = "$Latest"
  }
  tag {
    key                 = "Name"
    value               = "tooling-launch-template"
    propagate_at_launch = true
  }
}
# attaching autoscaling group of  tooling application to internal loadbalancer
resource "aws_autoscaling_attachment" "asg_attachment_tooling" {
  autoscaling_group_name = aws_autoscaling_group.tooling-asg.id
  alb_target_group_arn   = aws_lb_target_group.tooling-tgt.arn
}
```
</details>




### STORAGE AND DATABASE

Terraform Documentation:  
* [RDS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_subnet_group)
* [EFS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_file_system)
* [KMS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key)  
  
**Create Elastic File System (EFS)**
In order to create an EFS we need to create a [KMS key](https://aws.amazon.com/kms/getting-started/)  

**AWS Key Management Service (KMS)** makes it easy to create and manage cryptographic keys and control their use across a wide range of AWS services and in applications.  

Creating `efs.tf`  with the following code  

<details close>
<summary>efs.tf</summary>

``` bash
# create key from key management system
resource "aws_kms_key" "HRA-kms" {
  description = "KMS key "
  policy      = <<EOF
  {
  "Version": "2012-10-17",
  "Id": "kms-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::${var.account_no}:user/segun" },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
EOF
}
# create key alias
resource "aws_kms_alias" "alias" {
  name          = "alias/kms"
  target_key_id = aws_kms_key.HRA-kms.key_id
}

#Let us create EFS and it mount targets
# create Elastic file system
resource "aws_efs_file_system" "HRA-efs" {
  encrypted  = true
  kms_key_id = aws_kms_key.HRA-kms.arn
  tags = merge(
    var.tags,
    {
      Name = "HRA-efs"
    },
  )
}

# set first mount target for the EFS 
resource "aws_efs_mount_target" "subnet-1" {
  file_system_id  = aws_efs_file_system.HRA-efs.id
  subnet_id       = aws_subnet.private[2].id
  security_groups = [aws_security_group.datalayer-sg.id]
}
# set second mount target for the EFS 
resource "aws_efs_mount_target" "subnet-2" {
  file_system_id  = aws_efs_file_system.HRA-efs.id
  subnet_id       = aws_subnet.private[3].id
  security_groups = [aws_security_group.datalayer-sg.id]
}
# create access point for wordpress
resource "aws_efs_access_point" "wordpress" {
  file_system_id = aws_efs_file_system.HRA-efs.id
  posix_user {
    gid = 0
    uid = 0
  }
  root_directory {
    path = "/wordpress"
    creation_info {
      owner_gid   = 0
      owner_uid   = 0
      permissions = 0755
    }
  }
}
# create access point for tooling
resource "aws_efs_access_point" "tooling" {
  file_system_id = aws_efs_file_system.HRA-efs.id
  posix_user {
    gid = 0
    uid = 0
  }
  root_directory {
    path = "/tooling"
    creation_info {
      owner_gid   = 0
      owner_uid   = 0
      permissions = 0755
    }
  }
}
```
</details>

  
**Create** [**MySQL RDS**](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MySQL.html)  

Creating **RDS** itself using the following snippet of code in `rds.tf` file  
``` bash
# This section will create the subnet group for the RDS  instance using the private subnet
resource "aws_db_subnet_group" "HRA-rds" {
  name       = "HRA-rds"
  subnet_ids = [aws_subnet.private[2].id, aws_subnet.private[3].id]
  tags = merge(
    var.tags,
    {
      Name = "HRA-rds"
    },
  )
}
# create the RDS instance with the subnets group
resource "aws_db_instance" "HRA-rds" {
  allocated_storage      = 20
  storage_type           = "gp2"
  engine                 = "mysql"
  engine_version         = "5.7"
  instance_class         = "db.t2.micro"
  name                   = "daviddb"
  username               = var.master-username
  password               = var.master-password
  parameter_group_name   = "default.mysql5.7"
  db_subnet_group_name   = aws_db_subnet_group.HRA-rds.name
  skip_final_snapshot    = true
  vpc_security_group_ids = [aws_security_group.datalayer-sg.id]
  multi_az               = "true"
}
```


Declaring in `variables.tf` variables that we previosly gave reference to

<details close>
<summary>variables.tf</summary>

``` bash
variable "region" {
  type = string
  description = "The region to deploy resources"
}
variable "vpc_cidr" {
  type = string
  description = "The VPC cidr"
}
variable "enable_dns_support" {
  type = bool
}
variable "enable_dns_hostnames" {
  dtype = bool
}
variable "enable_classiclink" {
  type = bool
}
variable "enable_classiclink_dns_support" {
  type = bool
}
variable "preferred_number_of_public_subnets" {
  type        = number
  description = "Number of public subnets"
}
variable "preferred_number_of_private_subnets" {
  type        = number
  description = "Number of private subnets"
}
variable "name" {
  type    = string
  default = "ACS"
}
variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
variable "ami" {
  type        = string
  description = "AMI ID for the launch template"
}
variable "keypair" {
  type        = string
  description = "key pair for the instances"
}
variable "account_no" {
  type        = number
  description = "the account number"
}
variable "master-username" {
  type        = string
  description = "RDS admin username"
}
variable "master-password" {
  type        = string
  description = "RDS master password"
}
```
</details>


We need to update `terraform.tfvars`  to declare the **values** for the variables in our `varibales.tf `  

``` bash
region = "us-east-1"
vpc_cidr = "10.0.0.0/16"
enable_dns_support = "true"
enable_dns_hostnames = "true"
enable_classiclink = "false"
enable_classiclink_dns_support = "false"
preferred_number_of_public_subnets = "2"
preferred_number_of_private_subnets = "4"
environment = "production"
ami = "ami-0b0af3577fe5e3532"
keypair = "devops"
# Ensure to change this to your account number
account_no = "199055125796"
db-username = "Hector"
db-password = "Hect0rRodriguez!"
tags = {
  Enviroment      = "production" 
  Owner-Email     = "hectore@email.com"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
```

So far we have a long list of files that is not a bad start, but we are going to fix this using the concepts of **modules** in [Project 18](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/blob/main/PART3_PROJECT_18.md)  

<!--
Secondly, our application wont work because in out shell script that was passed into the launch some endpoints like the RDs and EFS point is needed in which they have not been created yet. So in project 19 we will use our Ansible knowledge to fix this.
-->


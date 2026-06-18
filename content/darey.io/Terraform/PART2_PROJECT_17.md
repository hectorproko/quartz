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
title: "Automating AWS Infrastructure with Terraform - Part 2: Building the Full Stack"
---

> **Part:** 2 of 4  
> **Topics:** Tagging · Internet Gateway · NAT Gateway · Route Tables · IAM Roles · Security Groups · ACM · Load Balancers · Auto Scaling · EFS · KMS · RDS

---

## **Overview**

In [[PART1_PROJECT_16|Part 1: Getting Started]] I set up the Terraform foundation: a dedicated IAM user, the AWS CLI, an S3 bucket, and a VPC with public subnets provisioned through refactored, variable-driven code.

This article builds the rest of the AWS stack on top of that foundation. I add the networking pieces that make the VPC usable (Internet Gateway, NAT Gateway, route tables), then move into compute and access control: IAM roles, security groups, TLS certificates, internal and external load balancers, auto scaling groups with launch templates, and finally the data layer with EFS and an RDS database.

A recurring theme throughout is **keeping the code DRY**, default tags applied everywhere through `merge()`, dynamic names through `format()`, and every value driven by variables rather than hard-coded strings.

---

## Prerequisites

- Completed [[PART1_PROJECT_16|Part 1: Getting Started]] (VPC, subnets, and the refactored `main.tf`/`variables.tf`/`terraform.tfvars` structure)
- The `terraform` IAM user and AWS CLI still configured locally

---

## Step 1 - Apply a Default Tagging Strategy

Before adding more resources, I set up tagging. Good tagging makes infrastructure far easier to operate:

- Resources are organized into logical "virtual" groups
- They can be filtered and searched from the console or programmatically
- The billing team can break down cost by department, environment, or resource type
- Unused resources become easy to spot and clean up
- When multiple teams share an account, tags make ownership clear

Rather than tagging each resource individually, I define a **default set of tags once** in `terraform.tfvars`:

```bash
tags = {
  Enviroment      = "development"
  Owner-Email     = "hector@email.com"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
```

Then every resource merges those defaults with its own `Name` tag using the built-in `merge()` function:

```bash
tags = merge(
  var.tags,
  {
    Name = "Name of the resource"
  },
)
```

For this to work, the `tags` variable must be declared in `variables.tf`:

```bash
variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```

> The payoff: when a tag value needs to change, I edit it in **one place** (`terraform.tfvars`) and it propagates to every resource.

---

## Step 2 - Internet Gateway (and the `format()` Function)

I create the **Internet Gateway** in its own file, `internet_gateway.tf`, so networking resources stay grouped.

```bash
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.main.id
  tags = merge(
    var.tags,
    {
      Name = format("%s-%s!", aws_vpc.main.id, "IG")
    },
  )
}
```

The `format()` function builds a dynamic, unique name. In the example above:

|Placeholder|Resolves to|
|---|---|
|First `%s`|The interpolated value of `aws_vpc.main.id`|
|Second `%s`|The literal string `IG`|
|`!`|A trailing literal character|

This becomes essential when a resource is created with `count` or in a loop, where each instance needs a **unique** key-value pair. For example, giving each subnet a distinct name:

```bash
tags = merge(
  var.tags,
  {
    Name = format("PrivateSubnet-%s", count.index)
  },
)
```

Which produces:

```
Name = PrivateSubnet-0
Name = PrivateSubnet-1
Name = PrivateSubnet-2
```

---

## Step 3 - NAT Gateway & Elastic IP

A NAT Gateway lets instances in private subnets reach the internet (for updates, package installs) without being publicly reachable. It needs an Elastic IP, and the Internet Gateway must exist first.

I create both in a new file, `natgateway.tf`, and use [`depends_on`](https://www.terraform.io/language/meta-arguments/depends_on) to make the dependency on the Internet Gateway **explicit**. Terraform usually infers dependencies, but being explicit here avoids race conditions.

```bash
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

---

## Step 4 - Route Tables (Public & Private)

To control traffic flow I create routes for both public and private subnets in `route_tables.tf`, using three resource types: `aws_route_table`, `aws_route`, and `aws_route_table_association`.

The pattern is: create a route table, associate every relevant subnet to it with a `count` loop, and (for the public side) add a route to the Internet Gateway.

<details><summary>route_tables.tf</summary>

```bash
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

</details>

**Networking checkpoint.** Running `terraform plan` and `terraform apply` now provisions a complete multi-AZ network:

|Resource|Count|
|---|---|
|VPC|1|
|Public subnets|2|
|Private subnets|4|
|Internet Gateway|1|
|NAT Gateway|1|
|Elastic IP|1|
|Route tables|2|

That completes the **networking** layer. Next comes **compute and access control**.

---

## Step 5 - IAM Roles & Instance Profile

I want to pass an **IAM role** to the EC2 instances so they can access specific AWS resources without embedding credentials. This is done in a new file, `roles.tf`, in four parts.

### 1. Create the AssumeRole

[AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) uses the Security Token Service (STS) to return **temporary** credentials (access key, secret key, and token) for accessing resources the entity wouldn't normally reach. Here the entity is the EC2 service.

```bash
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

### 2. Create the IAM Policy

The policy defines what the role is actually allowed to do. This example grants permission to run `describe` actions on EC2:

```bash
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
      Name = "aws assume policy"
    },
  )
}
```

### 3. Attach the Policy to the Role

```bash
resource "aws_iam_role_policy_attachment" "test-attach" {
  role       = aws_iam_role.ec2_instance_role.name
  policy_arn = aws_iam_policy.policy.arn
}
```

### 4. Create an Instance Profile

The instance profile is the wrapper that actually gets attached to EC2 instances, interpolating the IAM role:

```bash
resource "aws_iam_instance_profile" "ip" {
  name = "aws_instance_profile_test"
  role = aws_iam_role.ec2_instance_role.name
}
```

---

## Step 6 - Security Groups

All security groups live in a single file, `security.tf`, and each resource that needs one references it. I use the standalone `aws_security_group_rule` resource whenever a group needs to reference **another** security group as its source (a common pattern that avoids circular dependencies).

The design mirrors the reverse-proxy architecture: the external ALB is open to the internet, nginx accepts traffic only from the external ALB and the bastion, the internal ALB accepts only from nginx, the webservers accept only from the internal ALB and bastion, and the data layer accepts only NFS/MySQL traffic from the webservers and bastion.

<details><summary>security.tf</summary>

```bash
# security group for alb, to allow access from anywhere for HTTP and HTTPS traffic
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

# security group for bastion, to allow access into the bastion host from your IP
resource "aws_security_group" "bastion_sg" {
  name        = "vpc_web_sg"
  vpc_id      = aws_vpc.main.id
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

# security group for nginx reverse proxy, to allow access only from the external load balancer and bastion instance
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

# security group for internal alb, to have access only from nginx reverse proxy server
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

# security group for datalayer to allow traffic from webserver on NFS and MySQL port and bastion host on MySQL port
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

---

## Step 7 - TLS Certificate with ACM

To serve HTTPS, I create a wildcard certificate through AWS Certificate Manager and validate it via DNS, all in `cert.tf`. This block creates the certificate, a public Route 53 hosted zone, the validation records, and the A-records that point the application hostnames at the external ALB.

<details><summary>cert.tf</summary>

```bash
# This section creates a certificate, public zone, and validates the certificate using the DNS method
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
  depends_on   = [aws_route53_zone.hracompany]
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

> 👀 **Watch the references:** the listener in the next step must reference the **same** certificate validation resource name used here (`aws_acm_certificate_validation.hracompany`). A mismatched name is one of the most common copy-paste errors in this project and produces an "unsupported attribute" / unknown resource error at plan time.

---

## Step 8 - Application Load Balancers

### External (Internet-facing) ALB

In `alb.tf`, I create the external ALB, its target group, and a listener. The ALB sits in the two public subnets and uses the external ALB security group.

```bash
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

The **target group** tells the ALB where to route traffic, including health-check settings:

```bash
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

The **listener** ties the ALB to the target group on port 443, using the validated ACM certificate:

```bash
resource "aws_lb_listener" "nginx-listner" {
  load_balancer_arn = aws_lb.ext-alb.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.hracompany.certificate_arn
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nginx-tgt.arn
  }
}
```

I expose the ALB DNS name and target group ARN in `output.tf` so they print after `apply`:

```bash
output "alb_dns_name" {
  value = aws_lb.ext-alb.dns_name
}
output "alb_target_group_arn" {
  value = aws_lb_target_group.nginx-tgt.arn
}
```

### Internal ALB

The internal ALB follows the same pattern but sits in the **private** subnets (`internal = true`) and routes to the WordPress and tooling target groups. A single listener defaults to WordPress, and a listener **rule** routes to tooling when the host header matches.

<details><summary>alb.tf (internal load balancer)</summary>

```bash
# Internal Load Balancer for webservers
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

# --- target group for wordpress -------
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

# A single default listener for wordpress; a rule routes to tooling on host-header match
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

</details>

---

## Step 9 - Auto Scaling Groups & Launch Templates

Auto Scaling Groups (ASGs) scale the EC2 instances in and out with demand. Each ASG needs a **launch template** first. Based on the architecture there are four roles **bastion**, **nginx**, **wordpress**, and **tooling** split across two files:

- `asg-bastion-nginx.tf` - launch templates and ASGs for bastion and nginx
- `asg-wordpress-tooling.tf` - launch templates and ASGs for wordpress and tooling

I also create one SNS topic and an autoscaling notification so launch/terminate events (and their errors) are reported.

> 👀 **Consistent naming:** the SNS topic below is named `hector-sns`, so the notification's `topic_arn` must reference `aws_sns_topic.hector-sns.arn`. The original tutorial code mixed in a different name here, make sure the topic name and the reference match, or `terraform plan` fails with a reference error.

<details><summary>asg-bastion-nginx.tf</summary>

```bash
# Creating SNS Topic for all the Auto Scaling Groups
resource "aws_sns_topic" "hector-sns" {
  name = "Default_CloudWatch_Alarms_Topic"
}

# Creating notification for all the Auto Scaling Groups
resource "aws_autoscaling_notification" "hector_notifications" {
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
  topic_arn = aws_sns_topic.hector-sns.arn
}

# Randomly shuffle the AZ list for instance placement
resource "random_shuffle" "az_list" {
  input = data.aws_availability_zones.available.names
}

# Launch Template for Bastion
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

# ---- Autoscaling for bastion hosts ----
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

# Launch template for nginx
resource "aws_launch_template" "nginx-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.nginx-sg.id]
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
        Name = "nginx-launch-template"
      },
    )
  }
  user_data = filebase64("${path.module}/nginx.sh")
}

# ---- Autoscaling group for reverse proxy nginx ----
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

<details><summary>asg-wordpress-tooling.tf</summary>

```bash
# Launch template for wordpress
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

# ---- Autoscaling for wordpress application ----
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

# attaching autoscaling group of wordpress application to internal load balancer
resource "aws_autoscaling_attachment" "asg_attachment_wordpress" {
  autoscaling_group_name = aws_autoscaling_group.wordpress-asg.id
  alb_target_group_arn   = aws_lb_target_group.wordpress-tgt.arn
}

# Launch template for tooling
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

# ---- Autoscaling for tooling ----
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

# attaching autoscaling group of tooling application to internal load balancer
resource "aws_autoscaling_attachment" "asg_attachment_tooling" {
  autoscaling_group_name = aws_autoscaling_group.tooling-asg.id
  alb_target_group_arn   = aws_lb_target_group.tooling-tgt.arn
}
```

</details>

---

## Step 10 - Storage & Database (EFS, KMS, RDS)

The data layer is shared, encrypted storage plus a managed database.

**Terraform documentation:** [RDS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_subnet_group) · [EFS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_file_system) · [KMS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key)

### Elastic File System (EFS)

EFS needs a [KMS key](https://aws.amazon.com/kms/getting-started/) for encryption. **AWS Key Management Service** creates and manages the cryptographic keys used across AWS services. I create the key, an EFS file system encrypted with it, two mount targets (one per private data subnet), and access points for WordPress and tooling, all in `efs.tf`.

<details><summary>efs.tf</summary>

```bash
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
      "Principal": { "AWS": "arn:aws:iam::${var.account_no}:user/terraform" },
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

### MySQL RDS

The database goes in `rds.tf`. A DB subnet group places it across two private subnets, and the instance itself pulls credentials from variables (never hard-coded) and attaches the data-layer security group.

```bash
# This section creates the subnet group for the RDS instance using the private subnets
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

# create the RDS instance with the subnet group
resource "aws_db_instance" "HRA-rds" {
  allocated_storage      = 20
  storage_type           = "gp2"
  engine                 = "mysql"
  engine_version         = "5.7"
  instance_class         = "db.t2.micro"
  name                   = "hectordb"
  username               = var.master-username
  password               = var.master-password
  parameter_group_name   = "default.mysql5.7"
  db_subnet_group_name   = aws_db_subnet_group.HRA-rds.name
  skip_final_snapshot    = true
  vpc_security_group_ids = [aws_security_group.datalayer-sg.id]
  multi_az               = "true"
}
```

---

## Step 11 - Centralize Variables

Several resources above reference variables (`var.ami`, `var.keypair`, `var.account_no`, `var.master-username`, etc.). I declare all of them in `variables.tf` with types and descriptions.

<details><summary>variables.tf</summary>

```bash
variable "region" {
  type        = string
  description = "The region to deploy resources"
}
variable "vpc_cidr" {
  type        = string
  description = "The VPC cidr"
}
variable "enable_dns_support" {
  type = bool
}
variable "enable_dns_hostnames" {
  type = bool
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

And set their values in `terraform.tfvars`.

> 👀 **Match the variable names exactly.** The RDS instance uses `var.master-username` and `var.master-password`, so the tfvars keys must be `master-username` / `master-password` (not `db-username` / `db-password`). A name mismatch here leaves the RDS credentials unset at apply time. The corrected file is below.

<details><summary>terraform.tfvars</summary>

```bash
region                              = "us-east-1"
vpc_cidr                            = "10.0.0.0/16"
enable_dns_support                  = "true"
enable_dns_hostnames                = "true"
enable_classiclink                  = "false"
enable_classiclink_dns_support      = "false"
preferred_number_of_public_subnets  = "2"
preferred_number_of_private_subnets = "4"
environment                         = "production"
ami                                 = "ami-0b0af3577fe5e3532"
keypair                             = "devops"
# Ensure to change this to your account number
account_no      = "199055125796"
master-username = "Hector"
master-password = "<your-strong-password>"
tags = {
  Enviroment      = "production"
  Owner-Email     = "hector@email.com"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
```

</details>

> ⚠️ **Never commit real secrets.** `terraform.tfvars` contains the database password. Keep it out of version control (add it to `.gitignore`) and use a placeholder like `<your-strong-password>` in anything you publish.

---

## Summary

By the end of Part 2 the full application stack is defined as code:

- ✅ A default tagging strategy applied everywhere via `merge()`
- ✅ Internet Gateway, NAT Gateway + EIP, and public/private route tables (full multi-AZ networking)
- ✅ IAM role, policy, attachment, and instance profile for EC2
- ✅ A complete set of layered security groups
- ✅ A wildcard ACM certificate validated via Route 53 DNS
- ✅ External and internal Application Load Balancers with target groups and listener rules
- ✅ Auto Scaling Groups + launch templates for bastion, nginx, wordpress, and tooling
- ✅ Encrypted EFS (with KMS) and a multi-AZ MySQL RDS instance
- ✅ Every value driven by `variables.tf` and `terraform.tfvars`

At this point the project is **one long list of `.tf` files** functional, but hard to navigate. [[PART3_PROJECT18_Backends|Part 3: Remote Backends & Modules]] fixes that by moving state to a remote S3 backend with DynamoDB locking, and by refactoring everything into reusable **modules**.

---

_GitHub Repository: [AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4)_

<!--
ORIGINAL RAW NOTES (Project 17) - preserved for reference

Best practices Tagging
Tagging helps you manage your resources much more efficiently:
- Resources are much better organized in 'virtual' groups
- They can be easily filtered and searched from console or programmatically
- Billing team can easily generate reports and determine how much each part of infrastructure costs
- You can easily determine resources that are not being used and take actions accordingly
- If there are different teams in the organization using the same account, tagging can help differentiate who owns which resources

Internet Gateways & format() function
NAT Gateways - Create 1 NAT Gateways and 1 Elastic IP (EIP) addresses
AWS ROUTES - route_tables.tf with aws_route_table, aws_route, aws_route_table_association
After apply: VPC, 2 Public subnets, 4 Private subnets, 1 IG, 1 NAT GW, 1 EIP, 2 Route tables
AWS Identity and Access Management - AssumeRole, IAM policy, attach policy, instance profile
CREATE SECURITY GROUPS - security.tf, aws_security_group_rule to reference another SG
CREATE CERTIFICATE FROM AMAZON CERTIFICATE MANAGER - cert.tf
External + Internal ALB - alb.tf, target groups, listeners, output.tf
CREATING AUTOSCALING GROUPS - asg-bastion-nginx.tf, asg-wordpress-tooling.tf, SNS topic + notification
STORAGE AND DATABASE - efs.tf (KMS + EFS), rds.tf (MySQL)
variables.tf + terraform.tfvars

NOTE (original known issues to be aware of):
- SNS notification referenced aws_sns_topic.david-sns.arn while the topic was hector-sns
- nginx listener referenced aws_acm_certificate_validation.oyindamola while the cert was hracompany
- RDS name was "daviddb"; tfvars used db-username/db-password while variables.tf/rds.tf used master-username/master-password

NEXT (Project 19 note): the application won't fully work yet because the launch shell scripts
reference endpoints (RDS, EFS) that need to be wired in; Ansible is used later to fix this.
-->


<!--
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


# DEPLOYING A RENTAL COMPANY APP WITH TERRAFORM ON AWS (S3 & RDS)

Project overview:

The project encapsulated deploying a dynamic website on AWS using Terraform, focusing on
creating a robust, scalable infrastructure. The process started with version control setup in
GitHub, followed by configuring AWS connectivity and S3 bucket creation for the Terraform
state file. A comprehensive 3-tier VPC architecture was established, including internet gateway
setup, subnet creation, and NAT gateways for internet access in private subnets. Security groups
were configured for various components. The database was set up using an RDS snapshot,
followed by implementing an Application Load Balancer (ALB) with target groups and listeners
to manage traffic. Auto Scaling groups were created for dynamic scaling, and SNS topics were
used for notifications. Finally, DNS settings were configured in Route 53 to point a domain to
the ALB, successfully making the website accessible and secure.


## CONFIGURING AWS PROVIDER AND S3 BACKEND
After cloning my repository to my local machine, In my main.tf folder, I configured AWS provider to establish a secure connection between AWS and Terraform & store my terraform state file in S3 then initialised terraform with “terraform init”

```
# configure aws provider

provider "aws" {
  region = "us-east-1"
  profile = "programmatic-user"
}
```
```
# stores the terraform state file in s3
terraform {
  backend "s3" {
    bucket = "zayd-terraform-remote-state"
    key    = "terraform.tfstate.dev"
    region = "us-east-1"
    profile = "programmatic-user"
  }
}
```
![image](https://github.com/user-attachments/assets/6d572f1e-114c-4b54-9924-826016208aed)
## VPC, SUBNETS AND INTERNET GATEWAY

I created a 3-tier VPC architecture, comprising of two public subnet and four private subnets. In my
vpc.tf file I configured the AWS VPC with DNS hostnames enabled
```
# create vpc
# terraform aws create vpc
resource "aws_vpc" "vpc" {
  cidr_block              = var.vpc_cidr
  instance_tenancy        = "default"
  enable_dns_hostnames    = true

  tags      = {
    Name    = "dev vpc"
  }
}
```
I also created an Internet gateway and attached it to the vpc for access to the internet.
```
# create internet gateway and attach it to vpc

resource "aws_internet_gateway" "internet_gateway" {
  vpc_id    = aws_vpc.vpc.id

  tags      = {
    Name    = "dev internet gateway"
  }
}
```
The public subnets were created in the vpc to their respective availability zones and assigned
public IPv4 addresses on launch
```
# create public subnet az1
# terraform aws create subnet
resource "aws_subnet" "public_subnet_az1" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.public_subnet_az1_cidr
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags      = {
    Name    = "public subnet az1"
  }
}

# create public subnet az2
# terraform aws create subnet
resource "aws_subnet" "public_subnet_az2" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.public_subnet_az2_cidr
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true

  tags      = {
    Name    = "public subnet az2"
  }
}
```
In the next step, I created a public route table for access from anywhere on the internet through the
internet gateway then I associated both public subnets to the route table.
```
# create route table and add public route
# terraform aws create route table
resource "aws_route_table" "public_route_table" {
  vpc_id       = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.internet_gateway.id
  }

  tags       = {
    Name     = "public route table"
  }
}

# associate public subnet az1 to "public route table"
# terraform aws associate subnet with route table
resource "aws_route_table_association" "public_subnet_az1_route_table_association" {
  subnet_id           = aws_subnet.public_subnet_az1.id
  route_table_id      = aws_route_table.public_route_table.id
}

# associate public subnet az2 to "public route table"
# terraform aws associate subnet with route table
resource "aws_route_table_association" "public_subnet_2_route_table_association" {
  subnet_id           = aws_subnet.public_subnet_az2.id
  route_table_id      = aws_route_table.public_route_table.id
}
```
Then I created the two private app subnets and the two data subnet, these didn’t have public IPv4 address as they are private subnets and I dont want to expose them to direct traffic from the internet
```
# create private app subnet az1
# terraform aws create subnet
resource "aws_subnet" "private_app_subnet_az1" {
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = var.private_app_subnet_az1_cidr
  availability_zone        = "us-east-1a"
  map_public_ip_on_launch  = false

  tags      = {
    Name    = "private app subnet az1"
  }
}

# create private app subnet az2
# terraform aws create subnet
resource "aws_subnet" "private_app_subnet_az2" {
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = var.private_app_subnet_az2_cidr
  availability_zone        = "us-east-1b"
  map_public_ip_on_launch  = false

  tags      = {
    Name    = "private app subnet az2"
  }
}

# create private data subnet az1
# terraform aws create subnet
resource "aws_subnet" "private_data_subnet_az1" {
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = var.private_data_subnet_az1_cidr
  availability_zone        = "us-east-1a"
  map_public_ip_on_launch  = false

  tags      = {
    Name    = "private data subnet az1"
  }
}

# create private data subnet az2
# terraform aws create subnet
resource "aws_subnet" "private_data_subnet_az2" {
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = var.private_data_subnet_az2_cidr
  availability_zone        = "us-east-1b"
  map_public_ip_on_launch  = false

  tags      = {
    Name    = "private data subnet az2"
  }
}
```
## NAT GATEWAY 
I created two NAT gateways in public subnet AZ1 and AZ2 respectively and allocated elastic ip
addresses to them. This setup is designed to route internet traffic through the NAT gateways located in the public subnets, with dedicated private route tables established in each zone for this purpose. These
route tables are connected to the private subnets in their respective availability zones
```
# allocate elastic ip. this eip will be used for the nat-gateway in the public subnet az1 
# terraform aws allocate elastic ip
resource "aws_eip" "eip_for_nat_gateway_az1" {
  vpc    = true

  tags   = {
    Name = "nat gateway az1 eip"
  }
}

# allocate elastic ip. this eip will be used for the nat-gateway in the public subnet az2
# terraform aws allocate elastic ip
resource "aws_eip" "eip_for_nat_gateway_az2" {
  vpc    = true

  tags   = {
    Name = "nat gateway az2 eip"
  }
}

# create nat gateway in public subnet az1
# terraform create aws nat gateway
resource "aws_nat_gateway" "nat_gateway_az1" {
  allocation_id = aws_eip.eip_for_nat_gateway_az1.id
  subnet_id     = aws_subnet.public_subnet_az1.id

  tags   = {
    Name = "nat gateway az1"
  }

  # to ensure proper ordering, it is recommended to add an explicit dependency
  # on the internet gateway for the vpc.
  depends_on = [aws_internet_gateway.internet_gateway]
}

# create nat gateway in public subnet az2
# terraform create aws nat gateway
resource "aws_nat_gateway" "nat_gateway_az2" {
  allocation_id = aws_eip.eip_for_nat_gateway_az2.id
  subnet_id     = aws_subnet.public_subnet_az2.id

  tags   = {
    Name = "nat gateway az2"
  }

  # to ensure proper ordering, it is recommended to add an explicit dependency
  # on the internet gateway for the vpc.
  depends_on = [aws_internet_gateway.internet_gateway]
}

# create private route table az1 and add route through nat gateway az1
# terraform aws create route table
resource "aws_route_table" "private_route_table_az1" {
  vpc_id            = aws_vpc.vpc.id

  route {
    cidr_block      = "0.0.0.0/0"
    nat_gateway_id  = aws_nat_gateway.nat_gateway_az1.id
  }

  tags   = {
    Name = "private route table az1"
  }
}

# associate private app subnet az1 with private route table az1
# terraform aws associate subnet with route table
resource "aws_route_table_association" "private_app_subnet_az1_route_table_az1_association" {
  subnet_id         = aws_subnet.private_app_subnet_az1.id
  route_table_id    = aws_route_table.private_route_table_az1.id
}

# associate private data subnet az1 with private route table az1
# terraform aws associate subnet with route table
resource "aws_route_table_association" "private_data_subnet_az1_route_table_az1_association" {
  subnet_id         = aws_subnet.private_data_subnet_az1.id
  route_table_id    = aws_route_table.private_route_table_az1.id
}

# create private route table az2 and add route through nat gateway az2
# terraform aws create route table
resource "aws_route_table" "private_route_table_az2" {
  vpc_id            = aws_vpc.vpc.id

  route {
    cidr_block      = "0.0.0.0/0"
    nat_gateway_id  = aws_nat_gateway.nat_gateway_az2.id
  }

  tags   = {
    Name = "private route table az2"
  }
}

# associate private app subnet az2 with private route table az2
# terraform aws associate subnet with route table
resource "aws_route_table_association" "private_app_subnet_az2_route_table_az2_association" {
  subnet_id         = aws_subnet.private_app_subnet_az2.id
  route_table_id    = aws_route_table.private_route_table_az2.id
}

# associate private data subnet az2 with private route table az2
# terraform aws associate subnet with route table
resource "aws_route_table_association" "private_data_subnet_az2_route_table_az2_association" {
  subnet_id         = aws_subnet.private_data_subnet_az2.id
  route_table_id    = aws_route_table.private_route_table_az2.id
}
```
## SECURITY GROUPS
In this phase, I configured the security groups for the application load balancer (allowing inbound
http and https traffic from port 80 and 443), the bastion host (allowing ssh on port 22 from my IP),
the webserver EC2 Instance (inbound traffic on port 80,443 and 22) and the database (allowing
ingress traffic on port 3306 -MySQL)
```
# create security group for the application load balancer
# terraform aws create security group
resource "aws_security_group" "alb_security_group" {
  name        = "alb security group"
  description = "enable http/https access on port 80/443"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description      = "http access"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  ingress {
    description      = "https access"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = -1
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags   = {
    Name = "alb security group"
  }
}

# create security group for the bastion host aka jump box
# terraform aws create security group
resource "aws_security_group" "ssh_security_group" {
  name        = "ssh security group"
  description = "enable ssh access on port 22"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description      = "ssh access"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = [var.ssh_location]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = -1
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags   = {
    Name = "ssh security group"
  }
}

# create security group for the web server
# terraform aws create security group
resource "aws_security_group" "webserver_security_group" {
  name        = "webserver security group"
  description = "enable http/https access on port 80/443 via alb sg and access on port 22 via ssh sg"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description      = "http access"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    security_groups  = [aws_security_group.alb_security_group.id]
  }

  ingress {
    description      = "https access"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    security_groups  = [aws_security_group.alb_security_group.id]
  }

  ingress {
    description      = "ssh access"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    security_groups  = [aws_security_group.ssh_security_group.id]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = -1
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags   = {
    Name = "webserver security group"
  }
}

# create security group for the database
# terraform aws create security group
resource "aws_security_group" "database_security_group" {
  name        = "database security group"
  description = "enable mysql/aurora access on port 3306"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description      = "mysql/aurora access"
    from_port        = 3306
    to_port          = 3306
    protocol         = "tcp"
    security_groups  = [aws_security_group.webserver_security_group.id]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = -1
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags   = {
    Name = "database security group"
  }
}
```
## RDS INSTANCE
In this phase, I restored my RDS database from a snapshot I took when I built this website initially in
the management console. Firstly, I created the database subnet group then I retrieved the
database snapshot which I then used to created my database instance
```
# create database subnet group
# terraform aws db subnet group
resource "aws_db_subnet_group" "database_subnet_group" {
  name         = "database subnets"
  subnet_ids   = [aws_subnet.private_data_subnet_az1.id, aws_subnet.private_data_subnet_az2.id]
  description  = "subnets for database instance"

  tags   = {
    Name = "database subnets"
  }
}

# get the latest db snapshot
# terraform aws data db snapshot
data "aws_db_snapshot" "latest_db_snapshot" {
  db_snapshot_identifier = var.database_snapshot_identifier
  most_recent            = true
  snapshot_type          = "manual"
}

# create database instance restored from db snapshots
# terraform aws db instance
resource "aws_db_instance" "database_instance" {
  instance_class          = var.database_instance_class
  skip_final_snapshot     = true
  availability_zone       = "us-east-1b"
  identifier              = var.database_instance_identifier
  snapshot_identifier     = data.aws_db_snapshot.latest_db_snapshot.id
  db_subnet_group_name    = aws_db_subnet_group.database_subnet_group.name
  multi_az                = var.multi_az_deployment
  vpc_security_group_ids  = [aws_security_group.database_security_group.id]
}
```
## APPLICATION LOAD BALANCER & TARGET GROUPS
For the subsequent phase, I established an Application Load Balancer (ALB) to direct traffic towards both public subnets & positioning it behind the ALB security group. Next, I configured a target group behind the load balancer and a listener on port 80 that redirects traffic to port 443 to ensure a secure connection. Finally I added a listener on port 443 with forwarding action to forward traffic to my target group
```
# create application load balancer
# terraform aws create application load balancer
resource "aws_lb" "application_load_balancer" {
  name               = "dev-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_security_group.id]

  subnet_mapping {
    subnet_id = aws_subnet.public_subnet_az1.id
  }

  subnet_mapping {
    subnet_id = aws_subnet.public_subnet_az2.id
  }

  enable_deletion_protection = false

  tags   = {
    Name = "dev-alb"
  }
}

# create target group
# terraform aws create target group
resource "aws_lb_target_group" "alb_target_group" {
  name        = "dev-tg"
  target_type = "instance"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.vpc.id

  health_check {
    healthy_threshold   = 5
    interval            = 30
    matcher             = "200,301,302"
    path                = "/"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
}

# create a listener on port 80 with redirect action
# terraform aws create listener
resource "aws_lb_listener" "alb_http_listener" {
  load_balancer_arn = aws_lb.application_load_balancer.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      host        = "#{host}"
      path        = "/#{path}"
      port        = 443
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# create a listener on port 443 with forward action
# terraform aws create listener
resource "aws_lb_listener" "alb_https_listener" {
  load_balancer_arn  = aws_lb.application_load_balancer.arn
  port               = 443
  protocol           = "HTTPS"
  ssl_policy         = "ELBSecurityPolicy-2016-08"
  certificate_arn    = var.ssl_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.alb_target_group.arn
  }
}
```
## AUTO SCALING GROUPS, SNS AND ROUTE 53
I created a launch template; the launch template defines the configuration for instances that the
auto scaling group will use. the auto scaling group manages instance lifecycle and scaling based
on defined criteria, a minimum of one instance, a maximum of two and a desired capacity of 3.
and attaching the Auto Scaling group to the ALB target group ensures that incoming traffic is
distributed across healthy instances managed by the Auto Scaling group. Finally, an SNS topic to
notify me of changes to the instance status.
I concluded the project by configuring DNS settings for my application by retrieving details of
my hosted zone in AWS Route 53 and creating an A record that points to the Application Load
Balancer (ALB)
```
# create a launch template
# terraform aws launch template
resource "aws_launch_template" "webserver_launch_template" {
  name          = var.launch_template_name
  image_id      = var.ec2_image_id
  instance_type = var.ec2_instance_type
  key_name      = var.ec2_key_pair_name
  description   = "launch template for asg"

  monitoring {
    enabled = true
  }

  vpc_security_group_ids = [aws_security_group.webserver_security_group.id]
}

# create auto scaling group
# terraform aws autoscaling group
resource "aws_autoscaling_group" "auto_scaling_group" {
  vpc_zone_identifier = [aws_subnet.private_app_subnet_az1.id, aws_subnet.private_app_subnet_az2.id]
  desired_capacity    = 2
  max_size            = 4
  min_size            = 1
  name                = "dev-asg"
  health_check_type   = "ELB"

  launch_template {
    name    = aws_launch_template.webserver_launch_template.name
    version = "$Latest"
  }

  tag {
    key                 = "name"
    value               = "asg-webserver"
    propagate_at_launch = "true"
  }

  lifecycle {
    ignore_changes      = [target_group_arns]
  }
}

# attach auto scaling group to alb target group
# terraform aws autoscaling attachment
resource "aws_autoscaling_attachment" "asg_alb_target_group_attachment" {
  autoscaling_group_name = aws_autoscaling_group.auto_scaling_group.id
  lb_target_group_arn    = aws_lb_target_group.alb_target_group.arn
}

# create an auto scaling group notification
# terraform aws autoscaling notification
resource "aws_autoscaling_notification" "webserver_asg_notifications" {
  group_names = [aws_autoscaling_group.auto_scaling_group.name]

  notifications = [
    "autoscaling:EC2_INSTANCE_LAUNCH",
    "autoscaling:EC2_INSTANCE_TERMINATE",
    "autoscaling:EC2_INSTANCE_LAUNCH_ERROR",
    "autoscaling:EC2_INSTANCE_TERMINATE_ERROR",
  ]

  topic_arn = aws_sns_topic.user_updates.arn
}
```
```
# get hosted zone details
# terraform aws data hosted zone
data "aws_route53_zone" "hosted_zone" {
  name = var.domain_name
}

# create a record set in route 53
# terraform aws route 53 record
resource "aws_route53_record" "site_domain" {
  zone_id = data.aws_route53_zone.hosted_zone.zone_id
  name    = var.record_name
  type    = "A"

  alias {
    name                   = aws_lb.application_load_balancer.dns_name
    zone_id                = aws_lb.application_load_balancer.zone_id
    evaluate_target_health = true
  }
}
```
```
# create an sns topic
# terraform aws create sns topic
resource "aws_sns_topic" "user_updates" {
  name      = "dev-sns-topic"
}

# create an sns topic subscription
# terraform aws sns topic subscription
resource "aws_sns_topic_subscription" "notification_topic" {
  topic_arn = aws_sns_topic.user_updates.arn
  protocol  = "email"
  endpoint  = var.operator_email
}
```
![image](https://github.com/user-attachments/assets/fff5b280-f8ec-431a-b67e-f0dc6b8f1194)

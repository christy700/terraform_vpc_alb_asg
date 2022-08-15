# terraform_vpc_alb_asg

```
##keyname

ssh-keygen 

keyname: maininfra


##userdata
##vim mainsetup.sh

#!/bin/bash


echo "ClientAliveInterval 60" >> /etc/ssh/sshd_config
echo "LANG=en_US.utf-8" >> /etc/environment
echo "LC_ALL=en_US.utf-8" >> /etc/environment
service sshd restart


yum install httpd php git -y
systemctl restart httpd.service
systemctl enable httpd.service

git clone https://github.com/christy700/aws-elb-site.git /var/website/
cp -r /var/website/*  /var/www/html/
chown -R apache:apache /var/www/html/*





##vim provider.tf

provider "aws" {
  region = "ap-south-1"
}


##vim variables.tf

variable "aws_region" {
  default = "ap-south-1"
}

variable "vpc_cidr" {
    
  default = "172.17.0.0/16"
}

variable "project_name" {
  default = "timetrav"
}

variable "project_env" {
  default = "dev"
}

variable "sec_rules" {
  type = list
  default = ["22" , "80" , "443"]
}

variable "ami_name" {
  default = "amzn2-ami-kernel-5.10-hvm-*-x86_64-gp2"
}

variable "instance_type" {
  default = "t2.micro"
}

variable "domain_details" {
 type = map(any)
 default = {
  "root_dname" = "timetrav.online"
  "domain_name" = "main.timetrav.online"
 }
}

variable "asg_details" {
 type = map(any)
 default = {
  "desired_capacity" = 2
  "max_size" = 2
  "min_size" = 2
 }
}


##vim datasource.tf



data "aws_availability_zones" "available" {
  state = "available"
}


data "aws_ami" "main_ami" {
  most_recent      = true
  owners           = ["amazon"]

  filter {
    name   = "name"
    values = [var.ami_name]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

data "aws_acm_certificate" "ssl_certificate" {
  domain    = var.domain_details.root_dname
  statuses = ["ISSUED"]
}

data "aws_route53_zone" "myzone_id" {
  name         = var.domain_details.root_dname
}


data "aws_instances" "main_instance" {

  filter {
    name   = "tag:aws:autoscaling:groupName"
    values = [aws_autoscaling_group.main_asg.name]
  }
  instance_state_names = ["running"]
}






##vim main.tf


#++++++++++++++++++
#CREATING VPC INFRA
#++++++++++++++++++

resource "aws_vpc" "main_vpc" {
  cidr_block       = var.vpc_cidr
  instance_tenancy = "default"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = {
    Name = "${var.project_name}-${var.project_env}",
    Project = var.project_name,
    Env = var.project_env
  }
}


resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.project_name}-${var.project_env}",
    Project = var.project_name,
    Env = var.project_env
  }
}


resource "aws_subnet" "public_sub" {
  count = length(data.aws_availability_zones.available.names)
  vpc_id     = aws_vpc.main_vpc.id
  cidr_block = cidrsubnet(var.vpc_cidr,"4",count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-${var.project_env}-public_sub",
    Project = var.project_name,
    Env = var.project_env
  }
}


resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${var.project_name}-${var.project_env}-public_rt",
    Project = var.project_name,
    Env = var.project_env
  }
}


resource "aws_route_table_association" "public_rta" {
  count = length(aws_subnet.public_sub[*].id)
  subnet_id      = aws_subnet.public_sub[count.index].id
  route_table_id = aws_route_table.public_rt.id
}



#+++++++++++++++++++++++
#CREATING SECURITY GROUP
#+++++++++++++++++++++++

resource "aws_security_group" "main_sec" {
  name_prefix = "${var.project_name}-sec"
  description = "allow ports 22,80,443"
  vpc_id = aws_vpc.main_vpc.id
  
  egress {
    protocol = "-1"
    from_port = 0
    to_port = 0
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.project_name}-${var.project_env}-sec",
    Project = var.project_name,
    Env = var.project_env
  }
  lifecycle {
    create_before_destroy = true
  }  
}



#+++++++++++++++++++++++++++++++++++++
#CREATING SECURITY GROUP INGRESS RULES
#++++++++++++++++++++++++++++++++++++++

resource "aws_security_group_rule" "main_sec_rule" {
  for_each = toset(var.sec_rules)
  type              = "ingress"
  from_port         = each.key
  to_port           = each.key
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  ipv6_cidr_blocks  = ["::/0"]
  security_group_id = aws_security_group.main_sec.id
}



#+++++++++++++++++++++++++++++++
#CREATING KEY PAIR FOR INSTANCES
#+++++++++++++++++++++++++++++++


resource "aws_key_pair" "main_ssh_key" {
  key_name = "${var.project_name}-${var.project_env}"
  public_key = file("maininfra.pub")
  tags = {
    Name = "${var.project_name}-${var.project_env}-key",
    Project = var.project_name,
    Env = var.project_env
  }
}



#++++++++++++++++++++++++++++++++++++++
#CREATING TARGET GROUP FOR LOADBALANCER
#++++++++++++++++++++++++++++++++++++++


resource "aws_lb_target_group" "main_tg" {
  name     = "${var.project_name}-${var.project_env}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main_vpc.id
  target_type = "instance"
  deregistration_delay = 20
  health_check {
    enabled = true
    protocol = "HTTP"
    path = "/health.html"
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 5
    interval = 30
    matcher = 200
  }
    tags = {
    Name = "${var.project_name}-${var.project_env}-tg",
    Project = var.project_name,
    Env = var.project_env
  }
   lifecycle {
    create_before_destroy = true
  }
}



#++++++++++++++++++++++++++++++++++++++++
#CREATING LOADBALANCER AND LISTENER RULES
#++++++++++++++++++++++++++++++++++++++++


resource "aws_lb" "main_alb" {
  name               = "${var.project_name}-${var.project_env}-tg"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.main_sec.id]
  subnets            = [for subnet in aws_subnet.public_sub : subnet.id]
 
  tags = {
    Name = "${var.project_name}-${var.project_env}-tg",
    Project = var.project_name,
    Env = var.project_env
  }
  depends_on = [ aws_lb_target_group.main_tg ]
}


resource "aws_lb_listener" "lb_listener1" {
  load_balancer_arn = aws_lb.main_alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}


resource "aws_lb_listener" "lb_listener2" {
  load_balancer_arn = aws_lb.main_alb.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = data.aws_acm_certificate.ssl_certificate.arn

  default_action {
    type = "fixed-response"

    fixed_response {
      content_type = "text/plain"
      message_body = "No Page Found"
      status_code  = "200"
    }
  }
}

resource "aws_lb_listener_rule" "lb_lrule" {
  listener_arn = aws_lb_listener.lb_listener2.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main_tg.arn
  }

  condition {
    host_header {
      values = [var.domain_details.domain_name]
    }
  }
}



#+++++++++++++++++++++++++++++
#CREATING LAUNCH CONFIGURATION 
#+++++++++++++++++++++++++++++


resource "aws_launch_configuration" "main_lc" {
    
  name_prefix   = "${var.project_name}-${var.project_env}-"
  image_id      = data.aws_ami.main_ami.image_id
  instance_type = var.instance_type
  key_name      = aws_key_pair.main_ssh_key.id
  user_data     = file("mainsetup.sh")
  security_groups = [ aws_security_group.main_sec.id ]
  lifecycle {
    create_before_destroy = true
  }
}



#++++++++++++++++++++++++++
#CREATING AUTOSCALING GROUP
#++++++++++++++++++++++++++


resource "aws_autoscaling_group" "main_asg" {
    
  name_prefix = "${var.project_name}-${var.project_env}-"
  vpc_zone_identifier = aws_subnet.public_sub[*].id
  desired_capacity = var.asg_details.desired_capacity
  max_size = var.asg_details.max_size
  min_size = var.asg_details.min_size
  force_delete = true
  health_check_type = "EC2"
  launch_configuration = aws_launch_configuration.main_lc.id
  target_group_arns = [ aws_lb_target_group.main_tg.arn ]
  tag {
    key = "Name"
    value = "${var.project_name}-${var.project_env}"
    propagate_at_launch = true
  }
    
  tag {
    key = "Project"
    value = var.project_name
    propagate_at_launch = true
  }
    
  tag {
    key = "Env"
    value = var.project_env
    propagate_at_launch = true
  }
    
   
  depends_on = [ aws_lb_target_group.main_tg , aws_lb.main_alb ] 
  lifecycle {
    create_before_destroy = true
  }
}



#+++++++++++++++++++++
#ADDING ROUTE53 RECORD 
#+++++++++++++++++++++


resource "aws_route53_record" "record" {
  zone_id = data.aws_route53_zone.myzone_id.zone_id
  name    = var.domain_details.domain_name
  type    = "A"

  alias {
    name                   = aws_lb.main_alb.dns_name
    zone_id                = aws_lb.main_alb.zone_id
    evaluate_target_health = true
  }
}



##vim output.tf

output "asg_instance_public_ips" {
  value = data.aws_instances.main_instance.public_ips
}

output "asg_instance_private_ips" {
  value = data.aws_instances.main_instance.private_ips 
}

```

# Placing EC2 Webserver Instances in a Private Subnet with Internet Access via NAT Gateway using Terraform

In one of my earlier posts, we designed a website hosted on EC2 instances in a Multi-AZ environment behind an application load balancer. The EC2 instances were placed in public subnet therefore reachable from outside and they were accessing internate directly from Internet Gateway. In this post, I'll focus on placing EC2 instances in a private subnet and ensuring they have internet access via a NAT gateway using Terraform.

A NAT (Network Address Translation) gateway plays a crucial role in scenarios where instances in private subnets need access to the internet for updates, patches, or communication with external services, but you want to keep these instances secure from inbound internet traffic. The NAT gateway allows outbound connections to the internet while ensuring that inbound connections initiated from the internet are blocked.

# Architecture

![alt text](/images/diagram.png)

# Step 1: Creating the VPC and Network Components
First, we need to create the essential network components:
A VPC (Virtual Private Cloud) to contain all our resources.
Public and Private Subnets in two different availability zones to achieve high availability.
An Internet Gateway attached to the VPC to provide internet access to resources in the public subnets.
A NAT Gateway in each public subnet to route traffic from the private subnet to the internet.
Route tables and routes configured to direct traffic appropriately between the subnets, NAT gateway, and internet gateway.
```terraform
################################################################################
# Get list of available AZs
################################################################################
data "aws_availability_zones" "available_zones" {
  state = "available"
}

################################################################################
# Create the VPC
################################################################################
resource "aws_vpc" "app_vpc" {
  cidr_block           = var.vpc_cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-${var.name}"
  })
}

################################################################################
# Create the internet gateway
################################################################################
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.app_vpc.id

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-igw"
  })
}

################################################################################
# Create the public subnets
################################################################################
resource "aws_subnet" "public_subnets" {
  vpc_id = aws_vpc.app_vpc.id

  count             = 2
  cidr_block        = cidrsubnet(var.vpc_cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available_zones.names[count.index]

  map_public_ip_on_launch = true # This makes public subnet

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-pubsubnet-${count.index + 1}"
  })
}

################################################################################
# Create the private subnets
################################################################################
resource "aws_subnet" "private_subnets" {
  vpc_id = aws_vpc.app_vpc.id

  count             = 2
  cidr_block        = cidrsubnet(var.vpc_cidr_block, 8, 2 + count.index)
  availability_zone = data.aws_availability_zones.available_zones.names[count.index]

  map_public_ip_on_launch = false # This makes private subnet

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-privsubnet-${count.index + 1}"
  })
}

################################################################################
# Create the public route table
################################################################################
resource "aws_route_table" "public_route_table" {
  vpc_id = aws_vpc.app_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-pub-rtable"
  })

}

################################################################################
# Assign the public route table to the public subnet
################################################################################
resource "aws_route_table_association" "public_rt_asso" {
  count          = 2
  subnet_id      = element(aws_subnet.public_subnets[*].id, count.index)
  route_table_id = aws_route_table.public_route_table.id
}

################################################################################
# Set default route table as private route table
################################################################################
resource "aws_route_table" "private_route_table" {
  vpc_id = aws_vpc.app_vpc.id

  count = 2
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.natgateway[count.index].id
  }

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-priv-rtable"
  })
}

################################################################################
# Assign the private route table to the private subnet
################################################################################
resource "aws_route_table_association" "private_rt_asso" {
  count          = 2
  subnet_id      = element(aws_subnet.private_subnets[*].id, count.index)
  route_table_id = aws_route_table.private_route_table[count.index].id
}

################################################################################
# Create EIP for NAT Gateways
################################################################################
resource "aws_eip" "eip_natgw" {
  count = 2
}

################################################################################
# Create NAT Gateways in each public subnets
################################################################################
resource "aws_nat_gateway" "natgateway" {
  count         = 2
  allocation_id = aws_eip.eip_natgw[count.index].id
  subnet_id     = aws_subnet.public_subnets[count.index].id
}

```

# Step 2: Creating 2 Linux EC2 Web Server Instances in Separate AZs
Now, let's deploy our EC2 instances in private subnets in separate AZs for high availability. These instances will be our web servers. Since they are in a private subnet, they won’t have direct access to the internet unless we configure routing through the NAT gateway. Associate the instances with the private subnet route table that routes traffic through the NAT gateway.
EC2 instances will be created only after NAT gateways are functional because they require internet access to install httpd and other pacakges.
```terraform
################################################################################
# Get latest Amazon Linux 2023 AMI
################################################################################
data "aws_ami" "amazon-linux-2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

################################################################################
# Create the security group for EC2 Webservers
################################################################################
resource "aws_security_group" "ec2_security_group" {
  description = "Allow traffic for EC2 Webservers"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.sg_ingress_ports
    iterator = sg_ingress

    content {
      description = sg_ingress.value["description"]
      from_port   = sg_ingress.value["port"]
      to_port     = sg_ingress.value["port"]
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-sg-webserver"
  })
}


################################################################################
# Create the Linux EC2 Web server
################################################################################
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon-linux-2023.id
  instance_type          = var.instance_type
  key_name               = var.instance_key
  vpc_security_group_ids = [aws_security_group.ec2_security_group.id]

  count     = length(var.private_subnets)
  subnet_id = element(var.private_subnets, count.index)


  user_data = <<-EOF
  #!/bin/bash
  yum update -y
  yum install -y httpd.x86_64
  systemctl start httpd.service
  systemctl enable httpd.service

  TOKEN=$(curl --request PUT "http://169.254.169.254/latest/api/token" --header "X-aws-ec2-metadata-token-ttl-seconds: 3600")

  instanceId=$(curl -s http://169.254.169.254/latest/meta-data/instance-id --header "X-aws-ec2-metadata-token: $TOKEN")
  instanceAZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone --header "X-aws-ec2-metadata-token: $TOKEN")
  privHostName=$(curl -s http://169.254.169.254/latest/meta-data/local-hostname --header "X-aws-ec2-metadata-token: $TOKEN")
  privIPv4=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4 --header "X-aws-ec2-metadata-token: $TOKEN")
  
  echo "<font face = "Verdana" size = "5">"                               > /var/www/html/index.html
  echo "<center><h1>AWS Linux VM Deployed with Terraform</h1></center>"   >> /var/www/html/index.html
  echo "<center> <b>EC2 Instance Metadata</b> </center>"                  >> /var/www/html/index.html
  echo "<center> <b>Instance ID:</b> $instanceId </center>"               >> /var/www/html/index.html
  echo "<center> <b>AWS Availablity Zone:</b> $instanceAZ </center>"      >> /var/www/html/index.html
  echo "<center> <b>Private Hostname:</b> $privHostName </center>"        >> /var/www/html/index.html
  echo "<center> <b>Private IPv4:</b> $privIPv4 </center>"                >> /var/www/html/index.html
  echo "</font>"                                                          >> /var/www/html/index.html
EOF

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-ec2-${count.index + 1}"
  })
}
```

# Step 3: Creating an Application Load Balancer with HTTP Listener
Create an Application Load Balancer (ALB) to distribute traffic evenly between them. The ALB will be placed in the public subnets and will listen on port 80 (HTTP). The load balancer forwards traffic to your EC2 instances in the private subnets.

```terraform
################################################################################
# Define the security group for the Load Balancer
################################################################################
resource "aws_security_group" "aws-sg-load-balancer" {
  description = "Allow incoming connections for load balancer"
  vpc_id      = var.vpc_id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow incoming HTTP connections"
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-sg-alb"
  })
}

################################################################################
# Create application load balancer
################################################################################
resource "aws_lb" "aws-application_load_balancer" {
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.aws-sg-load-balancer.id]
  //subnets                    = [var.public_subnets[0],var.public_subnets[1] ,var.public_subnets[2],var.public_subnets[3]]
  subnets                    = tolist(var.public_subnets)
  enable_deletion_protection = false

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-alb"
  })
}
################################################################################
# create target group for ALB
################################################################################
resource "aws_lb_target_group" "alb_target_group" {
  target_type = "instance"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.vpc_id

  health_check {
    enabled             = true
    interval            = 300
    path                = "/"
    timeout             = 60
    matcher             = 200
    healthy_threshold   = 5
    unhealthy_threshold = 5
  }

  lifecycle {
    create_before_destroy = true
  }

  tags = merge(var.common_tags, {
    Name = "${var.naming_prefix}-alb-tg"
  })
}

################################################################################
# create a listener on port 80 with redirect action
################################################################################
resource "aws_lb_listener" "alb_http_listener" {
  load_balancer_arn = aws_lb.aws-application_load_balancer.id
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.alb_target_group.id
  }
}

################################################################################
# Target Group Attachment with Instance
################################################################################
resource "aws_alb_target_group_attachment" "tgattachment" {
  count            = length(var.instance_ids)
  target_group_arn = aws_lb_target_group.alb_target_group.arn
  target_id        = element(var.instance_ids, count.index)
}
```
# Steps to Run Terraform
Follow these steps to execute the Terraform configuration:
```terraform
terraform init
terraform plan 
terraform apply -auto-approve
```
Upon successful completion, Terraform will provide relevant outputs.
``` terraform
Apply complete! Resources: 26 added, 0 changed, 0 destroyed.
```

# Testing

Public Route Table with route to Internet Gateway and associated public subnets. 
![alt text](/images/public_rt.png)

Private Route Tables with route to NAT Gateway and associated private subnets
![alt text](/images/private_rt1.png)

![alt text](/images/private_rt2.png)

NAT Gateways with associated EIPs in each public subnet
![alt text](/images/nat_gateways.png)

EC2 instances in private subnets (without Public IP)
![alt text](/images/ec2_instances.png)

WebServers
![alt text](/images/website_1.png)

![alt text](/images/website_2.png)

# Cleanup
Remember to stop AWS components to avoid large bills.
```terraform
terraform destroy -auto-approve
```

# Conclusion
By placing your EC2 instances in a private subnet and enabling internet access via a NAT gateway, you've added an additional layer of security to your infrastructure. The instances remain isolated from direct internet exposure, yet they can still communicate with external services when needed.

# Resources
GitHub Repo: https://github.com/chinmayto/terraform-aws-private-EC2-website-alb-nat

AWS Reference: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html
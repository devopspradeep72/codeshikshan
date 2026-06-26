# Terraform - Infrastructure as Code (IaC)

## What is Terraform?
Terraform is an open-source IaC tool that lets you define cloud infrastructure using declarative configuration files. Write code to provision servers, databases, networks, and more across AWS, Azure, GCP, etc.

---

## Real-Time Scenario 1: Rapid Environment Setup

### Scenario Description
A startup needs to quickly provision production infrastructure on AWS:
- 3 EC2 instances for web servers
- 1 RDS PostgreSQL database
- 1 Load Balancer
- VPC with security groups
- S3 buckets for storage

### Problem Without Terraform
```
Manual Process (Error-Prone):
1. Log into AWS Console
2. Create VPC (wait for it to be ready)
3. Create subnets
4. Create security groups (remember rules?)
5. Launch EC2 instances
6. Configure RDS database
7. Set up load balancer
8. Configure auto-scaling
9. Create S3 buckets
10. Set up IAM roles and policies

Time: 2-3 hours
Errors: Easy to misconfigure (forgot to open port 443?)
Documentation: Manual tracking in spreadsheet
Reproducibility: Hard to recreate in another AWS account
```

### Terraform Solution

```hcl
# main.tf - Define all infrastructure

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
  backend "s3" {
    bucket         = "terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC and Networking
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = {
    Name = "production-vpc"
  }
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# Security Group for Web Servers
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_cidr]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 Instances
resource "aws_instance" "web" {
  count                = 3
  ami                  = data.aws_ami.ubuntu.id
  instance_type        = "t3.medium"
  subnet_id            = aws_subnet.public[count.index].id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = base64encode(file("${path.module}/scripts/init.sh"))
  
  tags = {
    Name = "web-server-${count.index + 1}"
  }
}

# Load Balancer
resource "aws_lb" "main" {
  name               = "production-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_target_group" "app" {
  name        = "app-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  
  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 3
    interval            = 30
    path                = "/health"
  }
}

resource "aws_lb_target_group_attachment" "app" {
  count            = 3
  target_group_arn = aws_lb_target_group.app.arn
  target_id        = aws_instance.web[count.index].id
  port             = 80
}

resource "aws_lb_listener" "app" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# RDS Database
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet"
  subnet_ids = aws_subnet.public[*].id
}

resource "aws_db_instance" "postgres" {
  identifier            = "production-db"
  engine                = "postgres"
  engine_version        = "13.7"
  instance_class        = "db.t3.small"
  allocated_storage     = 20
  storage_type          = "gp2"
  
  db_name              = var.db_name
  username             = var.db_username
  password             = var.db_password
  
  db_subnet_group_name = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"
  
  skip_final_snapshot = false
  
  tags = {
    Name = "production-database"
  }
}

# S3 Bucket
resource "aws_s3_bucket" "app_data" {
  bucket = "app-data-${var.environment}-${data.aws_caller_identity.current.account_id}"
  
  tags = {
    Name = "application-data"
  }
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

```hcl
# variables.tf - Input variables

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "environment" {
  type    = string
  default = "production"
}

variable "db_name" {
  type = string
}

variable "db_username" {
  type = string
}

variable "db_password" {
  type      = string
  sensitive = true
}

variable "allowed_ssh_cidr" {
  type = string
}
```

```hcl
# outputs.tf - Output values

output "load_balancer_dns" {
  value = aws_lb.main.dns_name
}

output "database_endpoint" {
  value = aws_db_instance.postgres.endpoint
}

output "s3_bucket_name" {
  value = aws_s3_bucket.app_data.id
}
```

```hcl
# terraform.tfvars - Input variable values

aws_region         = "us-east-1"
environment        = "production"
db_name            = "appdb"
db_username        = "admin"
db_password        = "SuperSecurePassword123!"
allowed_ssh_cidr   = "203.0.113.0/24"
```

### Deploy Infrastructure
```bash
# Initialize Terraform
terraform init

# Preview changes
terraform plan

# Apply infrastructure
terraform apply

# Output values
terraform output
```

### Result
- All infrastructure created in minutes
- Entire setup is version controlled
- Easy to recreate in another region
- Changes tracked in git
- Rollback capability

---

## Real-Time Scenario 2: Multi-Environment Management

### Scenario Description
Company needs to manage development, staging, and production environments with consistent infrastructure.

### Problem Without Terraform
- Dev has 1 web server, staging has 2, production has 5 (manual tracking)
- Database configs differ between environments
- Security group rules not consistent
- Hard to replicate production issues in dev

### Terraform Solution - Workspace Approach
```bash
# Create workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Select workspace
terraform workspace select prod

# Apply configuration specific to prod
terraform apply -var-file=prod.tfvars
```

```hcl
# dev.tfvars
environment = "dev"
instance_count = 1
instance_type = "t3.micro"
db_instance_class = "db.t3.micro"

# staging.tfvars
environment = "staging"
instance_count = 2
instance_type = "t3.small"
db_instance_class = "db.t3.small"

# prod.tfvars
environment = "prod"
instance_count = 5
instance_type = "t3.medium"
db_instance_class = "db.t3.medium"
backup_retention = 30
multi_az = true
```

---

## Real-Time Scenario 3: Disaster Recovery

### Scenario Description
AWS region goes down. Company needs to quickly spin up entire infrastructure in another region.

### Terraform Solution
```bash
# Current region has outage, switch to backup region
terraform var aws_region = "eu-west-1"
terraform plan -out=recovery.tfplan
terraform apply recovery.tfplan

# Entire infrastructure running in new region in minutes!
```

---

## Terraform Concepts

### State File
- Tracks current infrastructure
- Maps Terraform config to real resources
- Must be stored securely (S3 + encryption)

### Provider
- Plugin for specific cloud platform (AWS, Azure, GCP)
- Handles authentication and API calls

### Resource
- Real infrastructure object (EC2, RDS, S3)
- Declared in Terraform

### Module
- Reusable configuration package
- Encapsulates related resources

### Variable
- Input parameter
- Can be set via CLI, files, or environment

---

## When to Use Terraform

✅ **Good Use Cases:**
- Multi-cloud infrastructure
- Complex infrastructure
- Need for version control
- Multiple environments
- Infrastructure changes tracked

❌ **Avoid If:**
- Simple single resource
- Cloud-specific features only
- One-time setup

---

## Learning Path
1. Understand basic resource creation
2. Learn variables and outputs
3. Practice state management
4. Create reusable modules
5. Implement remote state
6. Master workspaces for environments

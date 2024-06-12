# AWS Cloud Security Project using IaC

This project demonstrates a secure AWS environment setup using Terraform.

## Features

- VPC with subnets
- IAM roles and policies
- Secure S3 buckets with encryption and logging
- EC2 instances with security groups
- RDS instance with encryption
- Monitoring with CloudWatch and CloudTrail
- Automated backups
- CI/CD pipeline

## Setup

1. Clone the repository.
2. Configure AWS CLI.
3. Initialize and apply Terraform configuration.

```bash
terraform init
terraform apply

### AWSIaCSecure
Creating a secure AWS environment using IaC

1. First we will create an AWS account (if you already have one, you are good to go). Then, we will install and configure the AWS CLI on our local machine.
# Install AWS CLI
pip install awscli

# Configure AWS CLI
aws configure

2. Next we will get started with the Infrastructure as Code (IaC) by creating a Terraform configuration file (main.tf) to define our AWS infrastructure.
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "subnet1" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

Initialize and apply

terraform init
terraform apply

Very important to secure everything with IAM and we will secure our S3 Buckets with a bucket policy

# IAM roles and policies
resource "aws_iam_role" "example" {
  name = "example-role"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}

resource "aws_iam_policy" "example_policy" {
  name        = "example_policy"
  description = "A test policy"
  policy      = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "ec2:Describe*",
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "example_attach" {
  role       = aws_iam_role.example.name
  policy_arn = aws_iam_policy.example_policy.arn
}

Securing our buckets and encrypting the data with server side encryption.

# S3 bucket with encryption and logging
resource "aws_s3_bucket" "example" {
  bucket = "example-bucket"
  acl    = "private"

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }

  logging {
    target_bucket = "log-bucket"
    target_prefix = "log/"
  }
}

Launch an EC2 instance. We use t2.micro because it saves cost.

# EC2 instance with security group
resource "aws_security_group" "web_sg" {
  name        = "web_sg"
  description = "Allow web traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  security_groups = [aws_security_group.web_sg.name]

  tags = {
    Name = "WebInstance"
  }
}

Every environment needs a database so we will launch a secure RDS instance. RDS automates database management tasks, such as provisioning, configuring, backups, and patching
# RDS instance
resource "aws_db_instance" "default" {
  identifier              = "mydb"
  allocated_storage       = 20
  storage_type            = "gp2"
  engine                  = "mysql"
  engine_version          = "8.0"
  instance_class          = "db.t2.micro"
  name                    = "mydatabase"
  username                = "foo"
  password                = "foobarbaz"
  parameter_group_name    = "default.mysql8.0"
  publicly_accessible     = false
  skip_final_snapshot     = true

  vpc_security_group_ids = [aws_security_group.web_sg.id]
}

CloudTrail is a service that helps enable operational and risk auditing, governance, and compliance of your AWS account. Actions taken by a user, role, or an AWS service are recorded as events in CloudTrail.

# CloudTrail
resource "aws_cloudtrail" "example" {
  name                          = "example"
  s3_bucket_name                = aws_s3_bucket.example.bucket
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
}

Cloudwatch provides insights on logs we monitor and can be used to determine system health.

# CloudWatch log group
resource "aws_cloudwatch_log_group" "example" {
  name = "/aws/cloudtrail/example"
}

Availability is a great reason why AWS is so amazing. We need to make sure we have backups so we never have downtime in the event of a problem.

# EC2 backup
resource "aws_backup_plan" "ec2_backup" {
  name = "ec2-backup-plan"

  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.example.name
    schedule          = "cron(0 12 * * ? *)"
  }
}

resource "aws_backup_selection" "ec2_selection" {
  name         = "ec2-backup-selection"
  iam_role_arn = aws_iam_role.example.arn
  plan_id      = aws_backup_plan.ec2_backup.id

  resources = [
    aws_instance.web.arn
  ]
}

# CodePipeline and CodeBuild setup
resource "aws_codepipeline" "example" {
  name = "example-pipeline"

  # Define stages and actions...
}


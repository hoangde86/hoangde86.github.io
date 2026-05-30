---
title: "Terraform: Infrastructure as Code từ đầu đến production"
date: 2026-05-20 09:00:00 +0700
categories: [DevOps, IaC]
tags: [terraform, iac, aws, infrastructure, devops]
---

Terraform cho phép quản lý infrastructure bằng code — tạo, cập nhật, xóa cloud resources một cách có kiểm soát.

## Tại sao dùng Terraform?

- **Reproducible**: tạo lại môi trường giống hệt nhau
- **Version control**: infrastructure thay đổi được track bằng Git
- **Plan trước khi apply**: xem những gì sẽ thay đổi
- **Multi-cloud**: AWS, GCP, Azure, cùng một syntax

## Cấu trúc project

```
infra/
├── main.tf           # resources chính
├── variables.tf      # input variables
├── outputs.tf        # output values
├── terraform.tfvars  # giá trị biến (không commit nếu có secret)
└── versions.tf       # provider version constraints
```

## Ví dụ: Deploy web server trên AWS

```hcl
# versions.tf
terraform {
  required_version = ">= 1.7"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "my-terraform-state"
    key    = "production/terraform.tfstate"
    region = "ap-southeast-1"
  }
}
```

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Tên môi trường: dev, staging, production"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "ssh_allowed_ips" {
  type    = list(string)
  default = []
}
```

```hcl
# main.tf
provider "aws" {
  region = "ap-southeast-1"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

resource "aws_security_group" "web" {
  name        = "${var.environment}-web-sg"
  description = "Allow HTTP, HTTPS, SSH"

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
    cidr_blocks = var.ssh_allowed_ips
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    apt update && apt install -y nginx
    systemctl enable --now nginx
  EOF

  tags = {
    Name        = "${var.environment}-web"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

```hcl
# outputs.tf
output "public_ip" {
  value       = aws_instance.web.public_ip
  description = "IP public của web server"
}

output "public_dns" {
  value = aws_instance.web.public_dns
}
```

## Workflow cơ bản

```bash
terraform init          # download providers, setup backend
terraform fmt           # format code
terraform validate      # kiểm tra syntax

terraform plan          # xem những gì sẽ thay đổi
terraform plan -out=tfplan   # lưu plan để apply sau

terraform apply tfplan  # thực hiện thay đổi
terraform destroy       # xóa toàn bộ resources
```

## Modules — Tái sử dụng code

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.environment}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-southeast-1a", "ap-southeast-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

## Workspace để quản lý nhiều môi trường

```bash
terraform workspace new staging
terraform workspace new production
terraform workspace select production
terraform workspace list

# Dùng workspace trong code
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "production" ? "t3.medium" : "t3.micro"
}
```

---

Luôn review kỹ `terraform plan` trước khi `apply` — một lệnh có thể xóa database production nếu không cẩn thận.

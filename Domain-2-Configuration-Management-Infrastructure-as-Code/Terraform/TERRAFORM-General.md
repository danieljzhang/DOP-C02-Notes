# Terraform - DOP-C02 Exam Notes

## 1. Overview

**Terraform** is an open-source Infrastructure as Code (IaC) tool that enables you to define and provision infrastructure using a declarative configuration language. It supports multiple cloud providers including AWS, Azure, and GCP.

### Key Characteristics
- **Declarative syntax** - Define desired state, not steps
- **Multi-cloud support** - Single tool for multiple providers
- **State management** - Tracks infrastructure state
- **Plan and apply** - Preview changes before execution
- **Resource graph** - Understands dependencies
- **Modular** - Reusable modules and configurations
- **Version control** - Infrastructure as code in Git

### What Problem Does It Solve?
- Eliminates manual infrastructure provisioning
- Provides consistent, repeatable deployments
- Enables infrastructure version control and collaboration
- Supports multi-cloud and hybrid environments
- Facilitates infrastructure testing and validation
- Enables infrastructure automation and CI/CD integration

---

## 2. Core Concepts

### Providers
```hcl
# AWS Provider configuration
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0"
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}
```

### Resources
```hcl
# EC2 Instance resource
resource "aws_instance" "web_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  key_name      = aws_key_pair.deployer.key_name
  
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public[0].id
  
  user_data = templatefile("${path.module}/user_data.sh", {
    db_endpoint = aws_db_instance.main.endpoint
  })
  
  tags = {
    Name = "${var.project_name}-web-server"
  }
}

# S3 Bucket resource
resource "aws_s3_bucket" "app_data" {
  bucket = "${var.project_name}-app-data-${random_id.bucket_suffix.hex}"
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### Variables
```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_config" {
  description = "EC2 instance configuration"
  type = object({
    type = string
    count = number
  })
  default = {
    type  = "t3.micro"
    count = 1
  }
}
```

### Outputs
```hcl
# outputs.tf
output "web_server_public_ip" {
  description = "Public IP of web server"
  value       = aws_instance.web_server.public_ip
}

output "database_endpoint" {
  description = "RDS instance endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

output "s3_bucket_name" {
  description = "Name of S3 bucket"
  value       = aws_s3_bucket.app_data.bucket
}
```

---

## 3. State Management

### Remote State Configuration
```hcl
# Backend configuration
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "environments/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

### State Locking with DynamoDB
```hcl
# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-state-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Terraform State Lock Table"
  }
}
```

### State Commands
```bash
# Initialize backend
terraform init

# Show current state
terraform show

# List resources in state
terraform state list

# Import existing resource
terraform import aws_instance.example i-1234567890abcdef0

# Remove resource from state
terraform state rm aws_instance.example

# Move resource in state
terraform state mv aws_instance.old aws_instance.new
```

---

## 4. Modules

### Module Structure
```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
├── ec2/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── rds/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

### VPC Module Example
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Type = "Public"
  })
}

# modules/vpc/variables.tf
variable "name" {
  description = "Name prefix for resources"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}
```

### Using Modules
```hcl
# main.tf
module "vpc" {
  source = "./modules/vpc"
  
  name                = var.project_name
  vpc_cidr           = "10.0.0.0/16"
  public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
  
  tags = local.common_tags
}

module "web_servers" {
  source = "./modules/ec2"
  
  name           = "${var.project_name}-web"
  instance_count = var.web_server_count
  subnet_ids     = module.vpc.public_subnet_ids
  
  tags = local.common_tags
}
```

---

## 5. Terraform with AWS CI/CD

### CodeBuild Integration
```yaml
# buildspec.yml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.9
    commands:
      - wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
      - unzip terraform_1.5.0_linux_amd64.zip
      - mv terraform /usr/local/bin/
      - terraform --version
  
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - terraform init
  
  build:
    commands:
      - echo Build started on `date`
      - terraform plan -out=tfplan
      - terraform apply tfplan
  
  post_build:
    commands:
      - echo Build completed on `date`

artifacts:
  files:
    - '**/*'
```

### CodePipeline with Terraform
```yaml
Resources:
  TerraformPipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt CodePipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactsBucket
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: S3
                Version: '1'
              Configuration:
                S3Bucket: !Ref SourceBucket
                S3ObjectKey: terraform-source.zip
              OutputArtifacts:
                - Name: SourceOutput
        
        - Name: Plan
          Actions:
            - Name: TerraformPlan
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref TerraformPlanProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: PlanOutput
        
        - Name: Approval
          Actions:
            - Name: ManualApproval
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              Configuration:
                CustomData: 'Please review the Terraform plan before applying'
        
        - Name: Apply
          Actions:
            - Name: TerraformApply
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref TerraformApplyProject
              InputArtifacts:
                - Name: PlanOutput
```

---

## 6. Best Practices

### Code Organization
```hcl
# Use consistent naming
resource "aws_instance" "web_server" {
  # Use descriptive names
  tags = {
    Name = "${var.environment}-${var.application}-web-server"
  }
}

# Use locals for computed values
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  }
  
  vpc_cidr_blocks = {
    dev     = "10.0.0.0/16"
    staging = "10.1.0.0/16"
    prod    = "10.2.0.0/16"
  }
}
```

### Security Best Practices
```hcl
# Use data sources for sensitive information
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db-password"
}

# Encrypt sensitive outputs
output "database_password" {
  value     = data.aws_secretsmanager_secret_version.db_password.secret_string
  sensitive = true
}

# Use least privilege IAM policies
resource "aws_iam_policy" "app_policy" {
  name = "${var.project_name}-app-policy"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "${aws_s3_bucket.app_data.arn}/*"
      }
    ]
  })
}
```

---

## 7. Terraform vs CloudFormation

| Feature | Terraform | CloudFormation |
|---------|-----------|----------------|
| **Language** | HCL (HashiCorp Configuration Language) | JSON/YAML |
| **Multi-cloud** | Yes (AWS, Azure, GCP, etc.) | AWS only |
| **State Management** | External state file | AWS managed |
| **Plan/Preview** | terraform plan | Change sets |
| **Modularity** | Modules | Nested stacks |
| **Community** | Large ecosystem | AWS native |
| **Learning Curve** | Moderate | Steep |
| **Rollback** | Manual | Automatic |

---

## 8. Common Exam Scenarios

### Scenario 1: Multi-Environment Infrastructure
```hcl
# terraform.tfvars.dev
environment = "dev"
instance_type = "t3.micro"
min_size = 1
max_size = 2

# terraform.tfvars.prod
environment = "prod"
instance_type = "t3.large"
min_size = 2
max_size = 10

# Deploy to different environments
terraform apply -var-file="terraform.tfvars.dev"
terraform apply -var-file="terraform.tfvars.prod"
```

### Scenario 2: Blue/Green Deployment
```hcl
variable "active_environment" {
  description = "Currently active environment (blue or green)"
  type        = string
  default     = "blue"
}

resource "aws_lb_target_group" "blue" {
  name     = "${var.project_name}-blue-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id
}

resource "aws_lb_target_group" "green" {
  name     = "${var.project_name}-green-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id
}

resource "aws_lb_listener" "main" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = var.active_environment == "blue" ? aws_lb_target_group.blue.arn : aws_lb_target_group.green.arn
  }
}
```

### Scenario 3: Disaster Recovery
```hcl
# Multi-region setup
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

module "primary_infrastructure" {
  source = "./modules/infrastructure"
  
  providers = {
    aws = aws.primary
  }
  
  environment = "prod-primary"
  region      = "us-east-1"
}

module "dr_infrastructure" {
  source = "./modules/infrastructure"
  
  providers = {
    aws = aws.dr
  }
  
  environment = "prod-dr"
  region      = "us-west-2"
}
```

---

## 9. CLI Commands

```bash
# Initialize Terraform
terraform init
terraform init -upgrade  # Upgrade providers

# Plan changes
terraform plan
terraform plan -out=tfplan
terraform plan -var-file="prod.tfvars"

# Apply changes
terraform apply
terraform apply tfplan
terraform apply -auto-approve

# Destroy infrastructure
terraform destroy
terraform destroy -target=aws_instance.example

# Validate configuration
terraform validate
terraform fmt

# Workspace management
terraform workspace list
terraform workspace new prod
terraform workspace select prod

# State management
terraform state list
terraform state show aws_instance.example
terraform refresh
```

---

## 10. Exam Tips

- **Understand state management** - Know how Terraform tracks resources
- **Master modules** - Reusability and organization are key
- **Know provider configuration** - Multi-cloud scenarios are common
- **Understand workspaces** - Environment separation strategies
- **Practice with AWS resources** - Focus on EC2, VPC, S3, RDS, IAM
- **Learn troubleshooting** - State corruption, import scenarios
- **Understand CI/CD integration** - CodePipeline and CodeBuild usage
- **Know security best practices** - State encryption, sensitive data handling
- **Compare with CloudFormation** - Understand when to use each tool
- **Practice disaster recovery** - Multi-region deployments and backup strategies

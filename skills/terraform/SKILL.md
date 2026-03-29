---
name: terraform
description: Terraform best practices — module structure, remote state (S3 + DynamoDB), drift detection, workspaces, tflint, checkov, and production-grade IaC patterns.
origin: custom
---

# Terraform Patterns

Production-grade Terraform for AWS infrastructure as code.

## When to Activate

- Writing or reviewing Terraform modules
- Setting up remote state backend
- Detecting and resolving infrastructure drift
- Structuring multi-environment Terraform projects
- Running security scanning on IaC (tflint, checkov)

## Module Structure

```
infra/
├── modules/                     # Reusable modules (no state)
│   ├── eks-cluster/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── rds-postgres/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── vpc/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/                # Environment-specific root modules
│   ├── dev/
│   │   ├── main.tf              # calls modules
│   │   ├── variables.tf
│   │   ├── terraform.tfvars     # dev-specific values (committed)
│   │   └── backend.tf           # remote state config
│   ├── staging/
│   │   └── ...
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
└── .terraform-version           # pin via tfenv
```

## Remote State — S3 + DynamoDB Lock

```hcl
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-prod"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true                          # ✅ encrypt at rest
    dynamodb_table = "terraform-state-lock-prod"   # ✅ prevent concurrent runs
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
  }
}
```

```hcl
# Bootstrap state bucket (run once, manually)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-prod"

  lifecycle {
    prevent_destroy = true    # ✅ never delete state bucket
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"        # ✅ version all state files
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

resource "aws_dynamodb_table" "terraform_state_lock" {
  name         = "terraform-state-lock-prod"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = local.common_tags
}
```

## Module — VPC Example

```hcl
# modules/vpc/variables.tf
variable "name" {
  type        = string
  description = "VPC name — used for resource naming and tagging"
}

variable "cidr" {
  type        = string
  description = "VPC CIDR block"
  default     = "10.0.0.0/16"
  validation {
    condition     = can(cidrnetmask(var.cidr))
    error_message = "Must be a valid CIDR block."
  }
}

variable "azs" {
  type        = list(string)
  description = "Availability zones"
}

variable "private_subnets" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "public_subnets" {
  type    = list(string)
  default = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/vpc/main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.2"    # ✅ always pin module versions

  name = var.name
  cidr = var.cidr

  azs             = var.azs
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  enable_nat_gateway   = true
  single_nat_gateway   = false     # HA: one NAT per AZ in prod
  enable_dns_hostnames = true
  enable_dns_support   = true

  # EKS-required tags
  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }

  tags = merge(var.tags, { managed_by = "terraform" })
}
```

## Calling Modules from Environment Root

```hcl
# environments/prod/main.tf
locals {
  common_tags = {
    env        = "prod"
    team       = "platform"
    managed_by = "terraform"
    cost_center = "infrastructure"
  }
}

module "vpc" {
  source = "../../modules/vpc"

  name            = "prod-vpc"
  cidr            = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  tags            = local.common_tags
}

module "eks" {
  source = "../../modules/eks-cluster"

  cluster_name    = "prod-eks"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets
  tags            = local.common_tags
}
```

## Provider & Version Pinning

```hcl
# environments/prod/versions.tf
terraform {
  required_version = "~> 1.8"     # ✅ pin Terraform version

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"         # ✅ pin provider version
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.28"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      managed_by  = "terraform"
      env         = "prod"
      team        = "platform"
    }
  }
}
```

## Drift Detection

```bash
# Detect drift — compare state vs real infra
terraform plan -detailed-exitcode
# Exit code 0 = no changes, 1 = error, 2 = changes detected (drift)

# Automated drift detection in CI (daily schedule)
# .github/workflows/drift-detection.yml
name: Drift Detection
on:
  schedule:
    - cron: '0 8 * * *'   # daily at 8am UTC

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.8.5"
      - run: terraform init
        working-directory: environments/prod
      - run: |
          terraform plan -detailed-exitcode -out=drift.tfplan
          EXIT_CODE=$?
          if [ $EXIT_CODE -eq 2 ]; then
            echo "::warning::DRIFT DETECTED in prod"
            # Send Slack notification or open GitHub Issue
          fi
        working-directory: environments/prod

# Import drifted resource back into state
terraform import aws_s3_bucket.my_bucket my-existing-bucket-name

# Moved block (rename resource without destroy/recreate)
moved {
  from = aws_s3_bucket.old_name
  to   = aws_s3_bucket.new_name
}
```

## Workspaces (Simple Multi-Env)

```bash
# Use workspaces for lightweight env separation
terraform workspace new staging
terraform workspace select prod
terraform workspace list

# Access workspace in config
locals {
  env = terraform.workspace   # "prod", "staging", "dev"

  instance_type = {
    prod    = "m5.xlarge"
    staging = "t3.medium"
    dev     = "t3.small"
  }[local.env]
}
```

> ⚠️ For complex multi-env: use separate directories (`environments/prod/`) instead of workspaces — easier to review, audit, and isolate blast radius.

## Sensitive Variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true    # ✅ marked sensitive — redacted from plan output
}

# Store in AWS Secrets Manager, not in .tfvars committed to git
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/postgres/password"
}

resource "aws_db_instance" "postgres" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

## Makefile for Common Operations

```makefile
ENV ?= dev

init:
	cd environments/$(ENV) && terraform init -upgrade

plan:
	cd environments/$(ENV) && terraform plan -out=tfplan

apply:
	cd environments/$(ENV) && terraform apply tfplan

destroy:
	cd environments/$(ENV) && terraform destroy

fmt:
	terraform fmt -recursive

validate:
	cd environments/$(ENV) && terraform validate

lint:
	tflint --recursive

security:
	checkov -d environments/$(ENV) --framework terraform

drift:
	cd environments/$(ENV) && terraform plan -detailed-exitcode
```

## tflint Configuration

```hcl
# .tflint.hcl
plugin "aws" {
  enabled = true
  version = "0.31.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform-required-version" {
  enabled = true
}

rule "terraform-required-providers" {
  enabled = true
}

rule "aws-instance-invalid-type" {
  enabled = true
}
```

## Anti-Patterns

| Anti-Pattern | Why Bad | Fix |
|---|---|---|
| ❌ Local state (`terraform.tfstate`) | Lost on machine failure, no team sharing | S3 + DynamoDB backend always |
| ❌ No version pinning | Breaking changes on `terraform init` | Pin `required_version` and all providers |
| ❌ Secrets in `.tfvars` committed | Secret leak | `sensitive = true` + Secrets Manager |
| ❌ Monolithic root module | 1000+ resources, slow plan, high blast radius | Split into modules + environment roots |
| ❌ `terraform apply` without plan review | Accidental destroy | Always `plan -out=tfplan` then `apply tfplan` |
| ❌ No `prevent_destroy` on critical resources | Accidental deletion of state bucket, DB | Add lifecycle `prevent_destroy = true` |
| ❌ Unpinned module source | `source = "git::..."` without `ref=` | Always pin `version = "5.5.2"` |


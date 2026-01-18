---
name: terraform-module
description: Create a Terraform module following best practices
---

# Workflow: Create Terraform Module

Create reusable, well-documented Terraform modules following HashiCorp best practices.

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| `terraform` | IaC tool | `brew install terraform` |
| `terraform-docs` | Documentation | `brew install terraform-docs` |
| `tflint` | Linting | `brew install tflint` |
| `checkov` | Security scanning | `pip install checkov` |

## 1. Module Structure

Create standard layout:

```
modules/<module-name>/
├── main.tf           # Primary resources
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── versions.tf       # Provider requirements
├── locals.tf         # Local values (optional)
├── data.tf           # Data sources (optional)
├── README.md         # Documentation (auto-generated)
├── examples/         # Usage examples
│   └── basic/
│       ├── main.tf
│       └── README.md
└── tests/            # Module tests (optional)
    └── basic.tftest.hcl
```

## 2. File Templates

### `versions.tf` - REQUIRED
```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0.0, < 6.0.0"
    }
  }
}
```

### `variables.tf` - REQUIRED

**Every variable MUST have:**
- `type` - Explicit type constraint
- `description` - Clear explanation
- `validation` - Where applicable

```hcl
variable "name" {
  type        = string
  description = "Name prefix for all resources"

  validation {
    condition     = length(var.name) > 0 && length(var.name) <= 32
    error_message = "Name must be 1-32 characters."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "tags" {
  type        = map(string)
  description = "Tags to apply to all resources"
  default     = {}
}

variable "enable_encryption" {
  type        = bool
  description = "Enable encryption at rest"
  default     = true
}
```

### `outputs.tf` - REQUIRED
```hcl
output "id" {
  description = "The ID of the created resource"
  value       = aws_resource.main.id
}

output "arn" {
  description = "The ARN of the created resource"
  value       = aws_resource.main.arn
}

# Mark sensitive outputs
output "connection_string" {
  description = "Database connection string"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}
```

### `locals.tf` - Common Patterns
```hcl
locals {
  # Merge default tags with user-provided tags
  default_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Module      = "module-name"
  }
  tags = merge(local.default_tags, var.tags)

  # Naming convention
  name_prefix = "${var.name}-${var.environment}"
}
```

### `main.tf` - Resources
```hcl
resource "aws_s3_bucket" "main" {
  bucket = "${local.name_prefix}-bucket"
  tags   = local.tags
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  count  = var.enable_encryption ? 1 : 0
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

## 3. Examples Directory

Create at least one working example:

### `examples/basic/main.tf`
```hcl
module "example" {
  source = "../../"

  name        = "my-app"
  environment = "dev"

  tags = {
    Project = "example"
  }
}

output "bucket_id" {
  value = module.example.id
}
```

### `examples/basic/README.md`
```markdown
# Basic Example

This example creates a basic instance of the module.

## Usage

```bash
terraform init
terraform plan
terraform apply
```
```

## 4. Module Tests (Terraform 1.6+)

### `tests/basic.tftest.hcl`
```hcl
run "basic" {
  command = plan

  variables {
    name        = "test"
    environment = "dev"
  }

  assert {
    condition     = aws_s3_bucket.main.bucket == "test-dev-bucket"
    error_message = "Bucket name should follow naming convention"
  }
}

run "validation_fails_invalid_env" {
  command = plan

  variables {
    name        = "test"
    environment = "invalid"
  }

  expect_failures = [var.environment]
}
```

## 5. Validation

```bash
# // turbo
terraform fmt -recursive -check
# STOP if not formatted

terraform init -backend=false
# STOP if init fails

terraform validate
# STOP if invalid

# Lint with TFLint
tflint --init
tflint --recursive
# STOP if errors

# Security scan
checkov -d . --compact
# Review HIGH/CRITICAL findings
```

## 6. Documentation

Auto-generate README:

```bash
# Generate markdown table format
terraform-docs markdown table --output-file README.md .

# Or generate with custom template
terraform-docs markdown --output-file README.md --config .terraform-docs.yml .
```

### `.terraform-docs.yml` (Optional)
```yaml
formatter: markdown table

sections:
  show:
    - header
    - requirements
    - providers
    - inputs
    - outputs
    - resources

content: |
  {{ .Header }}

  ## Usage

  ```hcl
  module "example" {
    source = "path/to/module"

    name        = "my-app"
    environment = "dev"
  }
  ```

  {{ .Requirements }}
  {{ .Providers }}
  {{ .Inputs }}
  {{ .Outputs }}
  {{ .Resources }}
```

## 7. Checklist

### Variables
- [ ] All variables have `type`
- [ ] All variables have `description`
- [ ] Sensitive variables marked as `sensitive = true`
- [ ] Validation blocks for constrained values
- [ ] Sensible defaults where appropriate

### Outputs
- [ ] All important resource attributes exported
- [ ] All outputs have `description`
- [ ] Sensitive outputs marked as `sensitive = true`

### Security
- [ ] No hardcoded secrets
- [ ] Encryption enabled by default
- [ ] Least privilege IAM policies
- [ ] No `0.0.0.0/0` in security groups (unless intended)

### Documentation
- [ ] README.md generated with terraform-docs
- [ ] At least one working example
- [ ] CHANGELOG.md for versioned modules

### Testing
- [ ] `terraform validate` passes
- [ ] `tflint` passes
- [ ] `checkov` has no HIGH/CRITICAL issues
- [ ] Example can be applied successfully

## Common Patterns

### Conditional Resources
```hcl
resource "aws_resource" "optional" {
  count = var.create_resource ? 1 : 0
  # ...
}

# Reference with splat
output "optional_id" {
  value = try(aws_resource.optional[0].id, null)
}
```

### Dynamic Blocks
```hcl
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port   = ingress.value.from_port
    to_port     = ingress.value.to_port
    protocol    = ingress.value.protocol
    cidr_blocks = ingress.value.cidr_blocks
  }
}
```

### Module Composition
```hcl
module "vpc" {
  source = "./modules/vpc"
  # ...
}

module "eks" {
  source = "./modules/eks"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}
```

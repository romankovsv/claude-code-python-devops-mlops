---
paths:
  - "**/*.tf"
  - "**/*.tfvars"
  - "**/*.yaml"
  - "**/*.yml"
  - "**/Dockerfile*"
  - "**/*.hcl"
  - ".github/workflows/**"
---

# Infrastructure Rules

Non-negotiable rules for all infrastructure code (Terraform, Helm, Kubernetes, Dockerfiles, CI/CD).

## 🔴 Rule 1: No Hardcoded Secrets

**Never** hardcode secrets, passwords, API keys, or credentials in any file.

```hcl
# ❌ FORBIDDEN
resource "aws_db_instance" "postgres" {
  password = "mysecretpassword123"   # NEVER
}

# ✅ REQUIRED
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/postgres/password"
}
resource "aws_db_instance" "postgres" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```

```yaml
# ❌ FORBIDDEN in K8s manifests
env:
  - name: DB_PASSWORD
    value: "mysecretpassword"   # NEVER

# ✅ REQUIRED — reference K8s Secret
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: db-password
```

**Enforcement**: gitleaks pre-commit hook + CI scan. Any secret found in git history = immediate rotation required.

---

## 🔴 Rule 2: Infrastructure as Code Only (No ClickOps)

Every AWS resource **must** be defined in Terraform or AWS CDK. No manual console changes.

```bash
# Before ANY manual AWS console change — ask:
# "Can this wait 30 minutes for IaC?"
# If yes → write the code
# If no (incident) → document the change, create IaC PR immediately after

# Detect drift (run daily in CI)
terraform plan -detailed-exitcode
# Exit code 2 = drift detected (someone did ClickOps)
```

**Why**: Manual changes are not reproducible, not audited, not reviewable, and create drift that causes future apply failures.

**Exceptions**: Incident response only — must create IaC PR within 24 hours of incident resolution.

---

## 🔴 Rule 3: Idempotency Required

All infrastructure operations must be safe to run multiple times with the same result.

```hcl
# ✅ Terraform is declarative — idempotent by design
# ✅ Helm upgrade --install is idempotent

# ❌ Shell scripts in user_data without idempotency guards
user_data = <<-EOF
  apt-get install -y nginx   # ✅ idempotent
  echo "setup done" >> /var/log/setup.log  # ❌ appends on every run
  useradd appuser  # ❌ fails on second run
EOF

# ✅ Correct: idempotent shell scripts
user_data = <<-EOF
  #!/bin/bash
  apt-get install -y nginx
  id -u appuser &>/dev/null || useradd -r appuser   # check before create
EOF
```

```yaml
# ✅ Kubernetes — all resources are idempotent via apply
kubectl apply -f deployment.yaml   # ✅ safe to run 100 times

# ❌ Never use kubectl create (fails if exists)
kubectl create -f deployment.yaml  # ❌ not idempotent
```

---

## 🔴 Rule 4: Observability Mandatory

Every deployed service **must** have metrics, logs, and health checks from day one.

```yaml
# ✅ Required on every Kubernetes Deployment
containers:
  - name: app
    readinessProbe:          # REQUIRED — traffic gate
      httpGet:
        path: /health/ready
        port: 8000
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:           # REQUIRED — restart gate
      httpGet:
        path: /health/live
        port: 8000
      initialDelaySeconds: 30
      periodSeconds: 30

# ✅ Required — Prometheus metrics endpoint
# App must expose /metrics or have sidecar exporter
```

```hcl
# ✅ Required on every RDS instance
resource "aws_db_instance" "postgres" {
  monitoring_interval          = 60        # Enhanced Monitoring
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  performance_insights_enabled = true
  performance_insights_retention_period = 7
}
```

```python
# ✅ Required — structured logging in every service
import logging
import json

logger = logging.getLogger(__name__)
logger.info("Request processed", extra={
    "order_id": order_id,
    "duration_ms": duration,
    "status": "success",
})
# NEVER: print("Request processed")
```

---

## 🔴 Rule 5: Rollback Strategy Always

Every deployment **must** have a documented and tested rollback path.

```bash
# ✅ Helm — rollback always available
helm upgrade --install myapp ./chart \
  --atomic \        # auto-rollback if health checks fail within timeout
  --wait \
  --timeout 5m

# Manual rollback:
helm history myapp -n production
helm rollback myapp 3 -n production --wait

# ✅ ArgoCD — rollback to previous revision
argocd app rollback myapp-prod

# ✅ Kubernetes deployment — always keep previous ReplicaSet
# revisionHistoryLimit: 3  (keep last 3 ReplicaSets for rollback)
```

```hcl
# ✅ Terraform — never apply without reviewed plan
terraform plan -out=tfplan        # save plan
terraform show tfplan              # human reviews it
terraform apply tfplan             # apply only the reviewed plan

# ✅ Critical resources — prevent accidental destroy
resource "aws_s3_bucket" "state" {
  lifecycle {
    prevent_destroy = true   # REQUIRED on: state bucket, prod DB, EKS cluster
  }
}
```

**Before every production deployment, confirm:**
- [ ] Previous version still available (Helm history / ArgoCD revision / Docker image in ECR)
- [ ] Rollback command tested or confirmed viable
- [ ] Database migrations are reversible (or have down migration script)
- [ ] Estimated rollback time < 5 minutes

---

## Additional Rules

### IAM Least Privilege
```hcl
# ❌ FORBIDDEN
resource "aws_iam_role_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "*"          # NEVER wildcard action
      Resource = "*"          # NEVER wildcard resource
    }]
  })
}

# ✅ REQUIRED
resource "aws_iam_role_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = ["arn:aws:s3:::my-bucket/*"]
    }]
  })
}
```

### Mandatory Resource Tagging
```hcl
# ✅ ALL resources must have these tags
locals {
  required_tags = {
    env         = var.environment          # dev / staging / prod
    team        = var.team                 # platform / ml / backend
    managed_by  = "terraform"             # never "manual"
    cost_center = var.cost_center         # for billing allocation
  }
}

# ✅ Use AWS provider default_tags to apply automatically
provider "aws" {
  default_tags {
    tags = local.required_tags
  }
}
```

### Version Pinning (All IaC)
```hcl
# ✅ REQUIRED — pin everything
terraform {
  required_version = "~> 1.8"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.40" }
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.2"    # ✅ exact or constrained version — never unpinned
}
```


---
name: security-infra
description: Infrastructure security patterns — Checkov, tfsec, OPA Gatekeeper, gitleaks, secrets management, Zero Trust, IAM least privilege, and supply chain security.
origin: custom
---

# Infrastructure Security Patterns

Security scanning, policy enforcement, and Zero Trust for IaC and Kubernetes.

## When to Activate

- Writing or reviewing Terraform / Kubernetes manifests
- Setting up pre-commit or CI security scanning
- Enforcing policy as code with OPA Gatekeeper
- Auditing IAM permissions for least privilege
- Detecting secrets in code or git history

## Checkov — IaC Static Analysis

```bash
# Scan Terraform
checkov -d infra/environments/prod --framework terraform \
  --output cli \
  --output sarif \
  --output-file-path results/ \
  --soft-fail \
  --compact

# Scan Kubernetes manifests
checkov -d k8s/ --framework kubernetes \
  --check CKV_K8S_8,CKV_K8S_9,CKV_K8S_10,CKV_K8S_14,CKV_K8S_22,CKV_K8S_28

# Scan Dockerfile
checkov -f Dockerfile --framework dockerfile

# Scan with custom skip list
checkov -d infra/ \
  --skip-check CKV_AWS_18,CKV_AWS_19 \
  --skip-check-path infra/modules/legacy/

# In CI — fail on HIGH severity only
checkov -d infra/ --severity HIGH --hard-fail-on HIGH
```

```yaml
# .checkov.yaml — project-wide config
framework:
  - terraform
  - kubernetes
  - dockerfile
  - github_actions

severity: HIGH
compact: true
output:
  - cli
  - sarif

skip-check:
  - CKV_AWS_144    # cross-region S3 replication — not required
  - CKV2_AWS_6     # S3 public access — handled at account level

soft-fail: false
```

## tfsec — Terraform Security Scanner

```bash
# Run tfsec
tfsec infra/ \
  --format lovely \
  --minimum-severity HIGH \
  --exclude-downloaded-modules

# With custom config
tfsec infra/ --config-file .tfsec.yaml

# Output SARIF for GitHub Security tab
tfsec infra/ --format sarif --out tfsec-results.sarif
```

```yaml
# .tfsec.yaml
severity_overrides:
  aws-s3-enable-versioning: warning    # downgrade to warning
  aws-rds-no-public-access: error      # keep as error

exclude_results:
  - rule_id: aws-ec2-no-public-egress-sgr
    description: "Temporary — tracked in JIRA-1234"
    expiry_date: "2026-06-01"          # ✅ expiry forces review

custom_checks:
  - code: custom-001
    description: "S3 buckets must have cost-center tag"
    block: resource
    filters:
      - attribute: type
        equals: aws_s3_bucket
    checks:
      - attribute: tags.cost_center
        is_present: true
```

## OPA Gatekeeper — Policy as Code (K8s)

```yaml
# Constraint Template — require resource limits
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' has no memory limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' has no CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.memory
          msg := sprintf("Container '%v' has no memory request", [container.name])
        }
---
# Constraint — enforce in production namespace
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  enforcementAction: deny     # deny | warn | dryrun
  match:
    namespaces:
      - production
      - staging
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
```

```yaml
# OPA policy — no privileged containers
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoprivilegedcontainer

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged container not allowed: '%v'", [container.name])
        }
```

## Gitleaks — Secret Scanning

```bash
# Scan current working tree
gitleaks detect --source . --verbose

# Scan entire git history (pre-push check)
gitleaks detect --source . --log-opts="--all" --verbose

# Output report
gitleaks detect --source . --report-path gitleaks-report.json --report-format json

# As pre-commit hook
gitleaks protect --staged --verbose
```

```toml
# .gitleaks.toml
[extend]
useDefault = true    # extend default ruleset

[[rules]]
id          = "custom-jwt-secret"
description = "JWT secret key"
regex       = '''(?i)jwt[_\-]?secret[_\-]?key\s*=\s*['"]?[a-zA-Z0-9+/]{32,}'''
severity    = "CRITICAL"
tags        = ["jwt", "secret"]

[[rules]]
id          = "custom-aws-account-id"
description = "AWS Account ID"
regex       = '''\b\d{12}\b'''
severity    = "WARNING"

[allowlist]
description = "Allowlist for known false positives"
regexes     = [
  "EXAMPLE",
  "PLACEHOLDER",
  "your-secret-here",
]
paths = [
  "tests/fixtures/",
  "docs/",
]
```

## Secrets Management — Best Practices

```python
# ✅ Correct — load from environment / Secrets Manager
import os
import boto3
import json


def get_secret(secret_name: str, region: str = "us-east-1") -> dict:
    """Fetch secret from AWS Secrets Manager."""
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])


# At application startup
secrets = get_secret("prod/app/db")
DB_PASSWORD = secrets["password"]
DB_HOST     = secrets["host"]

# For local dev — .env file (never committed)
from dotenv import load_dotenv
load_dotenv()  # reads .env — excluded from git via .gitignore
API_KEY = os.environ["OPENAI_API_KEY"]  # raises KeyError if missing — fail fast
```

```bash
# External Secrets Operator — sync K8s Secrets from AWS Secrets Manager
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: prod/app/db
        property: password
    - secretKey: api-key
      remoteRef:
        key: prod/app/keys
        property: openai_api_key
EOF
```

## Zero Trust Principles

```yaml
# Never trust, always verify
# 1. mTLS between all services (Istio)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT    # ✅ mTLS enforced — no plaintext traffic
---
# 2. Authorisation policy — deny by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}    # empty spec = deny all
---
# 3. Allow specific service-to-service communication
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/frontend-service-account"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/*"]
```

## Pre-commit Configuration

```yaml
# .pre-commit-config.yaml
repos:
  # Secret scanning
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks

  # Terraform
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      - id: terraform_checkov
        args:
          - --args=--config-file __GIT_WORKING_DIR__/.checkov.yaml

  # Kubernetes
  - repo: https://github.com/yannh/kubeconform
    rev: v0.6.4
    hooks:
      - id: kubeconform
        args:
          - -strict
          - -ignore-missing-schemas
          - -kubernetes-version
          - "1.30.0"

  # Python
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.1
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  # General
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key
      - id: no-commit-to-branch
        args: [--branch, main]
```

## IAM Least Privilege Checklist

```bash
# Find overly permissive policies
aws iam list-policies --scope Local --query \
  'Policies[?PolicyName!=`AdministratorAccess`].{Name:PolicyName,Arn:Arn}'

# Check for wildcard actions
aws iam get-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/MyPolicy \
  --version-id v1 \
  --query 'PolicyVersion.Document.Statement[?Action==`*`]'

# Use IAM Access Analyzer to find unused permissions
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/MyAnalyzer

# Generate least-privilege policy from CloudTrail
aws iam generate-policy \
  --policy-generation-details '{"PrincipalArn":"arn:aws:iam::123456789012:role/AppRole"}' \
  --cloud-trail-details '{"trailArn":"arn:aws:cloudtrail:us-east-1:123456789012:trail/MyTrail","startTime":"2026-01-01T00:00:00Z","endTime":"2026-03-29T00:00:00Z"}'
```

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| ❌ Secrets in git | Gitleaks pre-commit + External Secrets Operator |
| ❌ `"Action": "*"` IAM | Least privilege — specific actions and resources |
| ❌ Privileged containers | `securityContext.privileged: false` + OPA Gatekeeper |
| ❌ No network policy | Default deny + explicit allow rules |
| ❌ No IaC security scanning | Checkov + tfsec in every PR |
| ❌ Plain HTTP between services | Istio mTLS STRICT mode |
| ❌ Long-lived AWS credentials | IRSA (EKS) or OIDC federation — never static keys |


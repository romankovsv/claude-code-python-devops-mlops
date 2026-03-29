# DevOps & MLOps Tools Setup Guide

Complete installation guide for all DevOps/MLOps tools used in this project.

## macOS (Homebrew)

```bash
# Install Homebrew if not present
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

---

## IaC — Terraform & AWS CDK

### Terraform (via tfenv — version manager)

```bash
brew install tfenv
tfenv install 1.8.5
tfenv use 1.8.5

# Verify
terraform version
# Terraform v1.8.5
```

### tflint — Terraform linter

```bash
brew install tflint

# Init with AWS ruleset
tflint --init

# Create config
cat > .tflint.hcl <<'EOF'
plugin "aws" {
  enabled = true
  version = "0.31.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
EOF

# Verify
tflint --version
```

### AWS CDK

```bash
# Requires Node.js
brew install node

# Install CDK CLI globally
npm install -g aws-cdk

# Python CDK library (in your virtualenv)
pip install aws-cdk-lib constructs

# Verify
cdk --version
# 2.x.x

# Bootstrap your AWS account (run once per account/region)
cdk bootstrap aws://YOUR_ACCOUNT_ID/us-east-1
```

---

## Security Scanning

### checkov — IaC security scanner

```bash
# macOS / Linux
pip install checkov

# Verify
checkov --version

# Run
checkov -d infra/ --framework terraform
checkov -d k8s/ --framework kubernetes
```

### gitleaks — Secret scanner

```bash
# macOS
brew install gitleaks

# Linux
GITLEAKS_VERSION="8.18.2"
curl -sSL "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz" \
  | tar -xz -C /usr/local/bin gitleaks

# Verify
gitleaks version

# Run
gitleaks detect --source . --verbose
gitleaks protect --staged  # pre-commit mode
```

### trivy — Container & IaC vulnerability scanner

```bash
# macOS
brew install trivy

# Linux
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.50.4

# Verify
trivy --version

# Scan image
trivy image --severity CRITICAL,HIGH my-app:v1.0.0

# Scan Kubernetes cluster
trivy k8s --report summary cluster
```

### tfsec — Terraform security scanner

```bash
# macOS
brew install tfsec

# Linux
curl -sSL "https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64" \
  -o /usr/local/bin/tfsec && chmod +x /usr/local/bin/tfsec

# Verify
tfsec --version

# Run
tfsec infra/ --minimum-severity HIGH
```

---

## Kubernetes Tools

### kubectl

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

### helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version

# Add common repos
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add jetstack https://charts.jetstack.io
helm repo add seldonio https://storage.googleapis.com/seldon-charts
helm repo update
```

### kubeconform — Kubernetes manifest validator

```bash
# macOS
brew install kubeconform

# Linux
KUBECONFORM_VERSION="0.6.4"
curl -sSL "https://github.com/yannh/kubeconform/releases/download/v${KUBECONFORM_VERSION}/kubeconform-linux-amd64.tar.gz" \
  | tar -xz -C /usr/local/bin kubeconform

# Verify
kubeconform -v

# Validate
kubeconform -strict -kubernetes-version 1.30.0 k8s/*.yaml
```

### istioctl

```bash
# macOS / Linux
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.21.0 sh -
sudo mv istio-1.21.0/bin/istioctl /usr/local/bin/

# Verify
istioctl version

# Install Istio to cluster
istioctl install --set profile=production -y
```

### ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Verify
argocd version --client
```

---

## AWS Tools

### AWS CLI v2

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure
aws configure
# AWS Access Key ID: (use OIDC/SSO in CI — keys only for local dev)
# Default region: us-east-1
# Output format: json

# Verify
aws --version
aws sts get-caller-identity
```

### AWS SSO (recommended over static keys)

```bash
aws configure sso
# SSO start URL: https://my-org.awsapps.com/start
# SSO Region: us-east-1
# Follow browser prompts

# Login
aws sso login --profile my-profile
```

---

## MLOps Tools

### DVC — Data Version Control

```bash
# pip (all platforms)
pip install "dvc[s3]"     # with S3 support

# macOS (brew)
brew install dvc

# Verify
dvc --version

# Initialize in project
dvc init
dvc remote add -d myremote s3://my-bucket/dvc-cache
dvc remote modify myremote region us-east-1
```

### MLflow

```bash
pip install mlflow==2.12.0

# Verify
mlflow --version

# Start local tracking server (dev only)
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlruns \
  --host 0.0.0.0 \
  --port 5000

# For production — use Docker Compose (see skills/mlflow/SKILL.md)
```

### Weights & Biases

```bash
pip install wandb==0.16.6

# Login
wandb login  # prompts for API key from wandb.ai

# Verify
python -c "import wandb; print(wandb.__version__)"
```

### Evidently AI — Model Drift Detection

```bash
pip install evidently==0.4.28

# Verify
python -c "import evidently; print(evidently.__version__)"
```

### SageMaker SDK

```bash
pip install sagemaker==2.214.0 boto3==1.34.0

# Verify
python -c "import sagemaker; print(sagemaker.__version__)"
```

---

## Pre-commit — All Hooks Together

```bash
# Install pre-commit
pip install pre-commit
# or
brew install pre-commit

# Install hooks into git repo
pre-commit install
pre-commit install --hook-type pre-push

# Create config (copy to project root)
cat > .pre-commit-config.yaml <<'EOF'
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
EOF

# Run all hooks on all files (first time)
pre-commit run --all-files

# Verify
pre-commit --version
```

---

## Quick Verification Script

Run after setup to confirm all tools are installed:

```bash
#!/bin/bash
# scripts/check-devops-tools.sh

set -e
PASS="✅"
FAIL="❌"

check() {
  local name=$1
  local cmd=$2
  if command -v "$cmd" >/dev/null 2>&1; then
    version=$($cmd --version 2>/dev/null | head -1 || echo "ok")
    echo "$PASS $name: $version"
  else
    echo "$FAIL $name: NOT INSTALLED"
  fi
}

echo "=== IaC ==="
check "terraform"    terraform
check "tflint"       tflint
check "cdk"          cdk

echo ""
echo "=== Security ==="
check "checkov"      checkov
check "gitleaks"     gitleaks
check "trivy"        trivy
check "tfsec"        tfsec

echo ""
echo "=== Kubernetes ==="
check "kubectl"      kubectl
check "helm"         helm
check "kubeconform"  kubeconform
check "argocd"       argocd
check "istioctl"     istioctl

echo ""
echo "=== AWS ==="
check "aws"          aws

echo ""
echo "=== MLOps ==="
check "dvc"          dvc
check "mlflow"       mlflow
check "pre-commit"   pre-commit

echo ""
echo "=== Python packages ==="
python -c "import wandb; print('✅ wandb:', wandb.__version__)" 2>/dev/null || echo "❌ wandb: NOT INSTALLED"
python -c "import sagemaker; print('✅ sagemaker:', sagemaker.__version__)" 2>/dev/null || echo "❌ sagemaker: NOT INSTALLED"
python -c "import evidently; print('✅ evidently:', evidently.__version__)" 2>/dev/null || echo "❌ evidently: NOT INSTALLED"
```

```bash
chmod +x scripts/check-devops-tools.sh
./scripts/check-devops-tools.sh
```

---

## Linux (Ubuntu/Debian apt)

```bash
# Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# kubectl
sudo apt-get install -y kubectl

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# All pip tools
pip install checkov tflint dvc[s3] mlflow==2.12.0 wandb sagemaker evidently pre-commit
```


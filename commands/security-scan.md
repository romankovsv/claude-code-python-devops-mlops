---
description: Run comprehensive infrastructure security scan — IAM audit, secret detection, CVE scanning, OWASP infra checklist. Invokes security-auditor agent.
---

# /security-scan

Invokes the **security-auditor** agent to perform a comprehensive infrastructure security audit.

## What This Command Does

1. **Secret Scanning** — gitleaks on full git history + current working tree
2. **IaC Security Scan** — checkov + tfsec on all Terraform and K8s manifests
3. **IAM Audit** — wildcard policies, unused roles, missing MFA
4. **Container Security** — Trivy image scan for CVEs
5. **AWS Security Services** — GuardDuty findings, Security Hub score, CloudTrail coverage
6. **Prioritised Remediation** — CRITICAL / HIGH / MEDIUM with concrete fix commands

## When to Use

- Before any production release
- Monthly security review
- After a security incident
- When onboarding a new AWS account or environment
- When any new IAM role or policy is created

## Scope

```
# Automatically scans:
infra/**/*.tf          → checkov + tfsec
k8s/**/*.yaml          → checkov + kubeconform
.github/workflows/*.yml → checkov (GitHub Actions)
. (full tree)          → gitleaks (secrets)
AWS account            → IAM + GuardDuty + Security Hub (via AWS CLI)
Container images       → Trivy (if image tag provided)
```

## Output Format

```
## Security Audit Report — [Environment] — [Date]

### Scan Summary
| Tool      | Findings | Critical | High | Medium |
|-----------|---------|---------|------|--------|
| gitleaks  | X       | Y       | -    | -      |
| checkov   | X       | Y       | Z    | W      |
| tfsec     | X       | Y       | Z    | -      |
| Trivy     | X       | Y       | Z    | -      |

### 🔴 CRITICAL (fix within 24h)
...

### 🟠 HIGH (fix within 1 week)
...

### 🟡 MEDIUM (fix within 1 month)
...

### ✅ Passing Controls
...
```


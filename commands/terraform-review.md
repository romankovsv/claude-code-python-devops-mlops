---
description: Review Terraform, Helm, or Kubernetes manifests for security issues, missing tags, idempotency problems, and IaC best practices. Invokes infra-reviewer agent.
---

# /terraform-review

Invokes the **infra-reviewer** agent to perform a comprehensive IaC code review.

## What This Command Does

1. **Get Changed Files** — `git diff --name-only HEAD~1` scoped to `.tf`, `.yaml`, `values*.yaml`
2. **Security Scan** — checkov + tfsec + gitleaks on changed files
3. **Best Practices Check** — tagging, pinned versions, idempotency, rollback safety
4. **Rollback Verification** — confirm a rollback strategy exists
5. **Prioritised Report** — CRITICAL / HIGH / MEDIUM findings

## When to Use

- Before merging any Terraform PR
- Before applying Helm chart changes
- Before merging Kubernetes manifest changes
- As part of pre-deployment checklist

## Scope

Automatically detects and reviews:
- `**/*.tf` — Terraform resources, modules, providers
- `**/values*.yaml` — Helm chart values
- `**/Chart.yaml` — Helm chart metadata
- `**/k8s/*.yaml` or `**/manifests/*.yaml` — K8s manifests

## Output Format

```
## IaC Review — [Branch/PR Name]

### Security Scan Results
checkov: X findings (Y critical, Z high)
tfsec:   X findings
gitleaks: X secrets detected

### 🔴 CRITICAL (block merge)
...

### 🟠 HIGH (fix before merge)
...

### 🟡 MEDIUM (fix in follow-up PR)
...

### ✅ Approved Patterns
...

### Rollback Strategy
✅ Confirmed: helm history / terraform state backup / ArgoCD revision available
```


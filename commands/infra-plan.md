---
description: Design or review infrastructure plan — AWS service selection, IaC structure, cost vs reliability trade-offs. Invokes devops-architect. WAIT for user CONFIRM before generating any code.
---

# /infra-plan

Invokes the **devops-architect** agent to create a comprehensive infrastructure plan.

## What This Command Does

1. **Gather Requirements** — workload profile, SLO targets, budget, team constraints
2. **Propose Architecture** — AWS services, network topology, IaC approach (CDK vs Terraform)
3. **Trade-Off Analysis** — cost vs reliability for each key decision
4. **Cost Estimate** — monthly AWS bill breakdown
5. **Wait for Confirmation** — MUST receive user approval before generating any IaC code

## When to Use

- Starting a new AWS environment from scratch
- Migrating existing infrastructure to IaC
- Choosing between EKS vs ECS, CDK vs Terraform, Aurora vs RDS
- Planning multi-region or disaster recovery architecture

## Output Format

```
## Infrastructure Plan — [Project Name]

### Architecture Overview
[diagram / component list]

### AWS Services Selected
| Component | Service | Justification |
|-----------|---------|--------------|

### Trade-Offs
| Decision | Choice | Pros | Cons | Alternatives |
|----------|--------|------|------|-------------|

### Cost Estimate
| Resource | Monthly Cost |
|----------|-------------|
| Total    | $X,XXX/month |

### IaC Structure
[directory tree]

### Next Steps (after approval)
1. ...
```

**CRITICAL**: The devops-architect agent will NOT write any IaC code until you explicitly confirm the plan.


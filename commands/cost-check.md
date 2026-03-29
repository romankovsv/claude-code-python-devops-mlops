---
description: Analyse AWS costs, identify waste, and get rightsizing recommendations. Invokes cost-optimizer agent with Cost Explorer analysis and concrete savings actions.
---

# /cost-check

Invokes the **cost-optimizer** agent to analyse AWS spend and identify savings opportunities.

## What This Command Does

1. **Spend Overview** — last 30/90 days by service, team tag, and environment
2. **Zombie Resource Detection** — unattached EBS, unused EIPs, idle RDS, empty ALBs
3. **Rightsizing** — EC2/RDS/EKS Compute Optimizer recommendations
4. **Reserved vs Spot Analysis** — current coverage vs optimal strategy
5. **Tagging Audit** — resources without cost_center / team tags
6. **Anomaly Detection** — unusual cost spikes vs baseline

## When to Use

- Monthly cost review
- AWS bill exceeded budget
- Before committing to Reserved Instances
- After infrastructure change that might have increased costs
- Quarterly FinOps review

## Required Context

```
AWS Account: [account ID or profile]
Budget target: $X,XXX/month
Team/tag filter: [optional — leave blank for full account]
Focus area: [EC2 / RDS / EKS / S3 / all]
```

## Output Format

```
## AWS Cost Report — [Month Year]

### 📊 Spend Summary
- Total: $X,XXX/month
- vs Budget: [+/-X%]
- vs Last Month: [+/-X% MoM]
- Top 3 services: EC2 ($X), RDS ($X), Data Transfer ($X)

### 🔴 Immediate Savings (< 1 week)
| Resource | Monthly Waste | Action |
|----------|--------------|--------|

### 🟠 Medium-Term Savings (1-4 weeks)
| Opportunity | Monthly Saving | Effort |
|-------------|---------------|--------|

### 🟡 Long-Term Savings (1-3 months)
| Opportunity | Monthly Saving | Commitment |
|-------------|---------------|-----------|

### 💰 Total Potential Savings
$X,XXX/month (X% reduction)

### Tagging Gaps
X resources missing cost_center tag → unable to allocate to teams
```


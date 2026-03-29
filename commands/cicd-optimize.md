---
description: Optimise CI/CD pipeline for speed, reliability, and security. Invokes ci-cd-engineer agent to analyse bottlenecks, improve caching, and enforce security best practices.
---

# /cicd-optimize

Invokes the **ci-cd-engineer** agent to analyse and optimise CI/CD pipelines.

## What This Command Does

1. **Analyse Current Pipeline** — execution times, bottlenecks, parallelisation gaps
2. **Security Review** — OIDC vs static keys, secret handling, supply chain
3. **Performance Optimisation** — caching, parallelisation, Docker layer cache
4. **Reliability Improvements** — rollback, atomic deploys, health checks
5. **DORA Metrics Baseline** — deployment frequency, lead time, failure rate

## When to Use

- Pipeline takes > 10 minutes end-to-end
- Frequent flaky tests or deployment failures
- Migrating to OIDC from static AWS credentials
- Adding ArgoCD GitOps to existing push-based pipeline
- Setting up multi-environment promotion gates

## Required Context

```
Pipeline file: .github/workflows/ci-cd.yml (or provide path)
Current duration: X minutes
Main bottleneck: [if known]
Target environment: GitHub Actions / GitLab CI / Jenkins
```

## Output Format

```
## CI/CD Optimisation Report — [Pipeline Name]

### Current State
- Duration: X min | Target: Y min
- DORA Deployment Frequency: N deploys/day
- Security: [OIDC / static keys / unknown]

### 🔴 CRITICAL Security Issues
...

### Performance Optimisations
| Change | Effort | Time Saving |
|--------|--------|------------|

### Updated Pipeline
[modified workflow YAML with improvements applied]

### Expected After Optimisation
- Duration: Y min (Z% improvement)
- Estimated annual savings: X hours developer time
```


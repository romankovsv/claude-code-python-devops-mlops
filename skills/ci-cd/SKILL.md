---
name: ci-cd
description: CI/CD patterns — GitHub Actions with caching, ArgoCD GitOps, trunk-based development, rollback strategies, DORA metrics, and multi-environment promotion pipelines.
origin: custom
---

# CI/CD Patterns

Production CI/CD best practices for DevOps and MLOps pipelines.

## When to Activate

- Setting up or optimising GitHub Actions pipelines
- Implementing GitOps with ArgoCD
- Designing multi-environment promotion workflows
- Configuring rollback strategies
- Measuring pipeline health with DORA metrics

## GitHub Actions — Optimised Python Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  PYTHON_VERSION: "3.12"

jobs:
  # ─── CI ──────────────────────────────────────────────────────────────
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip                              # ✅ cache pip dependencies

      - name: Install dev dependencies
        run: pip install ruff black mypy

      - name: Ruff lint
        run: ruff check .

      - name: Black format check
        run: black . --check

      - name: Mypy type check
        run: mypy . --ignore-missing-imports

  test:
    runs-on: ubuntu-latest
    needs: lint-and-type-check
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run tests with coverage
        run: |
          pytest \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            --alluredir=allure-results \
            -n auto \                            # ✅ parallel execution
            -v
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost/testdb
          REDIS_URL: redis://localhost:6379

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.xml

      - name: Upload Allure results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-results
          path: allure-results/

  security:
    runs-on: ubuntu-latest
    needs: lint-and-type-check
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip

      - name: Install security tools
        run: pip install bandit pip-audit

      - name: Bandit SAST
        run: bandit -r src/ -f json -o bandit-report.json
        continue-on-error: true

      - name: pip-audit dependency scan
        run: pip-audit --require-hashes -r requirements.txt

      - name: Gitleaks secret scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ─── BUILD ───────────────────────────────────────────────────────────
  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=,format=short        # ✅ git SHA tag
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha                   # ✅ GitHub Actions layer cache
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true                             # ✅ SBOM for supply chain security

      - name: Image vulnerability scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

  # ─── DEPLOY STAGING ──────────────────────────────────────────────────
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - uses: actions/checkout@v4

      - name: Update Helm values (GitOps)
        run: |
          # Update image tag in GitOps repo → ArgoCD picks it up
          git clone https://x-token:${{ secrets.GITOPS_TOKEN }}@github.com/my-org/gitops-repo
          cd gitops-repo
          sed -i "s|tag:.*|tag: ${{ github.sha }}|" apps/staging/values.yaml
          git config user.email "ci@myapp.com"
          git config user.name "CI Bot"
          git commit -am "ci: deploy staging ${{ github.sha }}"
          git push

  # ─── DEPLOY PROD (manual gate) ───────────────────────────────────────
  deploy-prod:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    steps:
      - name: Deploy to production via ArgoCD
        run: |
          argocd app sync myapp-prod \
            --server $ARGOCD_SERVER \
            --auth-token $ARGOCD_TOKEN \
            --revision ${{ github.sha }} \
            --timeout 300
        env:
          ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
          ARGOCD_TOKEN: ${{ secrets.ARGOCD_TOKEN }}
```

## ArgoCD — GitOps Application

```yaml
# gitops-repo/apps/production/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/my-org/gitops-repo
    targetRevision: main
    path: apps/production
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # ✅ remove resources deleted from git
      selfHeal: true     # ✅ revert manual kubectl changes
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        maxDuration: 3m
        factor: 2
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas   # managed by HPA
```

## Rollback Strategies

```bash
# 1. ArgoCD rollback to previous revision
argocd app rollback myapp-prod

# 2. Helm rollback
helm history myapp -n production
helm rollback myapp 3 -n production --wait

# 3. Kubectl rollout undo
kubectl rollout undo deployment/myapp -n production
kubectl rollout status deployment/myapp -n production --timeout=5m

# 4. GitOps rollback (revert commit in gitops repo)
git revert HEAD --no-edit
git push origin main
# ArgoCD auto-syncs and rolls back

# 5. Blue-Green instant rollback (switch service selector)
kubectl patch service myapp -n production \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

## Trunk-Based Development

```
main (trunk)
├── feature branches — lived max 1-2 days
├── commits trigger CI immediately
├── feature flags control unfinished features in prod
└── release tags trigger prod deploy

# ✅ DO
git checkout -b feat/user-auth
# work for <2 days
git push origin feat/user-auth
# open PR → CI passes → merge → delete branch

# ❌ DON'T
# Long-lived feature branches (weeks)
# Gitflow with develop/release/hotfix branches
```

## DORA Metrics — Definition & Targets

| Metric | Definition | Elite | High | Medium | Low |
|--------|-----------|-------|------|--------|-----|
| **Deployment Frequency** | How often to prod | On demand (multiple/day) | Daily–weekly | Weekly–monthly | < Monthly |
| **Lead Time for Changes** | Commit → prod | < 1 hour | 1 day–1 week | 1 week–1 month | > 1 month |
| **Change Failure Rate** | % deploys causing incident | < 5% | 5-10% | 10-15% | > 15% |
| **MTTR** | Time to restore after failure | < 1 hour | < 1 day | < 1 day | > 1 day |

```python
# Measure deployment frequency via GitHub API
import requests
from datetime import datetime, timedelta

token = "ghp_..."
owner, repo = "my-org", "my-app"

# Count prod deployments in last 30 days
deployments = requests.get(
    f"https://api.github.com/repos/{owner}/{repo}/deployments",
    headers={"Authorization": f"Bearer {token}"},
    params={"environment": "production", "per_page": 100},
).json()

since = datetime.utcnow() - timedelta(days=30)
recent = [d for d in deployments if datetime.fromisoformat(d["created_at"][:-1]) > since]
print(f"Deployment frequency: {len(recent)/30:.1f} deploys/day")
```

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| ❌ Long-lived feature branches | Trunk-based: branch lives < 2 days |
| ❌ Manual deployments | ArgoCD GitOps — never `kubectl apply` by hand |
| ❌ No rollback plan | Helm history + ArgoCD rollback before every deploy |
| ❌ Secrets in GitHub Actions env vars | GitHub Secrets + OIDC for AWS (no long-lived keys) |
| ❌ No Docker layer cache | `cache-from: type=gha` in build-push-action |
| ❌ Deploying without image vulnerability scan | Trivy scan in build job, fail on CRITICAL |
| ❌ No deployment gate for production | GitHub Environment with required reviewers |


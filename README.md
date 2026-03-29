# Python + DevOps + MLOps Stack for Claude Code

**[English](#english)** | **[Русский](#russian)**

---

<a id="english"></a>

## What Is This?

A **ready-to-use toolkit** that supercharges Claude Code for Python development, QA automation, DevOps infrastructure, and MLOps pipelines. Once you drop it into your project, Claude Code instantly becomes a senior engineer who follows your team's exact standards.

**Without this toolkit** — Claude Code is a smart generalist.
**With this toolkit** — Claude Code writes Pythonic code, designs architectures, generates OOP test suites with Allure, reviews for OWASP vulnerabilities, designs AWS infrastructure with CDK/Terraform, manages ML pipelines with SageMaker/MLflow, and follows your exact conventions.

## How Claude Code Configuration Works

When you run Claude Code in a project, it reads special files from a `.claude/` directory:

```
your-project/
├── CLAUDE.md              # Main instructions — "the brain"
├── .pre-commit-config.yaml  # gitleaks + terraform + ruff + kubeconform
├── .tflint.hcl            # Terraform linting config
├── .checkov.yaml          # IaC security scanning config
├── .claude/
│   ├── rules/
│   │   ├── coding-style.md    # Python — activates on *.py
│   │   ├── testing.md         # Tests — activates on test_*.py
│   │   └── infra.md           # DevOps — activates on *.tf, *.yaml, Dockerfile
│   ├── skills/
│   │   ├── fastapi-patterns/  # Deep Python/framework knowledge
│   │   ├── kubernetes/        # Deep K8s knowledge
│   │   ├── terraform/         # Deep IaC knowledge
│   │   └── mlops/             # Deep ML pipeline knowledge
│   ├── agents/
│   │   ├── python-reviewer.md    # Python code review
│   │   ├── devops-architect.md   # AWS infrastructure design
│   │   ├── k8s-debugger.md       # Kubernetes debugging
│   │   └── mlops-engineer.md     # ML pipeline design
│   ├── commands/
│   │   ├── /plan, /tdd, /verify   # Python workflows
│   │   ├── /infra-plan            # Design AWS infrastructure
│   │   ├── /k8s-debug             # Debug Kubernetes
│   │   └── /security-scan         # Security audit
│   └── hooks.json         # Auto-actions: ruff, terraform fmt, checkov, gitleaks
└── src/  (or infra/, k8s/, ml/)
```

This toolkit provides all of these components — pre-configured for **Python, DevOps, and MLOps**.

---

## Components Explained

### CLAUDE.md — The Brain

The **single most important file**. Sits in your project root. Claude reads it at the start of every conversation and follows it as law.

Our CLAUDE.md includes:
- Python conventions (PEP 8, type hints, black/ruff/mypy)
- Strict OOP test rules (classes, service layers, Allure — mandatory)
- Framework rules (FastAPI, Django, SQLAlchemy, Celery)
- Database rules (PostgreSQL types, indexes, N+1 prevention)
- Security rules (no eval, no hardcoded secrets, bandit)
- **DevOps rules** (no ClickOps, IaC only, idempotency, rollback strategy)
- **MLOps rules** (reproducibility, feature store as single source, human gate to prod)
- All available commands and agents listed

**Think of it as**: an onboarding doc for a new team member. Everything they must always follow.

---

### Rules — Always-On Guidelines

**Location**: `rules/` → copy to `.claude/rules/`

Markdown files that **auto-activate by file pattern**. Edit a `.py` file — Python rules kick in automatically.

```yaml
---
paths:
  - "**/*.py"
---
# Python Coding Style
- Follow PEP 8
- Type annotations on all public functions
- black for formatting, ruff for linting
```

| File | What It Enforces |
|------|-----------------|
| `coding-style.md` | PEP 8, type hints, immutability, formatting tools |
| `patterns.md` | Protocol classes, dataclasses, context managers |
| `security.md` | Secrets via env vars, bandit scanning |
| `testing.md` | pytest, 80%+ coverage, test markers |
| `hooks.md` | Auto-format after edit, mypy, print() warnings |
| `infra.md` | **No hardcoded secrets, IaC only, idempotency, observability, rollback** |

**Rules vs CLAUDE.md**: CLAUDE.md = always on for everything. Rules = activate conditionally by file type. `infra.md` activates for `.tf`, `.yaml`, `.yml`, `Dockerfile*`, `.hcl`, and `.github/workflows/` files.

---

### Skills — Deep Knowledge On Demand

**Location**: `skills/` → copy to `.claude/skills/`

**Big difference from rules**: skills are deep reference documents (hundreds of lines) that Claude pulls in when the topic matches. They contain patterns, code examples, configuration templates, anti-patterns.

You don't call skills manually. Claude reads their descriptions and pulls them when relevant:

```
# Python scenario
You: "Create a FastAPI endpoint for user registration"
Claude pulls: fastapi-patterns → pydantic-patterns → sqlalchemy-patterns

# DevOps scenario
You: "Create an EKS cluster with RDS and autoscaling"
Claude pulls: kubernetes → terraform → aws → networking

# MLOps scenario
You: "Set up a fraud detection training pipeline with drift detection"
Claude pulls: mlops → sagemaker → mlflow → observability
```

**31 skills total:**

| Category | Skills |
|----------|--------|
| Core Python | `python-patterns`, `python-testing`, `pydantic-patterns`, `async-http-patterns` |
| Frameworks | `fastapi-patterns`, `django-patterns`, `django-security`, `django-tdd`, `django-verification`, `celery-patterns` |
| Data & Infra | `sqlalchemy-patterns`, `postgres-patterns`, `clickhouse-io`, `redis-patterns`, `database-migrations`, `deployment-patterns`, `docker-patterns` |
| QA Automation | `pytest-oop-patterns`, `allure-reporting`, `api-testing-patterns` |
| **DevOps** | `kubernetes`, `terraform`, `aws`, `ci-cd`, `observability`, `security-infra`, `networking` |
| **MLOps** | `sagemaker`, `seldon-core`, `mlflow`, `mlops` |

**Example — what's inside `fastapi-patterns`**:
- Project structure (app factory, routers, services, repos)
- Pydantic Settings for config
- Dependency injection patterns (`Depends()`)
- Request/Response schemas with validation
- Error handling (custom exceptions, handlers)
- Middleware (logging, CORS)
- Background tasks, WebSockets
- JWT authentication
- Testing with httpx AsyncClient

**Example — what's inside `kubernetes`**:
- Readiness / Liveness / Startup probes (code + config)
- HPA + VPA autoscaling with custom metrics
- PodDisruptionBudget for zero-downtime deploys
- RBAC: ServiceAccount → Role → RoleBinding
- NetworkPolicy: default-deny + explicit allow
- Helm chart structure and values best practices
- Anti-patterns: no resource limits, `:latest` tags, single replica

**Example — what's inside `terraform`**:
- Module structure (modules/ vs environments/)
- Remote state: S3 backend + DynamoDB locking
- Drift detection in CI (`terraform plan -detailed-exitcode`)
- Version pinning: `required_version`, providers, modules
- `moved` blocks for safe resource renames
- `prevent_destroy` on critical resources
- tflint + checkov integration

---

### Agents — Specialized Sub-Brains

**Location**: `agents/` → copy to `.claude/agents/`

Agents are **specialized personas** Claude can spawn. Each has specific tools, a model (opus/sonnet/haiku), and a focused mission.

**18 agents:**

| Agent | Model | What It Does |
|-------|-------|-------------|
| `architect` | opus | Designs system architecture, evaluates trade-offs |
| `planner` | opus | Creates phased implementation plans |
| `tdd-guide` | sonnet | Enforces test-first development (RED→GREEN→REFACTOR) |
| `python-reviewer` | sonnet | Reviews Python code: PEP 8, types, security, patterns |
| `code-reviewer` | sonnet | Universal code review: security, quality, structure |
| `security-reviewer` | sonnet | OWASP Top 10, secrets, dependency CVEs |
| `database-reviewer` | sonnet | PostgreSQL: queries, indexes, schema, N+1 |
| `refactor-cleaner` | sonnet | Dead code detection and safe removal |
| `doc-updater` | haiku | Documentation generation from code |
| `qa-architect` | opus | Test framework design, OOP architecture |
| `api-test-writer` | sonnet | Generates OOP API tests with httpx + Allure |
| `qa-reviewer` | sonnet | Reviews test code: OOP compliance, Allure, flakiness |
| **`devops-architect`** | opus | **Designs AWS infrastructure, service selection, cost vs reliability** |
| **`infra-reviewer`** | sonnet | **Reviews Terraform / Helm / K8s manifests before merge** |
| **`k8s-debugger`** | sonnet | **Diagnoses pod failures, networking, Istio issues** |
| **`ci-cd-engineer`** | sonnet | **Optimises pipelines, GitOps setup, OIDC migration** |
| **`security-auditor`** | opus | **IAM audit, secret scanning, CVE review, OWASP infra checklist** |
| **`cost-optimizer`** | sonnet | **Rightsizing, Reserved/Spot strategy, Cost Explorer analysis** |
| **`mlops-engineer`** | opus | **ML pipelines, feature stores, model registry, drift detection** |

**Model selection**: opus = deep thinking (slower, smarter), sonnet = fast execution, haiku = lightweight tasks.

---

### Commands — Your Slash-Command Toolkit

**Location**: `commands/` → copy to `.claude/commands/`

Type `/command` in Claude Code to trigger an action. Each command typically invokes an agent.

**28 commands:**

```
Planning & Development:
  /plan            Create implementation plan, WAIT for confirmation
  /tdd             Test-first: write failing test → implement → refactor
  /build-fix       Fix mypy/ruff errors one by one, minimal diffs
  /docs            Look up library docs ("FastAPI middleware")

Quality & Review:
  /code-review     Universal security & quality review
  /python-review   Python-specific: PEP 8, types, security
  /qa-review       Test code: OOP compliance, Allure, flaky patterns
  /verify          Full pipeline: types → lint → tests → security → git

Testing:
  /api-test        Generate OOP API test suite with Allure
  /test-coverage   Find gaps, generate tests to reach 80%+

Maintenance:
  /refactor-clean  Detect and safely remove dead code

DevOps:
  /infra-plan        Design AWS infrastructure — services, cost, trade-offs
  /k8s-debug         Diagnose CrashLoopBackOff / OOMKilled / Pending pods
  /terraform-review  Review Terraform/Helm/K8s for security & best practices
  /cicd-optimize     Optimise CI/CD pipeline speed, security, reliability
  /security-scan     Full infra security audit — IAM, secrets, CVE, OWASP
  /cost-check        AWS cost analysis — waste, rightsizing, savings

MLOps — Pipeline Design:
  /mlops-pipeline    Design end-to-end ML pipeline — DVC, SageMaker, MLflow, Feature Store
  /kubeflow-pipeline Design, review, or debug a Kubeflow Pipeline (KFP v2 SDK)

MLOps — Data & Features:
  /data-validate     Validate data quality — schema, drift gate, anomalies (GE / Pandera)
  /feature-review    Feature Store review — training-serving skew, freshness, schema

MLOps — Experiments & Registry:
  /experiment-review Review ML run — reproducibility, tracking completeness, model card
  /model-registry    Manage model versions — compare, promote stages, aliases, lineage

MLOps — Production:
  /model-health      Production model health — drift, latency, retraining trigger
  /ab-test           Design or analyse A/B / shadow test between model versions
  /retrain           Trigger and monitor model retraining — drift/scheduled/manual
  /model-promote     Safe model promotion — canary deploy, rollback plan, approval gate
```

**Example workflows:**

```bash
# New feature from scratch
/plan → confirm → /tdd → /python-review → /verify → push

# Write tests for existing API
/api-test POST /api/v1/orders → /qa-review → /verify

# Before PR
/verify pre-pr

# Clean up codebase
/refactor-clean → /verify

# Design new AWS infrastructure
/infra-plan → confirm → write Terraform/CDK → /terraform-review → apply

# Debug a failing K8s pod
/k8s-debug → fix → verify

# Monthly infrastructure review
/security-scan → /cost-check → create tickets for findings

# Optimise slow CI/CD pipeline
/cicd-optimize → apply changes → measure improvement
```

---

### Hooks — Automatic Actions

**Location**: `hooks/hooks.json` → merge into `.claude/hooks.json`

Hooks run automatically on events. You don't call them.

| When | What Happens |
|------|-------------|
| After editing `.py` | `ruff --fix` + `black` auto-format |
| After editing `.py` | `mypy` type check (async, non-blocking) |
| After editing `.py` | Warn if `print()` found → suggest `logging` |
| After editing `.tf` | `terraform fmt` auto-format |
| After editing `.tf` | `tflint` Terraform linting (async) |
| After editing `.yaml`/`.yml` | `kubeconform` K8s manifest validation (async) |
| After editing `.tf`/`.yaml` | `checkov` security scan (async) |
| When Claude stops | Scan for `pdb`, `breakpoint()`, `print()` in changed `.py` files |
| When Claude stops | `gitleaks` secret scan on staged files |
| When Claude stops | `terraform fmt` check on changed `.tf` files |

**Like a local CI pipeline running in real-time — for both Python and infrastructure code.**

---

### MCP Configs — External Integrations

**Location**: `mcp-configs/mcp-servers.json`

Give Claude Code access to external tools. Copy needed servers to `~/.claude.json`.

> ⚠️ **Keep under 8–10 active MCPs** — each one consumes context window.

#### General purpose (9 servers)

| Server | Package | Purpose |
|--------|---------|---------|
| `github` | `@modelcontextprotocol/server-github` | PRs, issues, repo operations, CI/CD triggers |
| `memory` | `@modelcontextprotocol/server-memory` | Persistent context across sessions |
| `sequential-thinking` | `@modelcontextprotocol/server-sequential-thinking` | Chain-of-thought for complex trade-offs |
| `context7` | `@upstash/context7-mcp` | Live docs — FastAPI, Django, SQLAlchemy, etc. |
| `clickhouse` | HTTP `mcp.clickhouse.cloud/mcp` | ClickHouse analytics queries |
| `filesystem` | `@modelcontextprotocol/server-filesystem` | Read/write local `.tf`, `.yaml`, project files |
| `insaits` | `pip install insa-its` | AI-to-AI security monitoring |
| `playwright` | `@playwright/mcp` | Browser automation / E2E testing |
| `exa-web-search` | `exa-mcp-server` | Web search, CVE research, runbook lookup |

#### DevOps / MLOps (8 servers)

| Server | Package | Priority | Best Used With | What It Enables |
|--------|---------|----------|----------------|-----------------|
| `aws-docs` | `@awslabs/aws-documentation-mcp-server` | 🔴 HIGH | `/infra-plan` `/terraform-review` `/experiment-review` | Official AWS docs — EKS, SageMaker, IAM, CDK inline |
| `kubernetes` | `mcp-server-kubernetes` | 🔴 HIGH | `/k8s-debug` `/security-scan` | Claude actually runs `kubectl` — not just generates commands |
| `terraform` | `@hashicorp/terraform-mcp-server` | 🔴 HIGH | `/terraform-review` `/infra-plan` | Terraform Registry — module docs, provider versions, resource schemas |
| `prometheus` | `mcp-prometheus` | 🔴 HIGH | `/model-health` `/k8s-debug` | Live PromQL queries — latency, error rates, SLO status |
| `postgres` | `@modelcontextprotocol/server-postgres` | 🟠 MEDIUM | `/experiment-review` `/feature-review` | MLflow backend store, Feast registry, ML metadata SQL |
| `aws-s3` | `aws-s3-mcp-server` | 🟠 MEDIUM | `/model-registry` `/retrain` `/data-validate` | Read model artifacts, DVC cache, Terraform state from S3 |
| `grafana` | `mcp-grafana` | 🟠 MEDIUM | `/model-health` `/k8s-debug` | Query dashboards and alert states in real-time |
| `jira` | `mcp-server-jira` | 🟡 NICE | `/security-scan` `/cost-check` `/model-health` | Auto-create tickets from scan findings |

#### Recommended profiles (activate per task)

```bash
# Python development
github + memory + sequential-thinking + context7 + filesystem

# DevOps / Infrastructure
github + aws-docs + kubernetes + terraform + prometheus + filesystem

# MLOps
github + aws-docs + postgres + aws-s3 + prometheus + grafana

# Security audit (/security-scan)
github + aws-docs + kubernetes + exa-web-search + jira

# Incident response (/k8s-debug + /model-health)
kubernetes + prometheus + grafana + sequential-thinking
```

---

## How Everything Works Together

### Python Workflow
```
You type: /plan Add order processing with Stripe webhooks

    CLAUDE.md loads          → Claude knows all project rules
    planner agent spawns     → Creates phased implementation plan
    You confirm              → Claude starts coding

    fastapi-patterns skill   → Knows how to structure endpoints
    sqlalchemy-patterns      → Knows async DB patterns
    pydantic-patterns        → Knows how to build request/response models

    After each file edit:
      hooks run              → ruff + black format, mypy checks types

You type: /verify pre-pr
    → mypy . → ruff check . → pytest --cov → bandit -r src/
    → Ready for PR: YES
```

### DevOps Workflow
```
You type: /infra-plan New EKS cluster for fraud detection service

    CLAUDE.md loads            → Knows IaC-only rule, no ClickOps
    devops-architect spawns    → Asks: SLO? budget? team size?
    You confirm plan           → Claude writes Terraform

    kubernetes skill           → HPA config, probes, PDB, RBAC
    terraform skill            → Remote state, modules, version pins
    aws skill                  → EKS node groups, IRSA, VPC endpoints
    networking skill           → 3-tier VPC, NAT per AZ, PrivateLink

    After each .tf edit:
      hooks run                → terraform fmt, tflint, checkov

You type: /terraform-review
    → checkov (HIGH findings) → tfsec → gitleaks → tagging audit
    → CRITICAL: RDS publicly_accessible=true → fix before apply

You type: /k8s-debug fraud-detector pod in CrashLoopBackOff
    k8s-debugger spawns        → kubectl logs --previous
    → Root cause: missing SECRET_DB_URL env var
    → Fix: add secretKeyRef to deployment
```

### MLOps Workflow
```
You type: /infra-plan ML training pipeline for fraud model

    mlops-engineer spawns      → Asks: data size? retraining freq? SLO?

    sagemaker skill            → Pipeline SDK v2, Spot instances
    mlops skill                → DVC for data, W&B for experiments
    observability skill        → Evidently drift detection, CloudWatch

    SageMaker pipeline created:
      preprocess (m5.xlarge) → train (ml.p3.2xlarge Spot) → evaluate → register

    After training:
      drift detection runs     → Evidently compares train vs prod dist
      alert fires              → Retraining triggered automatically
```

---

## Installation

### Option 1: Python project

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp hooks/hooks.json your-project/.claude/hooks.json
# Pick your skills
cp -r skills/fastapi-patterns/ your-project/.claude/skills/
cp -r skills/pytest-oop-patterns/ your-project/.claude/skills/
cp -r skills/sqlalchemy-patterns/ your-project/.claude/skills/
```

### Option 2: DevOps / Infrastructure project

```bash
cp CLAUDE.md your-project/
cp .pre-commit-config.yaml .tflint.hcl .checkov.yaml your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp hooks/hooks.json your-project/.claude/hooks.json
# DevOps skills
cp -r skills/kubernetes/ your-project/.claude/skills/
cp -r skills/terraform/ your-project/.claude/skills/
cp -r skills/aws/ your-project/.claude/skills/
cp -r skills/ci-cd/ your-project/.claude/skills/
cp -r skills/observability/ your-project/.claude/skills/
cp -r skills/security-infra/ your-project/.claude/skills/
cp -r skills/networking/ your-project/.claude/skills/
```

### Option 3: MLOps project

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp hooks/hooks.json your-project/.claude/hooks.json
# MLOps skills
cp -r skills/sagemaker/ your-project/.claude/skills/
cp -r skills/mlflow/ your-project/.claude/skills/
cp -r skills/seldon-core/ your-project/.claude/skills/
cp -r skills/mlops/ your-project/.claude/skills/
# Plus DevOps foundation
cp -r skills/kubernetes/ your-project/.claude/skills/
cp -r skills/observability/ your-project/.claude/skills/
```

### Option 4: Copy everything (full stack)

```bash
cp CLAUDE.md your-project/
cp .pre-commit-config.yaml .tflint.hcl .checkov.yaml your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp -r skills/ your-project/.claude/skills/
cp hooks/hooks.json your-project/.claude/hooks.json
```

### Option 5: Symlink (shared across projects)

```bash
ln -s /path/to/claude-code-python/agents/ your-project/.claude/agents
ln -s /path/to/claude-code-python/skills/ your-project/.claude/skills
ln -s /path/to/claude-code-python/commands/ your-project/.claude/commands
```

> **Tool installation**: See `docs/DEVOPS_SETUP.md` for installing terraform, kubectl, helm, checkov, gitleaks, trivy, DVC, MLflow, and all other DevOps/MLOps tools.

---

## Tech Stack Coverage

```
Python 3.9+
├── FastAPI          (async web framework)
├── Django + DRF     (full-stack web framework)
├── SQLAlchemy 2.0   (async ORM + Alembic migrations)
├── Pydantic v2      (data validation & settings)
├── Celery           (distributed task queues)
├── pytest           (testing framework)
├── httpx / aiohttp  (HTTP clients)
├── PostgreSQL       (primary database)
├── ClickHouse       (analytics database)
├── Redis            (caching, queues, locks)
├── Docker           (containerization)
├── Allure           (test reporting)
└── Tools: black, ruff, mypy, bandit, isort, faker

DevOps
├── Terraform        (IaC — modules, remote state, drift detection)
├── AWS CDK          (IaC — Python/TypeScript, L2/L3 constructs)
├── Kubernetes       (probes, HPA/VPA, RBAC, NetworkPolicy, Helm)
├── AWS              (EKS, RDS, S3, IAM, WAF, CloudWatch, Cost Explorer)
├── GitHub Actions   (CI/CD with OIDC, caching, SBOM)
├── ArgoCD           (GitOps — auto-sync, self-heal, rollback)
├── Istio            (service mesh — mTLS, traffic management)
├── Prometheus       (metrics, alerting rules, SLO/SLI)
├── Grafana          (dashboards as code)
├── OpenTelemetry    (distributed tracing)
├── cert-manager     (TLS with Let's Encrypt)
└── Security: checkov, tfsec, tflint, gitleaks, trivy, OPA Gatekeeper

MLOps
├── SageMaker        (training pipelines, Spot, Model Registry)
├── MLflow           (experiment tracking, model registry, serving)
├── Weights & Biases (experiment tracking, sweeps)
├── DVC              (data & pipeline versioning)
├── Feast            (open-source feature store)
├── AWS Feature Store (managed feature store)
├── Seldon Core      (K8s-native model serving, canary)
└── Evidently AI     (model drift detection)
```

---

## Quick Reference

```
CLAUDE.md           = Project brain — always loaded (Python + DevOps + MLOps rules)
.claude/rules/      = Auto-activate by file type:
                        *.py          → coding-style, testing, security
                        *.tf, *.yaml  → infra (no secrets, IaC only, rollback)
.claude/skills/     = Deep knowledge — Claude pulls when topic matches:
                        Python        → fastapi, django, sqlalchemy, celery...
                        DevOps        → kubernetes, terraform, aws, ci-cd...
                        MLOps         → sagemaker, mlflow, mlops, seldon-core...
.claude/agents/     = Specialists — invoked by commands:
                        Python        → python-reviewer, architect, tdd-guide...
                        DevOps        → devops-architect, k8s-debugger, infra-reviewer...
                        MLOps         → mlops-engineer, cost-optimizer...
.claude/commands/   = Slash commands:
                        Python        → /plan /tdd /verify /api-test /python-review
                        DevOps        → /infra-plan /k8s-debug /terraform-review /security-scan
                        MLOps         → /mlops-pipeline /experiment-review /model-health
                                        /feature-review /model-promote
                        Cost          → /cicd-optimize /cost-check
.claude/hooks.json  = Auto-actions on save:
                        *.py          → ruff, black, mypy
                        *.tf          → terraform fmt, tflint, checkov
                        *.yaml        → kubeconform, checkov
                        on stop       → gitleaks secret scan
mcp-configs/        = External: GitHub, Context7 docs, ClickHouse, Playwright
docs/DEVOPS_SETUP.md = Tool installation guide (terraform, kubectl, helm, DVC, MLflow...)
```

---
---

<a id="russian"></a>

# Python + DevOps + MLOps Stack for Claude Code — Русская версия

## Что это?

**Готовый набор инструментов**, который превращает Claude Code в senior Python/DevOps/MLOps-инженера. Когда вы подключаете этот набор к проекту, Claude Code мгновенно знает как:

- Писать код по PEP 8 с type hints и правильной обработкой ошибок
- Проектировать архитектуры на FastAPI/Django/SQLAlchemy
- Генерировать ООП-тесты с pytest, httpx и Allure-отчётами
- Ревьюить код на уязвимости (OWASP Top 10)
- Работать с PostgreSQL, Redis, ClickHouse, Celery, Docker
- **Проектировать AWS инфраструктуру** с CDK (Python/TypeScript) и Terraform
- **Управлять Kubernetes** — Helm, HPA, RBAC, NetworkPolicy, Istio
- **Строить ML пайплайны** — SageMaker, MLflow, DVC, Feature Store
- **Оптимизировать CI/CD** — GitHub Actions, ArgoCD GitOps, OIDC
- **Проводить security-аудиты** — checkov, tfsec, gitleaks, OPA Gatekeeper

**Без этого набора** Claude Code — умный генералист. **С ним** — senior инженер, который следует стандартам вашей команды.

---

## Как работает конфигурация Claude Code

Когда вы запускаете Claude Code в директории проекта, он читает специальные файлы из папки `.claude/`:

```
your-project/
├── CLAUDE.md              # Главный файл инструкций — "мозг"
├── .pre-commit-config.yaml  # gitleaks + terraform + ruff + kubeconform
├── .tflint.hcl            # Конфиг линтинга Terraform
├── .checkov.yaml          # Конфиг security-скана IaC
├── .claude/
│   ├── rules/
│   │   ├── coding-style.md    # Python — активируется на *.py
│   │   ├── testing.md         # Тесты — активируется на test_*.py
│   │   └── infra.md           # DevOps — активируется на *.tf, *.yaml, Dockerfile
│   ├── skills/
│   │   ├── fastapi-patterns/  # Глубокие знания Python/фреймворков
│   │   ├── kubernetes/        # Глубокие знания K8s
│   │   ├── terraform/         # Глубокие знания IaC
│   │   └── mlops/             # Глубокие знания ML пайплайнов
│   ├── agents/
│   │   ├── python-reviewer.md    # Ревью Python кода
│   │   ├── devops-architect.md   # Проектирование AWS инфраструктуры
│   │   ├── k8s-debugger.md       # Дебаг Kubernetes
│   │   └── mlops-engineer.md     # ML пайплайны
│   ├── commands/
│   │   ├── /plan /tdd /verify    # Python воркфлоу
│   │   ├── /infra-plan           # Проектирование AWS инфраструктуры
│   │   ├── /k8s-debug            # Дебаг Kubernetes
│   │   └── /security-scan        # Security-аудит
│   └── hooks.json         # Авто-действия: ruff, terraform fmt, checkov, gitleaks
└── src/  (или infra/, k8s/, ml/)
```

Этот тулкит предоставляет все эти компоненты — настроенные для **Python, DevOps и MLOps**.

---

## Компоненты

### CLAUDE.md — Мозг

**Самый важный файл.** Лежит в корне проекта. Claude читает его в начале каждого разговора и следует как закону.

Наш CLAUDE.md содержит:
- Конвенции Python (PEP 8, type hints, black/ruff/mypy)
- Строгие правила ООП-тестов (классы, сервисные слои, Allure — обязательно)
- Правила по фреймворкам (FastAPI, Django, SQLAlchemy, Celery)
- Правила по БД (PostgreSQL, индексы, предотвращение N+1)
- Правила безопасности (никакого eval, никаких хардкодов секретов)
- **Правила DevOps** (IaC only, идемпотентность, стратегия отката, observability)
- **Правила MLOps** (воспроизводимость, Feature Store, human gate для прода)
- Все доступные команды и агенты

**Представьте**: это как документ для онбординга нового разработчика. Всё, что он должен всегда соблюдать.

---

### Rules — Правила (Всегда активны)

**Расположение**: `rules/` → копировать в `.claude/rules/`

Markdown-файлы, которые **автоматически активируются** по паттерну файлов. Редактируете `.py` файл — включаются Python-правила.

```yaml
---
paths:
  - "**/*.py"
---
# Python Coding Style
- Следовать PEP 8
- Type annotations на всех публичных функциях
```

| Файл | Что контролирует |
|------|-----------------|
| `coding-style.md` | PEP 8, type hints, иммутабельность, black/ruff/isort |
| `patterns.md` | Protocol, dataclasses, контекстные менеджеры |
| `security.md` | Секреты через env vars, bandit-сканирование |
| `testing.md` | pytest, покрытие 80%+, маркеры тестов |
| `hooks.md` | Автоформат после редактирования, mypy, предупреждения о print() |
| `infra.md` | **Без хардкодов секретов, IaC only, идемпотентность, observability, rollback** |

**Разница с CLAUDE.md**: CLAUDE.md = всегда включён для всего. Rules = включаются условно по типу файла. `infra.md` активируется для `.tf`, `.yaml`, `.yml`, `Dockerfile*`, `.hcl` и `.github/workflows/`.

---

### Skills — Глубокие знания по запросу

**Расположение**: `skills/` → копировать в `.claude/skills/`

**Главное отличие от правил**: скиллы — это большие справочные документы (сотни строк) с паттернами, примерами кода, конфигурациями, антипаттернами. Claude подтягивает их когда тема совпадает.

Вы не вызываете скиллы вручную. Claude сам решает когда они нужны:

```
# Python сценарий
Вы: "Создай FastAPI эндпоинт для регистрации пользователя"
Claude подтягивает: fastapi-patterns → pydantic-patterns → sqlalchemy-patterns

# DevOps сценарий
Вы: "Создай EKS кластер с RDS и автоскейлингом"
Claude подтягивает: kubernetes → terraform → aws → networking

# MLOps сценарий
Вы: "Настрой ML пайплайн обучения с детекцией дрифта"
Claude подтягивает: mlops → sagemaker → mlflow → observability
```

**31 скиллов:**

| Категория | Скиллы |
|-----------|--------|
| Core Python | `python-patterns`, `python-testing`, `pydantic-patterns`, `async-http-patterns` |
| Фреймворки | `fastapi-patterns`, `django-patterns`, `django-security`, `django-tdd`, `django-verification`, `celery-patterns` |
| Данные и инфра | `sqlalchemy-patterns`, `postgres-patterns`, `clickhouse-io`, `redis-patterns`, `database-migrations`, `deployment-patterns`, `docker-patterns` |
| QA автоматизация | `pytest-oop-patterns`, `allure-reporting`, `api-testing-patterns` |
| **DevOps** | `kubernetes`, `terraform`, `aws`, `ci-cd`, `observability`, `security-infra`, `networking` |
| **MLOps** | `sagemaker`, `seldon-core`, `mlflow`, `mlops` |

**Пример — что внутри `fastapi-patterns`:**
- Структура проекта (app factory, routers, services, repos)
- Pydantic Settings для конфигурации
- Паттерны Dependency Injection (`Depends()`)
- Схемы Request/Response с валидацией
- Обработка ошибок (кастомные исключения)
- Middleware (логирование, CORS)
- Background tasks, WebSockets
- JWT аутентификация
- Тестирование через httpx AsyncClient

---

### Agents — Специализированные суб-агенты

**Расположение**: `agents/` → копировать в `.claude/agents/`

Агенты — это **специализированные персоны**, которых Claude может вызвать. У каждого свои инструменты, своя модель и своя миссия.

**18 агентов:**

| Агент | Модель | Что делает |
|-------|--------|-----------|
| `architect` | opus | Проектирует архитектуру, оценивает trade-offs |
| `planner` | opus | Создаёт пошаговые планы реализации |
| `tdd-guide` | sonnet | Контролирует TDD: RED → GREEN → REFACTOR |
| `python-reviewer` | sonnet | Ревью Python: PEP 8, типы, безопасность |
| `code-reviewer` | sonnet | Универсальный ревью: безопасность, качество |
| `security-reviewer` | sonnet | OWASP Top 10, секреты, CVE в зависимостях |
| `database-reviewer` | sonnet | PostgreSQL: запросы, индексы, схема, N+1 |
| `refactor-cleaner` | sonnet | Поиск и безопасное удаление мёртвого кода |
| `doc-updater` | haiku | Генерация документации из кода |
| `qa-architect` | opus | Проектирование тест-фреймворка, ООП архитектура |
| `api-test-writer` | sonnet | Генерация ООП API-тестов с httpx + Allure |
| `qa-reviewer` | sonnet | Ревью тест-кода: ООП, Allure, flaky-паттерны |
| **`devops-architect`** | opus | **Проектирует AWS инфраструктуру, выбор сервисов, cost vs reliability** |
| **`infra-reviewer`** | sonnet | **Ревью Terraform / Helm / K8s манифестов** |
| **`k8s-debugger`** | sonnet | **Диагностика pod failures, сетевых проблем, Istio** |
| **`ci-cd-engineer`** | sonnet | **Оптимизация пайплайнов, GitOps, OIDC миграция** |
| **`security-auditor`** | opus | **IAM аудит, поиск секретов, CVE, OWASP инфра** |
| **`cost-optimizer`** | sonnet | **Rightsizing, Spot/Reserved стратегия, Cost Explorer** |
| **`mlops-engineer`** | opus | **ML пайплайны, feature stores, model registry, drift detection** |

**Выбор модели**: opus = глубокое мышление (медленнее, умнее), sonnet = быстрое выполнение, haiku = лёгкие задачи.

---

### Commands — Слэш-команды

**Расположение**: `commands/` → копировать в `.claude/commands/`

Набираете `/команда` в Claude Code — запускается действие. Каждая команда обычно вызывает агента.

**28 команд:**

```
Планирование и разработка:
  /plan            Создать план реализации, ЖДАТЬ подтверждения
  /tdd             Тест-первый подход: RED → GREEN → REFACTOR
  /build-fix       Починить ошибки mypy/ruff по одной
  /docs            Посмотреть актуальную документацию библиотеки

Качество и ревью:
  /code-review     Универсальный ревью безопасности и качества
  /python-review   Python-специфичный: PEP 8, типы, безопасность
  /qa-review       Тест-код: ООП, Allure, flaky-паттерны
  /verify          Полный пайплайн: типы → линт → тесты → безопасность

Тестирование:
  /api-test        Сгенерировать ООП тест-сьюит с Allure
  /test-coverage   Найти пробелы, дописать тесты до 80%+

Обслуживание:
  /refactor-clean  Найти и безопасно удалить мёртвый код

DevOps:
  /infra-plan        Проектирование AWS инфраструктуры — сервисы, стоимость, trade-offs
  /k8s-debug         Диагностика CrashLoopBackOff / OOMKilled / Pending подов
  /terraform-review  Ревью Terraform/Helm/K8s на безопасность и best practices
  /cicd-optimize     Оптимизация CI/CD пайплайна — скорость, безопасность, надёжность
  /security-scan     Полный security-аудит инфраструктуры — IAM, секреты, CVE, OWASP
  /cost-check        Анализ затрат AWS — waste, rightsizing, экономия

MLOps — Дизайн пайплайнов:
  /mlops-pipeline    Проектирование ML пайплайна — DVC, SageMaker, MLflow, Feature Store
  /kubeflow-pipeline Дизайн, ревью или дебаг Kubeflow Pipeline (KFP v2 SDK)

MLOps — Данные и фичи:
  /data-validate     Валидация данных — схема, drift gate, аномалии (GE / Pandera)
  /feature-review    Ревью Feature Store — training-serving skew, свежесть, схема

MLOps — Эксперименты и реестр:
  /experiment-review Ревью ML эксперимента — воспроизводимость, артефакты, model card
  /model-registry    Управление версиями — сравнение, алиасы, аудит lineage

MLOps — Продакшн:
  /model-health      Здоровье модели в проде — дрифт, задержки, триггер переобучения
  /ab-test           A/B тест или shadow mode между версиями модели
  /retrain           Переобучение — drift/scheduled/manual с gates и мониторингом
  /model-promote     Безопасный деплой в прод — canary, план отката, approval gate
```

**Примеры рабочих процессов:**

```bash
# Новая фича с нуля
/plan → подтверждаем → /tdd → /python-review → /verify → push

# Написать тесты для существующего API
/api-test POST /api/v1/orders → /qa-review → /verify

# Перед PR
/verify pre-pr

# Навести порядок в коде
/refactor-clean → /verify

# Проектирование новой AWS инфраструктуры
/infra-plan → подтверждаем → пишем Terraform/CDK → /terraform-review → apply

# Дебаг падающего K8s пода
/k8s-debug → исправляем → /verify

# Ежемесячный обзор инфраструктуры
/security-scan → /cost-check → создаём тикеты по находкам

# Оптимизация медленного CI/CD пайплайна
/cicd-optimize → применяем изменения → измеряем улучшение
```

---

### Hooks — Автоматические действия

**Расположение**: `hooks/hooks.json` → добавить в `.claude/hooks.json`

Хуки срабатывают автоматически на события. Вы их не вызываете.

| Когда | Что происходит |
|-------|---------------|
| После редактирования `.py` | `ruff --fix` + `black` автоформат |
| После редактирования `.py` | `mypy` проверка типов (в фоне) |
| После редактирования `.py` | Предупреждение если найден `print()` → предложит `logging` |
| После редактирования `.tf` | `terraform fmt` автоформат |
| После редактирования `.tf` | `tflint` линтинг Terraform (в фоне) |
| После редактирования `.yaml`/`.yml` | `kubeconform` валидация K8s манифестов (в фоне) |
| После редактирования `.tf`/`.yaml` | `checkov` security-скан (в фоне) |
| Когда Claude останавливается | Поиск `pdb`, `breakpoint()`, `print()` в изменённых `.py` файлах |
| Когда Claude останавливается | `gitleaks` скан секретов в staged файлах |
| Когда Claude останавливается | `terraform fmt` проверка на изменённых `.tf` файлах |

**Как локальный CI-пайплайн, работающий в реальном времени — и для Python, и для инфраструктурного кода.**

---

### MCP Configs — Внешние интеграции

**Расположение**: `mcp-configs/mcp-servers.json`

Дают Claude Code доступ к внешним инструментам. Скопируйте нужные серверы в `~/.claude.json`.

> ⚠️ **Держите не больше 8–10 активных MCP одновременно** — каждый потребляет context window.

#### Общего назначения (9 серверов)

| Сервер | Пакет | Для чего |
|--------|-------|----------|
| `github` | `@modelcontextprotocol/server-github` | PR, issues, операции с репозиториями |
| `memory` | `@modelcontextprotocol/server-memory` | Persistent контекст между сессиями |
| `sequential-thinking` | `@modelcontextprotocol/server-sequential-thinking` | Chain-of-thought для сложных trade-off анализов |
| `context7` | `@upstash/context7-mcp` | Актуальные доки — FastAPI, Django, SQLAlchemy и др. |
| `clickhouse` | HTTP `mcp.clickhouse.cloud/mcp` | Аналитические запросы ClickHouse |
| `filesystem` | `@modelcontextprotocol/server-filesystem` | Чтение/запись `.tf`, `.yaml`, файлов проекта |
| `insaits` | `pip install insa-its` | AI-to-AI мониторинг безопасности |
| `playwright` | `@playwright/mcp` | Автоматизация браузера / E2E тесты |
| `exa-web-search` | `exa-mcp-server` | Веб-поиск, исследование CVE, поиск runbooks |

#### DevOps / MLOps (8 серверов)

| Сервер | Пакет | Приоритет | Лучшие команды | Что даёт |
|--------|-------|-----------|----------------|---------|
| `aws-docs` | `@awslabs/aws-documentation-mcp-server` | 🔴 HIGH | `/infra-plan` `/terraform-review` | Официальные AWS доки — EKS, SageMaker, IAM, CDK прямо в контексте |
| `kubernetes` | `mcp-server-kubernetes` | 🔴 HIGH | `/k8s-debug` `/security-scan` | Claude реально запускает `kubectl` — не просто генерирует команды |
| `terraform` | `@hashicorp/terraform-mcp-server` | 🔴 HIGH | `/terraform-review` `/infra-plan` | Terraform Registry — документация ресурсов, версии провайдеров |
| `prometheus` | `mcp-prometheus` | 🔴 HIGH | `/model-health` `/k8s-debug` | Live PromQL запросы — latency, error rates, SLO прямо сейчас |
| `postgres` | `@modelcontextprotocol/server-postgres` | 🟠 MEDIUM | `/experiment-review` `/feature-review` | MLflow backend store, Feast registry, ML metadata SQL |
| `aws-s3` | `aws-s3-mcp-server` | 🟠 MEDIUM | `/model-registry` `/retrain` | Чтение артефактов моделей, DVC cache, Terraform state |
| `grafana` | `mcp-grafana` | 🟠 MEDIUM | `/model-health` `/k8s-debug` | Реальные панели и алерты Grafana в контексте |
| `jira` | `mcp-server-jira` | 🟡 NICE | `/security-scan` `/cost-check` | Автоматическое создание тикетов по findings |

#### Рекомендованные профили

```bash
# Python разработка
github + memory + sequential-thinking + context7 + filesystem

# DevOps / Инфраструктура
github + aws-docs + kubernetes + terraform + prometheus + filesystem

# MLOps
github + aws-docs + postgres + aws-s3 + prometheus + grafana

# Security аудит (/security-scan)
github + aws-docs + kubernetes + exa-web-search + jira

# Инцидент (/k8s-debug + /model-health)
kubernetes + prometheus + grafana + sequential-thinking
```

---

## Как всё работает вместе

### Python воркфлоу
```
Вы: /plan Добавить обработку заказов с Stripe webhooks

    CLAUDE.md загружается      → Claude знает все правила проекта
    planner агент запускается  → Создаёт план по фазам
    Вы подтверждаете           → Claude начинает кодить

    fastapi-patterns скилл     → Структура эндпоинтов
    sqlalchemy-patterns        → Async паттерны БД
    После каждого сохранения:
      хуки                     → ruff + black, mypy проверяет типы

Вы: /verify pre-pr
    → mypy . → ruff check . → pytest --cov → bandit
    → Готов к PR: ДА
```

### DevOps воркфлоу
```
Вы: /infra-plan Новый EKS кластер для сервиса детекции фрода

    CLAUDE.md загружается      → Знает: IaC only, без ClickOps
    devops-architect запускается → Спрашивает: SLO? бюджет? размер команды?
    Вы подтверждаете план      → Claude пишет Terraform

    kubernetes скилл           → HPA, probes, PDB, RBAC
    terraform скилл            → Remote state, модули, version pins
    aws скилл                  → EKS node groups, IRSA, VPC endpoints
    После каждого .tf файла:
      хуки                     → terraform fmt, tflint, checkov

Вы: /terraform-review
    → checkov HIGH → tfsec → gitleaks → аудит тегов
    → CRITICAL: RDS publicly_accessible=true → исправить до apply
```

### MLOps воркфлоу
```
Вы: /infra-plan ML пайплайн обучения модели

    mlops-engineer запускается  → Спрашивает: объём данных? частота переобучения?

    sagemaker скилл            → Pipeline SDK v2, Spot инстансы
    mlops скилл                → DVC для данных, W&B для экспериментов
    observability скилл        → Evidently детекция дрифта, CloudWatch

    После обучения:
      drift detection          → Evidently сравнивает train vs prod
      алерт срабатывает        → Автоматически запускает переобучение
```

---

## Установка

### Вариант 1: Python проект

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp hooks/hooks.json your-project/.claude/hooks.json
# Выберите нужные скиллы
cp -r skills/fastapi-patterns/ your-project/.claude/skills/
cp -r skills/pytest-oop-patterns/ your-project/.claude/skills/
cp -r skills/sqlalchemy-patterns/ your-project/.claude/skills/
```

### Вариант 2: DevOps / Infrastructure проект

```bash
cp CLAUDE.md your-project/
cp .pre-commit-config.yaml .tflint.hcl .checkov.yaml your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp hooks/hooks.json your-project/.claude/hooks.json
# DevOps скиллы
cp -r skills/kubernetes/ your-project/.claude/skills/
cp -r skills/terraform/ your-project/.claude/skills/
cp -r skills/aws/ your-project/.claude/skills/
cp -r skills/ci-cd/ your-project/.claude/skills/
cp -r skills/observability/ your-project/.claude/skills/
cp -r skills/security-infra/ your-project/.claude/skills/
cp -r skills/networking/ your-project/.claude/skills/
```

### Вариант 3: MLOps проект

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp hooks/hooks.json your-project/.claude/hooks.json
# MLOps скиллы
cp -r skills/sagemaker/ your-project/.claude/skills/
cp -r skills/mlflow/ your-project/.claude/skills/
cp -r skills/seldon-core/ your-project/.claude/skills/
cp -r skills/mlops/ your-project/.claude/skills/
# DevOps основа
cp -r skills/kubernetes/ your-project/.claude/skills/
cp -r skills/observability/ your-project/.claude/skills/
```

### Вариант 4: Скопировать всё

```bash
cp CLAUDE.md your-project/
cp .pre-commit-config.yaml .tflint.hcl .checkov.yaml your-project/
cp -r rules/ your-project/.claude/rules/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp -r skills/ your-project/.claude/skills/
cp hooks/hooks.json your-project/.claude/hooks.json
```

### Вариант 5: Симлинки (общие для нескольких проектов)

```bash
ln -s /path/to/claude-code-python/agents/ your-project/.claude/agents
ln -s /path/to/claude-code-python/skills/ your-project/.claude/skills
ln -s /path/to/claude-code-python/commands/ your-project/.claude/commands
```

> **Установка инструментов**: см. `docs/DEVOPS_SETUP.md` — инструкции по установке terraform, kubectl, helm, checkov, gitleaks, trivy, DVC, MLflow и всех остальных DevOps/MLOps инструментов.

---

## Покрытие технологий

```
Python 3.9+
├── FastAPI          (async веб-фреймворк)
├── Django + DRF     (full-stack веб-фреймворк)
├── SQLAlchemy 2.0   (async ORM + Alembic миграции)
├── Pydantic v2      (валидация данных и настройки)
├── Celery           (распределённые очереди задач)
├── pytest           (тестирование)
├── httpx / aiohttp  (HTTP-клиенты)
├── PostgreSQL       (основная БД)
├── ClickHouse       (аналитическая БД)
├── Redis            (кэш, очереди, блокировки)
├── Docker           (контейнеризация)
├── Allure           (отчёты по тестам)
└── Инструменты: black, ruff, mypy, bandit, isort, faker

DevOps
├── Terraform        (IaC — модули, удалённое состояние, детекция дрифта)
├── AWS CDK          (IaC — Python/TypeScript, L2/L3 конструкты)
├── Kubernetes       (probes, HPA/VPA, RBAC, NetworkPolicy, Helm)
├── AWS              (EKS, RDS, S3, IAM, WAF, CloudWatch, Cost Explorer)
├── GitHub Actions   (CI/CD с OIDC, кэширование, SBOM)
├── ArgoCD           (GitOps — авто-синк, self-heal, откат)
├── Istio            (service mesh — mTLS, управление трафиком)
├── Prometheus       (метрики, правила алертинга, SLO/SLI)
├── Grafana          (дашборды как код)
├── OpenTelemetry    (распределённая трассировка)
├── cert-manager     (TLS с Let's Encrypt)
└── Безопасность: checkov, tfsec, tflint, gitleaks, trivy, OPA Gatekeeper

MLOps
├── SageMaker        (обучающие пайплайны, Spot, Model Registry)
├── MLflow           (трекинг экспериментов, реестр моделей, сервинг)
├── Weights & Biases (трекинг экспериментов, sweeps)
├── DVC              (версионирование данных и пайплайнов)
├── Feast            (open-source feature store)
├── AWS Feature Store (управляемый feature store)
├── Seldon Core      (K8s-нативный сервинг моделей, canary)
└── Evidently AI     (детекция дрифта моделей)
```

---

## Шпаргалка

```
CLAUDE.md           = Мозг проекта — всегда загружен (Python + DevOps + MLOps правила)
.claude/rules/      = Авто-активация по типу файла:
                        *.py          → coding-style, testing, security
                        *.tf, *.yaml  → infra (no secrets, IaC only, rollback)
.claude/skills/     = Глубокие знания — Claude подтягивает по теме:
                        Python        → fastapi, django, sqlalchemy, celery...
                        DevOps        → kubernetes, terraform, aws, ci-cd...
                        MLOps         → sagemaker, mlflow, mlops, seldon-core...
.claude/agents/     = Специалисты — вызываются командами:
                        Python        → python-reviewer, architect, tdd-guide...
                        DevOps        → devops-architect, k8s-debugger, infra-reviewer...
                        MLOps         → mlops-engineer, cost-optimizer...
.claude/commands/   = Слэш-команды:
                        Python        → /plan /tdd /verify /api-test /python-review
                        DevOps        → /infra-plan /k8s-debug /terraform-review /security-scan
                        MLOps дизайн  → /mlops-pipeline /kubeflow-pipeline
                        MLOps данные  → /data-validate /feature-review
                        MLOps реестр  → /experiment-review /model-registry
                        MLOps прод    → /model-health /ab-test /retrain /model-promote
                        Стоимость     → /cicd-optimize /cost-check
.claude/hooks.json  = Авто-действия при сохранении:
                        *.py          → ruff, black, mypy
                        *.tf          → terraform fmt, tflint, checkov
                        *.yaml        → kubeconform, checkov
                        при остановке → gitleaks сканирование секретов
mcp-configs/        = Внешние: GitHub, Context7 доки, ClickHouse, Playwright
docs/DEVOPS_SETUP.md = Установка инструментов (terraform, kubectl, helm, DVC, MLflow...)
```

---

## Directory Structure

```
python-stack/
├── README.md                           # This file (EN + RU tutorial)
├── CLAUDE.md                           # Main brain — copy to project root
├── .pre-commit-config.yaml             # Pre-commit hooks (gitleaks, terraform, ruff, kubeconform)
├── .tflint.hcl                         # tflint config for Terraform linting
├── .checkov.yaml                       # checkov config for IaC security scanning
├── .gitignore                          # Python + macOS + DevOps + MLOps + secrets
│
├── rules/                              # 6 auto-activate rules
│   ├── coding-style.md                 #   PEP 8, type hints, formatting
│   ├── hooks.md                        #   Auto-format, mypy, print warnings
│   ├── patterns.md                     #   Protocol, dataclasses, context managers
│   ├── security.md                     #   Secrets, bandit, OWASP
│   ├── testing.md                      #   pytest, coverage, markers
│   └── infra.md                        #   IaC rules: no secrets, idempotency, rollback
│
├── skills/                             # 31 deep knowledge skills
│   │
│   │  # ─── Core Python ─────────────────────
│   ├── python-patterns/SKILL.md
│   ├── python-testing/SKILL.md
│   ├── pydantic-patterns/SKILL.md
│   ├── async-http-patterns/SKILL.md
│   │
│   │  # ─── Frameworks ──────────────────────
│   ├── fastapi-patterns/SKILL.md
│   ├── django-patterns/SKILL.md
│   ├── django-security/SKILL.md
│   ├── django-tdd/SKILL.md
│   ├── django-verification/SKILL.md
│   ├── celery-patterns/SKILL.md
│   │
│   │  # ─── Data & Infrastructure ───────────
│   ├── sqlalchemy-patterns/SKILL.md
│   ├── postgres-patterns/SKILL.md
│   ├── clickhouse-io/SKILL.md
│   ├── redis-patterns/SKILL.md
│   ├── database-migrations/SKILL.md
│   ├── deployment-patterns/SKILL.md
│   ├── docker-patterns/SKILL.md
│   │
│   │  # ─── QA Automation ───────────────────
│   ├── pytest-oop-patterns/SKILL.md
│   ├── allure-reporting/SKILL.md
│   ├── api-testing-patterns/SKILL.md
│   │
│   │  # ─── DevOps ──────────────────────────
│   ├── kubernetes/SKILL.md             #   Probes, HPA/VPA, RBAC, Helm, NetworkPolicy
│   ├── terraform/SKILL.md              #   Modules, remote state, drift detection
│   ├── aws/SKILL.md                    #   CDK vs Terraform, EKS/RDS/S3/IAM, Cost Explorer, WAF
│   ├── ci-cd/SKILL.md                  #   GitHub Actions, ArgoCD, DORA metrics
│   ├── observability/SKILL.md          #   Prometheus, OpenTelemetry, SLO/SLI, alerting
│   ├── security-infra/SKILL.md         #   Checkov, tfsec, OPA Gatekeeper, Zero Trust
│   ├── networking/SKILL.md             #   VPC, Ingress, Istio, cert-manager, Route53
│   │
│   │  # ─── MLOps ───────────────────────────
│   ├── sagemaker/SKILL.md              #   Pipelines, Spot, Model Registry, IAM
│   ├── seldon-core/SKILL.md            #   SeldonDeployment CRD, canary, gRPC
│   ├── mlflow/SKILL.md                 #   Tracking, Registry, Projects, Serving
│   └── mlops/SKILL.md                  #   DVC, W&B, Feast vs AWS FS, Evidently
│
├── agents/                             # 18 specialized agents
│   │
│   │  # ─── Python & Architecture ───────────
│   ├── architect.md
│   ├── planner.md
│   ├── tdd-guide.md
│   ├── python-reviewer.md
│   ├── code-reviewer.md
│   ├── security-reviewer.md
│   ├── database-reviewer.md
│   ├── refactor-cleaner.md
│   ├── doc-updater.md
│   │
│   │  # ─── QA Automation ───────────────────
│   ├── qa-architect.md
│   ├── api-test-writer.md
│   ├── qa-reviewer.md
│   │
│   │  # ─── DevOps & MLOps ──────────────────
│   ├── devops-architect.md             #   AWS infra design, cost vs reliability
│   ├── infra-reviewer.md               #   Terraform / Helm / K8s review
│   ├── k8s-debugger.md                 #   Pod failures, networking, Istio
│   ├── ci-cd-engineer.md               #   Pipeline optimisation, GitOps, OIDC
│   ├── security-auditor.md             #   IAM audit, secrets, CVE, OWASP infra
│   ├── cost-optimizer.md               #   Rightsizing, Spot/Reserved, Cost Explorer
│   └── mlops-engineer.md               #   ML pipelines, feature stores, drift
│
├── commands/                           # 22 slash commands
│   │
│   │  # ─── Planning & Development ──────────
│   ├── plan.md
│   ├── tdd.md
│   ├── build-fix.md
│   ├── docs.md
│   │
│   │  # ─── Quality & Review ────────────────
│   ├── code-review.md
│   ├── python-review.md
│   ├── qa-review.md
│   ├── verify.md
│   │
│   │  # ─── Testing ─────────────────────────
│   ├── api-test.md
│   ├── test-coverage.md
│   │
│   │  # ─── Maintenance ─────────────────────
│   ├── refactor-clean.md
│   │
│   │  # ─── DevOps ──────────────────────────
│   ├── infra-plan.md                   #   Design AWS infrastructure
│   ├── k8s-debug.md                    #   Diagnose K8s pod failures
│   ├── terraform-review.md             #   Review IaC before merge
│   ├── cicd-optimize.md                #   Optimise CI/CD pipelines
│   ├── security-scan.md                #   Full infra security audit
│   ├── cost-check.md                   #   AWS cost analysis & savings
│   │
│   │  # ─── MLOps ───────────────────────────
│   ├── mlops-pipeline.md               #   Design end-to-end ML pipeline
│   ├── experiment-review.md            #   Review ML run reproducibility & model card
│   ├── model-health.md                 #   Production drift, latency, retraining trigger
│   ├── feature-review.md               #   Feature Store skew, freshness, schema
│   └── model-promote.md                #   Canary deploy, rollback plan, approval gate
│
├── hooks/
│   └── hooks.json                      # Auto-format, mypy, terraform fmt, checkov, gitleaks
│
├── mcp-configs/
│   └── mcp-servers.json                # GitHub, Context7, ClickHouse, Playwright
│
└── docs/
    ├── TUTORIAL_EN.md                  # Extended English tutorial
    └── DEVOPS_SETUP.md                 # DevOps/MLOps tools installation guide
```

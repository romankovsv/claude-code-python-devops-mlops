# CLAUDE.md

This file provides guidance to Claude Code when working in Python, DevOps, and MLOps projects that use this stack.

## Project Type

This is a **Python + DevOps + MLOps toolkit** — a collection of agents, skills, commands, hooks, and rules for building, testing, and operating production-grade applications and ML systems on AWS and Kubernetes.

## Mandatory Conventions

### Python Code

- Follow **PEP 8** strictly
- **Type annotations** on all public functions — no exceptions
- **Black** (88 chars) + **isort** + **ruff** for formatting/linting
- **mypy** in strict mode for type checking
- Use `logging` module, NEVER `print()` in production code
- Prefer `pathlib.Path` over `os.path`
- Use `f-strings` for string formatting
- Immutable by default: `@dataclass(frozen=True)`, `NamedTuple`, `tuple`
- Explicit `None` checks: `if value is None`, never `if value == None`
- Specific exceptions: never bare `except:`, always `except SpecificError`

### Test Code (STRICT OOP — Non-Negotiable)

- **ALL tests MUST be in classes** — no bare `def test_*()` functions, ever
- **ALL test classes MUST inherit from `BaseTest`** or a domain-specific base class
- **ALL API calls MUST go through service classes** (`UsersAPI`, `AuthAPI`, etc.) — never raw `httpx.get()` or `requests.post()` in tests
- **ALL API service methods MUST have `@allure.step`** decorators
- **ALL request/response bodies MUST use Pydantic models** — no raw dicts
- **ALL test data MUST come from `DataGenerator`/factories** — never hardcode emails, passwords, IDs
- **EVERY test follows Arrange-Act-Assert** pattern
- **Allure metadata is required on every test**:
  - `@allure.epic("...")` on class
  - `@allure.feature("...")` on class
  - `@allure.story("...")` on method
  - `@allure.severity(...)` on method
- **NEVER use `time.sleep()`** in tests — use `Waiter` with polling
- **Tests MUST be independent** — no shared mutable state, no execution order dependency

### Frameworks

- **FastAPI**: async-first, Pydantic Settings for config, `Depends()` for DI
- **Django**: split settings (base/dev/prod), custom User model, service layer pattern
- **SQLAlchemy 2.0**: `select()` style (not legacy `Query`), async sessions, `selectinload`/`joinedload`
- **Celery**: idempotent tasks, `task_acks_late=True`, batch by queue
- **pytest**: fixtures in conftest.py hierarchy, markers for test categorization

### Database

- **PostgreSQL**: `bigint` for IDs, `text` not `varchar(255)`, `timestamptz` not `timestamp`
- Always index foreign keys
- `CREATE INDEX CONCURRENTLY` for zero-downtime
- Parameterized queries only — never f-strings in SQL
- Django: `select_related`/`prefetch_related` to prevent N+1
- SQLAlchemy: `selectinload`/`joinedload` to prevent N+1

### Security

- Secrets via environment variables (`os.environ`, Pydantic `SecretStr`)
- `bandit -r src/` before every PR
- `pip-audit` for dependency vulnerabilities
- Never `eval()`, `exec()`, `pickle.loads()` with user data
- Never `yaml.load()` — use `yaml.safe_load()`
- Never `subprocess(shell=True)` with user input
- Django: `DEBUG=False` in production, CSRF enabled, HSTS headers

## Running Tests

```bash
# Run all tests with Allure
pytest --alluredir=allure-results -v

# Run with coverage
pytest --cov=src --cov-report=term-missing --alluredir=allure-results

# Run smoke suite
pytest -m smoke --alluredir=allure-results

# Run parallel
pytest -n auto --alluredir=allure-results

# Generate Allure report
allure serve allure-results

# Linting & type checking
ruff check .
black . --check
mypy .

# Security scan
bandit -r src/
pip-audit
```

## Project Structure Expected

```
project/
├── src/                        # Application source code
│   └── app/
│       ├── api/                # Routes/endpoints
│       ├── models/             # DB models (SQLAlchemy/Django)
│       ├── schemas/            # Pydantic schemas
│       ├── services/           # Business logic
│       ├── repositories/       # Data access layer
│       └── core/               # Config, security, database setup
├── tests/
│   ├── conftest.py             # Root fixtures (session-scoped)
│   ├── framework/              # Test framework (reusable)
│   │   ├── base/               # BaseTest, BaseAPIClient
│   │   ├── clients/            # HTTPClient wrappers
│   │   ├── models/             # Request/response Pydantic DTOs
│   │   ├── helpers/            # Assertions, DataGenerator, Allure helpers
│   │   └── config/             # Settings, endpoints
│   ├── api/                    # API service classes
│   │   ├── auth_api.py
│   │   └── users_api.py
│   └── tests/                  # Test classes
│       ├── test_auth/
│       ├── test_users/
│       └── test_orders/
├── alembic/                    # Migrations (if SQLAlchemy)
├── pyproject.toml
├── Dockerfile
└── docker-compose.yml
```

## Key Commands

| Command | Purpose |
|---------|---------|
| `/plan` | Create implementation plan before coding |
| `/tdd` | Test-driven development: RED → GREEN → REFACTOR |
| `/api-test` | Generate OOP API test suite with Allure |
| `/python-review` | Python code review (PEP 8, types, security) |
| `/qa-review` | Test code review (OOP, Allure, flaky patterns) |
| `/code-review` | Universal security & quality review |
| `/verify` | Full verification: types + lint + tests + security |
| `/build-fix` | Fix mypy/ruff errors with minimal diffs |
| `/test-coverage` | Find gaps, generate tests to reach 80%+ |
| `/refactor-clean` | Remove dead code safely with test verification |
| `/docs` | Look up library documentation via Context7 |
| `/update-docs` | Update project documentation, docstrings, and diagrams from code |

### DevOps & MLOps Commands

| Command | Purpose |
|---------|---------|
| `/infra-plan` | Design AWS infrastructure — services, cost, trade-offs |
| `/k8s-debug` | Diagnose CrashLoopBackOff / OOMKilled / Pending pods |
| `/terraform-review` | Review Terraform/Helm/K8s manifests for security & best practices |
| `/cicd-optimize` | Optimise CI/CD pipeline speed, security, and reliability |
| `/security-scan` | Full infra security audit — IAM, secrets, CVE, OWASP |
| `/cost-check` | AWS cost analysis — waste detection, rightsizing, savings |
| `/mlops-pipeline` | Design end-to-end ML pipeline — DVC, SageMaker, MLflow, Feature Store |
| `/kubeflow-pipeline` | Design, review, or debug a Kubeflow Pipeline (KFP v2 SDK) |
| `/data-validate` | Validate data quality — schema, drift gate, anomalies (GE / Pandera) |
| `/feature-review` | Feature Store review — training-serving skew, freshness, schema |
| `/experiment-review` | Review ML run — reproducibility, tracking completeness, model card |
| `/model-registry` | Manage model versions — compare, promote stages, aliases, lineage |
| `/model-health` | Production model health — drift, latency, retraining trigger |
| `/ab-test` | Design or analyse A/B / shadow test between model versions |
| `/retrain` | Trigger and monitor model retraining — drift/scheduled/manual |
| `/model-promote` | Safe model promotion — canary deploy, rollback plan, approval gate |

## Available Skills (Deep Knowledge)
*Claude pulls these in automatically when context matches. Do not run them.*

| Domain | Skills |
|--------|--------|
| **Core Python** | `python-patterns`, `python-testing`, `pydantic-patterns`, `async-http-patterns` |
| **Frameworks** | `fastapi-patterns`, `django-patterns`, `django-security`, `django-tdd`, `django-verification`, `celery-patterns` |
| **Data & Infra** | `sqlalchemy-patterns`, `postgres-patterns`, `clickhouse-io`, `redis-patterns`, `database-migrations`, `deployment-patterns`, `docker-patterns` |
| **QA Auto** | `pytest-oop-patterns`, `allure-reporting`, `api-testing-patterns` |
| **DevOps** | `kubernetes`, `terraform`, `aws`, `ci-cd`, `observability`, `security-infra`, `networking` |
| **MLOps** | `sagemaker`, `seldon-core`, `mlflow`, `mlops` |

## Agents Available

| Agent | Use When |
|-------|----------|
| `architect` | Designing system architecture, making tech decisions |
| `planner` | Planning features, breaking down complex work |
| `tdd-guide` | Writing new features test-first |
| `python-reviewer` | Reviewing Python code changes |
| `qa-architect` | Designing test framework, OOP test architecture |
| `api-test-writer` | Generating new API test suites |
| `qa-reviewer` | Reviewing test code before merge |
| `security-reviewer` | Checking for OWASP Top 10, secrets, vulns |
| `database-reviewer` | Reviewing SQL, migrations, query performance |
| `code-reviewer` | General code quality and security review |
| `refactor-cleaner` | Cleaning up dead code and duplicates |
| `doc-updater` | Keeping documentation in sync with code |

### DevOps & MLOps Agents

| Agent | Use When |
|-------|----------|
| `devops-architect` | Designing AWS infrastructure, choosing services, cost vs reliability trade-offs |
| `infra-reviewer` | Reviewing Terraform / Helm / K8s manifests before merge |
| `k8s-debugger` | Diagnosing pod failures, networking, Istio issues |
| `ci-cd-engineer` | Optimising pipelines, GitOps setup, OIDC migration |
| `security-auditor` | IAM audit, secrets scan, CVE review, OWASP infra checklist |
| `cost-optimizer` | Rightsizing, Reserved/Spot strategy, Cost Explorer analysis |
| `mlops-engineer` | ML pipelines, feature stores, model registry, drift detection |

## DevOps & MLOps Rules (Non-Negotiable)

> Full rules in `rules/infra.md` — applies to all `.tf`, `.yaml`, `.yml`, `Dockerfile*` files.

1. **No hardcoded secrets** — Secrets Manager / IRSA / env vars only; never in `.tf`, `.yaml`, or code
2. **Infrastructure as Code only** — No ClickOps; every resource in Terraform or CDK; drift = bug
3. **Idempotency required** — All infra operations safe to run multiple times
4. **Observability mandatory** — `readinessProbe` + `livenessProbe` + metrics + structured logs from day one
5. **Rollback strategy always** — `helm upgrade --atomic`, ArgoCD rollback, `prevent_destroy` on critical resources
6. **Least privilege IAM** — No wildcard actions; IRSA for EKS workloads; no long-lived access keys
7. **Mandatory tagging** — All AWS resources tagged: `env`, `team`, `managed_by`, `cost_center`

## IaC Project Structure Expected

```
project/
├── infra/                          # Infrastructure as Code
│   ├── modules/                    # Reusable Terraform modules (no state)
│   │   ├── vpc/
│   │   ├── eks-cluster/
│   │   └── rds-postgres/
│   ├── environments/               # Environment root modules
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   └── .terraform-version          # tfenv version pin
├── k8s/                            # Kubernetes manifests / Helm charts
│   ├── charts/
│   │   └── myapp/
│   │       ├── Chart.yaml
│   │       ├── values.yaml
│   │       ├── values-prod.yaml
│   │       └── templates/
│   └── manifests/                  # Raw K8s manifests (if not Helm)
├── ml/                             # MLOps
│   ├── pipelines/                  # SageMaker / Kubeflow pipeline definitions
│   ├── scripts/                    # preprocess.py, train.py, evaluate.py
│   ├── feature_store/              # Feast feature repo
│   └── dvc.yaml                    # DVC pipeline
├── .github/
│   └── workflows/
│       ├── ci-cd.yml               # Main CI/CD pipeline
│       └── drift-detection.yml     # Daily Terraform drift check
├── .pre-commit-config.yaml         # Pre-commit hooks (see docs/DEVOPS_SETUP.md)
├── .tflint.hcl                     # tflint config
├── .checkov.yaml                   # checkov config
└── .gitleaks.toml                  # gitleaks config
```

## What NOT To Do

### Python
- Do not write bare test functions — always use classes
- Do not call `httpx`/`requests` directly in tests — use API service classes
- Do not hardcode test data — use DataGenerator
- Do not use `time.sleep()` — use Waiter/polling
- Do not skip Allure decorators — they are mandatory
- Do not use `print()` — use `logging` or Allure attachments
- Do not use `SELECT *` — specify columns
- Do not use `OFFSET` pagination — use cursor pagination
- Do not use `eval()`, `pickle`, `yaml.load()` with untrusted data

### Database & Migrations (CRITICAL)
- **Do not execute schema migrations (`alembic upgrade`, `manage.py migrate`) against production from your local machine.**
- **Do not run destructive data operations (`DROP TABLE`, `DELETE FROM` without `WHERE` limit).**
- Do not run database migrations inside application startup (e.g. FastAPI app lifespan). Always use a dedicated Kubernetes Job (Helm pre-install hook) or a dedicated CI/CD pipeline step.
- Do not write migration scripts that lock large tables for hours (`ALTER TABLE ADD COLUMN` without default is safe, but adding constraints via `UPDATE` requires batched operations).

### Terraform & State (CRITICAL)
- **Do not manually edit, delete, or commit `.tfstate` files.**
- **Do not run `terraform state rm` or `terraform state push` without strictly reviewing the blast radius.**
- Always use a remote backend (S3/GCS) with state locking (DynamoDB) enabled. Never use local state for production.

### DevOps / MLOps
- Do not hardcode secrets anywhere — Secrets Manager / External Secrets Operator
- Do not make manual AWS console changes (ClickOps) — IaC only
- Do not use `:latest` image tags — pin to git SHA or semver
- Do not run `terraform apply` without reviewing `terraform plan` first
- Do not deploy to production without a canary / rollback strategy
- Do not commit `mlruns/`, `wandb/`, `.dvc/cache` to git
- Do not use static AWS access keys in CI/CD — use OIDC
- Do not skip resource limits on K8s containers — OOMKilled in production

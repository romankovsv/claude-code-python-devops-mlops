---
name: mlflow
description: MLflow patterns for experiment tracking, model registry, artifact storage, Projects reproducibility, and model serving — with best practices for Python ML teams.
origin: custom
---

# MLflow Patterns

MLflow tracking, model registry, and serving for Python ML projects.

## When to Activate

- Tracking ML experiments (parameters, metrics, artifacts)
- Registering and versioning trained models
- Comparing runs across experiments
- Serving models via REST API or Docker
- Setting up reproducible training with MLflow Projects

## Experiment Tracking

```python
import mlflow
import mlflow.sklearn
import mlflow.xgboost
import git
from mlflow.models.signature import infer_signature

mlflow.set_tracking_uri("http://mlflow-server:5000")      # remote tracking server
mlflow.set_experiment("fraud-detection-v2")

# Auto-tag run with git SHA for full reproducibility
repo = git.Repo(search_parent_directories=True)
git_sha = repo.head.commit.hexsha

with mlflow.start_run(run_name="xgboost-baseline") as run:
    # Tag for governance
    mlflow.set_tags({
        "git_sha": git_sha,
        "dataset_version": "2026-03-29",
        "team": "ml-platform",
        "env": "staging",
    })

    # Log parameters
    params = {
        "learning_rate": 0.05,
        "max_depth": 6,
        "n_estimators": 300,
        "subsample": 0.8,
        "colsample_bytree": 0.8,
    }
    mlflow.log_params(params)

    model = train_model(X_train, y_train, **params)

    # Log metrics per epoch (step=epoch for time-series view)
    for epoch, (auc, f1) in enumerate(training_history):
        mlflow.log_metrics({"train_auc": auc, "train_f1": f1}, step=epoch)

    # Final evaluation metrics
    mlflow.log_metrics({
        "val_auc_roc": evaluate_auc(model, X_val, y_val),
        "val_f1_score": evaluate_f1(model, X_val, y_val),
        "val_precision": evaluate_precision(model, X_val, y_val),
        "val_recall": evaluate_recall(model, X_val, y_val),
    })

    # Log model with signature — enables input validation on serve
    signature = infer_signature(X_train, model.predict(X_train))
    input_example = X_train.iloc[:3]

    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="model",
        signature=signature,
        input_example=input_example,
        registered_model_name="FraudDetector",   # ✅ auto-register
    )

    # Log artifacts
    mlflow.log_artifact("reports/feature_importance.png")
    mlflow.log_artifact("reports/confusion_matrix.png")
    mlflow.log_artifact("reports/shap_summary.png")

    print(f"Run ID: {run.info.run_id}")
    print(f"Artifact URI: {run.info.artifact_uri}")
```

## Custom Training Loop (PyTorch)

```python
import mlflow.pytorch
import torch

with mlflow.start_run():
    mlflow.log_params({"lr": 1e-3, "epochs": 50, "batch_size": 64})

    model = MyNeuralNet()
    optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

    for epoch in range(50):
        loss = train_epoch(model, optimizer, train_loader)
        val_loss, val_acc = evaluate(model, val_loader)
        mlflow.log_metrics(
            {"train_loss": loss, "val_loss": val_loss, "val_acc": val_acc},
            step=epoch,
        )

    # Log PyTorch model
    signature = infer_signature(sample_input, model(sample_input).detach().numpy())
    mlflow.pytorch.log_model(
        pytorch_model=model,
        artifact_path="model",
        signature=signature,
        registered_model_name="MyNeuralNet",
    )
```

## Model Registry — Lifecycle Management

```python
from mlflow.tracking import MlflowClient

client = MlflowClient(tracking_uri="http://mlflow-server:5000")

# View all versions
versions = client.search_model_versions("name='FraudDetector'")
for v in versions:
    print(f"Version {v.version} | Stage: {v.current_stage} | Run: {v.run_id}")

# Promote: None → Staging
client.transition_model_version_stage(
    name="FraudDetector",
    version=3,
    stage="Staging",
)

# After staging validation → Production
client.transition_model_version_stage(
    name="FraudDetector",
    version=3,
    stage="Production",
    archive_existing_versions=True,   # ✅ auto-archive old prod
)

# Add description / model card
client.update_model_version(
    name="FraudDetector",
    version=3,
    description=(
        "XGBoost v2 — val_auc=0.97, val_f1=0.94\n"
        "Trained on 6M samples, 2026-03-29\n"
        "Approved by: ml-team"
    ),
)
```

## Load Model for Inference

```python
import mlflow.pyfunc

# Load by stage (always use Production in prod services)
model = mlflow.pyfunc.load_model("models:/FraudDetector/Production")
predictions = model.predict(X_new)

# Load by specific version (for reproducibility in tests)
model_v3 = mlflow.pyfunc.load_model("models:/FraudDetector/3")
```

## MLflow Projects (Reproducible Training)

```yaml
# MLproject
name: fraud-detector
conda_env: conda.yaml

entry_points:
  train:
    parameters:
      learning_rate: {type: float, default: 0.05}
      max_depth:     {type: int,   default: 6}
      n_estimators:  {type: int,   default: 300}
      data_version:  {type: string, default: "latest"}
    command: >
      python scripts/train.py
        --lr {learning_rate}
        --depth {max_depth}
        --estimators {n_estimators}
        --data-version {data_version}

  evaluate:
    parameters:
      run_id: {type: string}
    command: "python scripts/evaluate.py --run-id {run_id}"
```

```bash
# Run locally
mlflow run . -P learning_rate=0.01 -P max_depth=8

# Run from git (fully reproducible — pins commit)
mlflow run git+https://github.com/my-org/fraud-model.git@v1.2.0 \
  -P learning_rate=0.01 \
  -P max_depth=8
```

## MLflow Serving

```bash
# Serve registered model locally
mlflow models serve \
  -m "models:/FraudDetector/Production" \
  -p 8080 \
  --no-conda

# Predict via REST
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"dataframe_records": [{"feature_1": 0.5, "feature_2": 1.2}]}'

# Build production Docker image
mlflow models build-docker \
  -m "models:/FraudDetector/Production" \
  -n "my-registry/fraud-detector:v3" \
  --enable-mlserver

docker push my-registry/fraud-detector:v3
```

## Remote Tracking Server (Docker Compose)

```yaml
# docker-compose.mlflow.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mlflow
      POSTGRES_USER: mlflow
      POSTGRES_PASSWORD: ${MLFLOW_DB_PASSWORD}
    volumes:
      - mlflow-postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mlflow"]
      interval: 10s
      timeout: 5s
      retries: 5

  mlflow:
    image: ghcr.io/mlflow/mlflow:v2.12.0
    ports:
      - "5000:5000"
    environment:
      MLFLOW_BACKEND_STORE_URI: postgresql://mlflow:${MLFLOW_DB_PASSWORD}@postgres/mlflow
      MLFLOW_ARTIFACT_ROOT: s3://my-bucket/mlflow-artifacts/
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_DEFAULT_REGION: us-east-1
    depends_on:
      postgres:
        condition: service_healthy
    command: >
      mlflow server
        --backend-store-uri postgresql://mlflow:${MLFLOW_DB_PASSWORD}@postgres/mlflow
        --default-artifact-root s3://my-bucket/mlflow-artifacts/
        --host 0.0.0.0
        --port 5000
        --workers 4
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  mlflow-postgres:
```

## Hyperparameter Tuning with MLflow

```python
import optuna
import mlflow

def objective(trial: optuna.Trial) -> float:
    lr = trial.suggest_float("learning_rate", 1e-4, 1e-1, log=True)
    depth = trial.suggest_int("max_depth", 3, 10)
    n_est = trial.suggest_int("n_estimators", 100, 500, step=50)

    with mlflow.start_run(nested=True):
        mlflow.log_params({"lr": lr, "depth": depth, "n_est": n_est})
        model = train_model(X_train, y_train, lr=lr, depth=depth, n_est=n_est)
        auc = evaluate_auc(model, X_val, y_val)
        mlflow.log_metric("val_auc", auc)
        return auc

with mlflow.start_run(run_name="optuna-sweep"):
    study = optuna.create_study(direction="maximize")
    study.optimize(objective, n_trials=50)
    mlflow.log_params(study.best_params)
    mlflow.log_metric("best_val_auc", study.best_value)
```

## Best Practices

| Practice | Detail |
|----------|--------|
| ✅ Remote tracking server | Never use local `mlruns/` in team/prod — use PostgreSQL + S3 |
| ✅ S3 artifact store | All models, plots, data profiles in versioned S3 |
| ✅ Model signature | Always `infer_signature()` — enables input validation on serve |
| ✅ Tag runs with git SHA | Full reproducibility — link run → code |
| ✅ Archive old prod versions | `archive_existing_versions=True` on every promotion |
| ✅ Human approval gate | Staging → Production requires explicit review |
| ✅ Pin mlflow version | `mlflow==2.12.0` in `requirements.txt` — never `>=` |

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| ❌ Local `mlruns/` committed to git | Remote PostgreSQL + S3 backend; add `mlruns/` to `.gitignore` |
| ❌ No model signature | `infer_signature()` on every `log_model()` call |
| ❌ Promoting without metrics review | Always log and review metrics before staging → prod |
| ❌ Secrets in MLproject params | Use env vars or AWS Secrets Manager |
| ❌ `mlflow>=1.0` in requirements | Pin exact version: `mlflow==2.12.0` |
| ❌ No `input_example` | Add `input_example=X_train.iloc[:3]` for schema docs |


---
description: Design, review, or debug a Kubeflow Pipeline — component structure, caching, artifacts, resource requests, recurring runs, and KFP v2 SDK patterns. Invokes mlops-engineer agent.
---

# /kubeflow-pipeline

Invokes the **mlops-engineer** agent to design, review, or debug a Kubeflow Pipeline.

## What This Command Does

1. **Design** — component breakdown, DAG topology, caching strategy, artifact passing
2. **Review** — KFP v2 SDK compliance, resource requests, idempotency, error handling
3. **Debug** — diagnose failed runs, stuck components, OOMKilled containers
4. **Recurring Runs** — schedule setup, concurrency policy, failure notifications
5. **Compare vs SageMaker Pipelines** — when to use which

## When to Use

- Building a new training or batch-inference pipeline on Kubeflow
- Migrating a notebook or script to a reproducible KFP pipeline
- Debugging a failed or stuck pipeline run in KFP UI
- Setting up scheduled recurring runs
- Deciding between Kubeflow Pipelines and SageMaker Pipelines

## Required Context (provide when running)

```
Action:         [design / review / debug]
Pipeline steps: [preprocess → train → evaluate → register, or describe]
Cluster:        [GKE / EKS / on-prem]
KFP version:    [v1 / v2 — default v2]
Problem:        [if debugging: component name + error message]
```

## KFP v2 SDK — Core Patterns

### Component Definition
```python
from kfp import dsl
from kfp.dsl import Dataset, Input, Output, Model, Metrics

# ✅ Typed component — inputs/outputs as artifacts
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["scikit-learn==1.4.0", "pandas==2.2.0"],
)
def preprocess(
    raw_data: Input[Dataset],
    processed_data: Output[Dataset],
    test_size: float = 0.2,
) -> None:
    import pandas as pd
    from sklearn.model_selection import train_test_split

    df = pd.read_parquet(raw_data.path)
    train, test = train_test_split(df, test_size=test_size, random_state=42)
    train.to_parquet(f"{processed_data.path}/train.parquet")
    test.to_parquet(f"{processed_data.path}/test.parquet")


@dsl.component(base_image="python:3.11-slim", packages_to_install=["scikit-learn==1.4.0"])
def train(
    processed_data: Input[Dataset],
    model: Output[Model],
    metrics: Output[Metrics],
    n_estimators: int = 100,
    learning_rate: float = 0.1,
) -> None:
    import joblib
    import pandas as pd
    from sklearn.ensemble import GradientBoostingClassifier
    from sklearn.metrics import roc_auc_score

    train_df = pd.read_parquet(f"{processed_data.path}/train.parquet")
    test_df  = pd.read_parquet(f"{processed_data.path}/test.parquet")

    X_train, y_train = train_df.drop("label", axis=1), train_df["label"]
    X_test,  y_test  = test_df.drop("label", axis=1),  test_df["label"]

    clf = GradientBoostingClassifier(n_estimators=n_estimators, learning_rate=learning_rate)
    clf.fit(X_train, y_train)

    auc = roc_auc_score(y_test, clf.predict_proba(X_test)[:, 1])
    metrics.log_metric("auc_roc", auc)
    metrics.log_metric("n_estimators", n_estimators)

    joblib.dump(clf, model.path + "/model.joblib")


@dsl.component(base_image="python:3.11-slim", packages_to_install=["mlflow==2.12.0"])
def register_model(
    model: Input[Model],
    metrics: Input[Metrics],
    model_name: str = "fraud-detector",
    auc_threshold: float = 0.90,
) -> str:
    import mlflow
    import mlflow.sklearn
    import joblib

    auc = metrics.metadata["auc_roc"]
    if auc < auc_threshold:
        raise ValueError(f"AUC {auc:.3f} below threshold {auc_threshold} — not registering")

    mlflow.set_tracking_uri("http://mlflow-service.mlops.svc.cluster.local:5000")
    with mlflow.start_run():
        mlflow.log_metric("auc_roc", auc)
        clf = joblib.load(model.path + "/model.joblib")
        mlflow.sklearn.log_model(clf, "model", registered_model_name=model_name)

    return f"registered:{model_name}"
```

### Pipeline DAG with Caching
```python
@dsl.pipeline(
    name="fraud-detection-training",
    description="Weekly retraining pipeline for fraud detection model",
)
def fraud_pipeline(
    data_path: str = "gs://my-bucket/data/fraud/v2.0",
    n_estimators: int = 200,
    learning_rate: float = 0.05,
    auc_threshold: float = 0.90,
) -> None:

    preprocess_task = preprocess(raw_data_uri=data_path)
    preprocess_task.set_caching_options(enable_caching=True)  # cache if input unchanged
    preprocess_task.set_memory_limit("4G")
    preprocess_task.set_cpu_limit("2")

    train_task = train(
        processed_data=preprocess_task.outputs["processed_data"],
        n_estimators=n_estimators,
        learning_rate=learning_rate,
    )
    train_task.set_caching_options(enable_caching=False)  # always retrain
    train_task.set_memory_limit("8G")
    train_task.set_cpu_limit("4")
    # GPU training:
    # train_task.set_accelerator_type("NVIDIA_TESLA_T4").set_accelerator_limit(1)

    register_model(
        model=train_task.outputs["model"],
        metrics=train_task.outputs["metrics"],
        auc_threshold=auc_threshold,
    )


# Compile to YAML
from kfp import compiler
compiler.Compiler().compile(fraud_pipeline, "fraud_pipeline.yaml")
```

### Submit & Schedule
```python
import kfp

client = kfp.Client(host="http://ml-pipeline.kubeflow.svc.cluster.local:8888")

# One-off run
run = client.create_run_from_pipeline_func(
    fraud_pipeline,
    arguments={"n_estimators": 200, "auc_threshold": 0.90},
    run_name="fraud-training-2026-03-29",
    experiment_name="fraud-detection",
)
print(f"Run URL: {client.get_run(run.run_id).run.id}")

# Recurring run (weekly on Monday 02:00 UTC)
client.create_recurring_run(
    experiment_id=client.get_experiment(experiment_name="fraud-detection").id,
    job_name="fraud-weekly-retrain",
    cron_expression="0 2 * * 1",
    max_concurrency=1,
    pipeline_func=fraud_pipeline,
    arguments={"n_estimators": 200},
)
```

## Debugging Failed Runs

```bash
# Get pod logs for a failed component
# 1. Find the pod name from KFP UI or:
kubectl get pods -n kubeflow-user \
  -l pipeline/runid=<run-id> \
  --sort-by=.metadata.creationTimestamp

# 2. Check logs
kubectl logs <pod-name> -n kubeflow-user -c main
kubectl logs <pod-name> -n kubeflow-user --previous  # if CrashLoopBackOff

# 3. Common errors:
# OOMKilled → increase set_memory_limit()
# ImagePullBackOff → check base_image tag, registry auth
# Permission denied on GCS/S3 → check Workload Identity / IRSA on service account
# Component stuck "pending" → check node resources: kubectl top nodes

# 4. Describe pod for events
kubectl describe pod <pod-name> -n kubeflow-user | tail -30
```

## KFP v2 vs SageMaker Pipelines — When to Use Which

| Factor | Kubeflow Pipelines | SageMaker Pipelines |
|--------|-------------------|---------------------|
| Infrastructure | Your K8s cluster (EKS/GKE/on-prem) | Fully managed AWS |
| Cost | Pay for K8s nodes only | Pay per pipeline execution step |
| Portability | ✅ Cloud-agnostic | ❌ AWS-only |
| Managed scaling | Manual (HPA on nodes) | ✅ Auto (managed instances) |
| Built-in spot | Manual (tolerations) | ✅ Native Spot support |
| UI | KFP UI (self-hosted) | SageMaker Studio |
| Best for | Multi-cloud, on-prem, cost control | AWS-only, managed, fast setup |

## Anti-Patterns

```python
# ❌ Passing large data as parameters (use Dataset artifacts)
def train(data: str = "huge json string..."):  # breaks at scale

# ✅ Pass as artifact
def train(data: Input[Dataset]):
    df = pd.read_parquet(data.path)

# ❌ Hardcoding credentials inside component
def train():
    boto3.client("s3", aws_access_key_id="AKIA...")  # NEVER

# ✅ Use Workload Identity (GKE) or IRSA (EKS) — no keys in code

# ❌ No resource limits (OOMKills neighbours)
train_task = train(...)  # no limits set

# ✅ Always set limits
train_task.set_memory_limit("8G").set_cpu_limit("4")

# ❌ enable_caching=True on training step
# → stale model if data changed but path didn't
train_task.set_caching_options(enable_caching=True)  # dangerous for train

# ✅ Cache preprocessing, never cache training
preprocess_task.set_caching_options(enable_caching=True)
train_task.set_caching_options(enable_caching=False)
```

## Output Format

```
## Kubeflow Pipeline Review — [pipeline_name]

### Design
DAG: preprocess(cached) → train → evaluate → [gate] → register
Components: 4 | Estimated duration: ~35 min

### 🔴 CRITICAL
- train component: no memory/CPU limits → risk of OOMKill on shared cluster
- credentials hardcoded in register component → use IRSA/Workload Identity

### 🟠 HIGH
- enable_caching=True on train step → stale model risk if data path doesn't change
- No auc_threshold gate before register → bad model can reach registry

### 🟡 MEDIUM
- base_image uses :latest tag → non-reproducible builds; pin to SHA

### ✅ Passing
- Typed artifacts used throughout (Dataset, Model, Metrics)
- Recurring run configured with max_concurrency=1
```


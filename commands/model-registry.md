---
description: Manage ML model lifecycle in the registry — promote stages (Staging → Production), compare versions, archive old models, and audit lineage. Works with MLflow Model Registry and SageMaker Model Registry. Invokes mlops-engineer agent.
---

# /model-registry

Invokes the **mlops-engineer** agent to manage model versions in the registry.

## What This Command Does

1. **Compare Versions** — metrics side-by-side across registered model versions
2. **Promote Stage** — move model: `None → Staging → Production` with approval gate
3. **Archive** — safely retire old model versions (with traffic check first)
4. **Lineage Audit** — which data, code, and hyperparams produced this model
5. **Alias Management** — MLflow aliases (`champion`, `challenger`) vs stages

## When to Use

- After a training run to register a new candidate model
- Before `/model-promote` — verify registry state is clean
- Auditing which model version is in production and why
- Retiring old model versions after successful promotion
- Setting up champion/challenger A/B test

## Required Context (provide when running)

```
Registry type:    [MLflow / SageMaker / both]
Model name:       [registered model name]
Action:           [compare / promote / archive / lineage / alias]
Version:          [version number or run_id]
```

## MLflow Model Registry

### Register & Promote
```python
import mlflow
from mlflow import MlflowClient

client = MlflowClient(tracking_uri="http://mlflow-service.mlops.svc:5000")
MODEL_NAME = "fraud-detector"

# Register from a run
run_id = "abc123def456"
model_uri = f"runs:/{run_id}/model"

# Register to registry
result = mlflow.register_model(model_uri=model_uri, name=MODEL_NAME)
version = result.version
print(f"Registered as version {version}")

# Add descriptive tags
client.set_model_version_tag(MODEL_NAME, version, "dataset_dvc_tag", "v2.4.0")
client.set_model_version_tag(MODEL_NAME, version, "git_sha", "a1b2c3d4")
client.set_model_version_tag(MODEL_NAME, version, "auc_roc", "0.931")

# ─── MLflow v2 recommended: use ALIASES instead of stages ───
# Set challenger alias (new candidate)
client.set_registered_model_alias(MODEL_NAME, "challenger", version)

# After validation — promote challenger to champion
client.set_registered_model_alias(MODEL_NAME, "champion", version)

# Load by alias (stable reference in serving code)
model = mlflow.sklearn.load_model(f"models:/{MODEL_NAME}@champion")
```

### Compare Versions
```python
def compare_model_versions(model_name: str, versions: list[int]) -> None:
    client = MlflowClient()
    rows = []
    for v in versions:
        mv = client.get_model_version(model_name, str(v))
        run = client.get_run(mv.run_id)
        rows.append({
            "version":    v,
            "alias":      mv.aliases,
            "auc_roc":    run.data.metrics.get("eval/auc_roc", "N/A"),
            "f1":         run.data.metrics.get("eval/f1", "N/A"),
            "dataset":    mv.tags.get("dataset_dvc_tag", "N/A"),
            "git_sha":    mv.tags.get("git_sha", "N/A")[:8],
            "created":    mv.creation_timestamp,
        })

    import pandas as pd
    df = pd.DataFrame(rows).sort_values("version", ascending=False)
    print(df.to_string(index=False))

compare_model_versions("fraud-detector", versions=[8, 9, 10])
# version  alias        auc_roc   f1     dataset  git_sha
#      10  [champion]   0.931     0.882  v2.4.0   a1b2c3d4
#       9  []           0.924     0.876  v2.3.0   f9e8d7c6
#       8  []           0.918     0.871  v2.2.0   b5a4c3d2
```

### Archive Old Versions
```python
# ✅ Safe archive: check no active aliases before archiving
def safe_archive(model_name: str, version: int) -> None:
    mv = client.get_model_version(model_name, str(version))
    if mv.aliases:
        raise ValueError(
            f"Cannot archive v{version} — active aliases: {mv.aliases}. "
            "Remove aliases first."
        )
    client.update_model_version(
        name=model_name,
        version=str(version),
        description=f"Archived — superseded by v{version + 2}",
    )
    # MLflow v2: no "Archived" stage — just remove from use and delete if needed
    # client.delete_model_version(model_name, str(version))  # permanent deletion
    print(f"✅ v{version} archived")

safe_archive("fraud-detector", 7)
```

### Lineage Audit
```python
def audit_model_lineage(model_name: str, alias: str = "champion") -> None:
    client = MlflowClient()
    mv = client.get_model_version_by_alias(model_name, alias)
    run = client.get_run(mv.run_id)

    print(f"=== Model Lineage: {model_name}@{alias} (v{mv.version}) ===")
    print(f"Run ID:       {mv.run_id}")
    print(f"Git SHA:      {mv.tags.get('git_sha', 'NOT LOGGED ⚠️')}")
    print(f"Dataset:      {mv.tags.get('dataset_dvc_tag', 'NOT LOGGED ⚠️')}")
    print(f"Created:      {mv.creation_timestamp}")
    print(f"\nHyperparams:")
    for k, v in run.data.params.items():
        print(f"  {k}: {v}")
    print(f"\nMetrics:")
    for k, v in run.data.metrics.items():
        print(f"  {k}: {v:.4f}")

audit_model_lineage("fraud-detector", alias="champion")
```

## SageMaker Model Registry

```python
import boto3

sm = boto3.client("sagemaker")

# List versions
response = sm.list_model_packages(
    ModelPackageGroupName="fraud-detector",
    SortBy="CreationTime",
    SortOrder="Descending",
)
for pkg in response["ModelPackageSummaryList"][:5]:
    print(f"v{pkg['ModelPackageVersion']} | {pkg['ModelApprovalStatus']} | {pkg['CreationTime']}")

# Approve a version (Pending → Approved)
sm.update_model_package(
    ModelPackageName=f"arn:aws:sagemaker:us-east-1:123456789012:model-package/fraud-detector/10",
    ModelApprovalStatus="Approved",
    ApprovalDescription="AUC-ROC 0.931, load test passed, model card complete",
)

# Reject (failed evaluation)
sm.update_model_package(
    ModelPackageName="...",
    ModelApprovalStatus="Rejected",
    ApprovalDescription="AUC-ROC 0.87 below 0.90 threshold",
)
```

## Output Format

```
## Model Registry Report — fraud-detector

### Current State
champion (v10):    AUC-ROC=0.931  F1=0.882  dataset=v2.4.0  git=a1b2c3d
challenger (v11):  AUC-ROC=0.934  F1=0.885  dataset=v2.5.0  git=b2c3d4e  ← candidate

### Version Comparison (last 5)
v11  AUC=0.934  F1=0.885  ← challenger
v10  AUC=0.931  F1=0.882  ← champion  
v9   AUC=0.924  F1=0.876
v8   AUC=0.918  F1=0.871
v7   AUC=0.901  F1=0.863  (archived)

### Lineage Warnings
🟠 v9: dataset_dvc_tag NOT LOGGED — lineage incomplete
✅ v10, v11: full lineage (git SHA + DVC tag + hyperparams)

### Recommendations
1. Promote v11 challenger to champion: run /model-promote
2. Archive v8 (superseded, no active aliases)
3. Add dataset DVC tag to v9 run retrospectively
```


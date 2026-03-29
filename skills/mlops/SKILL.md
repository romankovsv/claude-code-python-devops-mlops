---
name: mlops
description: MLOps patterns — MLflow, DVC, Weights & Biases, Feast vs AWS Feature Store (comparison + examples), SageMaker, Seldon Core, model drift detection, and end-to-end pipeline design.
origin: custom
---

# MLOps Patterns

End-to-end ML lifecycle: data versioning, experiment tracking, feature stores, model serving, and drift detection.

## When to Activate

- Designing or reviewing end-to-end ML pipelines
- Choosing between Feast and AWS Feature Store
- Setting up DVC for data and pipeline versioning
- Configuring W&B sweeps for hyperparameter tuning
- Implementing model drift detection and retraining triggers

## ML Lifecycle Overview

```
Data → Feature Store → Training → Experiment Tracking → Model Registry → Serving → Monitoring
 DVC      Feast/AWS       SageMaker    MLflow/W&B          MLflow/SM        Seldon    Evidently
```

## DVC — Data & Pipeline Versioning

```bash
# Initialize DVC in git repo
dvc init
git add .dvc .gitignore
git commit -m "chore: initialize DVC"

# Add remote storage (S3)
dvc remote add -d myremote s3://my-bucket/dvc-cache
dvc remote modify myremote region us-east-1
git add .dvc/config
git commit -m "chore: add S3 DVC remote"

# Track dataset
dvc add data/raw/transactions.csv
git add data/raw/transactions.csv.dvc data/raw/.gitignore
git commit -m "data: add transactions dataset v1"
dvc push   # uploads to S3

# Reproduce on another machine
git clone https://github.com/my-org/fraud-model
dvc pull   # downloads from S3
```

```yaml
# dvc.yaml — pipeline definition
stages:
  preprocess:
    cmd: python scripts/preprocess.py --input data/raw --output data/processed
    deps:
      - scripts/preprocess.py
      - data/raw/transactions.csv
    outs:
      - data/processed/train.parquet
      - data/processed/test.parquet
    params:
      - params.yaml:
          - preprocess.test_size
          - preprocess.random_seed

  train:
    cmd: python scripts/train.py
    deps:
      - scripts/train.py
      - data/processed/train.parquet
    outs:
      - models/model.pkl
    params:
      - params.yaml:
          - train.learning_rate
          - train.max_depth
          - train.n_estimators
    metrics:
      - metrics/train_metrics.json:
          cache: false

  evaluate:
    cmd: python scripts/evaluate.py
    deps:
      - scripts/evaluate.py
      - models/model.pkl
      - data/processed/test.parquet
    metrics:
      - metrics/eval_metrics.json:
          cache: false
    plots:
      - metrics/roc_curve.csv:
          x: fpr
          y: tpr
```

```bash
# Run pipeline (only changed stages)
dvc repro

# Compare metrics across commits
dvc metrics diff HEAD~1

# Show DAG
dvc dag
```

## Weights & Biases — Experiment Tracking

```python
import wandb
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import roc_auc_score, f1_score

# Initialize run
wandb.init(
    project="fraud-detection",
    name="gbm-baseline-v2",
    tags=["baseline", "gbm", "2026-03"],
    config={
        "learning_rate": 0.05,
        "max_depth": 6,
        "n_estimators": 300,
        "subsample": 0.8,
        "dataset_version": "2026-03-29",
        "architecture": "gbm",
    },
    notes="Gradient boosting baseline with SMOTE oversampling",
)

# Log metrics during training
for epoch in range(config.n_estimators):
    train_loss = ...
    wandb.log({"train/loss": train_loss, "epoch": epoch})

# Final evaluation
model = GradientBoostingClassifier(**wandb.config)
model.fit(X_train, y_train)

metrics = {
    "eval/auc_roc": roc_auc_score(y_test, model.predict_proba(X_test)[:, 1]),
    "eval/f1":      f1_score(y_test, model.predict(X_test)),
}
wandb.log(metrics)

# Log model artifact
artifact = wandb.Artifact("fraud-detector", type="model")
artifact.add_file("models/model.pkl")
wandb.log_artifact(artifact)

# Log feature importance plot
wandb.log({"feature_importance": wandb.plot.bar(
    wandb.Table(
        data=[[f, i] for f, i in zip(feature_names, model.feature_importances_)],
        columns=["feature", "importance"],
    ),
    "feature",
    "importance",
)})

wandb.finish()
```

```python
# W&B Sweeps — hyperparameter optimisation
sweep_config = {
    "method": "bayes",         # bayesian optimisation
    "metric": {"name": "eval/auc_roc", "goal": "maximize"},
    "parameters": {
        "learning_rate": {"distribution": "log_uniform_values", "min": 0.001, "max": 0.1},
        "max_depth":     {"values": [3, 4, 5, 6, 7, 8]},
        "n_estimators":  {"values": [100, 200, 300, 500]},
        "subsample":     {"distribution": "uniform", "min": 0.6, "max": 1.0},
    },
    "early_terminate": {
        "type": "hyperband",
        "min_iter": 5,
    },
}

sweep_id = wandb.sweep(sweep_config, project="fraud-detection")
wandb.agent(sweep_id, function=train, count=50)
```

## Feature Store — Feast vs AWS Feature Store

### Comparison Table

| Dimension | Feast (Open Source) | AWS Feature Store |
|-----------|--------------------|--------------------|
| **Cost** | Free (infra costs only) | Pay per write/read + storage |
| **Managed** | Self-hosted | Fully managed by AWS |
| **Online store** | Redis, DynamoDB, Bigtable | Managed (low-latency, ~1ms p99) |
| **Offline store** | S3, BigQuery, Redshift | S3 + Athena (managed) |
| **Feature freshness** | Depends on materialisation job | Near real-time (streaming ingestion) |
| **SageMaker integration** | Manual | ✅ Native (training datasets, endpoints) |
| **Portability** | ✅ Multi-cloud | AWS lock-in |
| **Team size** | Any (self-service) | Better for large AWS orgs |
| **Best for** | Multi-cloud, existing infra, open-source preference | AWS-native, SageMaker pipelines |

---

### Feast — Python Examples

```python
# feature_store/feature_repo/features.py
from datetime import timedelta
from feast import Entity, FeatureView, Field, FileSource, ValueType
from feast.types import Float32, Int64, String

# Entity — the object features describe
customer = Entity(
    name="customer_id",
    value_type=ValueType.INT64,
    description="Customer identifier",
    join_keys=["customer_id"],
)

# Source — historical offline data
transactions_source = FileSource(
    path="s3://my-bucket/feast/transactions/",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_timestamp",
)

# Feature View
customer_transaction_features = FeatureView(
    name="customer_transaction_stats",
    entities=[customer],
    ttl=timedelta(days=7),              # features expire after 7 days
    schema=[
        Field(name="avg_transaction_amount_7d", dtype=Float32),
        Field(name="transaction_count_7d",       dtype=Int64),
        Field(name="max_transaction_amount_7d",  dtype=Float32),
        Field(name="days_since_last_transaction", dtype=Int64),
        Field(name="preferred_merchant_category", dtype=String),
    ],
    online=True,                        # ✅ enable online store
    source=transactions_source,
)
```

```python
# Materialise to online store (run on schedule)
from feast import FeatureStore
from datetime import datetime

store = FeatureStore(repo_path="feature_store/feature_repo/")

# Materialise last 7 days
store.materialize_incremental(end_date=datetime.utcnow())

# Training — get historical features (offline)
entity_df = pd.DataFrame({
    "customer_id": [1001, 1002, 1003],
    "event_timestamp": pd.to_datetime(["2026-03-29", "2026-03-29", "2026-03-29"]),
})

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "customer_transaction_stats:avg_transaction_amount_7d",
        "customer_transaction_stats:transaction_count_7d",
        "customer_transaction_stats:days_since_last_transaction",
    ],
).to_df()

# Serving — get online features (low-latency)
features = store.get_online_features(
    features=[
        "customer_transaction_stats:avg_transaction_amount_7d",
        "customer_transaction_stats:transaction_count_7d",
    ],
    entity_rows=[{"customer_id": 1001}, {"customer_id": 1002}],
).to_dict()
```

---

### AWS Feature Store — Python Examples

```python
import boto3
import sagemaker
from sagemaker.feature_store.feature_group import FeatureGroup
from sagemaker.feature_store.feature_definition import (
    FeatureDefinition, FeatureTypeEnum,
)

sagemaker_session = sagemaker.Session()
role = "arn:aws:iam::123456789012:role/SageMakerExecutionRole"

# Define Feature Group
customer_feature_group = FeatureGroup(
    name="customer-transaction-stats",
    sagemaker_session=sagemaker_session,
)

# Define features
customer_feature_group.feature_definitions = [
    FeatureDefinition(feature_name="customer_id",                  feature_type=FeatureTypeEnum.INTEGRAL),
    FeatureDefinition(feature_name="avg_transaction_amount_7d",    feature_type=FeatureTypeEnum.FRACTIONAL),
    FeatureDefinition(feature_name="transaction_count_7d",         feature_type=FeatureTypeEnum.INTEGRAL),
    FeatureDefinition(feature_name="max_transaction_amount_7d",    feature_type=FeatureTypeEnum.FRACTIONAL),
    FeatureDefinition(feature_name="days_since_last_transaction",  feature_type=FeatureTypeEnum.INTEGRAL),
    FeatureDefinition(feature_name="event_time",                   feature_type=FeatureTypeEnum.FRACTIONAL),
]

# Create Feature Group
customer_feature_group.create(
    s3_uri=f"s3://my-bucket/feature-store/",
    record_identifier_name="customer_id",
    event_time_feature_name="event_time",
    role_arn=role,
    enable_online_store=True,    # ✅ low-latency online store
    description="Customer transaction statistics for fraud detection",
    tags=[
        {"Key": "team", "Value": "ml"},
        {"Key": "env", "Value": "prod"},
    ],
)

# Ingest features
import time
records = [
    {
        "customer_id": 1001,
        "avg_transaction_amount_7d": 125.50,
        "transaction_count_7d": 8,
        "max_transaction_amount_7d": 499.99,
        "days_since_last_transaction": 0,
        "event_time": time.time(),
    }
]
customer_feature_group.ingest(data_frame=pd.DataFrame(records), max_workers=4, wait=True)

# Online serving — get record (p99 < 10ms)
sm_featurestore_runtime = boto3.client("sagemaker-featurestore-runtime")

response = sm_featurestore_runtime.get_record(
    FeatureGroupName="customer-transaction-stats",
    RecordIdentifierValueAsString="1001",
    FeatureNames=["avg_transaction_amount_7d", "transaction_count_7d"],
)
features = {r["FeatureName"]: r["ValueAsString"] for r in response["Record"]}

# Offline store — Athena query for training
import awswrangler as wr

training_df = wr.athena.read_sql_query(
    sql="""
        SELECT customer_id, avg_transaction_amount_7d, transaction_count_7d
        FROM "sagemaker_featurestore"."customer_transaction_stats_1234567890"
        WHERE event_time > to_unixtime(date_add('day', -30, now()))
    """,
    database="sagemaker_featurestore",
    workgroup="primary",
)
```

## Model Drift Detection — Evidently AI

```python
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset, ClassificationPreset
from evidently.test_suite import TestSuite
from evidently.tests import (
    TestShareOfMissingValues,
    TestColumnDrift,
    TestPrecisionScore,
    TestRecallScore,
)
import pandas as pd

# Reference data (training distribution)
reference_data = pd.read_parquet("data/processed/train.parquet")

# Current production data (last 7 days)
current_data = pd.read_parquet("data/production/last_7d.parquet")

# Data drift report
drift_report = Report(metrics=[DataDriftPreset()])
drift_report.run(reference_data=reference_data, current_data=current_data)
drift_report.save_html("reports/drift_report.html")

# Automated test suite
test_suite = TestSuite(tests=[
    TestShareOfMissingValues(lt=0.05),          # < 5% missing
    TestColumnDrift(column_name="amount"),       # no distribution drift
    TestColumnDrift(column_name="merchant_cat"),
    TestPrecisionScore(gte=0.90),               # precision >= 90%
    TestRecallScore(gte=0.85),                  # recall >= 85%
])
test_suite.run(reference_data=reference_data, current_data=current_data)

# Fail pipeline if drift detected
if not test_suite.as_dict()["summary"]["all_passed"]:
    print("⚠️ Drift detected — triggering retraining pipeline")
    # Trigger SageMaker retraining pipeline
    pipeline.start(parameters={"TriggerReason": "drift-detected"})
```

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| ❌ Features computed differently in training vs serving | Shared Feature Store (Feast / AWS FS) |
| ❌ Data not versioned | DVC with S3 remote — every dataset commit |
| ❌ No experiment tracking | MLflow or W&B from first experiment |
| ❌ `mlruns/` committed to git | Remote tracking server + `.gitignore` |
| ❌ No drift detection | Evidently AI on schedule; alert on threshold breach |
| ❌ Direct prod deploy without canary | Seldon canary split; SageMaker shadow mode |
| ❌ Retraining triggered manually | Automate via drift detection → SageMaker Pipeline |
| ❌ Hardcoded feature computation | Feature Store materialisation job — single source of truth |


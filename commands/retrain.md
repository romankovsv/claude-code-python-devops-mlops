---
description: Trigger and manage a model retraining run — drift-triggered, scheduled, or manual. Generates retraining pipeline invocation, monitors training, and gates promotion. Invokes mlops-engineer agent.
---

# /retrain

Invokes the **mlops-engineer** agent to trigger and monitor a model retraining run.

## What This Command Does

1. **Trigger Analysis** — why is retraining needed? (drift / schedule / manual / accuracy drop)
2. **Data Preparation** — fetch latest data, run `/data-validate` gate, DVC snapshot
3. **Launch Pipeline** — SageMaker Pipeline / Kubeflow / DVC repro
4. **Monitor Run** — track metrics live, compare to baseline
5. **Promotion Gate** — if metrics pass → auto-tag registry → hand off to `/model-promote`

## When to Use

- `/model-health` detected drift above threshold
- Accuracy dropped > threshold in production
- Scheduled weekly/monthly retraining
- New labelled data available
- Upstream feature logic changed

## Required Context (provide when running)

```
Model:            [model name / endpoint]
Trigger reason:   [drift / accuracy-drop / scheduled / new-data / manual]
Pipeline type:    [SageMaker / Kubeflow / DVC / other]
New data:         [S3 path / DVC tag / date range]
Baseline version: [current champion version to compare against]
```

## Trigger Types & Auto-Detection

```python
# Evidently drift-triggered retraining
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset

DRIFT_THRESHOLD = 0.30
ACCURACY_DROP_THRESHOLD = 0.03   # 3pp drop triggers retraining

def should_retrain(
    reference_df: pd.DataFrame,
    current_df: pd.DataFrame,
    current_auc: float,
    baseline_auc: float,
) -> tuple[bool, str]:

    # Check 1: data drift
    report = Report(metrics=[DataDriftPreset(drift_share_threshold=DRIFT_THRESHOLD)])
    report.run(reference_data=reference_df, current_data=current_df)
    drift_share = report.as_dict()["metrics"][0]["result"]["share_of_drifted_columns"]

    if drift_share > DRIFT_THRESHOLD:
        return True, f"Data drift: {drift_share:.0%} columns drifted (threshold: {DRIFT_THRESHOLD:.0%})"

    # Check 2: accuracy drop
    accuracy_drop = baseline_auc - current_auc
    if accuracy_drop > ACCURACY_DROP_THRESHOLD:
        return True, f"Accuracy drop: {accuracy_drop:.3f} (threshold: {ACCURACY_DROP_THRESHOLD})"

    return False, "No retraining needed"

needs_retrain, reason = should_retrain(ref_df, current_df, current_auc=0.891, baseline_auc=0.924)
```

## SageMaker Pipeline — Trigger

```python
import boto3
import json
from datetime import datetime

sm = boto3.client("sagemaker")

def trigger_sagemaker_retraining(
    pipeline_name: str = "fraud-detector-training",
    data_s3_uri: str = "s3://my-bucket/data/fraud/latest/",
    reason: str = "drift-triggered",
) -> str:

    # Snapshot current data version before training
    dvc_tag = f"train-{datetime.utcnow().strftime('%Y%m%d-%H%M')}"
    # (assumes dvc push + git tag already done by data pipeline)

    response = sm.start_pipeline_execution(
        PipelineName=pipeline_name,
        PipelineExecutionDisplayName=f"retrain-{reason}-{datetime.utcnow().date()}",
        PipelineParameters=[
            {"Name": "InputDataS3Uri",  "Value": data_s3_uri},
            {"Name": "DVCDataTag",      "Value": dvc_tag},
            {"Name": "RetrainReason",   "Value": reason},
        ],
        PipelineExecutionDescription=f"Auto-triggered: {reason}",
    )

    execution_arn = response["PipelineExecutionArn"]
    print(f"✅ Pipeline started: {execution_arn}")
    return execution_arn


def wait_for_pipeline(execution_arn: str, poll_interval: int = 60) -> str:
    import time
    while True:
        response = sm.describe_pipeline_execution(PipelineExecutionArn=execution_arn)
        status = response["PipelineExecutionStatus"]
        print(f"  Status: {status}")
        if status in ("Succeeded", "Failed", "Stopped"):
            return status
        time.sleep(poll_interval)


execution_arn = trigger_sagemaker_retraining(reason="drift-triggered")
status = wait_for_pipeline(execution_arn)
print(f"Pipeline finished: {status}")
```

## Kubeflow Pipeline — Trigger

```python
import kfp

client = kfp.Client(host="http://ml-pipeline.kubeflow.svc:8888")

run = client.create_run_from_pipeline_package(
    pipeline_file="fraud_pipeline.yaml",
    arguments={
        "data_path":      "gs://my-bucket/data/fraud/latest/",
        "n_estimators":   200,
        "auc_threshold":  0.90,
    },
    run_name=f"retrain-drift-triggered-{datetime.utcnow().date()}",
    experiment_name="fraud-detection-retraining",
)
print(f"Run ID: {run.run_id}")
print(f"UI: http://kubeflow.internal/#/runs/details/{run.run_id}")
```

## DVC Repro — Trigger

```bash
# Pull latest data
dvc pull

# Run full pipeline (only reruns stages with changed inputs)
dvc repro

# Tag the run
git add dvc.lock
git commit -m "retrain: drift-triggered 2026-03-29"
git tag train-2026-03-29-drift
git push origin main --tags
dvc push

# View metrics
dvc metrics show
dvc plots show
```

## Post-Training Promotion Gate

```python
import mlflow
from mlflow import MlflowClient

def post_training_gate(
    run_id: str,
    model_name: str,
    baseline_alias: str = "champion",
    auc_threshold: float = 0.90,
    min_improvement: float = 0.0,  # must be at least as good as baseline
) -> bool:

    client = MlflowClient()
    run = client.get_run(run_id)
    new_auc = run.data.metrics["eval/auc_roc"]

    # Get baseline
    champion = client.get_model_version_by_alias(model_name, baseline_alias)
    champion_run = client.get_run(champion.run_id)
    baseline_auc = champion_run.data.metrics["eval/auc_roc"]

    print(f"New model AUC:   {new_auc:.4f}")
    print(f"Champion AUC:    {baseline_auc:.4f}")
    print(f"Delta:           {new_auc - baseline_auc:+.4f}")

    if new_auc < auc_threshold:
        print(f"❌ GATE FAILED: AUC {new_auc:.4f} below minimum {auc_threshold}")
        return False

    if new_auc < baseline_auc + min_improvement:
        print(f"❌ GATE FAILED: no improvement over champion (delta={new_auc-baseline_auc:+.4f})")
        return False

    # Register as challenger
    result = mlflow.register_model(f"runs:/{run_id}/model", model_name)
    client.set_registered_model_alias(model_name, "challenger", result.version)
    print(f"✅ Registered as v{result.version} (challenger) → ready for /model-promote")
    return True
```

## Automated Retraining — EventBridge + Lambda (AWS)

```python
# EventBridge rule: trigger weekly + on drift alarm
# terraform/modules/retraining-trigger/main.tf

resource "aws_cloudwatch_event_rule" "weekly_retrain" {
  name                = "fraud-detector-weekly-retrain"
  schedule_expression = "cron(0 2 ? * MON *)"  # Monday 02:00 UTC
}

resource "aws_cloudwatch_event_rule" "drift_retrain" {
  name        = "fraud-detector-drift-triggered"
  event_pattern = jsonencode({
    source      = ["custom.mlops"]
    detail-type = ["ModelDriftDetected"]
    detail      = { model = ["fraud-detector"], severity = ["HIGH"] }
  })
}

resource "aws_lambda_function" "retrain_trigger" {
  function_name = "fraud-detector-retrain-trigger"
  handler       = "handler.lambda_handler"
  runtime       = "python3.11"
  # Calls sm.start_pipeline_execution()
}
```

## Output Format

```
## Retraining Report — fraud-detector — 2026-03-29

### Trigger
Type: drift-triggered
Reason: days_since_last_tx PSI=0.31 > threshold 0.30

### Data
Input:    s3://my-bucket/data/fraud/2026-03-29/
DVC tag:  train-20260329-0200
Rows:     1,842,310 (previous: 1,791,024 +2.9%)
Period:   2026-02-27 → 2026-03-28

### Data Validation Gate
✅ Schema: all columns valid
✅ Drift vs training baseline: PSI=0.31 on days_since_last_tx — REASON for retraining
✅ Missing values: 0.2% across all features

### Training Run
Platform:   SageMaker Pipeline
Duration:   47 min (Spot ml.p3.2xlarge)
Run ID:     abc123def456
Cost:       ~$1.84

### Metrics vs Champion (v10)
AUC-ROC:  0.934 vs 0.931 (+0.003) ✅
F1:       0.886 vs 0.882 (+0.004) ✅
Precision: 0.901 vs 0.897        ✅

### Promotion Gate
✅ AUC 0.934 >= threshold 0.900
✅ Improvement over champion: +0.003
✅ Registered as fraud-detector v11 (challenger)

### Next Step
→ Run /model-promote fraud-detector v11
  or: /ab-test to compare v10 vs v11 before full promotion
```


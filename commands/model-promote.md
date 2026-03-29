---
description: Safely promote a model from staging to production — canary deploy, rollback plan, monitoring checks, and approval gate. Invokes mlops-engineer agent.
---

# /model-promote

Invokes the **mlops-engineer** agent to safely promote a model from staging to production.

## What This Command Does

1. **Pre-Promotion Checklist** — model card, metrics gate, load test, rollback plan
2. **Canary Deploy** — route X% traffic to new model, monitor for 24h
3. **Shadow Mode Option** — run new model in parallel with no real traffic impact
4. **Metrics Gate** — auto-rollback if latency/error/drift threshold breached
5. **Full Promotion** — 0% → 10% canary → 50% → 100% with checkpoints
6. **Rollback Procedure** — documented and tested before promotion starts

## When to Use

- Promoting a retrained model to replace the current production model
- Deploying a new model version after experiment review passed
- Running A/B test between two model versions
- Emergency rollback after production incident

## Required Context (provide when running)

```
Model name:          [mlflow model name / seldon deployment]
New version:         [v1.2.0 / run_id / SHA]
Current production:  [version currently serving 100% traffic]
Serving platform:    [Seldon Core / SageMaker Endpoint / BentoML]
Canary percentage:   [10% recommended, or specify]
Monitoring window:   [24h recommended, or specify]
```

## Pre-Promotion Checklist (must all pass)

### Model Quality Gate
- [ ] `eval/auc_roc` >= baseline + 0% (no regression)
- [ ] `eval/f1` >= baseline - 2% (small tolerance for minor trade-offs)
- [ ] No data leakage confirmed (train/test split reviewed)
- [ ] Model card filled (description, input/output schema, limitations)
- [ ] Load test passed: p99 latency < SLO at 2× expected peak QPS

### Reproducibility Gate
- [ ] `git_sha` logged in MLflow/W&B run
- [ ] Dataset DVC version tagged
- [ ] Training script version matches run artifact

### Rollback Ready
- [ ] Previous model version still available in registry (not deleted)
- [ ] Rollback command documented and tested in staging
- [ ] On-call engineer notified of promotion window

## Canary Deploy — Seldon Core

```yaml
# values-prod.yaml — 10% canary
predictors:
  - name: current
    replicas: 3
    traffic: 90
    graph:
      name: fraud-detector
      modelUri: s3://models/fraud-detector/v1.1.0

  - name: canary
    replicas: 1
    traffic: 10
    graph:
      name: fraud-detector
      modelUri: s3://models/fraud-detector/v1.2.0
```

```bash
# Apply canary
helm upgrade fraud-detector ./chart -f values-prod.yaml -n ml-serving

# Monitor canary (watch for 24h)
kubectl get --watch seldondeployment fraud-detector -n ml-serving

# Promote to 100% (after passing monitoring window)
# Edit values-prod.yaml: traffic: 100 on new version, remove old predictor
helm upgrade fraud-detector ./chart -f values-prod.yaml -n ml-serving

# Emergency rollback
helm rollback fraud-detector -n ml-serving
```

## Canary Deploy — SageMaker Endpoint

```python
import boto3

sm = boto3.client("sagemaker")

# Update endpoint with canary split
sm.update_endpoint(
    EndpointName="fraud-detector-prod",
    DeploymentConfig={
        "BlueGreenUpdatePolicy": {
            "TrafficRoutingConfiguration": {
                "Type": "CANARY",
                "CanarySize": {"Type": "CAPACITY_PERCENT", "Value": 10},
                "WaitIntervalInSeconds": 86400,  # 24h canary window
            },
            "MaximumExecutionTimeoutInSeconds": 21600,
        }
    },
)

# Rollback if needed
sm.update_endpoint(
    EndpointName="fraud-detector-prod",
    EndpointConfigName="fraud-detector-prod-v1.1.0",  # previous config
)
```

## Automated Rollback Triggers

Set these CloudWatch alarms — auto-rollback if breached during canary:

```python
# Error rate > 1%
sm.put_metric_alarm(
    AlarmName="fraud-detector-canary-error-rate",
    Namespace="AWS/SageMaker",
    MetricName="ModelError",
    Threshold=0.01,
    ComparisonOperator="GreaterThanThreshold",
    EvaluationPeriods=3,
    Period=300,
    AlarmActions=["arn:aws:sns:...rollback-topic"],
)

# Latency p99 > 200ms
sm.put_metric_alarm(
    AlarmName="fraud-detector-canary-latency",
    Namespace="AWS/SageMaker",
    MetricName="ModelLatency",
    ExtendedStatistic="p99",
    Threshold=200,
    ComparisonOperator="GreaterThanThreshold",
    EvaluationPeriods=3,
    Period=300,
    AlarmActions=["arn:aws:sns:...rollback-topic"],
)
```

## Output Format

```
## Model Promotion Plan — [model_name] v[X] → v[Y]

### Pre-Promotion Checklist
✅ Metrics gate passed: AUC-ROC 0.924 → 0.931 (+0.8%)
✅ Load test passed: p99 87ms at 500 QPS (SLO: <200ms)
✅ Rollback confirmed: v1.1.0 available in registry
🟠 Model card incomplete: missing "limitations" section → fill before promoting

### Promotion Schedule
Step 1 (now):     10% canary → monitor 24h
Step 2 (+24h):    50% if metrics OK → monitor 4h
Step 3 (+28h):    100% full promotion
Rollback trigger: error rate > 1% OR p99 > 200ms OR AUC-ROC drop > 3%

### Rollback Command
helm rollback fraud-detector -n ml-serving
# OR
aws sagemaker update-endpoint --endpoint-name fraud-detector-prod \
  --endpoint-config-name fraud-detector-prod-v1.1.0

### On-call
Notify: @ml-oncall on Slack before starting canary
Window: avoid Friday 16:00+ and weekends
```


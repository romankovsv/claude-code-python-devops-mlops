---
description: Check model health in production — prediction latency, data drift, feature drift, error rate, and retraining triggers. Invokes mlops-engineer agent with Evidently AI / AWS Model Monitor analysis.
---

# /model-health

Invokes the **mlops-engineer** agent to run a production model health check.

## What This Command Does

1. **Prediction Latency** — p50/p95/p99 vs SLO threshold
2. **Data Quality** — missing values, out-of-range inputs, schema violations
3. **Feature Drift** — compare current input distribution vs training baseline (Evidently)
4. **Target Drift / Concept Drift** — prediction distribution shift over time
5. **Model Performance** — precision/recall/AUC if ground truth available
6. **Retraining Trigger** — should retraining be kicked off now?

## When to Use

- Weekly or monthly model health review
- After a significant traffic spike or data pipeline change
- After a product or business rule change that might affect input distribution
- When model accuracy complaints come from the business
- As part of on-call runbook for ML services

## Required Context (provide when running)

```
Model name / endpoint:    [seldon deployment name / sagemaker endpoint]
Reference dataset:        [S3 path or DVC tag of training data]
Current window:           [last 7 days / last 30 days / custom date range]
Ground truth available:   [yes / no / delayed (specify lag)]
Drift threshold:          [default 0.1 PSI / Jensen-Shannon / custom]
```

## What Gets Checked

### Latency (CloudWatch / Prometheus)
```bash
# SageMaker endpoint
aws cloudwatch get-metric-statistics \
  --namespace AWS/SageMaker \
  --metric-name ModelLatency \
  --dimensions Name=EndpointName,Value=<endpoint> Name=VariantName,Value=AllTraffic \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics p50 p95 p99

# Seldon / Prometheus
rate(seldon_api_executor_client_requests_seconds_bucket[5m])
```

### Data Drift (Evidently)
```python
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset

report = Report(metrics=[DataDriftPreset()])
report.run(reference_data=reference_df, current_data=current_df)
# Drift detected if share_of_drifted_columns > 0.3
```

### Retraining Decision Matrix

| Condition | Action |
|-----------|--------|
| Drift score > 0.3 on key features | 🔴 Trigger retraining immediately |
| Precision/Recall drop > 5% from baseline | 🔴 Trigger retraining immediately |
| Drift score 0.1–0.3 | 🟠 Schedule retraining within 1 week |
| Missing values > 5% on any feature | 🟠 Investigate data pipeline |
| Latency p99 > 2× SLO | 🟠 Scale endpoint / investigate |
| All metrics within bounds | ✅ Healthy — next check in 7 days |

## Output Format

```
## Model Health Report — [model_name] — [date range]

### 📊 Prediction Latency
p50: 45ms  ✅ (SLO: <100ms)
p95: 89ms  ✅ (SLO: <200ms)
p99: 210ms ⚠️  (SLO: <200ms) — investigate slow tail

### 📉 Data Drift (Evidently — last 7 days vs training)
amount:              PSI=0.18 🟠 moderate drift
merchant_category:   PSI=0.04 ✅ stable
days_since_last_tx:  PSI=0.31 🔴 HIGH DRIFT
Overall drift score: 0.28 (threshold: 0.10)

### 🎯 Model Performance (ground truth available)
AUC-ROC: 0.891 (baseline: 0.924) 🔴 -3.6% — exceeds 3% threshold

### 🔔 Retraining Decision
🔴 TRIGGER RETRAINING — 2 critical conditions met:
  1. days_since_last_tx drift PSI=0.31 > 0.30 threshold
  2. AUC-ROC dropped 3.6% from baseline

### Next Actions
1. kubectl create job retrain-fraud-model --from=cronjob/fraud-model-retrain
   (or trigger SageMaker Pipeline manually)
2. Investigate data pipeline for days_since_last_tx — possible upstream change
3. After retraining: canary 10% → monitor 24h → promote to 100%
```


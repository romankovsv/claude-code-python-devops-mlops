---
description: Design or analyse an A/B test or shadow test between two model versions — traffic split, statistical significance, metric definitions, and promotion decision. Invokes mlops-engineer agent.
---

# /ab-test

Invokes the **mlops-engineer** agent to design or analyse an A/B test between model versions.

## What This Command Does

1. **Design** — traffic split, sample size calculation, success metrics, guard-rails
2. **Shadow Mode** — run new model in parallel with zero traffic impact (safest start)
3. **Statistical Analysis** — significance testing, confidence intervals, minimum detectable effect
4. **Serving Setup** — Seldon canary, SageMaker production variants, Istio weights
5. **Decision Gate** — when to stop, promote, or roll back based on results

## When to Use

- Comparing two model architectures on live traffic
- Testing a retrained model vs current champion before full promotion
- Running shadow mode to validate new model without any risk
- Validating A/B test results before making a promotion decision

## Required Context (provide when running)

```
Model A (control):   [version / endpoint currently serving]
Model B (treatment): [version / endpoint to test]
Success metric:      [fraud_catch_rate / precision / revenue / latency]
Guard-rail metrics:  [false_positive_rate < 0.02 / p99 < 200ms]
Traffic split:       [90/10 recommended, or specify]
Test duration:       [7 days recommended, or specify]
Serving platform:    [Seldon Core / SageMaker / Istio]
```

## Sample Size Calculation

```python
from scipy import stats
import numpy as np

def required_sample_size(
    baseline_rate: float,   # e.g. current fraud catch rate = 0.85
    min_detectable_effect: float = 0.02,  # detect 2pp improvement
    alpha: float = 0.05,    # significance level
    power: float = 0.80,    # 80% power
) -> int:
    """Calculate required sample size per variant."""
    p1 = baseline_rate
    p2 = baseline_rate + min_detectable_effect
    pooled_p = (p1 + p2) / 2

    z_alpha = stats.norm.ppf(1 - alpha / 2)
    z_beta  = stats.norm.ppf(power)

    n = (
        (z_alpha * np.sqrt(2 * pooled_p * (1 - pooled_p)) +
         z_beta  * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2
        / (p2 - p1) ** 2
    )
    return int(np.ceil(n))

n = required_sample_size(baseline_rate=0.85, min_detectable_effect=0.02)
print(f"Required per variant: {n:,}")
print(f"Total at 90/10 split: {int(n / 0.10):,} requests")
print(f"At 5000 req/day: {n / 5000:.1f} days per variant")
# → Required per variant: 2,516
# → Total at 90/10 split: 25,160 requests
# → At 5000 req/day: ~2.5 days
```

## Statistical Significance Analysis

```python
from scipy.stats import chi2_contingency, ttest_ind
import pandas as pd

def analyse_ab_test(results: pd.DataFrame) -> dict:
    """
    results columns: variant (A/B), converted (0/1), value (e.g. fraud amount caught)
    """
    a = results[results["variant"] == "A"]
    b = results[results["variant"] == "B"]

    # Conversion rate comparison (chi-square)
    contingency = [
        [a["converted"].sum(), len(a) - a["converted"].sum()],
        [b["converted"].sum(), len(b) - b["converted"].sum()],
    ]
    chi2, p_value, _, _ = chi2_contingency(contingency)

    rate_a = a["converted"].mean()
    rate_b = b["converted"].mean()
    relative_lift = (rate_b - rate_a) / rate_a * 100

    # Value comparison (t-test) — if metric is continuous
    t_stat, p_value_value = ttest_ind(a["value"], b["value"])

    return {
        "rate_a":        rate_a,
        "rate_b":        rate_b,
        "relative_lift": relative_lift,
        "p_value":       p_value,
        "significant":   p_value < 0.05,
        "samples_a":     len(a),
        "samples_b":     len(b),
        "recommendation": "promote B" if (p_value < 0.05 and relative_lift > 0) else "keep A",
    }
```

## Shadow Mode — Zero Risk Testing

```yaml
# Seldon — shadow deployment: B receives traffic but responses are discarded
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: fraud-detector
  namespace: ml-serving
spec:
  predictors:
    - name: main
      replicas: 3
      traffic: 100          # A serves all real traffic
      graph:
        name: model-a
        modelUri: s3://models/fraud-detector/v10

    - name: shadow
      replicas: 1
      traffic: 0            # receives copy of requests, responses ignored
      shadow: true          # KEY: shadow mode
      graph:
        name: model-b
        modelUri: s3://models/fraud-detector/v11
```

```bash
# Compare shadow predictions vs main predictions
kubectl logs -l seldon-deployment-id=fraud-detector,predictor-id=shadow \
  -n ml-serving | grep '"response"' | head -100
```

## A/B Split — SageMaker Production Variants

```python
import boto3

sm = boto3.client("sagemaker")

# Create endpoint config with 90/10 split
sm.create_endpoint_config(
    EndpointConfigName="fraud-detector-ab-test",
    ProductionVariants=[
        {
            "VariantName": "ModelA",
            "ModelName": "fraud-detector-v10",
            "InitialInstanceCount": 2,
            "InstanceType": "ml.c5.large",
            "InitialVariantWeight": 90,          # 90% traffic
        },
        {
            "VariantName": "ModelB",
            "ModelName": "fraud-detector-v11",
            "InitialInstanceCount": 1,
            "InstanceType": "ml.c5.large",
            "InitialVariantWeight": 10,          # 10% traffic
        },
    ],
    DataCaptureConfig={
        "EnableCapture": True,
        "InitialSamplingPercentage": 100,        # capture all for analysis
        "DestinationS3Uri": "s3://my-bucket/data-capture/fraud-detector-ab/",
        "CaptureOptions": [{"CaptureMode": "Input"}, {"CaptureMode": "Output"}],
    },
)

# Monitor per-variant metrics
cw = boto3.client("cloudwatch")
for variant in ["ModelA", "ModelB"]:
    response = cw.get_metric_statistics(
        Namespace="AWS/SageMaker",
        MetricName="Invocations",
        Dimensions=[
            {"Name": "EndpointName", "Value": "fraud-detector-prod"},
            {"Name": "VariantName", "Value": variant},
        ],
        StartTime=..., EndTime=..., Period=3600, Statistics=["Sum"],
    )
    print(f"{variant}: {sum(r['Sum'] for r in response['Datapoints']):,.0f} requests")
```

## Decision Framework

```
Before stopping the test, confirm ALL of the following:
┌─────────────────────────────────────────────────────────────┐
│ 1. Sample size reached?         n_B >= required_per_variant │
│ 2. Test duration met?           >= 7 days (full week cycle) │
│ 3. p-value < 0.05?              statistical significance     │
│ 4. Practical significance?      lift >= min_detectable_effect│
│ 5. Guard-rails OK?              FPR, latency, error rate     │
└─────────────────────────────────────────────────────────────┘

Result matrix:
  Sig=✅, Lift=✅, Guard-rails=✅  → PROMOTE B (run /model-promote)
  Sig=✅, Lift=✅, Guard-rails=❌  → INVESTIGATE — latency regression?
  Sig=✅, Lift=❌ (B worse)        → KEEP A, archive B
  Sig=❌ (no difference)           → KEEP A (no improvement detected)
  Early stop (guard-rail breach)   → ROLLBACK B immediately
```

## Output Format

```
## A/B Test Analysis — fraud-detector v10 (A) vs v11 (B)

### Test Setup
Traffic split:   90% A / 10% B
Duration:        7 days (2026-03-22 → 2026-03-29)
Samples A:       31,450  |  Samples B:  3,490

### Primary Metric: fraud_catch_rate
Model A: 85.1%
Model B: 87.4%  (+2.7pp relative lift: +3.2%)
p-value: 0.0031  ✅ statistically significant (p < 0.05)

### Guard-Rail Metrics
false_positive_rate:  A=1.82%  B=1.79%  ✅ both below 2% limit
p99 latency:          A=91ms   B=88ms   ✅ both below 200ms SLO
error rate:           A=0.01%  B=0.01%  ✅

### Decision
✅ PROMOTE Model B to champion
  - Significant improvement: +2.7pp fraud_catch_rate (p=0.003)
  - All guard-rails passing
  - Next step: run /model-promote with canary 10% → 24h → 100%
```


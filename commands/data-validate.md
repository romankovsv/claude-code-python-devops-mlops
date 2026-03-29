---
description: Validate data quality before training or serving — schema validation (Great Expectations / Pandera), distribution checks, anomaly detection, and pipeline failure gate. Invokes mlops-engineer agent.
---

# /data-validate

Invokes the **mlops-engineer** agent to run data validation before a training run or serving pipeline.

## What This Command Does

1. **Schema Validation** — column names, types, nullability, value ranges
2. **Distribution Checks** — compare current batch vs reference baseline (drift gate)
3. **Anomaly Detection** — sudden cardinality spikes, unexpected nulls, outliers
4. **Pipeline Gate** — fail fast before training if data quality is below threshold
5. **Tool Selection** — Great Expectations vs Pandera vs AWS Deequ vs TFDV

## When to Use

- Before every training run (as first pipeline stage)
- After a data pipeline change or upstream schema migration
- When a model's accuracy suddenly drops (suspect bad data)
- When onboarding a new data source into the Feature Store
- Setting up data contracts between data engineering and ML teams

## Required Context (provide when running)

```
Dataset:          [S3 path / DVC tag / table name]
Reference:        [S3 path of baseline / previous version]
Tool preference:  [Great Expectations / Pandera / auto-select]
Strictness:       [fail-fast / warn-only / custom thresholds]
Pipeline stage:   [before training / before serving / both]
```

## Tool Selection Guide

| Tool | Best For | Integration |
|------|----------|-------------|
| **Pandera** | DataFrame schema in Python code — type annotations | pytest, inline validation |
| **Great Expectations** | Data warehouse, complex suites, data docs | Airflow, Kubeflow, Spark |
| **AWS Deequ** | PySpark on EMR / Glue at scale | AWS Glue, EMR |
| **TFDV** | TensorFlow-based pipelines, TFX | Kubeflow TFX |
| **Evidently** | Distribution drift (train vs current) | Any Python pipeline |

## Pandera — Inline Schema Validation

```python
import pandera as pa
from pandera.typing import DataFrame, Series

class FraudFeatureSchema(pa.DataFrameModel):
    """Schema for fraud detection training data."""

    customer_id:        Series[str]   = pa.Field(nullable=False, unique=True)
    amount:             Series[float] = pa.Field(ge=0.0, le=1_000_000.0, nullable=False)
    merchant_category:  Series[str]   = pa.Field(isin=["retail", "food", "travel", "other"])
    days_since_last_tx: Series[int]   = pa.Field(ge=0, le=3650, nullable=False)
    label:              Series[int]   = pa.Field(isin=[0, 1])

    class Config:
        coerce = True       # auto-cast types where possible
        strict = True       # fail on unexpected columns
        ordered = False

# ✅ Validate at pipeline entry point
@pa.check_types
def preprocess(df: DataFrame[FraudFeatureSchema]) -> pd.DataFrame:
    # schema is validated automatically before this function runs
    return df.dropna()

# Manual validation with detailed error report
try:
    FraudFeatureSchema.validate(df, lazy=True)  # collect ALL errors, not just first
except pa.errors.SchemaErrors as e:
    print(e.failure_cases)  # DataFrame of all failures
    raise
```

## Great Expectations — Expectation Suite

```python
import great_expectations as gx

context = gx.get_context()

# Build expectation suite
suite = context.add_expectation_suite("fraud_training_data")
validator = context.get_validator(
    datasource_name="fraud_s3",
    data_asset_name="training_2026_03",
)

# Schema expectations
validator.expect_column_to_exist("amount")
validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_be_of_type("label", "int64")

# Distribution expectations (use reference stats)
validator.expect_column_mean_to_be_between("amount", min_value=50.0, max_value=500.0)
validator.expect_column_values_to_be_between("amount", min_value=0.0, max_value=1_000_000.0)
validator.expect_column_proportion_of_unique_values_to_be_between(
    "merchant_category", min_value=0.0, max_value=0.01
)

# Label balance check (fraud rate 0.5% – 5%)
validator.expect_column_mean_to_be_between("label", min_value=0.005, max_value=0.05)

# Save suite + run checkpoint
validator.save_expectation_suite()
checkpoint = context.add_checkpoint(
    name="fraud_training_checkpoint",
    validations=[{"batch_request": ..., "expectation_suite_name": "fraud_training_data"}],
)
result = checkpoint.run()

if not result.success:
    raise ValueError(f"Data validation failed: {result.statistics}")
```

## Distribution Drift Gate (Evidently)

```python
from evidently.report import Report
from evidently.metric_preset import DataQualityPreset, DataDriftPreset
from evidently.metrics import DatasetMissingValuesMetric

report = Report(metrics=[
    DataQualityPreset(),
    DataDriftPreset(drift_share_threshold=0.3),   # fail if >30% columns drift
    DatasetMissingValuesMetric(),
])

report.run(
    reference_data=reference_df,   # training baseline
    current_data=current_df,       # today's batch
)

results = report.as_dict()

# Pipeline gate
drift_share = results["metrics"][1]["result"]["share_of_drifted_columns"]
missing_share = results["metrics"][2]["result"]["share_of_missing_values"]

if drift_share > 0.30:
    raise ValueError(f"Data drift too high: {drift_share:.0%} columns drifted")
if missing_share > 0.05:
    raise ValueError(f"Too many missing values: {missing_share:.0%}")

print(f"✅ Data quality OK — drift: {drift_share:.0%}, missing: {missing_share:.0%}")
```

## Kubeflow Pipeline Integration

```python
from kfp import dsl
from kfp.dsl import Dataset, Input, Output

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandera==0.18.0", "evidently==0.4.28", "pandas==2.2.0"],
)
def validate_data(
    raw_data: Input[Dataset],
    reference_data: Input[Dataset],
    validation_report: Output[Dataset],
    drift_threshold: float = 0.30,
    missing_threshold: float = 0.05,
) -> None:
    import pandas as pd
    import pandera as pa
    from evidently.report import Report
    from evidently.metric_preset import DataDriftPreset

    df = pd.read_parquet(raw_data.path)
    ref_df = pd.read_parquet(reference_data.path)

    # 1. Schema check
    FraudFeatureSchema.validate(df, lazy=True)  # raises on failure → stops pipeline

    # 2. Drift check
    report = Report(metrics=[DataDriftPreset(drift_share_threshold=drift_threshold)])
    report.run(reference_data=ref_df, current_data=df)
    report.save_html(f"{validation_report.path}/report.html")

    drift_share = report.as_dict()["metrics"][0]["result"]["share_of_drifted_columns"]
    if drift_share > drift_threshold:
        raise ValueError(f"Drift {drift_share:.0%} > threshold {drift_threshold:.0%}")
```

## Output Format

```
## Data Validation Report — [dataset] — [date]

### Schema
✅ All 12 columns present with correct types
🔴 FAIL: 2,341 rows (0.8%) have amount < 0 — schema violation

### Distribution (vs reference 2026-02-28)
amount:              PSI=0.04 ✅ stable
merchant_category:   new value "crypto" not in allowed set 🔴
days_since_last_tx:  PSI=0.28 🟠 moderate drift
label (fraud rate):  0.023 ✅ within expected 0.005–0.05

### Missing Values
customer_id:  0.0%  ✅
amount:       0.0%  ✅
merchant_category: 1.2% 🟠 above 1% warning threshold

### Verdict
🔴 FAIL — pipeline should NOT proceed:
  1. Fix negative amount values in upstream data pipeline
  2. Add "crypto" to allowed merchant_category values (or investigate if error)
```


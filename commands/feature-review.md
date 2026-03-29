---
description: Review feature engineering code and Feature Store setup — training-serving skew, feature freshness, materialisation schedule, and online/offline parity. Invokes mlops-engineer agent.
---

# /feature-review

Invokes the **mlops-engineer** agent to review feature engineering and Feature Store configuration.

## What This Command Does

1. **Training-Serving Skew Audit** — is feature computation identical in training and serving?
2. **Feature Freshness** — TTL, materialisation schedule, staleness risk
3. **Online/Offline Parity** — same features available in both stores with matching logic
4. **Data Quality** — nulls, outliers, cardinality, expected value ranges
5. **Feature Store Config Review** — Feast repo or AWS Feature Group definition
6. **Schema Validation** — input schema matches model expectation

## When to Use

- Before launching a new model to production
- After changing feature computation logic
- When model predictions seem inconsistent between training and serving
- When setting up a Feature Store for the first time
- During quarterly feature health review

## Required Context (provide when running)

```
Feature store type:     [Feast / AWS Feature Store / custom]
Feature names:          [list key features or point to feature_repo/]
Model using features:   [model name / endpoint]
Last materialisation:   [datetime or "unknown"]
```

## Checklist Applied

### Training-Serving Skew (CRITICAL — most common production ML bug)
- [ ] Feature computation logic in training script == materialisation job (single source)
- [ ] No manual transformations applied after `get_online_features()` in serving
- [ ] Timestamp handling consistent (same timezone, same truncation)
- [ ] Categorical encoding consistent (same vocabulary / encoder artifact used)
- [ ] Null handling identical in training and serving (same imputation strategy)

### Feature Freshness (HIGH)
- [ ] TTL set and documented for each feature view
- [ ] Materialisation schedule covers expected serving frequency
- [ ] Monitoring alert set if materialisation job fails or is delayed
- [ ] Staleness acceptable for model use case (real-time vs batch)

### Online/Offline Parity (HIGH)
- [ ] Same feature names and types in online and offline store
- [ ] `get_historical_features()` and `get_online_features()` return same schema
- [ ] No features computed differently for offline training vs online serving

### Schema (MEDIUM)
- [ ] All features have `dtype` defined (Float32, Int64, String)
- [ ] Features have valid `description` fields
- [ ] Entity join keys documented and consistent

## Output Format

```
## Feature Review — [feature_group / feature_view name]

### 🔴 Training-Serving Skew (CRITICAL)
- `amount_log_transform` applied in training script but NOT in serving code
  → Fix: move transformation to materialisation job or Feature Store transform
- `merchant_category` encoded with LabelEncoder saved locally — version mismatch risk
  → Fix: save encoder as DVC artifact; load same artifact in serving

### 🟠 Freshness Issues (HIGH)
- `days_since_last_transaction` TTL = 7 days but materialisation runs weekly
  → Risk: stale features up to 14 days if job is delayed
  → Fix: run materialisation daily; set alert if lag > 25h

### 🟡 Schema Issues (MEDIUM)
- 3 features missing `description` fields
- `customer_age` defined as Float32 but model expects Int64

### ✅ Passing
- Online/offline feature names match (all 12 features)
- Entity join key `customer_id` consistent across all feature views
- TTL set on all feature views
```


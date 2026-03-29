---
description: Review an existing ML experiment or training run — metrics quality, reproducibility, artifact logging, hyperparameter coverage, and model card completeness. Invokes mlops-engineer agent.
---

# /experiment-review

Invokes the **mlops-engineer** agent to review ML experiments for reproducibility, completeness, and production-readiness.

## What This Command Does

1. **Reproducibility Audit** — pinned deps, git SHA in run, data version logged
2. **Tracking Completeness** — all params, metrics, artifacts logged to MLflow/W&B
3. **Hyperparameter Coverage** — search space reasonable, enough trials
4. **Model Card Review** — input schema, output schema, bias evaluation, limitations
5. **Production Readiness Gate** — is this run safe to promote to staging?

## When to Use

- Before promoting a model from experiment to staging
- Reviewing a teammate's training run before sign-off
- Auditing experiment hygiene on a new ML project
- Checking if a run is reproducible on a fresh machine

## Required Context (provide when running)

```
MLflow run ID / W&B run URL: <id or url>
Model type:                  [GBM / NN / transformer / other]
Task:                        [classification / regression / ranking / NLP]
Dataset version:             [DVC tag / S3 path / date]
```

## Checklist Applied

### Reproducibility (CRITICAL — must pass to promote)
- [ ] `git_sha` or `mlflow.source.git.commit` logged in run
- [ ] Dataset version (DVC tag / S3 URI) logged as param or tag
- [ ] All library versions pinned (`requirements.txt` or `conda.yaml` attached to run)
- [ ] Random seeds fixed and logged (`random_seed` param)
- [ ] Run can be reproduced from scratch with `dvc repro` or `mlflow run`

### Tracking Completeness (HIGH)
- [ ] All hyperparameters logged (not just final values — full config)
- [ ] Training metrics logged per epoch/step (not just final)
- [ ] Evaluation metrics on held-out test set logged
- [ ] Model artifact logged with `input_example` and `signature`
- [ ] Feature importance or SHAP values logged (if applicable)
- [ ] Confusion matrix / ROC curve logged as artifact

### Model Card (HIGH — required for production)
- [ ] `description` field filled (what the model does, training data summary)
- [ ] `input_schema` defined (feature names, types, expected ranges)
- [ ] `output_schema` defined (prediction format, confidence range)
- [ ] Known limitations documented
- [ ] Bias evaluation results logged (if applicable)

### Production Gate (evaluate before promotion)
- [ ] eval/auc_roc >= baseline (or defined threshold)
- [ ] No data leakage detected (train/test split verified)
- [ ] Training-serving skew risk assessed (feature computation matches Feature Store)
- [ ] Model size acceptable for serving latency SLO

## Output Format

```
## Experiment Review — [run_id / model_name]

### Reproducibility
🔴 FAIL: git_sha not logged — run cannot be traced to code version
✅ PASS: dataset DVC tag v1.4.2 logged
✅ PASS: requirements.txt attached to run

### Tracking Completeness
🟠 WARN: input_example missing from logged model — serving schema unknown
✅ PASS: all 12 hyperparameters logged
✅ PASS: ROC curve logged as artifact

### Model Card
🟠 WARN: no description field — add before promotion

### Metrics vs Baseline
AUC-ROC: 0.924 (baseline: 0.901) ✅ +2.5%
F1:      0.881 (baseline: 0.863) ✅ +2.1%

### Verdict
⚠️  NOT READY for staging — fix CRITICAL items first:
1. Re-run with git SHA logging enabled
2. Add input_example to mlflow.sklearn.log_model()
```


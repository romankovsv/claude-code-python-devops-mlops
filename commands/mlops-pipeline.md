---
description: Design or review an end-to-end ML pipeline — data versioning (DVC), training (SageMaker), experiment tracking (MLflow/W&B), feature store, model registry, and serving (Seldon). Invokes mlops-engineer agent. WAIT for user CONFIRM before generating any code.
---

# /mlops-pipeline

Invokes the **mlops-engineer** agent to design or review an end-to-end ML pipeline.

## What This Command Does

1. **Gather Requirements** — workload type, data volume, retraining frequency, latency SLO
2. **Propose Pipeline Architecture** — preprocessing → training → evaluation → registry → serving
3. **Tool Selection** — DVC vs no DVC, MLflow vs W&B, Feast vs AWS Feature Store, Seldon vs SageMaker Endpoint
4. **Wait for Confirmation** — MUST receive user approval before generating any code
5. **Generate Pipeline Code** — SageMaker Pipeline SDK / Kubeflow, DVC yaml, tracking setup

## When to Use

- Designing a new ML pipeline from scratch
- Migrating a notebook to a production pipeline
- Choosing between MLflow and W&B for experiment tracking
- Setting up DVC for data and pipeline versioning
- Deciding between Feast and AWS Feature Store

## Required Context (provide when running)

```
Use case:         [fraud detection / recommendation / forecasting / NLP / CV / other]
Data size:        [GB/TB, rows/day]
Retraining:       [daily / weekly / on-drift / manual]
Serving latency:  [p99 < Xms / batch / stream]
Cloud:            [AWS / GCP / Azure / on-prem]
Existing stack:   [what's already in use]
```

## Output Format

```
## ML Pipeline Design — [Use Case]

### Architecture
preprocessing → training → evaluation → [approve] → registry → serving
     DVC           SageMaker    Evidently    human        MLflow       Seldon

### Tool Decisions
| Layer              | Tool                | Reason |
|--------------------|---------------------|--------|
| Data versioning    | DVC + S3            | ...    |
| Experiment tracking| MLflow / W&B        | ...    |
| Feature store      | Feast / AWS FS      | ...    |
| Model serving      | Seldon / SM Endpoint| ...    |

### Pipeline Stages
1. Preprocess  — instance type, input/output, SLA
2. Train       — instance type, Spot?, checkpointing
3. Evaluate    — metrics threshold for promotion
4. Register    — staging gate, model card
5. Serve       — replicas, HPA, canary split

### Cost Estimate
Training (Spot p3.2xlarge, 2h, weekly): ~$X/month

### Next Steps (after your approval)
1. dvc.yaml pipeline definition
2. SageMaker Pipeline SDK code
3. MLflow tracking integration
4. Seldon/Endpoint deployment manifest
```

**CRITICAL**: No code is generated until you explicitly confirm the pipeline design.


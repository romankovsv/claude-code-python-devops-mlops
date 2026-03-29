---
description: Debug a failing Kubernetes pod or service — CrashLoopBackOff, OOMKilled, Pending, networking issues. Invokes k8s-debugger agent with systematic diagnosis.
---

# /k8s-debug

Invokes the **k8s-debugger** agent to systematically diagnose and fix Kubernetes issues.

## What This Command Does

1. **Identify Failure Type** — CrashLoopBackOff / OOMKilled / Pending / NetworkError
2. **Collect Evidence** — logs, events, pod description, resource usage
3. **Root Cause Analysis** — pinpoint exact cause with supporting evidence
4. **Remediation Steps** — concrete kubectl commands to fix the issue
5. **Prevention** — what to change to prevent recurrence

## When to Use

- Pod in CrashLoopBackOff
- Pod stuck in Pending state
- OOMKilled errors
- Service unreachable / DNS not resolving
- HPA not scaling
- Istio / service mesh connectivity issues

## Required Context (provide when running)

```
Namespace: <namespace>
Pod/Deployment name: <name>
Symptom: <what you observe>
Last working state: <when it last worked>
Recent changes: <what changed before it broke>
```

## Output Format

```
## K8s Debug Report: <resource> in <namespace>

### Diagnosis
- **Failure type**: CrashLoopBackOff
- **Root cause**: [identified cause]
- **Evidence**: [log lines / describe output]

### Immediate Fix
[kubectl commands to run now]

### Verification
[commands to confirm fix worked]

### Prevention
[what to change in manifests/Helm values]
```


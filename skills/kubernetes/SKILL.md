---
name: kubernetes
description: Kubernetes best practices — liveness/readiness/startup probes, HPA/VPA, RBAC, Helm, NetworkPolicy, PodDisruptionBudget, anti-patterns, and production hardening.
origin: custom
---

# Kubernetes Patterns

Production Kubernetes best practices for application deployment and operations.

## When to Activate

- Writing or reviewing Kubernetes manifests
- Setting up Helm charts for applications
- Configuring autoscaling (HPA/VPA)
- Debugging pod failures (CrashLoopBackOff, OOMKilled, Pending)
- Designing RBAC and NetworkPolicy

## Probes — Liveness, Readiness, Startup

```yaml
containers:
  - name: app
    image: my-registry/app:v1.2.0    # ✅ never :latest
    ports:
      - containerPort: 8000

    # Startup probe — gives slow-starting apps time to boot
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8000
      failureThreshold: 30           # 30 * 10s = 5 min max startup
      periodSeconds: 10

    # Readiness — traffic only when ready
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8000
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
      successThreshold: 1
      timeoutSeconds: 5

    # Liveness — restart if stuck
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8000
      initialDelaySeconds: 30        # after startupProbe succeeds
      periodSeconds: 30
      failureThreshold: 3
      timeoutSeconds: 5

    # ✅ Always set resource requests and limits
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

## HPA — Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2          # ✅ never 0 for production
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

## VPA — Vertical Pod Autoscaler

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  updatePolicy:
    updateMode: "Off"    # "Off" = recommendation only; "Auto" = live updates
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
```

## PodDisruptionBudget

```yaml
# ✅ Always define PDB for production workloads
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
  namespace: production
spec:
  minAvailable: 2        # or use maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

## RBAC — Least Privilege

```yaml
# ServiceAccount per application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/AppRole  # IRSA
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]           # ✅ minimal permissions
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: app-service-account
    namespace: production
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: app-role
```

## NetworkPolicy — Zero Trust

```yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow only what's needed
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-allow-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - podSelector:
            matchLabels:
              app: nginx-ingress
      ports:
        - protocol: TCP
          port: 8000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53    # DNS
```

## Helm Best Practices

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 0.1.0            # chart version (SemVer)
appVersion: "1.2.0"       # application version
```

```yaml
# values.yaml — sensible defaults
replicaCount: 2

image:
  repository: my-registry/my-app
  tag: ""                  # override in CI/CD with --set image.tag=$GIT_SHA
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 1

serviceAccount:
  create: true
  annotations: {}

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com
```

```bash
# Helm best practices
helm lint ./chart                                      # validate chart
helm template ./chart --values values-prod.yaml        # dry-run render
helm diff upgrade my-app ./chart --values values-prod.yaml  # see diff before apply
helm upgrade --install my-app ./chart \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml \
  --set image.tag=$GIT_SHA \
  --atomic \                  # rollback on failure
  --timeout 5m \
  --wait
```

## Anti-Patterns

| Anti-Pattern | Why Bad | Fix |
|---|---|---|
| ❌ No resource limits | OOMKilled, noisy neighbour | Always set `requests` and `limits` |
| ❌ `image: app:latest` | Non-reproducible, random updates | Pin to `app:v1.2.0` or `app:$GIT_SHA` |
| ❌ No PodDisruptionBudget | Nodes drain kills all pods | Define PDB with `minAvailable: 1` |
| ❌ No readiness probe | Traffic to unready pods | Always configure `readinessProbe` |
| ❌ Running as root | Security risk | `securityContext.runAsNonRoot: true` |
| ❌ No NetworkPolicy | East-west traffic unrestricted | Default deny + explicit allow |
| ❌ Secrets in ConfigMap | Plain text in etcd | Use Secret or external secrets operator |
| ❌ Single replica in prod | SPOF | `minReplicas: 2` + PDB |
| ❌ No namespace isolation | Blast radius is whole cluster | Separate namespaces per team/env |

## Security Context (Hardened)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

## Useful Debugging Commands

```bash
# Pod status
kubectl get pods -n production -o wide
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production --previous   # crashed container logs

# Events (great for PVC, scheduling issues)
kubectl get events -n production --sort-by='.lastTimestamp'

# Resource usage
kubectl top pods -n production
kubectl top nodes

# Exec into container
kubectl exec -it <pod-name> -n production -- /bin/sh

# Port forward for local debugging
kubectl port-forward svc/my-app 8080:80 -n production

# Check RBAC permissions
kubectl auth can-i get pods --as=system:serviceaccount:production:app-service-account
```


---
name: networking
description: AWS and Kubernetes networking patterns — VPC 3-tier design, Ingress NGINX vs ALB, Istio service mesh, DNS with Route53, TLS with cert-manager, and PrivateLink.
origin: custom
---

# Networking Patterns

Production networking for AWS and Kubernetes environments.

## When to Activate

- Designing VPC architecture for new environments
- Choosing between Ingress NGINX and AWS ALB Ingress
- Setting up TLS with cert-manager
- Configuring Istio service mesh
- Troubleshooting DNS or connectivity issues

## VPC — 3-Tier Design (AWS)

```
┌─────────────────────────────────────────────────────────┐
│ VPC  10.0.0.0/16                                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ AZ-1a       │  │ AZ-1b       │  │ AZ-1c       │    │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │
│  │ Public      │  │ Public      │  │ Public      │    │
│  │ 10.0.1.0/24 │  │ 10.0.2.0/24 │  │ 10.0.3.0/24 │    │
│  │ (ALB, NAT)  │  │ (ALB, NAT)  │  │ (ALB, NAT)  │    │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │
│  │ Private App │  │ Private App │  │ Private App │    │
│  │ 10.0.11.0/24│  │ 10.0.12.0/24│  │ 10.0.13.0/24│    │
│  │ (EKS nodes) │  │ (EKS nodes) │  │ (EKS nodes) │    │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │
│  │ Private DB  │  │ Private DB  │  │ Private DB  │    │
│  │ 10.0.21.0/24│  │ 10.0.22.0/24│  │ 10.0.23.0/24│    │
│  │ (RDS, Redis)│  │ (RDS, Redis)│  │ (RDS, Redis)│    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
```

```hcl
# Terraform VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.2"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b", "us-east-1c"]

  # 3-tier subnets
  public_subnets   = ["10.0.1.0/24",  "10.0.2.0/24",  "10.0.3.0/24"]
  private_subnets  = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
  database_subnets = ["10.0.21.0/24", "10.0.22.0/24", "10.0.23.0/24"]

  # NAT: one per AZ for HA
  enable_nat_gateway     = true
  single_nat_gateway     = false     # false = HA; true = cost saving for non-prod
  one_nat_gateway_per_az = true

  enable_dns_hostnames = true
  enable_dns_support   = true

  # VPC Endpoints — avoid NAT costs for AWS services
  enable_s3_endpoint       = true
  enable_dynamodb_endpoint = true

  # EKS tags — required for ALB controller
  public_subnet_tags = {
    "kubernetes.io/role/elb"                  = "1"
    "kubernetes.io/cluster/prod-eks"          = "shared"
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"         = "1"
    "kubernetes.io/cluster/prod-eks"          = "shared"
  }
}
```

## VPC Endpoints (PrivateLink) — Reduce NAT Costs

```hcl
# Interface endpoints for ECR, Secrets Manager, STS
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.us-east-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true    # ✅ use standard ECR endpoint URL
}

resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.us-east-1.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# Gateway endpoints (free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids
}
```

## Ingress NGINX vs AWS ALB — Comparison

| Dimension | Ingress NGINX | AWS ALB Ingress Controller |
|-----------|--------------|---------------------------|
| **Layer** | L7 (in-cluster) | L7 (managed AWS ALB) |
| **Cost** | EC2 nodes (existing) | ~$20/ALB/month per ALB |
| **SSL termination** | cert-manager (Let's Encrypt) | ACM (free certs) |
| **WebSocket** | ✅ Native | ✅ Supported |
| **gRPC** | ✅ Supported | ✅ Supported |
| **Multi-cluster** | Complex | ✅ Target group binding |
| **WAF integration** | Manual (ModSecurity) | ✅ Native AWS WAF |
| **Best for** | Many services, cost-optimised | AWS-native, WAF required |

## Ingress NGINX Setup

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values values-ingress.yaml \
  --version 4.10.0
```

```yaml
# values-ingress.yaml
controller:
  replicaCount: 2
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  config:
    use-real-ip: "true"
    use-forwarded-headers: "true"
    proxy-real-ip-cidr: "0.0.0.0/0"
    ssl-protocols: "TLSv1.2 TLSv1.3"
    ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
    enable-brotli: "true"
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
  podDisruptionBudget:
    enabled: true
    minAvailable: 1
```

```yaml
# Ingress resource
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"            # req/sec
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

## TLS — cert-manager + Let's Encrypt

```bash
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --version v1.14.4
```

```yaml
# ClusterIssuer — Let's Encrypt production
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@myorg.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
      # DNS-01 for wildcard certs (Route53)
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: Z1234567890ABC
        selector:
          dnsNames:
            - "*.myapp.com"
```

## DNS — Route53

```hcl
# Terraform Route53
resource "aws_route53_zone" "main" {
  name = "myapp.com"
}

# ALB alias record
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.com"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# Health check + failover
resource "aws_route53_health_check" "api" {
  fqdn              = "api.myapp.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "api_failover_primary" {
  zone_id         = aws_route53_zone.main.zone_id
  name            = "api.myapp.com"
  type            = "A"
  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.api.id

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}
```

## Istio Service Mesh

```bash
# Install Istio
istioctl install --set profile=production -y

# Enable sidecar injection for namespace
kubectl label namespace production istio-injection=enabled
```

```yaml
# VirtualService — traffic routing + canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
  namespace: production
spec:
  hosts:
    - api.myapp.com
    - api.production.svc.cluster.local
  gateways:
    - istio-system/main-gateway
    - mesh
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: api
            subset: v2
    - route:
        - destination:
            host: api
            subset: v1
          weight: 90
        - destination:
            host: api
            subset: v2
          weight: 10
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: 5xx,reset,connect-failure
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
  namespace: production
spec:
  host: api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

## Useful Networking Debug Commands

```bash
# Test DNS resolution inside pod
kubectl exec -it <pod> -n production -- nslookup api.production.svc.cluster.local
kubectl exec -it <pod> -n production -- dig api.myapp.com

# Test connectivity
kubectl exec -it <pod> -n production -- curl -v https://api.production.svc.cluster.local/health
kubectl exec -it <pod> -n production -- nc -zv postgres 5432

# Check Istio sidecar proxy
istioctl proxy-status
istioctl proxy-config listeners <pod-name>.production
istioctl proxy-config routes <pod-name>.production
istioctl analyze -n production

# Check network policies
kubectl get networkpolicies -n production
kubectl describe networkpolicy default-deny-all -n production

# Check service endpoints
kubectl get endpoints -n production
kubectl describe service api -n production
```

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| ❌ Single NAT gateway in prod | One NAT per AZ — single NAT = AZ SPOF |
| ❌ No VPC endpoints for S3/ECR | Traffic via NAT costs $0.045/GB — use PrivateLink (free for S3/DynamoDB) |
| ❌ All resources in public subnets | Private subnets for EKS nodes, RDS, Redis |
| ❌ Self-signed TLS certs | cert-manager + Let's Encrypt or ACM |
| ❌ Wildcard security group rules (`0.0.0.0/0` inbound) | Specific CIDR or SG references only |
| ❌ No health checks on DNS records | Route53 health check + failover routing |
| ❌ Plain HTTP between services | Istio mTLS STRICT or at minimum TLS with cert-manager |


---
name: aws
description: AWS patterns — CDK (Python primary, TypeScript alternative) vs Terraform trade-offs, EKS/RDS/S3/IAM best practices, Cost Explorer, Well-Architected Framework 6 pillars, WAF, CloudWatch.
origin: custom
---

# AWS Patterns

AWS infrastructure best practices with CDK and Terraform examples side by side.

## When to Activate

- Designing AWS architecture
- Choosing between CDK and Terraform
- Setting up EKS, RDS, S3, or IAM resources
- Cost optimisation via Cost Explorer
- Applying Well-Architected Framework review
- Configuring WAF and CloudWatch alarms

## CDK vs Terraform — Trade-Off Comparison

| Dimension | AWS CDK | Terraform |
|-----------|---------|-----------|
| **Language** | Python / TypeScript / Java / Go | HCL (domain-specific) |
| **Learning curve** | Lower for devs; higher for ops | Moderate — HCL is simple |
| **AWS coverage** | 100% — always latest | ~95% — occasional lag |
| **Multi-cloud** | AWS only | AWS + GCP + Azure + 100s more |
| **State management** | CloudFormation (managed by AWS) | Manual — S3 + DynamoDB required |
| **Import existing** | `cdk import` (limited) | `terraform import` (mature) |
| **Drift detection** | CloudFormation drift detection | `terraform plan -detailed-exitcode` |
| **Testing** | Unit tests in pytest/jest (assertions) | `terraform validate` + `checkov` |
| **Abstraction** | L2/L3 constructs — high-level | Low-level resources — explicit |
| **Community** | Growing | Largest IaC community |
| **Best for** | Pure AWS, dev-owned infra | Multi-cloud, ops-owned, enterprise |

**Rule of thumb**: CDK if team is Python/TS devs and AWS-only. Terraform if ops team or multi-cloud.

---

## CDK — Python (Primary)

### Setup

```bash
pip install aws-cdk-lib constructs
npm install -g aws-cdk   # CLI
cdk bootstrap aws://123456789012/us-east-1
```

### EKS Cluster (CDK Python)

```python
# infra/stacks/eks_stack.py
from aws_cdk import (
    Stack,
    aws_eks as eks,
    aws_ec2 as ec2,
    aws_iam as iam,
    Tags,
)
from constructs import Construct


class EksStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # VPC
        vpc = ec2.Vpc(
            self,
            "EksVpc",
            max_azs=3,
            nat_gateways=3,                        # ✅ one per AZ for HA
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    name="Public",
                    subnet_type=ec2.SubnetType.PUBLIC,
                    cidr_mask=24,
                ),
                ec2.SubnetConfiguration(
                    name="Private",
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
                    cidr_mask=24,
                ),
            ],
        )

        # EKS Cluster
        cluster = eks.Cluster(
            self,
            "EksCluster",
            cluster_name="prod-eks",
            version=eks.KubernetesVersion.V1_30,
            vpc=vpc,
            vpc_subnets=[ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS)],
            default_capacity=0,                    # manage node groups separately
            endpoint_access=eks.EndpointAccess.PRIVATE,  # ✅ private endpoint
            secrets_encryption_key=None,           # add KMS key in prod
        )

        # Managed Node Group
        cluster.add_nodegroup_capacity(
            "AppNodes",
            instance_types=[
                ec2.InstanceType("m5.xlarge"),
                ec2.InstanceType("m5a.xlarge"),    # ✅ multiple types for Spot
            ],
            capacity_type=eks.CapacityType.SPOT,   # ✅ cost optimisation
            min_size=2,
            desired_size=3,
            max_size=10,
            disk_size=50,
            node_role=self._create_node_role(),
        )

        # Output cluster name
        self.cluster = cluster
        Tags.of(self).add("env", "prod")
        Tags.of(self).add("managed_by", "cdk")

    def _create_node_role(self) -> iam.Role:
        role = iam.Role(
            self,
            "NodeRole",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKSWorkerNodePolicy"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEC2ContainerRegistryReadOnly"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKS_CNI_Policy"),
            ],
        )
        return role
```

### RDS PostgreSQL (CDK Python)

```python
from aws_cdk import (
    aws_rds as rds,
    aws_ec2 as ec2,
    aws_secretsmanager as sm,
    Duration,
    RemovalPolicy,
)


class RdsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, vpc: ec2.Vpc, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # Password in Secrets Manager — never hardcoded
        db_secret = sm.Secret(
            self,
            "DbSecret",
            secret_name="prod/postgres/app",
            generate_secret_string=sm.SecretStringGenerator(
                secret_string_template='{"username": "appuser"}',
                generate_string_key="password",
                exclude_characters="/@\"' \\",
                password_length=32,
            ),
        )

        # Security Group — only EKS can connect
        db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc, description="RDS SG")
        db_sg.add_ingress_rule(
            peer=ec2.SecurityGroup.from_security_group_id(self, "EksSg", eks_sg_id),
            connection=ec2.Port.tcp(5432),
        )

        # RDS Aurora PostgreSQL Serverless v2
        cluster = rds.DatabaseCluster(
            self,
            "AuroraPostgres",
            engine=rds.DatabaseClusterEngine.aurora_postgres(
                version=rds.AuroraPostgresEngineVersion.VER_15_4
            ),
            credentials=rds.Credentials.from_secret(db_secret),
            default_database_name="appdb",
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS),
            security_groups=[db_sg],
            writer=rds.ClusterInstance.serverless_v2("Writer"),
            readers=[
                rds.ClusterInstance.serverless_v2("Reader1", scale_with_writer=True),
            ],
            serverless_v2_min_capacity=0.5,
            serverless_v2_max_capacity=16,
            backup=rds.BackupProps(retention=Duration.days(14)),    # ✅ 14-day backup
            deletion_protection=True,                                # ✅ no accidental delete
            removal_policy=RemovalPolicy.RETAIN,                    # ✅ retain on stack delete
            storage_encrypted=True,                                  # ✅ KMS encryption
            cloudwatch_logs_exports=["postgresql"],
        )
```

### S3 Bucket (CDK Python)

```python
from aws_cdk import aws_s3 as s3, RemovalPolicy, Duration


bucket = s3.Bucket(
    self,
    "DataBucket",
    bucket_name=f"my-data-{self.account}-{self.region}",
    versioned=True,                         # ✅ versioning
    encryption=s3.BucketEncryption.KMS,    # ✅ KMS encryption
    block_public_access=s3.BlockPublicAccess.BLOCK_ALL,   # ✅ no public access
    removal_policy=RemovalPolicy.RETAIN,
    lifecycle_rules=[
        s3.LifecycleRule(
            transitions=[
                s3.Transition(
                    storage_class=s3.StorageClass.INTELLIGENT_TIERING,
                    transition_after=Duration.days(30),
                )
            ],
            noncurrent_version_expiration=Duration.days(90),
        )
    ],
    server_access_logs_bucket=access_logs_bucket,  # ✅ access logging
)
```

### IAM Role (CDK Python — IRSA for EKS)

```python
from aws_cdk import aws_iam as iam


# IRSA — IAM Role for Service Account (EKS workload identity)
irsa_role = iam.Role(
    self,
    "AppIrsaRole",
    role_name="prod-app-irsa",
    assumed_by=iam.FederatedPrincipal(
        cluster.open_id_connect_provider.open_id_connect_provider_arn,
        conditions={
            "StringEquals": {
                f"{oidc_issuer}:sub": "system:serviceaccount:production:app-service-account",
                f"{oidc_issuer}:aud": "sts.amazonaws.com",
            }
        },
        assume_role_action="sts:AssumeRoleWithWebIdentity",
    ),
)

# Least privilege — only what the app needs
irsa_role.add_to_policy(
    iam.PolicyStatement(
        effect=iam.Effect.ALLOW,
        actions=["s3:GetObject", "s3:PutObject"],
        resources=[f"{bucket.bucket_arn}/*"],
    )
)
irsa_role.add_to_policy(
    iam.PolicyStatement(
        effect=iam.Effect.ALLOW,
        actions=["secretsmanager:GetSecretValue"],
        resources=["arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/app/*"],
    )
)
```

---

## CDK — TypeScript (Alternative)

```typescript
// infra/stacks/eks-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as eks from 'aws-cdk-lib/aws-eks';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export class EksStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'EksVpc', {
      maxAzs: 3,
      natGateways: 3,
    });

    const cluster = new eks.Cluster(this, 'EksCluster', {
      clusterName: 'prod-eks',
      version: eks.KubernetesVersion.V1_30,
      vpc,
      defaultCapacity: 0,
      endpointAccess: eks.EndpointAccess.PRIVATE,
    });

    cluster.addNodegroupCapacity('AppNodes', {
      instanceTypes: [
        new ec2.InstanceType('m5.xlarge'),
        new ec2.InstanceType('m5a.xlarge'),
      ],
      capacityType: eks.CapacityType.SPOT,
      minSize: 2,
      desiredSize: 3,
      maxSize: 10,
    });
  }
}
```

---

## Same Resources in Terraform (for comparison)

```hcl
# EKS via Terraform
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.4"

  cluster_name    = "prod-eks"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  cluster_endpoint_public_access  = false   # private endpoint
  cluster_endpoint_private_access = true

  eks_managed_node_groups = {
    app = {
      instance_types = ["m5.xlarge", "m5a.xlarge"]
      capacity_type  = "SPOT"
      min_size       = 2
      desired_size   = 3
      max_size       = 10
      disk_size      = 50
    }
  }

  tags = local.common_tags
}
```

---

## Cost Explorer & Optimisation

```python
import boto3
from datetime import date, timedelta

ce = boto3.client("ce", region_name="us-east-1")

# Get last 30 days cost by service
response = ce.get_cost_and_usage(
    TimePeriod={
        "Start": (date.today() - timedelta(days=30)).isoformat(),
        "End": date.today().isoformat(),
    },
    Granularity="MONTHLY",
    Metrics=["UnblendedCost", "UsageQuantity"],
    GroupBy=[
        {"Type": "DIMENSION", "Key": "SERVICE"},
        {"Type": "TAG",       "Key": "env"},
    ],
    Filter={
        "Tags": {"Key": "env", "Values": ["prod"]}
    },
)

for group in response["ResultsByTime"][0]["Groups"]:
    service = group["Keys"][0]
    cost    = group["Metrics"]["UnblendedCost"]["Amount"]
    print(f"{service}: ${float(cost):.2f}")
```

### Cost Optimisation Checklist

| Strategy | Potential Saving | Action |
|----------|-----------------|--------|
| EC2 Spot instances | Up to 90% | Use Spot for stateless workloads / training |
| Reserved Instances (1yr) | ~40% | Commit for steady-state prod workloads |
| RDS Aurora Serverless v2 | ~60% vs provisioned for variable load | Scale-to-zero in non-prod |
| S3 Intelligent-Tiering | ~40% on infrequent data | Enable lifecycle rule |
| NAT Gateway (PrivateLink) | Varies | Use VPC endpoints for S3, DynamoDB, ECR |
| Right-sizing | 20-40% | Use Compute Optimizer recommendations |
| Delete unattached EBS | 100% waste | `aws ec2 describe-volumes --filters Name=status,Values=available` |

---

## Well-Architected Framework — 6 Pillars

| Pillar | Key Practices |
|--------|--------------|
| **Operational Excellence** | IaC only (no ClickOps), runbooks, observability, chaos engineering |
| **Security** | Least privilege IAM, encryption at rest + transit, WAF, GuardDuty, SCPs |
| **Reliability** | Multi-AZ, auto-healing, backup + restore tested, health checks, circuit breakers |
| **Performance Efficiency** | Right instance types, caching (ElastiCache), CDN (CloudFront), async processing |
| **Cost Optimisation** | Tagging strategy, Spot/Reserved mix, delete waste, Cost Anomaly Detection |
| **Sustainability** | Graviton instances, serverless, efficient algorithms, minimize data transfer |

### WAF (Web Application Firewall)

```python
# CDK Python — WAF for ALB
from aws_cdk import aws_wafv2 as wafv2

web_acl = wafv2.CfnWebACL(
    self,
    "AppWaf",
    scope="REGIONAL",
    default_action=wafv2.CfnWebACL.DefaultActionProperty(allow={}),
    visibility_config=wafv2.CfnWebACL.VisibilityConfigProperty(
        cloud_watch_metrics_enabled=True,
        metric_name="AppWaf",
        sampled_requests_enabled=True,
    ),
    rules=[
        # AWS Managed Rules — Common Rule Set
        wafv2.CfnWebACL.RuleProperty(
            name="AWSManagedRulesCommonRuleSet",
            priority=1,
            override_action=wafv2.CfnWebACL.OverrideActionProperty(none={}),
            statement=wafv2.CfnWebACL.StatementProperty(
                managed_rule_group_statement=wafv2.CfnWebACL.ManagedRuleGroupStatementProperty(
                    vendor_name="AWS",
                    name="AWSManagedRulesCommonRuleSet",
                )
            ),
            visibility_config=wafv2.CfnWebACL.VisibilityConfigProperty(
                cloud_watch_metrics_enabled=True,
                metric_name="CommonRuleSet",
                sampled_requests_enabled=True,
            ),
        ),
        # Rate limiting — 1000 req/5min per IP
        wafv2.CfnWebACL.RuleProperty(
            name="RateLimitRule",
            priority=2,
            action=wafv2.CfnWebACL.RuleActionProperty(block={}),
            statement=wafv2.CfnWebACL.StatementProperty(
                rate_based_statement=wafv2.CfnWebACL.RateBasedStatementProperty(
                    limit=1000,
                    aggregate_key_type="IP",
                )
            ),
            visibility_config=wafv2.CfnWebACL.VisibilityConfigProperty(
                cloud_watch_metrics_enabled=True,
                metric_name="RateLimitRule",
                sampled_requests_enabled=True,
            ),
        ),
    ],
)
```

### CloudWatch Alarms

```python
from aws_cdk import aws_cloudwatch as cw, aws_cloudwatch_actions as cw_actions, Duration

# ALB 5xx error rate alarm
alarm_5xx = cw.Alarm(
    self,
    "Alb5xxAlarm",
    metric=alb.metric_http_code_elb(
        code=elbv2.HttpCodeElb.ELB_5XX_COUNT,
        period=Duration.minutes(5),
        statistic="Sum",
    ),
    threshold=10,
    evaluation_periods=2,
    datapoints_to_alarm=2,
    comparison_operator=cw.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
    alarm_description="ALB 5xx errors > 10 in 10 minutes",
    treat_missing_data=cw.TreatMissingData.NOT_BREACHING,
)
alarm_5xx.add_alarm_action(cw_actions.SnsAction(alert_topic))

# RDS CPU alarm
cw.Alarm(
    self,
    "RdsCpuAlarm",
    metric=db_cluster.metric_cpu_utilization(period=Duration.minutes(5)),
    threshold=80,
    evaluation_periods=3,
    alarm_description="RDS CPU > 80% for 15 minutes",
)
```

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| ❌ ClickOps (manual console changes) | IaC only — CDK or Terraform |
| ❌ Wildcard IAM permissions (`"*"`) | Least privilege — specific actions and resources |
| ❌ Public S3 buckets | `block_public_access=BlockPublicAccess.BLOCK_ALL` |
| ❌ No multi-AZ | Deploy across 3 AZs minimum for prod |
| ❌ No backup / restore tested | Schedule backups + test restore quarterly |
| ❌ No cost tags | Tag everything: env, team, cost_center |
| ❌ Secrets in environment variables | Secrets Manager + IRSA |
| ❌ No VPC endpoints | Traffic to S3/DynamoDB/ECR goes through NAT — use PrivateLink |


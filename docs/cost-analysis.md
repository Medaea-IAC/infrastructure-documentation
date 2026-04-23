# Infrastructure Cost Analysis

> Monthly cost estimates for the current Medaea EHR infrastructure on AWS (us-east-1).
> All figures are based on AWS public pricing as of 2024–2025 and reflect the dev environment
> as deployed. Production estimates are included separately.

---

## Summary — Dev Environment Monthly Cost

| Layer | Service | Monthly Cost |
|---|---|---|
| dns | ACM Certificate | **Free** |
| network | VPC, NAT Gateway, data transfer | **~$47** |
| data | RDS PostgreSQL db.t3.medium | **~$62** |
| data | ElastiCache Redis cache.t3.micro | **~$17** |
| data | S3 media bucket (50 GB) | **~$1.15** |
| data | KMS keys (2 keys) | **~$2** |
| secrets | Secrets Manager (4 secrets) | **~$1.60** |
| compute | ECS Fargate (API + Frontend tasks) | **~$28** |
| compute | Application Load Balancer | **~$22** |
| compute | ECR repositories (storage) | **~$1** |
| compute | CloudWatch Logs + metrics | **~$5** |
| edge | CloudFront (3 distributions, 10 GB/mo) | **~$1** |
| edge | WAFv2 (1 WebACL + managed rules) | **~$13** |
| edge | Route53 (1 hosted zone) | **~$0.50** |
| **Total** | | **~$201 / month** |

> **Note:** Dev runs minimal Fargate task sizes and single-AZ RDS. Production costs
> will be significantly higher — see the production estimate section below.

---

## Detailed Breakdown — Per Service

### ACM Certificate (dns layer)
- **Cost: $0/month**
- AWS Certificate Manager is free for certificates used with AWS services (CloudFront, ALB)
- Wildcard cert `*.medaea.net` + SANs `*.ehr.medaea.net`, `*.api.medaea.net` — still free

---

### VPC + NAT Gateway (network layer)
- **Cost: ~$47/month**

| Component | Pricing | Monthly |
|---|---|---|
| VPC | Free | $0 |
| Subnets, route tables, IGW | Free | $0 |
| NAT Gateway (1x) | $0.045/hr × 730 hrs | $32.85 |
| NAT Gateway data processed | $0.045/GB × 100 GB est. | $4.50 |
| VPC Flow Logs (CloudWatch) | $0.50/GB × 5 GB est. | $2.50 |
| Elastic IP (1x, attached to NAT) | Free while attached | $0 |
| Elastic IP (unattached) | $0.005/hr | **avoid — release after destroy** |

> **Cost optimization:** NAT Gateway is the most expensive network item at $33/month.
> In dev, consider using a NAT instance (t3.nano ~$3.80/mo) or VPC endpoints for
> S3/ECR/Secrets to reduce NAT data transfer costs.

---

### RDS PostgreSQL (data layer)
- **Cost: ~$62/month**

| Component | Pricing | Monthly |
|---|---|---|
| db.t3.medium instance (Single-AZ) | $0.068/hr × 730 hrs | $49.64 |
| gp3 storage (20 GB) | $0.115/GB-mo × 20 GB | $2.30 |
| Automated backups (7 days, ~20 GB) | Free up to DB storage size | $0 |
| RDS CloudWatch logs | ~$0.50/GB × 1 GB est. | $0.50 |
| Data transfer out | $0.09/GB × 1 GB est. | $0.09 |

> **Cost optimization:**
> - Use `db.t3.small` ($0.034/hr = ~$25/mo) for dev if load is minimal
> - Stop dev RDS during off-hours using AWS Instance Scheduler (saves ~65%)
> - Production: `db.r6g.large` Multi-AZ = ~$350/month

---

### ElastiCache Redis (data layer)
- **Cost: ~$17/month**

| Component | Pricing | Monthly |
|---|---|---|
| cache.t3.micro (1 node) | $0.017/hr × 730 hrs | $12.41 |
| KMS key for encryption | $1.00/key/mo | $1.00 |
| Backup storage | Free up to cache size | $0 |
| Data transfer | Minimal for dev | ~$0.50 |

> **Cost optimization:**
> - `cache.t3.micro` is already the smallest available — appropriate for dev
> - Production: `cache.r6g.large` cluster mode = ~$150/month

---

### KMS Keys (data layer)
- **Cost: ~$2/month**

| Key | Cost |
|---|---|
| RDS encryption key | $1.00/key/mo |
| ElastiCache encryption key | $1.00/key/mo |
| API requests (< 20,000/mo) | Free tier covers dev |

---

### Secrets Manager (secrets layer)
- **Cost: ~$1.60/month**

| Secret | Cost |
|---|---|
| `django-secret-key` | $0.40/secret/mo |
| `openai-api-key` | $0.40/secret/mo |
| `jwt-secret` | $0.40/secret/mo |
| `redis-password` | $0.40/secret/mo |
| API calls (< 10,000/mo) | Free for dev traffic |

---

### ECS Fargate (compute layer)
- **Cost: ~$28/month** (dev — minimal task sizes)

Current task configuration estimate:

| Service | vCPU | Memory | Count | Monthly |
|---|---|---|---|---|
| `medaea-dev-api` (FastAPI) | 0.5 vCPU | 1 GB | 1 | $11.20 |
| `medaea-dev-frontend` | 0.25 vCPU | 0.5 GB | 1 | $5.60 |

Fargate pricing (us-east-1):
- vCPU: $0.04048/vCPU-hr
- Memory: $0.004445/GB-hr

> **Cost optimization:**
> - Scale to 0 tasks during off-hours for dev (ECS scheduled scaling)
> - Production: 2+ tasks per service for HA = 2× cost minimum
> - Consider Fargate Spot for dev (up to 70% cheaper, acceptable interruption for non-prod)

---

### Application Load Balancer (compute layer)
- **Cost: ~$22/month**

| Component | Pricing | Monthly |
|---|---|---|
| ALB base charge | $0.008/ALB-hr × 730 hrs | $5.84 |
| LCU (Load Balancer Capacity Units) | $0.008/LCU-hr, est. 2 LCU | $11.68 |
| Data processed | ~$0.008/GB × 10 GB | $0.08 |

---

### CloudFront (edge layer)
- **Cost: ~$1/month** (dev — low traffic)

| Distribution | Traffic | Monthly |
|---|---|---|
| Marketing (`dev.medaea.net`) | ~3 GB out | $0.27 |
| EHR frontend (`dev.ehr.medaea.net`) | ~3 GB out | $0.27 |
| API (`dev.api.medaea.net`) | ~4 GB out | $0.36 |

CloudFront pricing: $0.0085/GB (first 10 TB)
HTTPS requests: $0.0100/10,000 requests — minimal for dev

> **Note:** CloudFront pricing scales very favorably. Even at 10× dev traffic
> the cost is only ~$10/month. For production at 1 TB/month: ~$85/month.

---

### WAFv2 (edge layer)
- **Cost: ~$13/month**

| Component | Pricing | Monthly |
|---|---|---|
| WebACL (1x, CLOUDFRONT scope) | $5.00/mo | $5.00 |
| Managed rule groups (3 groups) | $1.00/group/mo | $3.00 |
| Rules processed (est. 1M req/mo) | $0.60/1M requests | $0.60 |
| Bot Control rule group | $10.00/mo (if enabled) | optional |

> **Cost optimization:** Disable Bot Control rule group in dev (saves $10/month).
> Enable for production — bot traffic is heavier in production environments.

---

### Route53 (edge layer)
- **Cost: ~$0.50/month**

| Component | Cost |
|---|---|
| Hosted zone `medaea.net` | $0.50/mo |
| DNS queries (< 1B/mo) | $0.40/1M queries — minimal for dev |

---

### S3 Media Bucket (data layer)
- **Cost: ~$1.15/month**

| Component | Pricing | Monthly |
|---|---|---|
| Storage (50 GB est.) | $0.023/GB-mo | $1.15 |
| PUT/GET requests | $0.0004/1,000 | minimal |
| Data transfer out | $0.09/GB | minimal |

---

## Cost by Environment

### Dev (Current)
```
$201 / month  ← minimal sizes, single-AZ, 1 task per service
```

### Staging (Estimated — if stood up)
```
$260 / month  ← same as dev + slightly larger RDS (db.t3.large)
               + 2 tasks per service for realistic load testing
```

### Production (Estimated — when ready)
```
$1,100–$1,800 / month  ← see breakdown below
```

---

## Production Cost Estimate

Production requires high availability (Multi-AZ), larger instances, and redundant tasks.

| Service | Prod Config | Monthly |
|---|---|---|
| RDS PostgreSQL | db.r6g.large, Multi-AZ | ~$350 |
| ElastiCache Redis | cache.r6g.large, cluster mode, 2 shards | ~$200 |
| ECS Fargate — API | 1 vCPU / 2 GB, 3 tasks minimum | ~$100 |
| ECS Fargate — Frontend | 0.5 vCPU / 1 GB, 3 tasks minimum | ~$45 |
| ALB | Higher LCU at production traffic | ~$40 |
| NAT Gateways | 2x (one per AZ) | ~$95 |
| CloudFront | 1 TB/month out | ~$85 |
| WAFv2 | + Bot Control + Shield Advanced | ~$130 |
| CloudWatch | More logs, longer retention | ~$30 |
| Shield Advanced | Optional — $3,000/month flat + data | $3,000+ |
| **Total (without Shield Advanced)** | | **~$1,075–$1,200/mo** |
| **Total (with Shield Advanced)** | | **~$4,100+/mo** |

> Shield Advanced is only recommended if the platform processes >$10M ARR or is
> a target for volumetric DDoS attacks (e.g. high-profile public-facing EHR).

---

## Cost Optimization Recommendations

### Immediate (Dev)

| Action | Saving | Effort |
|---|---|---|
| Stop RDS dev instance overnight (8pm–8am weekdays) | ~$37/mo (60%) | Low — AWS Instance Scheduler |
| Scale ECS to 0 tasks overnight in dev | ~$18/mo | Low — scheduled scaling |
| Use Fargate Spot for dev tasks | ~$14/mo (50%) | Low — spot capacity provider |
| Remove unattached Elastic IPs | $3.65/mo each | None — release after destroy |
| Switch NAT GW to NAT instance (t3.nano) in dev | ~$29/mo | Medium — module change |

**Potential dev saving: ~$100/month (50% reduction)**

### Before Production

| Action | Saving | Effort |
|---|---|---|
| Reserved Instances for RDS (1-year) | ~35% on RDS = ~$120/mo | Low — purchase RI |
| Compute Savings Plan for Fargate (1-year) | ~20% on Fargate | Low — purchase plan |
| S3 Intelligent-Tiering for media bucket | Varies by access pattern | Low — bucket config |
| CloudFront cache optimization | Reduces ALB + ECS compute | Medium — cache-control headers |
| Right-size after load testing | Depends on actual utilization | Medium — monitor then resize |

---

## Cost Monitoring Setup

### AWS Cost Anomaly Detection (recommended — free)

```bash
# Create a cost anomaly monitor for the Medaea account
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "MedaeaSpendMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# Alert if daily spend exceeds $20 over expected
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "MedaeaDailyAlert",
    "MonitorArnList": ["<monitor-arn>"],
    "Subscribers": [{"Address": "devops@medaea.com", "Type": "EMAIL"}],
    "Threshold": 20,
    "Frequency": "DAILY"
  }'
```

### Monthly Budget Alert

```bash
# Alert at 80% of $300 dev budget
aws budgets create-budget \
  --account-id 009952409575 \
  --budget '{
    "BudgetName": "MedaeaDevMonthly",
    "BudgetLimit": {"Amount": "300", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "devops@medaea.com"}]
  }]'
```

### Cost by Layer (using AWS Cost Allocation Tags)

All Terraform resources are tagged with `ManagedBy = terraform` and `Environment = dev`.
Use the AWS Cost Explorer to filter by tag to see per-layer spend:

```
Cost Explorer → Filter by tag → ManagedBy: terraform
                              → Group by: Service
```

Add `Layer` tags to each Terraform module's `tags` map to get per-layer cost breakdown.

---

## Load-Based Scaling Cost Projection

| Monthly Active Users | API Requests/mo | Add'l Fargate Cost | Add'l CloudFront | Total Delta |
|---|---|---|---|---|
| 100 (dev) | 500K | baseline | baseline | $0 |
| 1,000 | 5M | +$20 | +$1 | +$21 |
| 10,000 | 50M | +$80 | +$4 | +$84 |
| 100,000 | 500M | +$400 (auto-scale) | +$40 | +$440 |
| 500,000 | 2.5B | +$1,800 (auto-scale) | +$190 | +$1,990 |

> ECS Fargate auto-scales horizontally — costs scale linearly with task count.
> CloudFront and WAFv2 costs are request-based and scale proportionally.
> RDS is the bottleneck at high load — upgrade instance class before scaling ECS tasks.

---

## Billing References

| Link | Purpose |
|---|---|
| [AWS Pricing Calculator](https://calculator.aws/pricing/2/home) | Build and share estimates |
| [AWS Cost Explorer](https://console.aws.amazon.com/cost-management/home) | View actual spend |
| [EC2 / Fargate Pricing](https://aws.amazon.com/fargate/pricing/) | Current Fargate rates |
| [RDS Pricing](https://aws.amazon.com/rds/postgresql/pricing/) | RDS instance pricing |
| [NAT Gateway Pricing](https://aws.amazon.com/vpc/pricing/) | NAT Gateway rates |
| [CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/) | CDN rates |

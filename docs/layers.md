# Layer Reference

> Detailed breakdown of each Terraform layer — what it manages, its dependencies, state location, and key outputs.

---

## Layer Overview

```
Deploy order:  dns ─┐
                    ├─► network ─► data ─┬─► secrets ─────────────────┐
                    │                    └─► compute ─► edge ─► platform┘
                    │                                       │
                    └───────────────────────────────────────┘

Destroy order (reverse): platform → edge → secrets → compute → data → network → dns
```

---

## Layer: dns

| Item | Value |
|---|---|
| **Directory** | `environments/dev/dns/` |
| **State key** | `dev/dns/terraform.tfstate` |
| **Depends on** | Nothing (must run first) |
| **Consumed by** | edge (reads ACM cert ARN) |

**What it manages:**
- ACM wildcard TLS certificate for `medaea.net`
- Subject alternative names: `*.medaea.net`, `*.ehr.medaea.net`, `*.api.medaea.net`
- DNS validation records (auto-created via Route53)
- Certificate must be in `us-east-1` (CloudFront requirement)

**Key output read by edge layer:**
```hcl
data.terraform_remote_state.dns.outputs.acm_certificate_arn
```

**Notes:**
- The Route53 hosted zone (`medaea.net`) must already exist before this layer runs
- Replacing the cert (e.g. adding SANs) takes 5–10 minutes for DNS validation to complete
- `*.medaea.net` covers one level: `dev.medaea.net` ✓, but NOT `dev.ehr.medaea.net` ✗ — that's why `*.ehr.medaea.net` and `*.api.medaea.net` are separate SANs

---

## Layer: network

| Item | Value |
|---|---|
| **Directory** | `environments/dev/network/` |
| **State key** | `dev/network/terraform.tfstate` |
| **Depends on** | ensure-backend |
| **Consumed by** | data, compute |

**What it manages:**
- VPC (`10.0.0.0/16`)
- Public subnets (for ALB, NAT gateway)
- Private subnets (for ECS tasks, RDS, ElastiCache)
- Internet gateway, NAT gateway, route tables
- Security groups (ALB, ECS tasks, RDS, ElastiCache)

---

## Layer: data

| Item | Value |
|---|---|
| **Directory** | `environments/dev/data/` |
| **State key** | `dev/data/terraform.tfstate` |
| **Depends on** | network |
| **Consumed by** | secrets, compute |

**What it manages:**
- RDS PostgreSQL 16.4 (single-AZ in dev, Multi-AZ in prod)
- ElastiCache Redis (replication group, 1 node in dev)
- KMS keys for encryption at rest (RDS, ElastiCache, S3)
- S3 media bucket
- DB subnet groups, parameter groups

**Important notes:**
- RDS version `16.4` is the working version in `us-east-1` (16.1, 16.2, 15.5 not available)
- ElastiCache uses a KMS grant so `elasticache.amazonaws.com` can use the key
- `recovery_window_in_days = 0` for dev secrets (instant delete, no recovery period)

---

## Layer: secrets

| Item | Value |
|---|---|
| **Directory** | `environments/dev/secrets/` |
| **State key** | `dev/secrets/terraform.tfstate` |
| **Depends on** | data |
| **Consumed by** | platform |

**What it manages:**
- AWS Secrets Manager entries for application secrets:
  - `medaea/dev/django-secret-key`
  - `medaea/dev/jwt-secret`
  - `medaea/dev/openai-api-key`
  - `medaea/dev/redis-password`
- SSM Parameter Store parameters:
  - `/<env>/database-url`
  - `/<env>/secret-key`

**Notes:**
- Secrets imported via `import {}` blocks after being manually created during early bootstrap
- Import blocks are idempotent — safe to leave in permanently

---

## Layer: compute

| Item | Value |
|---|---|
| **Directory** | `environments/dev/compute/` |
| **State key** | `dev/compute/terraform.tfstate` |
| **Depends on** | network, data |
| **Consumed by** | edge, platform |

**What it manages:**
- ECS cluster (`medaea-dev`)
- ECR repositories (`medaea-dev/api`, `medaea-dev/frontend`, `medaea-dev/ehr`)
- Application Load Balancer (ALB) + HTTPS/HTTP listeners + target groups
- CloudWatch metric alarms (ECS CPU/memory)
- SNS topic for alerts → email subscription

**Key outputs read by edge layer:**
```hcl
data.terraform_remote_state.compute.outputs.alb_dns_name
data.terraform_remote_state.compute.outputs.alb_zone_id
```

**Notes:**
- `alert_email` variable defaults to `ops@medaea.net`; override via GitHub repo variable `ALERT_EMAIL`
- `acm_wildcard_cert_arn` is read automatically from dns layer remote state (not a variable)
- If compute state doesn't exist yet, edge layer uses defaults (`alb_dns_name = ""`) so it can still plan

---

## Layer: edge

| Item | Value |
|---|---|
| **Directory** | `environments/dev/edge/` |
| **State key** | `dev/edge/terraform.tfstate` |
| **Depends on** | dns, compute |
| **Consumed by** | Nothing |

**What it manages:**
- S3 buckets for marketing website and EHR portal static assets
- CloudFront distribution — Marketing website (`dev.medaea.net` → S3)
- CloudFront distribution — EHR portal (`dev.ehr.medaea.net` → S3)
- CloudFront distribution — API (`dev.api.medaea.net` → ALB)
- WAFv2 WebACL (CLOUDFRONT scope, us-east-1) with managed rules
- WAF CloudWatch log group + log delivery resource policy
- Route53 A alias records for all three domains

**Domain routing:**

| Domain | CloudFront origin | Target |
|---|---|---|
| `dev.medaea.net` | S3 (OAC) | Marketing website |
| `dev.ehr.medaea.net` | S3 (OAC) | EHR React SPA |
| `dev.api.medaea.net` | ALB | FastAPI backend |

**Notes:**
- All Route53 records use `allow_overwrite = true` — safe if records already exist from a previous partial apply
- WAF log group name must start with `aws-waf-logs-` (AWS requirement)
- CloudFront distributions can take 10–15 minutes to deploy

---

## Layer: platform

| Item | Value |
|---|---|
| **Directory** | `environments/dev/platform/` |
| **State key** | `dev/platform/terraform.tfstate` |
| **Depends on** | compute, secrets |
| **Consumed by** | Nothing |

**What it manages:**
- ECS task definitions (backend API, frontend, EHR portal)
- ECS Fargate services
- CodeDeploy applications and deployment groups (blue/green)
- IAM roles for ECS task execution and CodeDeploy
- SSM parameter references in container environment variables

**Notes:**
- Account ID is derived from `data.aws_caller_identity.current.account_id` — no variable needed
- Blue/green deployments swap traffic between blue (current) and green (new) target groups via CodeDeploy

---

## Terraform State Layout

```
S3: medaea-dev-terraform-state/
├── dev/dns/terraform.tfstate
├── dev/network/terraform.tfstate
├── dev/data/terraform.tfstate
├── dev/secrets/terraform.tfstate
├── dev/compute/terraform.tfstate
├── dev/edge/terraform.tfstate
└── dev/platform/terraform.tfstate
```

Native S3 state locking used (Terraform 1.7+ feature — no DynamoDB table required).

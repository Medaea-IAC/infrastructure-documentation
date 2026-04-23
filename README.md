# Medaea Infrastructure Documentation

> Infrastructure-as-code documentation for the Medaea EHR platform.
> Deployed on AWS ECS Fargate via Terraform 1.7.5 + GitHub Actions.

---

## Quick Links

| Document | What it covers |
|---|---|
| [Pipeline Architecture](docs/pipeline.md) | Three-workflow model — plan, apply, destroy |
| [Deploy Guide](docs/deploy.md) | Step-by-step deployment instructions |
| [Destroy Guide](docs/destroy.md) | Safe teardown — full or single layer |
| [Layers Reference](docs/layers.md) | What each of the 7 layers contains |
| [IAM Reference](docs/iam.md) | IAM user, policies, self-service bootstrap |
| [Troubleshooting](docs/troubleshooting.md) | Common errors and fixes |
| [Current State](docs/current-state.md) | Live environment status and known issues |

---

## Infrastructure Repositories

| Repo | Purpose |
|---|---|
| `Medaea-IAC/infra-live` | Terraform root modules, pipeline workflows, IAM policies |
| `Medaea-IAC/infra-modules` | Reusable Terraform modules (VPC, ECS, RDS, CloudFront, etc.) |
| `Medaea-IAC/infrastructure-documentation` | This repo |

---

## Pipeline Summary

### How code changes deploy

```
Developer pushes to develop
         │
         ▼
   plan.yml runs          ← Auto. Detects changed layers.
   terraform plan         ← Plan only, never apply.
   (per changed layer)    ← Results visible in Actions UI.
         │
         │ (Human reviews plan output)
         │
         ▼
   apply.yml triggered    ← Manual. Actions → Apply Infrastructure.
   terraform apply        ← Choose: all layers or a specific one.
```

### Three workflows

**Push to develop** → `plan.yml` runs automatically  
- Detects which layer files changed  
- Runs `terraform plan` only for those layers  
- Never applies. Safe on every push.  

**Deploy** → `apply.yml` triggered manually  
- Choose target: `all` or a specific layer  
- Toggle `plan_only` to preview first  
- Applies changes in correct dependency order  

**Destroy** → `destroy-all.yml` or `destroy.yml` triggered manually  
- Always runs `ensure-iam` first (policy sync before destroy)  
- Full destroy: reverse layer order  
- Single layer: pick from dropdown  

---

## AWS Account

| Item | Value |
|---|---|
| Account ID | `009952409575` |
| Region | `us-east-1` |
| Deploy user | `github-deploy-user-medaea-01` |
| State bucket | `medaea-dev-terraform-state` |
| Terraform version | `1.7.5` |

---

## Environment Domains

| Domain | Environment | Target |
|---|---|---|
| `dev.medaea.net` | dev | Marketing website (S3 + CloudFront) |
| `dev.ehr.medaea.net` | dev | EHR frontend (React, S3 + CloudFront) |
| `dev.api.medaea.net` | dev | FastAPI backend (ECS Fargate + ALB + CloudFront) |

---

## Layer Overview

| # | Layer | Key Resources | Deploy time |
|---|---|---|---|
| 1 | `dns` | ACM wildcard cert | 5–30 min (cert validation) |
| 2 | `network` | VPC, subnets, NAT, SGs | 2–5 min |
| 3 | `data` | RDS PostgreSQL 16.4, ElastiCache Redis, KMS | 10–20 min |
| 4 | `secrets` | SSM parameters | < 1 min |
| 5 | `compute` | ECS cluster, ALB, ECR repos, CloudWatch | 3–8 min |
| 6 | `edge` | CloudFront (3 distros), WAFv2, Route53 aliases | 5–15 min |
| 7 | `platform` | ECS services (Fargate), CodeDeploy blue/green | 5–10 min |

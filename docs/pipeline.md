# Deploy Pipeline Architecture

## Overview

The `Deploy Infrastructure` workflow (`infra-live/.github/workflows/deploy.yml`) manages all AWS infrastructure via Terraform in strict layer order. Every layer is isolated into its own Terraform state file stored in S3.

## Job Execution Order

```
push to develop / workflow_dispatch
         │
         ▼
  ┌─────────────┐
  │ safety-gate │  Blocks prod deploys from push (must use workflow_dispatch)
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │ ensure-iam  │  Creates + attaches MedaeaGitHubDeployPolicy-{1,2,3}
  │             │  Uses deploy user's own credentials (self-managed)
  └──────┬──────┘
         │
         ▼
  ┌────────────────┐
  │ ensure-backend │  Creates S3 state bucket if missing (idempotent)
  └────────┬───────┘
           │
     ┌─────┴──────┐
     │            │
     ▼            ▼
  ┌─────┐    ┌─────────┐
  │ dns │    │ network │
  └──┬──┘    └────┬────┘
     │             │
     │        ┌────▼────┐
     │        │  data   │  RDS + ElastiCache + S3 media
     │        └────┬────┘
     │        ┌────┴──────────────┐
     │        │                  │
     │   ┌────▼─────┐     ┌──────▼───┐
     │   │ secrets  │     │ compute  │  ECS cluster + ALB + ECR
     │   └──────────┘     └────┬─────┘
     │                         │
     └─────────────────────────┤
                               │
                    ┌──────────┴──────────┐
                    │                     │
              ┌─────▼──────┐       ┌──────▼───┐
              │    edge    │       │ platform │
              │ CloudFront │       │  ECS     │
              │ WAF+Route53│       │ services │
              └────────────┘       └──────────┘
```

## Layer Details

| Layer | Terraform Dir | Key Resources | Depends On |
|---|---|---|---|
| `dns` | `environments/dev/dns` | ACM wildcard cert (`*.medaea.net`) | ensure-backend |
| `network` | `environments/dev/network` | VPC, subnets, IGW, NAT GW, route tables, SGs | ensure-backend |
| `data` | `environments/dev/data` | RDS PostgreSQL, ElastiCache Redis, S3 media bucket | network |
| `secrets` | `environments/dev/secrets` | SSM params: `/dev/db-host`, `/dev/redis-url` | data |
| `compute` | `environments/dev/compute` | ECS cluster (`medaea-dev`), ECR repos (`medaea-dev-*`), ALB | network + data |
| `edge` | `environments/dev/edge` | CloudFront distros (website/EHR/API), WAFv2, Route53 aliases | dns + compute |
| `platform` | `environments/dev/platform` | ECS services (Fargate), CodeDeploy blue/green | compute + secrets |

## Terraform State Layout

All state stored in S3 bucket `medaea-dev-terraform-state`:

```
medaea-dev-terraform-state/
├── dev/dns/terraform.tfstate
├── dev/network/terraform.tfstate
├── dev/data/terraform.tfstate
├── dev/secrets/terraform.tfstate
├── dev/compute/terraform.tfstate
├── dev/edge/terraform.tfstate
└── dev/platform/terraform.tfstate
```

Native S3 state locking is used (no DynamoDB required — Terraform 1.7+ feature).

## Module References

All modules in `Medaea-IAC/infra-modules` are referenced with `?ref=develop`. No version pinning yet — `develop` is the only valid branch.

## Domain Routing

| Domain | Route | Target |
|---|---|---|
| `dev.medaea.net` | CloudFront → S3 | Marketing website |
| `dev.ehr.medaea.net` | CloudFront → S3 | EHR frontend (React) |
| `dev.api.medaea.net` | CloudFront → ALB | FastAPI backend (ECS) |

## ECS Configuration

- **Cluster name:** `medaea-dev` (NOT `medaea-dev-cluster`)
- **ECR repo prefix:** `medaea-dev` (e.g. `medaea-dev/api`, `medaea-dev/frontend`)
- **Launch type:** Fargate
- **Deployment:** CodeDeploy blue/green

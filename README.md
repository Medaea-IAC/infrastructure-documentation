# Medaea Infrastructure Documentation

Central reference for all infrastructure decisions, pipeline architecture, IAM policy history, and operational runbooks for the Medaea EHR platform.

## Repository Index

| Document | Purpose |
|---|---|
| [Pipeline Architecture](docs/pipeline.md) | Full deploy pipeline, job order, layer diagram |
| [IAM Policy History](docs/iam-policy.md) | Permission decisions, split rationale, bootstrap approach |
| [Manual Steps Log](docs/manual-steps.md) | Every manual action required, why it was needed, and how to avoid it |
| [Deployment History](docs/deployment-history.md) | Chronological change log with PR references |

---

## Platform Overview

Medaea is a dual-application EHR platform deployed on AWS via GitHub Actions + Terraform.

```
┌─────────────────────────────────────────────────────────────────┐
│                        medaea.net (Route53)                      │
├──────────────────┬──────────────────────┬───────────────────────┤
│  dev.medaea.net  │  dev.ehr.medaea.net  │  dev.api.medaea.net   │
│  Marketing Site  │     EHR Frontend     │     FastAPI Backend    │
│  (CloudFront→S3) │  (CloudFront→S3)     │  (CloudFront→ALB)     │
└──────────────────┴──────────────────────┴───────────────────────┘
                                                    │
                                            ┌───────┴────────┐
                                            │  ECS Fargate   │
                                            │  medaea-dev    │
                                            │  cluster       │
                                            └───────┬────────┘
                                                    │
                          ┌─────────────────────────┼──────────────────────┐
                          │                         │                      │
                   ┌──────┴──────┐          ┌───────┴──────┐      ┌───────┴──────┐
                   │  RDS        │          │  ElastiCache │      │  SSM         │
                   │  PostgreSQL │          │  Redis       │      │  Parameters  │
                   └─────────────┘          └──────────────┘      └──────────────┘
```

## GitHub Repositories

| Repo | Purpose |
|---|---|
| `Medaea-IAC/infra-live` | Terraform live configs — all environments |
| `Medaea-IAC/infra-modules` | Reusable Terraform modules |
| `Medaea-Development/medaea-platform` | Application monorepo (frontend + backend) |
| `Medaea-IAC/infrastructure-documentation` | This repo — all docs and history |

## AWS Account

- **Account ID:** `009952409575`
- **Primary region:** `us-east-1`
- **Deploy IAM user:** `github-deploy-user-medaea-01`
- **Terraform state:** S3 bucket `medaea-dev-terraform-state` (native S3 locking, no DynamoDB)

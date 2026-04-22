# Medaea Infrastructure Documentation

Central reference for all infrastructure decisions, pipeline architecture, IAM policies, and operational runbooks for the Medaea EHR platform.

---

## Quick Navigation

| Document | What it covers |
|---|---|
| [**Deploy Guide**](docs/deploy.md) | Step-by-step: first-time bootstrap through fully live deployment |
| [**Destroy Guide**](docs/destroy.md) | Tear down a single layer or everything at once |
| [**Layer Reference**](docs/layers.md) | What each Terraform layer manages, inputs/outputs, state keys |
| [**IAM Reference**](docs/iam.md) | Deploy user policies — what each contains, sizes, how they self-update |
| [**Troubleshooting**](docs/troubleshooting.md) | Every production error and exactly how it was fixed |
| [**Current State**](docs/current-state.md) | Live snapshot — policy sizes, module defaults, open PRs |
| [**Pipeline Architecture**](docs/pipeline.md) | Full deploy pipeline, job order, layer diagram |
| [**Deployment History**](docs/deployment-history.md) | Chronological change log with PR references |

---

## Platform Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        medaea.net (Route53)                     │
├──────────────────┬──────────────────────┬───────────────────────┤
│  dev.medaea.net  │  dev.ehr.medaea.net  │  dev.api.medaea.net   │
│  Marketing Site  │     EHR Frontend     │     FastAPI Backend    │
│  (CloudFront→S3) │  (CloudFront→S3)     │  (CloudFront→ALB)     │
└──────────────────┴──────────────────────┴───────────────────────┘
                                                    │
                                            ┌───────┴────────┐
                                            │  ECS Fargate   │
                                            │  medaea-dev    │
                                            └───────┬────────┘
                          ┌─────────────────────────┼─────────────────────┐
                    ┌─────┴──────┐          ┌────────┴─────┐    ┌─────────┴──────┐
                    │ RDS 16.4   │          │ ElastiCache  │    │ Secrets Manager│
                    │ PostgreSQL │          │ Redis        │    │ + SSM Params   │
                    └────────────┘          └──────────────┘    └────────────────┘
```

## Key Facts

| Item | Value |
|---|---|
| AWS Account | `009952409575` |
| Primary region | `us-east-1` |
| Deploy IAM user | `github-deploy-user-medaea-01` |
| Terraform version | `1.7.5` |
| State backend | S3 `medaea-dev-terraform-state` (native S3 locking — no DynamoDB) |
| Module source | `Medaea-IAC/infra-modules @ develop` |

## GitHub Repositories

| Repository | Purpose |
|---|---|
| `Medaea-IAC/infra-live` | Terraform live configs, pipeline YAML, IAM policy JSON |
| `Medaea-IAC/infra-modules` | Reusable Terraform modules |
| `Medaea-Development/medaea-platform` | Application monorepo (frontend + EHR + backend) |
| `Medaea-IAC/infrastructure-documentation` | This repo |

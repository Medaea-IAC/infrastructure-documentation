# Pipeline Architecture

> Enterprise-grade, Jenkins-style infrastructure pipeline for the Medaea EHR platform.
> Three separate workflows handle planning, deployment, and destruction independently.

---

## Overview — Three Workflows

| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| **Plan** | `plan.yml` | Auto on push to `develop` | Runs `terraform plan` only for changed layers |
| **Apply** | `apply.yml` | Manual only | Applies changes to selected layers or all at once |
| **Destroy All** | `destroy-all.yml` | Manual only | Tears down all 7 layers in reverse order |
| **Destroy Layer** | `destroy.yml` | Manual only | Tears down a single selected layer |
| **Deploy (legacy)** | `deploy.yml` | Manual only | Old combined workflow, kept for compatibility |

**Key principle:** Code commits never automatically apply infrastructure. They only run a plan preview. A human must explicitly trigger Apply.

---

## Workflow 1 — Plan (Auto on Push)

**File:** `.github/workflows/plan.yml`
**Trigger:** Every push to `develop`

```
push to develop
      │
      ▼
 detect-changes   ← dorny/paths-filter reads git diff
      │
      ├─ ensure-iam ──────────────────────────────────────┐
      │                                                    │
      │  (only runs plan for layers where files changed)   │
      │                                                    ▼
      ├─ plan-dns         (if environments/dev/dns/** changed)
      ├─ plan-network     (if environments/dev/network/** changed)
      ├─ plan-data        (if environments/dev/data/** changed)
      ├─ plan-secrets     (if environments/dev/secrets/** changed)
      ├─ plan-compute     (if environments/dev/compute/** changed)
      ├─ plan-edge        (if environments/dev/edge/** changed)
      └─ plan-platform    (if environments/dev/platform/** changed)
```

- `ensure-iam` always runs — keeps policies current even on non-apply runs
- Layer plans run in **parallel** (no ordering needed for plan-only)
- No apply, no state changes. Safe to run on every commit.
- Results appear in the GitHub Actions run as pass/fail per layer

**What "changed" means:**  
`environments/dev/dns/**` → dns layer plan runs  
`environments/dev/network/**` → network layer plan runs  
`iam/**` → only ensure-iam runs (policy update), no layer plans  

---

## Workflow 2 — Apply (Manual Deploy)

**File:** `.github/workflows/apply.yml`
**Trigger:** `Actions → Apply Infrastructure → Run workflow` (manual only)

### Inputs

| Input | Options | Default | Description |
|---|---|---|---|
| `environment` | dev / staging / prod | dev | Target AWS environment |
| `target` | all / dns / network / data / secrets / compute / edge / platform | all | What to deploy |
| `plan_only` | true / false | false | Preview only — no apply |

### Layer execution order for `target=all`

```
                    ensure-iam
                         │
                    ensure-backend
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
            dns                  network
              │                     │
              │                ┌────▼────┐
              │                │  data   │
              │                └────┬────┘
              │           ┌─────────┴──────────┐
              │           ▼                    ▼
              │        secrets             compute
              │           │                   │
              └───────────┴──────────┬────────┘
                                     ▼
                              ┌──────┴──────┐
                              ▼             ▼
                             edge        platform
```

### Single-layer deploys (`target=dns`, `target=compute`, etc.)

Only the selected layer's job runs — all other layer jobs are skipped via `if: inputs.target == 'all' || inputs.target == 'layername'`. `ensure-iam` and `ensure-backend` always run first regardless of target.

### Plan-only mode (`plan_only=true`)

Every layer runs `terraform plan` but skips `terraform apply`. Use this to preview changes before committing to apply.

---

## Workflow 3 — Destroy All (Manual)

**File:** `.github/workflows/destroy-all.yml`
**Trigger:** Manual only — must type `destroy-all` to confirm

### Execution order (reverse of deploy)

```
confirm → ensure-iam → platform → edge → secrets → compute → data → network → dns
```

Each layer waits for the previous to complete before starting. This ensures:
- ECS services (platform) are gone before their cluster (compute) is destroyed
- CloudFront (edge) is gone before ACM certificate (dns) is destroyed
- Compute is gone before the network it runs in (network) is destroyed

### Why ensure-iam runs on destroy

The deploy user needs `terraform destroy` permissions (same as apply — `ec2:DeleteVpc`, `ecs:DeleteCluster`, etc.). If a new permission was added to the policy JSON but the pipeline hasn't deployed since that update, the AWS policy would be stale. Running ensure-iam first guarantees the live AWS policy matches the repo JSON before any destroy operation.

---

## Workflow 4 — Destroy Layer (Manual)

**File:** `.github/workflows/destroy.yml`
**Trigger:** Manual — choose environment + layer from dropdown

Destroys a single layer. Same ensure-iam pre-step as destroy-all. Use this when:
- A single layer is broken and you need to recreate it
- You want to destroy platform only (to redeploy ECS services from scratch)
- You want to rebuild data layer (RDS) after a schema change requires recreation

---

## IAM Policy Self-Management

All workflows use a single deploy IAM user (`github-deploy-user-medaea-01`). Permissions are split across three managed policies (6,144-char AWS limit per policy):

| Policy | Covers |
|---|---|
| `MedaeaGitHubDeployPolicy-1` | S3, Route53, ACM, CloudFront, WAFv2, VPC/EC2 |
| `MedaeaGitHubDeployPolicy-2` | ELB, ECS, ECR, RDS, ElastiCache, Secrets, SSM |
| `MedaeaGitHubDeployPolicy-3` | CloudWatch, Logs, SNS, IAM, CodeDeploy, KMS |

The `ensure-iam` job runs at the top of every workflow. It:
1. Reads the three `iam/github-deploy-policy-*.json` files from the repo
2. Creates a new policy version and sets it as default (pruning oldest if at AWS's 5-version limit)
3. Creates the policy + attaches to the user if it doesn't exist yet

This means IAM permissions are always up-to-date before any terraform operation runs — both on deploy and destroy.

---

## Terraform State

All state stored in S3 `medaea-dev-terraform-state`, one file per layer:

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

Native S3 state locking is used (Terraform 1.7+ feature — no DynamoDB required).

---

## Module References

All infrastructure modules live in `Medaea-IAC/infra-modules`, referenced via:
```hcl
source = "git::https://github.com/Medaea-IAC/infra-modules.git//modules/<name>?ref=develop"
```

No version pinning — `develop` is the single source of truth for modules.

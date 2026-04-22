# Deployment History

Chronological record of all infrastructure changes, decisions made, errors encountered, and resolutions.

---

## Phase 1 — Initial Repository Setup

**Repos created:**
- `Medaea-IAC/infra-live` — live Terraform configurations
- `Medaea-IAC/infra-modules` — reusable Terraform modules
- `Medaea-Development/medaea-platform` — application monorepo

**Key decisions:**
- Terraform state: S3 with native locking (Terraform 1.7+), no DynamoDB
- State bucket: `medaea-dev-terraform-state`
- Module ref strategy: `?ref=develop` (only valid branch on infra-modules)
- Monorepo structure: `frontend:5000`, `marketing:3000`, `backend:8000`

---

## Phase 2 — Edge Layer (CloudFront) — PR #17

**Branch:** `feature/MEP-52-fix-pipeline-cloudfront-iam` → `develop`  
**Status:** Merged

**What was built:**
- Full CloudFront structure with 3 distributions:
  - `dev.medaea.net` → S3 (marketing website)
  - `dev.ehr.medaea.net` → S3 (EHR frontend)
  - `dev.api.medaea.net` → ALB custom origin (FastAPI)
- Shared WAFv2 WebACL attached to all CloudFront distributions
- Route53 alias records for all three domains
- ACM wildcard cert for `*.medaea.net` (DNS validated)

**Secrets layer created:**
- `environments/dev/secrets/` — reads RDS/Redis endpoints from remote state
- Creates SSM params: `/dev/db-host`, `/dev/redis-url`
- Uses `ignore_changes` on SecureString placeholder values

**IAM permissions file:**
- `iam/github-deploy-policy.json` — 412-line comprehensive policy (later found to exceed AWS limits)
- `iam/attach-deploy-policy.sh` — manual attach script

**Pipeline structure established:**
```
ensure-backend → dns → network → data → secrets → compute → edge → platform
```

---

## Phase 3 — Platform Deploy Fix — medaea-platform PR

**Branch:** `feature/MEP-52-fix-deploy-pipeline` → `develop`  
**Status:** Pushed, pending merge

**Errors fixed:**
- ECS cluster name: `medaea-dev-cluster` → `medaea-dev` (wrong name in deploy workflow)
- ECR repo prefix: `medaea/dev` → `medaea-dev` (wrong prefix caused image push failures)

**Compute outputs clarified:**
- Exports: `ecs_cluster_arn`, `ecs_cluster_name`
- Does NOT export `ecs_cluster_id` (field doesn't exist — use ARN or name)

---

## Phase 4 — IAM Bootstrap Workflow — PR #18

**Branch:** `feature/MEP-52-iam-bootstrap-workflow` → `develop`  
**Status:** Merged

**Problem:** `github-deploy-user-medaea-01` had no IAM permissions, causing AccessDenied on every layer:
- `route53:ListHostedZones` (dns layer)
- `ec2:CreateVpc`, `ec2:AllocateAddress` (network layer)

**First approach:** Separate `iam-bootstrap.yml` workflow requiring admin credentials as GitHub secrets.  
**Limitation:** Required user to manually trigger a separate workflow with temporary admin secrets.

---

## Phase 5 — Self-Managed IAM Pipeline — PR (pending)

**Branch:** `feature/MEP-52-self-managed-iam-permissions` → `develop`  
**Status:** Pushed, pending merge

**Improvement:** Integrated IAM bootstrap directly into `deploy.yml` as `ensure-iam` job.  
**Design:** Deploy user manages its own permissions using a narrow inline policy applied once via CloudShell.

**New files:**
- `iam/self-service-bootstrap-policy.json` — 9 IAM actions scoped tightly to own policy and user ARN
- `iam/BOOTSTRAP.md` — CloudShell one-liner + Console instructions

**Pipeline change:**
```
safety-gate → ensure-iam → ensure-backend → [all layers]
```

**Error encountered:** `iam:CreatePolicy` still failed — CloudShell command not yet run.

---

## Phase 6 — Policy Size Fix — PR (pending)

**Branch:** `feature/MEP-52-split-iam-policy` → `develop`  
**Status:** Pushed, pending merge

**Error:** `LimitExceeded: Cannot exceed quota for PolicySize: 6144`  
**Cause:** `github-deploy-policy.json` was 12,757 chars — over AWS's 6,144-char managed policy limit.

**Fix:** Split into three policies, all verified under the limit:

| Policy | Size | Covers |
|---|---|---|
| `MedaeaGitHubDeployPolicy-1` | 4,804 chars | S3, Route53, ACM, CloudFront, WAFv2, VPC/EC2 |
| `MedaeaGitHubDeployPolicy-2` | 4,785 chars | ELB, ECS, ECR, RDS, ElastiCache, Secrets, SSM |
| `MedaeaGitHubDeployPolicy-3` | 3,267 chars | CloudWatch, Logs, SNS, IAM, CodeDeploy, KMS |

**self-service-bootstrap-policy.json updated:**
- Resource changed from exact ARN to wildcard `MedaeaGitHubDeployPolicy*`
- Covers all three split policies with one inline bootstrap policy

**ensure-iam job updated:**
- Single reusable `upsert_policy()` bash function
- Called three times — once per policy file
- Each call: create-or-update + attach (idempotent)

---

## Current State

| Item | Status |
|---|---|
| infra-live repo | Active on `develop` |
| infra-modules repo | Active on `develop` |
| medaea-platform repo | Active on `develop` |
| CloudFront edge layer | Deployed via PR #17 |
| IAM policy split | Pushed, pending merge |
| One-time CloudShell bootstrap | **Pending — must be run before pipeline succeeds** |
| Full pipeline end-to-end | Pending first successful run |

## Next Steps

1. Merge `feature/MEP-52-split-iam-policy` → `develop`
2. Run the one-time CloudShell command (see [Manual Steps](manual-steps.md))
3. Re-run the "Deploy Infrastructure" workflow — all layers should succeed

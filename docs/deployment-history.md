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

---

## Phase 7 — EC2 Describe Permissions Patch — PR (pending)

**Branch:** `feature/MEP-52-patch-ec2-describe-permissions` → `develop`
**Status:** Pushed, pending merge

**DNS layer:** Passed successfully.

**Network layer errors:**

| Error | Action Missing | Triggered By |
|---|---|---|
| `UnauthorizedOperation` | `ec2:DescribeVpcAttribute` | Terraform waiter after `aws_vpc` creation checks DNS hostname attribute |
| `UnauthorizedOperation` | `ec2:DescribeAddressesAttribute` | Terraform reads back EIP domain name attribute after `aws_eip` creation |

**Fix:** Added 5 actions to the `VPCAndNetworking` statement in `github-deploy-policy-1.json`:

| Action Added | Reason |
|---|---|
| `ec2:DescribeVpcAttribute` | Required by CreateVpc waiter |
| `ec2:DescribeAddressesAttribute` | Required by EIP read-back |
| `ec2:DescribePrefixLists` | Security group rules that reference prefix lists |
| `ec2:DescribeVpcClassicLink` | Terraform VPC module compatibility checks |
| `ec2:DescribeFlowLogs` | Terraform state read for VPC flow log resources |

**Policy-1 size:** 4,804 → 4,987 chars (still under 6,144 limit).

**Pattern:** AWS Terraform provider frequently reads back resource attributes after creation using `Describe*` API calls that are separate from the write permission. All EC2 `Describe*` actions needed by the VPC module are now covered.

---

## Phase 8 — S3 Module Replication Fix (infra-modules)

**Branch:** `feature/MEP-52-optional-s3-replication` → `develop` on `infra-modules`
**Status:** Pushed, pending merge

**Error:**
```
Error: Missing required argument
on main.tf line 89, in module "media_s3_dev"
The argument "replication_destination_bucket_arn" is required, but no definition was found.
```

**Root cause:** The `s3` module in `infra-modules` declared `replication_destination_bucket_arn` as a required variable with no default. Cross-region replication is not needed in dev, so the live config correctly omitted it — but the module enforced it unconditionally.

**Fix in `infra-modules/modules/s3`:**

- `variables.tf`: Added `default = null` to `replication_destination_bucket_arn`
- `main.tf`: Gated all 4 replication resources with `count = var.replication_destination_bucket_arn != null ? 1 : 0`
  - `aws_iam_role.replication`
  - `aws_iam_policy.replication`
  - `aws_iam_role_policy_attachment.replication`
  - `aws_s3_bucket_replication_configuration.this`

**No change required in infra-live** — the `media_s3_dev` module call already omits the argument and now correctly uses the null default, creating a plain encrypted + versioned bucket with no replication.

**Behaviour after fix:**

| `replication_destination_bucket_arn` | Resources created |
|---|---|
| `null` (default) | Bucket + KMS + versioning + lifecycle only |
| `arn:aws:s3:::bucket-name` | Full cross-region replication stack |

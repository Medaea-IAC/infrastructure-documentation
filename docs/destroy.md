# Destroy Guide

> How to safely tear down Medaea infrastructure — full environment or individual layers.

---

## Before You Destroy

### Why ensure-iam runs first

Both destroy workflows run `ensure-iam` before touching any Terraform state. This is critical because:

- The deploy user needs `terraform destroy` permissions (same action set as apply — `ec2:DeleteVpc`, `ecs:DeleteCluster`, etc.)
- IAM policies in AWS can fall out of sync with the repo JSON if deploys haven't run recently
- Stale permissions are the most common cause of destroy failures

The destroy pipeline now guarantees the live AWS policy matches the repo JSON before any layer is destroyed.

### Check what exists before destroying

```bash
# List all active resources in the environment
aws ecs list-clusters
aws rds describe-db-instances
aws ec2 describe-vpcs --filters "Name=tag:Environment,Values=dev"
aws cloudfront list-distributions
```

---

## Destroy All Layers — Full Teardown

**Workflow:** `Actions → Destroy All Infrastructure → Run workflow`

This tears down all 7 layers in reverse dependency order.  
Production is blocked — only `dev` and `staging` are available.

### Inputs

| Input | Required | Notes |
|---|---|---|
| `environment` | Yes | `dev` or `staging` only |
| `confirm` | Yes | Must type exactly `destroy-all` |

### Execution order

```
confirm
   │
   ▼
ensure-iam          ← Updates AWS policies from repo JSON
   │
   ▼
1/7  platform       ← ECS services + CodeDeploy (removed first — no traffic)
   │
   ▼
2/7  edge           ← CloudFront + WAF + Route53 (DNS detached before compute gone)
   │
   ▼
3/7  secrets        ← SSM / Secrets Manager
   │
   ▼
4/7  compute        ← ECS cluster + ALB + ECR
   │
   ▼
5/7  data           ← RDS + ElastiCache + KMS (removed after compute is gone)
   │
   ▼
6/7  network        ← VPC + subnets + NAT (removed after all workloads are gone)
   │
   ▼
7/7  dns            ← ACM certificate (removed last)
```

Each layer waits for the previous to complete. If any layer fails, subsequent layers are blocked automatically by GitHub Actions job dependencies.

### What is NOT destroyed

- S3 state bucket (`medaea-dev-terraform-state`) — preserved intentionally to avoid losing Terraform state
- IAM user and policies — preserved so the pipeline can be re-run after destroy
- ECR images — preserved (ECR repositories are destroyed but images in them are deleted as part of that)

---

## Destroy a Single Layer

**Workflow:** `Actions → Destroy Layer → Run workflow`

Use this when:
- One layer is broken and you need to recreate it cleanly
- You want to rebuild data (RDS) after a schema change requires recreation
- You want to destroy platform (ECS services) only, without touching compute or network

### Inputs

| Input | Options | Notes |
|---|---|---|
| `environment` | dev / staging | |
| `layer` | dns / network / data / secrets / compute / edge / platform | Layer to destroy |

### Important — destroy order matters

When destroying manually, respect the dependency chain or dependent layers will have dangling references:

```
Safe single-layer destroy order:
  platform → edge → secrets → compute → data → network → dns

NEVER destroy network before compute (ECS tasks still running in the VPC)
NEVER destroy dns before edge (CloudFront origin still using the cert)
NEVER destroy data before compute (ECS tasks still reading from RDS/Redis)
```

### Example — rebuild the compute layer from scratch

```
# Step 1: Destroy platform first (ECS services depend on compute)
Actions → Destroy Layer
  environment: dev
  layer: platform

# Step 2: Destroy compute
Actions → Destroy Layer
  environment: dev
  layer: compute

# Step 3: Redeploy compute
Actions → Apply Infrastructure
  environment: dev
  target: compute

# Step 4: Redeploy platform
Actions → Apply Infrastructure
  environment: dev
  target: platform
```

---

## If Destroy Fails Mid-Run

When a layer fails mid-destroy, GitHub Actions stops subsequent layers automatically (job dependency). The environment is in a partial state.

### Recovery options

**Option A — Fix and retry the failed layer**

Check the failed job output for the specific error. Common causes:
- IAM permission gap (should be prevented by ensure-iam, but check anyway)
- Resource has a dependency that wasn't destroyed first
- AWS rate limiting (retry usually works)

Fix the issue, then re-run just that layer:
```
Actions → Destroy Layer → environment: dev, layer: <failed-layer>
```

**Option B — Force-delete stuck resources via AWS CLI**

If Terraform can't delete a resource (e.g. RDS with deletion protection, or a security group with active ENIs):

```bash
# Remove RDS deletion protection
aws rds modify-db-instance \
  --db-instance-identifier medaea-dev-postgres \
  --no-deletion-protection

# Detach security group dependencies
aws ec2 describe-network-interfaces \
  --filters "Name=group-id,Values=<sg-id>" \
  --query 'NetworkInterfaces[*].NetworkInterfaceId'

# Then retry the Terraform destroy
```

**Option C — Manual cleanup**

If Terraform state is corrupt or the destroy can't proceed, use the AWS Console or CLI to manually delete resources, then run `terraform state rm` to remove the resource from state.

---

## Verifying Successful Destroy

After all layers report green:

```bash
# VPC should be gone
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=medaea-dev-vpc" \
  --query 'Vpcs[].VpcId'
# Expected: []

# RDS should be gone
aws rds describe-db-instances \
  --query 'DBInstances[?DBInstanceIdentifier==`medaea-dev-postgres`]'
# Expected: []

# ECS cluster should be gone
aws ecs list-clusters \
  --query 'clusterArns[?contains(@, `medaea-dev`)]'
# Expected: []

# CloudFront distributions
aws cloudfront list-distributions \
  --query 'DistributionList.Items[?contains(Comment, `medaea-dev`)]'
# Expected: []
```

---

## Re-deploying After a Full Destroy

After a successful full destroy, you can redeploy from scratch by following the full deploy sequence in `docs/deploy.md`:

```
Apply dns → Apply network → Apply data → Apply secrets →
Apply compute → Apply edge → Apply platform
```

The S3 state bucket is preserved, so Terraform starts with empty state (all resources recreated).

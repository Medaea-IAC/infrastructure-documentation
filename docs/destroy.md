# Destroy Guide

> How to tear down Medaea infrastructure — a single layer or everything at once.

---

## Important Rules

- **Production (`prod`) is fully blocked** — the pipeline aborts if you attempt it
- **Destroy order is strict** — always reverse of deploy order; skipping steps causes errors because lower layers depend on upper layers' resources (security groups, VPCs, etc.)
- **The S3 state bucket is never auto-destroyed** — it holds all Terraform state; remove it manually at the very end if you want a full wipe

---

## Option A — Destroy Everything (Recommended for full teardown)

Uses the **Destroy All Infrastructure** workflow which runs all 7 layers in strict sequence automatically.

### Steps

1. Go to: **GitHub → `Medaea-IAC/infra-live` → Actions → Destroy All Infrastructure → Run workflow**

2. Fill in the form:

   | Field | What to enter |
   |---|---|
   | `environment` | `dev` or `staging` |
   | `confirm` | Type `destroy-all` exactly — anything else aborts |

3. Click **Run workflow**

4. The pipeline runs 7 jobs sequentially:

   ```
   Safety gate
       │
       ▼
   1. platform  — ECS services, CodeDeploy           (~3 min)
       │
       ▼
   2. edge      — CloudFront, WAF, Route53            (~15 min — CF takes time)
       │
       ▼
   3. secrets   — Secrets Manager entries             (~1 min)
       │
       ▼
   4. compute   — ALB, ECS cluster, ECR, monitoring   (~5 min)
       │
       ▼
   5. data      — RDS, ElastiCache, KMS               (~12 min — RDS takes time)
       │
       ▼
   6. network   — VPC, subnets, NAT gateway            (~3 min)
       │
       ▼
   7. dns       — ACM certificate                     (~2 min)
       │
       ▼
   Summary job  — prints confirmation + next steps
   ```

   **Expected total time: 30–45 minutes.**

5. After all jobs are green, optionally remove the state bucket:

   ```bash
   # Empty the bucket first (required before deletion)
   aws s3 rm s3://medaea-dev-terraform-state --recursive

   # Delete the bucket
   aws s3 rb s3://medaea-dev-terraform-state
   ```

---

## Option B — Destroy a Single Layer

Use the **Destroy Infrastructure (single layer)** workflow when you only need to tear down one layer (e.g., redeploy data layer from scratch).

### Steps

1. Go to: **GitHub → `Medaea-IAC/infra-live` → Actions → Destroy Infrastructure (single layer) → Run workflow**

2. Fill in the form:

   | Field | What to enter |
   |---|---|
   | `environment` | `dev`, `staging`, or `prod` |
   | `layer` | The layer name (see table below) |
   | `confirm` | Type `destroy` exactly |

3. Available layers and their destroy impact:

   | Layer | Destroys | Safe to destroy if… |
   |---|---|---|
   | `platform` | ECS services, CodeDeploy apps | Always safe (stateless) |
   | `edge` | CloudFront distros, WAF, Route53 records, S3 buckets | Always safe (stateless) |
   | `secrets` | Secrets Manager secrets | Safe — dev secrets have 0-day recovery window |
   | `compute` | ALB, ECS cluster, ECR repos, CloudWatch alarms | Safe after platform is destroyed |
   | `data` | RDS instance, ElastiCache cluster, KMS keys | **Destroys all data** — ensure backups exist |
   | `network` | VPC, subnets, NAT gateway, security groups | Safe after data + compute are destroyed |
   | `dns` | ACM wildcard certificate | Safe after edge is destroyed |

### Single-layer destroy order

If manually destroying multiple layers, follow this order — **each layer must complete before starting the next**:

```
platform → edge → secrets → compute → data → network → dns
```

---

## Option C — Emergency: Destroy from the CLI

If the pipeline is unavailable, destroy directly with Terraform:

```bash
# Clone the repo
git clone https://github.com/Medaea-IAC/infra-live.git
cd infra-live

# Set AWS credentials
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-east-1

# Configure GH_TOKEN for private module access
git config --global url."https://x-access-token:<GH_TOKEN>@github.com".insteadOf "https://github.com"

# Destroy each layer in order
for layer in platform edge secrets compute data network dns; do
  echo "=== Destroying $layer ==="
  cd environments/dev/$layer
  terraform init -input=false
  terraform destroy -input=false -auto-approve
  cd ../../..
done
```

---

## After a Full Destroy — Re-Deploying

After all layers are destroyed, re-deploy from scratch the same way as the first deploy:

1. Push to `develop` (or trigger `workflow_dispatch`)
2. The `ensure-iam` job recreates all IAM policies automatically
3. The `ensure-backend` job recreates the S3 state bucket automatically
4. All layers deploy in sequence

**No manual steps required** — the pipeline is fully self-healing.

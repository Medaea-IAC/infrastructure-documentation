# Deploy Guide

> Step-by-step instructions for deploying Medaea infrastructure — from a clean AWS account to fully live.

---

## Prerequisites (One-time Setup)

### 1. AWS — Bootstrap IAM user

Run once in AWS CloudShell or locally with admin credentials:

```bash
# Create the deploy user
aws iam create-user --user-name github-deploy-user-medaea-01

# Create access keys and save them — you'll add these to GitHub Secrets
aws iam create-access-key --user-name github-deploy-user-medaea-01

# Apply the self-service bootstrap inline policy (lets the user manage its own policies)
aws iam put-user-policy \
  --user-name github-deploy-user-medaea-01 \
  --policy-name MedaeaSelfServiceBootstrap \
  --policy-document file://iam/self-service-bootstrap-policy.json
```

See `iam/BOOTSTRAP.md` for the full inline policy content.

### 2. GitHub — Add repository secrets

Go to **Settings → Secrets and variables → Actions** in the `Medaea-IAC/infra-live` repo and add:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | From step 1 access key output |
| `AWS_SECRET_ACCESS_KEY` | From step 1 access key output |
| `GH_TOKEN` | GitHub PAT with `repo` scope (for cloning infra-modules) |

### 3. (Optional) Create S3 state bucket manually

The pipeline creates this automatically. Skip this step unless you need it ahead of time:

```bash
aws s3api create-bucket \
  --bucket medaea-dev-terraform-state \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket medaea-dev-terraform-state \
  --versioning-configuration Status=Enabled
```

---

## Day-to-Day Workflow

### What happens when you push to `develop`

Every push to `develop` **only runs terraform plan** — never apply. The `plan.yml` workflow:

1. Detects which layer files changed (e.g. `environments/dev/compute/**`)
2. Runs `terraform plan` only for those layers
3. Reports pass/fail per layer in the Actions UI

No infrastructure is created or changed by a push. This is intentional.

### How to actually deploy

Go to **Actions → Apply Infrastructure → Run workflow**:

```
Environment:  dev        ← always dev unless staging/prod is ready
Target:       all        ← "all" for a full deploy, or pick a specific layer
Plan only:    false      ← set true to preview first
```

Click **Run workflow**.

---

## Full Deploy — Clean Account (First Time)

Run these steps in order. Each `Apply Infrastructure` run is a manual trigger.

### Step 1 — Apply dns (ACM certificate)

ACM certificate validation can take 5–30 minutes. This must complete before edge.

```
Target: dns
```

Wait for the certificate to reach `Issued` status in ACM before proceeding.

```bash
aws acm list-certificates --region us-east-1 \
  --query 'CertificateSummaryList[*].{Domain:DomainName,Status:Status}'
```

### Step 2 — Apply network (VPC + subnets + NAT)

```
Target: network
```

Verify after:

```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=medaea-dev-vpc" \
  --query 'Vpcs[].{VpcId:VpcId,CIDR:CidrBlock,State:State}'
```

### Step 3 — Apply data (RDS + ElastiCache + KMS)

RDS PostgreSQL provisioning takes 5–15 minutes.

```
Target: data
```

Verify RDS is available:

```bash
aws rds describe-db-instances \
  --query 'DBInstances[*].{ID:DBInstanceIdentifier,Status:DBInstanceStatus,Engine:EngineVersion}'
```

### Step 4 — Apply secrets (Secrets Manager / SSM)

Populates `/dev/db-host`, `/dev/redis-url`, and other SSM parameters that ECS tasks read at startup.

```
Target: secrets
```

### Step 5 — Apply compute (ECS cluster + ALB + ECR)

Creates the ECS cluster (`medaea-dev`), Application Load Balancer, ECR repositories, and CloudWatch log groups.

```
Target: compute
```

Verify ALB is active:

```bash
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[*].{Name:LoadBalancerName,State:State.Code,DNS:DNSName}'
```

### Step 6 — Apply edge (CloudFront + WAF + Route53)

Depends on both `dns` (ACM cert must be Issued) and `compute` (ALB DNS needed as origin).
CloudFront distribution deployment takes 5–15 minutes.

```
Target: edge
```

Verify distributions are deployed:

```bash
aws cloudfront list-distributions \
  --query 'DistributionList.Items[*].{Domain:DomainName,Status:Status,Origins:Origins.Items[0].DomainName}'
```

### Step 7 — Apply platform (ECS services + CodeDeploy)

Push Docker images to ECR **before** running this step, or ECS tasks will fail to start.

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  009952409575.dkr.ecr.us-east-1.amazonaws.com

# Push API image
docker build -t medaea-api ./api
docker tag medaea-api:latest \
  009952409575.dkr.ecr.us-east-1.amazonaws.com/medaea-dev/api:latest
docker push 009952409575.dkr.ecr.us-east-1.amazonaws.com/medaea-dev/api:latest

# Push frontend image (if applicable)
docker build -t medaea-frontend ./frontend
docker tag medaea-frontend:latest \
  009952409575.dkr.ecr.us-east-1.amazonaws.com/medaea-dev/frontend:latest
docker push 009952409575.dkr.ecr.us-east-1.amazonaws.com/medaea-dev/frontend:latest
```

Then trigger:

```
Target: platform
```

---

## Partial Deploy — Single Layer

Use this when only one layer changed and you don't need to re-run everything.

**Example: compute layer changed (new ECS task definition)**

```
Actions → Apply Infrastructure → Run workflow
  Environment: dev
  Target:      compute
  Plan only:   false
```

Only the `compute` job runs. All other layers are skipped. `ensure-iam` and `ensure-backend` still run first (they are always part of every apply).

---

## Preview Before Applying (Plan Only)

To see what Terraform would change without making any changes:

```
Actions → Apply Infrastructure → Run workflow
  Environment: dev
  Target:      all          ← or specific layer
  Plan only:   true
```

Check the Actions run output — each layer's plan shows resources to be added/changed/destroyed.

---

## Verifying a Successful Full Deploy

After all 7 layers are green, verify end-to-end:

```bash
# DNS resolves
dig dev.medaea.net
dig dev.ehr.medaea.net
dig dev.api.medaea.net

# HTTPS responds
curl -I https://dev.medaea.net
curl -I https://dev.ehr.medaea.net
curl -I https://dev.api.medaea.net/health
```

In the AWS Console:
- **CloudFront** — 3 distributions in `Deployed` state
- **ECS** — services running, desired count = running count
- **RDS** — instance in `available` state
- **ElastiCache** — cluster in `available` state

---

## Deploying Application Updates (No Infrastructure Change)

After pushing new Docker images to ECR, trigger a CodeDeploy blue/green deployment:

```bash
# Force new ECS deployment (CodeDeploy handles the blue/green swap)
aws ecs update-service \
  --cluster medaea-dev \
  --service medaea-dev-api \
  --force-new-deployment

aws ecs update-service \
  --cluster medaea-dev \
  --service medaea-dev-frontend \
  --force-new-deployment
```

Or trigger through the CodeDeploy console for a controlled rollout with health check gates.

**Note:** If you changed the ECS task definition in Terraform, run `Target: platform` instead.

---

## Common Issues

### "Error: AccessDenied — iam:CreatePolicyVersion"

The deploy user is missing the self-service bootstrap inline policy.  
Run the bootstrap step from Prerequisites again with admin credentials.

### "Error: NoSuchBucket"

The S3 state bucket doesn't exist yet. Run `Target: dns` or `Target: network` first — `ensure-backend` creates the bucket automatically.

### "Error: InvalidParameterException — RDS engine version"

Only `16.4` is available for PostgreSQL in us-east-1 at the time of writing. Check `environments/dev/data/main.tf` and ensure `engine_version = "16.4"`.

### Plan passes but apply fails on edge

Usually means ACM certificate is not yet `Issued`. Wait and retry edge after the cert validates.

See `docs/troubleshooting.md` for more.

# Deploy Guide

> Step-by-step instructions for deploying the Medaea infrastructure from a clean AWS account to fully live.

---

## Prerequisites

Before running the pipeline for the first time, complete these one-time manual steps.

### 1. AWS — Create the deploy IAM user

```bash
aws iam create-user --user-name github-deploy-user-medaea-01

# Create and download access keys
aws iam create-access-key --user-name github-deploy-user-medaea-01
# ↑ Save AccessKeyId and SecretAccessKey — you'll need them for GitHub secrets
```

### 2. AWS — Attach the self-service bootstrap policy

The pipeline manages its own IAM policies (creates and updates them). For this to work, the deploy user needs a small inline policy that allows it to manage only the three `MedaeaGitHubDeployPolicy-*` policies.

```bash
aws iam put-user-policy \
  --user-name github-deploy-user-medaea-01 \
  --policy-name MedaeaSelfServiceBootstrap \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "iam:CreatePolicy",
        "iam:CreatePolicyVersion",
        "iam:DeletePolicyVersion",
        "iam:GetPolicy",
        "iam:GetPolicyVersion",
        "iam:ListPolicyVersions",
        "iam:AttachUserPolicy",
        "iam:DetachUserPolicy",
        "iam:ListAttachedUserPolicies"
      ],
      "Resource": "arn:aws:iam::009952409575:policy/MedaeaGitHubDeployPolicy-*"
    }]
  }'
```

### 3. AWS — Create the Route53 hosted zone

The `medaea.net` hosted zone must exist before any Terraform runs (the `dns` layer validates against it).

```bash
aws route53 create-hosted-zone \
  --name medaea.net \
  --caller-reference medaea-$(date +%s)
# Note the hosted zone ID from the output (format: Z0392791162ZTG8WYEICP)
```

Update your domain registrar's nameservers to the four NS records Route53 gives you.

### 4. GitHub — Add repository secrets

In `Medaea-IAC/infra-live` → Settings → Secrets and variables → Actions:

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key from step 1 |
| `AWS_SECRET_ACCESS_KEY` | Secret key from step 1 |
| `GH_TOKEN` | GitHub PAT with `repo` scope (for reading private infra-modules) |

### 5. GitHub — Add repository variables (optional)

| Variable name | Default if not set | Purpose |
|---|---|---|
| `ALERT_EMAIL` | `ops@medaea.net` | Email address for CloudWatch alarm SNS notifications |

---

## First Deploy — From Scratch

Once prerequisites are complete, push any change to the `develop` branch (or trigger manually via `workflow_dispatch`).

The pipeline runs all layers automatically in dependency order:

```
push to develop
      │
      ▼
 safety-gate ──► ensure-iam ──► ensure-backend
                                      │
                          ┌───────────┴───────────┐
                          ▼                       ▼
                         dns                   network
                          │                       │
                          │                  ┌────┴────┐
                          │                  ▼         ▼
                          │                data     (data)
                          │                  │         │
                          │             secrets    compute
                          │                           │
                          └──────────┬────────────────┘
                                     ▼
                              ┌──────┴──────┐
                              ▼             ▼
                             edge        platform
```

**Expected total time for a clean first deploy: 25–40 minutes.**
Most of that is RDS (~10 min to create) and CloudFront (~5–10 min to propagate).

---

## Triggering a Deploy Manually

Go to: **GitHub → `Medaea-IAC/infra-live` → Actions → Deploy Infrastructure → Run workflow**

| Input | Options | Default | Notes |
|---|---|---|---|
| `environment` | `dev`, `staging`, `prod` | `dev` | `prod` requires `workflow_dispatch` — push is blocked |
| `plan_only` | `true` / `false` | `false` | Set to `true` to preview changes without applying |
| `deploy_dns` | `true` / `false` | `false` | Force-run the dns layer even without changes |
| `deploy_network` | `true` / `false` | `false` | Force-run the network layer |
| `deploy_data` | `true` / `false` | `false` | Force-run the data layer |
| `deploy_compute` | `true` / `false` | `false` | Force-run the compute layer |
| `deploy_edge` | `true` / `false` | `false` | Force-run the edge layer |
| `deploy_platform` | `true` / `false` | `false` | Force-run the platform layer |

On a normal `push` to `develop`, all layers run automatically — the individual flags are only needed for manual targeted re-runs.

---

## Re-Deploying After IAM Policy Changes

When a policy JSON file (`iam/github-deploy-policy-*.json`) is updated on a branch:

1. Merge the branch into `develop`
2. The next pipeline run will call `ensure-iam` which automatically creates a new policy version and sets it as default
3. Old policy versions are pruned automatically (AWS allows max 5 versions)

**You do not need to manually apply policy changes** — the `ensure-iam` job handles it on every run.

---

## Re-Running a Single Failed Layer

If one layer fails and you want to re-run just that layer without re-running everything:

1. Go to **Actions → Deploy Infrastructure → Run workflow**
2. Set `plan_only = false`
3. Set **only** the failed layer's flag to `true` (e.g. `deploy_compute = true`)
4. All other layer flags stay `false`

This skips unchanged layers and only applies the one you need.

---

## Verifying a Successful Deploy

After all jobs turn green, verify end-to-end:

```bash
# DNS resolves correctly
dig dev.medaea.net
dig dev.ehr.medaea.net
dig dev.api.medaea.net

# HTTPS responds
curl -I https://dev.medaea.net
curl -I https://dev.ehr.medaea.net
curl -I https://dev.api.medaea.net/health
```

In the AWS Console, confirm:
- **CloudFront** — 3 distributions in `Deployed` state
- **ECS** — services running with desired count = actual count
- **RDS** — instance in `available` state
- **ElastiCache** — cluster in `available` state

---

## Updating Application Images (ECS)

After pushing new Docker images to ECR:

1. The `platform` layer CodeDeploy deployment group automatically handles blue/green
2. Trigger a new ECS deployment:

```bash
aws ecs update-service \
  --cluster medaea-dev \
  --service <service-name> \
  --force-new-deployment
```

Or trigger through the CodeDeploy console for a controlled blue/green swap.

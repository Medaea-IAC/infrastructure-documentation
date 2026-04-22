# Manual Steps Log

This document records every manual action that was required during infrastructure setup, the reason it was needed, and the current status.

---

## Step 1 — Apply MedaeaSelfServiceBootstrap Inline Policy

**Status:** Required once, never again  
**When required:** First-ever pipeline run (or after account reset)  
**Reason:** AWS IAM cannot bootstrap itself. A user cannot grant itself permissions it doesn't already have. This is a hard security constraint in every AWS account.

**What to do:**

Open AWS CloudShell (terminal icon in top-right of AWS Console) and run:

```bash
aws iam put-user-policy \
  --user-name github-deploy-user-medaea-01 \
  --policy-name MedaeaSelfServiceBootstrap \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Sid":"ManageOwnDeployPolicies","Effect":"Allow","Action":["iam:CreatePolicy","iam:GetPolicy","iam:GetPolicyVersion","iam:CreatePolicyVersion","iam:ListPolicyVersions","iam:DeletePolicyVersion","iam:ListPolicies"],"Resource":"arn:aws:iam::009952409575:policy/MedaeaGitHubDeployPolicy*"},{"Sid":"AttachOwnDeployPolicies","Effect":"Allow","Action":["iam:AttachUserPolicy","iam:ListAttachedUserPolicies"],"Resource":"arn:aws:iam::009952409575:user/github-deploy-user-medaea-01"}]}'
```

**Why CloudShell:** CloudShell automatically uses your console login session as credentials. No keys to configure. Runs in browser. Free.

**After this:** Every future pipeline run (including destroy/rehydrate) is fully automatic.

**Does not survive destroy?** The inline policy is on the IAM user itself, not on any infrastructure resource. Destroying and re-creating VPCs, ECS clusters, RDS, etc. does NOT affect this inline policy. It only needs to be reapplied if the IAM user itself is deleted.

---

## Step 2 — Set GitHub Secrets

**Status:** One-time setup in GitHub repository settings  
**When required:** Before first pipeline run

Go to `Medaea-IAC/infra-live` → Settings → Secrets and variables → Actions → New repository secret:

| Secret Name | Value | Notes |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Deploy user access key | From IAM → Users → github-deploy-user-medaea-01 → Security credentials |
| `AWS_SECRET_ACCESS_KEY` | Deploy user secret key | Only shown once at creation time |
| `GH_TOKEN` | GitHub PAT | Needs `repo` scope for private module access in infra-modules |

These secrets are permanent — they do not need to be rotated unless the key is compromised.

---

## Step 3 — GitHub Environment Setup

**Status:** One-time setup  
**When required:** Before first pipeline run using `workflow_dispatch`

Go to `Medaea-IAC/infra-live` → Settings → Environments → New environment:
- Name: `dev`
- No protection rules required for dev

This is referenced by `environment: dev` in each Terraform layer job.

---

## What Has Never Required Manual Action

Once the above steps are done:

- Creating/destroying VPC, subnets, route tables — automatic
- Creating/destroying ECS cluster, services, task definitions — automatic
- Creating/destroying RDS, ElastiCache — automatic
- Creating/destroying CloudFront distributions — automatic
- ACM certificate issuance and DNS validation — automatic
- SSM parameter creation — automatic
- IAM role creation for ECS tasks — automatic
- CodeDeploy application/deployment group creation — automatic
- Terraform state management — automatic (S3 bucket created by ensure-backend)
- IAM managed policy creation and attachment — automatic (after Step 1 above)

---

## One-time: Restore Secrets Manager Secrets Scheduled for Deletion

**When:** After a partial Terraform apply creates secrets, then fails, leaving them
in scheduled-deletion state. Symptom: `You can't create this secret because a secret
with this name is already scheduled for deletion.`

**Run in AWS CloudShell (account 009952409575, region us-east-1):**

```bash
for secret in django-secret-key openai-api-key jwt-secret redis-password; do
  aws secretsmanager restore-secret \
    --secret-id "medaea/dev/${secret}" \
    --region us-east-1 \
    && echo "Restored: medaea/dev/${secret}" \
    || echo "Not scheduled / already active: medaea/dev/${secret}"
done
```

**After restoring:** re-run the pipeline. Terraform will import/adopt the existing secrets
rather than recreating them (assuming they are already in state, or run `terraform import`
if not).

**Going forward:** `recovery_window_in_days = 0` is now set for dev in the secrets module,
so future `terraform destroy` calls will force-delete immediately without the recovery window.

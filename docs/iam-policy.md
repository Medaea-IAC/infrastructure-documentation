# IAM Policy Architecture

## Overview

The deploy user `github-deploy-user-medaea-01` (AWS account `009952409575`) runs all GitHub Actions jobs. It manages its own permissions via a self-managed bootstrap pattern, requiring no admin credentials in the pipeline.

## The Bootstrap Problem

AWS IAM has a hard constraint: a user cannot grant itself permissions it doesn't already have. This creates a one-time chicken-and-egg bootstrap requirement.

**Resolution:** A narrow "self-service" inline policy is applied to the user **once** via AWS CloudShell. After that, the pipeline is fully self-managing forever.

## Bootstrap Inline Policy — `MedaeaSelfServiceBootstrap`

Applied once to `github-deploy-user-medaea-01`. Grants only:

| Action | Scoped To |
|---|---|
| `iam:CreatePolicy` | `arn:aws:iam::009952409575:policy/MedaeaGitHubDeployPolicy*` |
| `iam:GetPolicy` | same |
| `iam:GetPolicyVersion` | same |
| `iam:CreatePolicyVersion` | same |
| `iam:ListPolicyVersions` | same |
| `iam:DeletePolicyVersion` | same |
| `iam:ListPolicies` | same |
| `iam:AttachUserPolicy` | `arn:aws:iam::009952409575:user/github-deploy-user-medaea-01` |
| `iam:ListAttachedUserPolicies` | same |

**This is not admin access.** It cannot touch any other IAM resource.

### One-Time CloudShell Command

```bash
aws iam put-user-policy \
  --user-name github-deploy-user-medaea-01 \
  --policy-name MedaeaSelfServiceBootstrap \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Sid":"ManageOwnDeployPolicies","Effect":"Allow","Action":["iam:CreatePolicy","iam:GetPolicy","iam:GetPolicyVersion","iam:CreatePolicyVersion","iam:ListPolicyVersions","iam:DeletePolicyVersion","iam:ListPolicies"],"Resource":"arn:aws:iam::009952409575:policy/MedaeaGitHubDeployPolicy*"},{"Sid":"AttachOwnDeployPolicies","Effect":"Allow","Action":["iam:AttachUserPolicy","iam:ListAttachedUserPolicies"],"Resource":"arn:aws:iam::009952409575:user/github-deploy-user-medaea-01"}]}'
```

## Deploy Permissions — Three Managed Policies

AWS managed policies have a **6,144-character size limit**. The full permission set (12,757 chars) was split into three policies:

### MedaeaGitHubDeployPolicy-1 (4,804 chars)
Edge and network layer permissions:
- **S3** — state bucket, media bucket (CreateBucket, PutObject, GetObject, etc.)
- **Route53** — hosted zone reads/writes for DNS records and ACM validation
- **ACM** — certificate request, describe, delete, list
- **CloudFront** — distributions, OAC, invalidations
- **WAFv2** — WebACL, IP sets, association
- **EC2/VPC** — VPC, subnets, IGW, NAT, EIPs, route tables, security groups

### MedaeaGitHubDeployPolicy-2 (4,785 chars)
Compute and data layer permissions:
- **ELB** — load balancers, target groups, listeners, rules
- **ECS** — clusters, task definitions, services
- **ECR** — repositories, lifecycle policies, image scanning
- **RDS** — DB instances, subnet groups, parameter groups
- **ElastiCache** — cache clusters, subnet groups, parameter groups
- **Secrets Manager** — create, get, update, delete secrets
- **SSM** — get/put/delete parameters

### MedaeaGitHubDeployPolicy-3 (3,267 chars)
Operations and orchestration permissions:
- **CloudWatch** — metric alarms
- **Logs** — log groups, retention policies, streams
- **SNS** — topics, subscriptions (for CloudWatch alerts)
- **IAM** — roles, instance profiles, policies (for ECS task roles, CodeDeploy)
- **CodeDeploy** — applications, deployment groups, deployments
- **KMS** — decrypt, generate data key (for encrypted secrets)

## Pipeline ensure-iam Job

The `ensure-iam` job runs automatically at the start of every pipeline execution:

```
For each of the 3 policies:
  1. Check if policy ARN exists in IAM
  2a. If NOT exists → iam:CreatePolicy
  2b. If EXISTS → check version count → prune oldest if ≥5 → iam:CreatePolicyVersion
  3. Check if policy is attached to the deploy user
  4a. If NOT attached → iam:AttachUserPolicy
  4b. If ATTACHED → log SKIP, no API call
```

This means:
- **First run:** creates all 3 policies + attaches them (~15s)
- **Subsequent runs:** verifies attachment, logs SKIP (~5s)
- **After destroy/rehydrate:** same as first run — policies recreated automatically
- **After policy change:** new policy version created, old one pruned

## Why Not Admin Credentials?

Admin credentials in a CI pipeline are a security risk:
- If secrets are leaked, the attacker has full AWS access
- Rotating admin credentials is complex and disruptive
- Audit trails become unclear (pipeline actions mixed with human actions)

The self-managed pattern limits the blast radius: if `AWS_ACCESS_KEY_ID` is leaked, an attacker can only manage the three `MedaeaGitHubDeployPolicy*` policies and nothing else — until the deploy user's own broader permissions are active.

# IAM Reference

> IAM policies for `github-deploy-user-medaea-01` — what each policy contains, current sizes, and how they are automatically maintained.

---

## How IAM Policies Are Managed

The pipeline is **self-managing**. On every run, the `ensure-iam` job:

1. Checks whether each of the three `MedaeaGitHubDeployPolicy-*` policies exists
2. Creates it if missing, or creates a new policy version if the JSON has changed
3. Prunes the oldest version if the 5-version AWS limit would be exceeded
4. Checks whether each policy is attached to `github-deploy-user-medaea-01`
5. Attaches it if not already attached

**This means:** after merging a branch that updates a policy JSON file, the next pipeline run automatically applies the change — no manual `aws iam` commands needed.

---

## Bootstrap Requirement (one-time only)

For the `ensure-iam` job to work, the deploy user needs a small inline policy before the first run:

```bash
aws iam put-user-policy \
  --user-name github-deploy-user-medaea-01 \
  --policy-name MedaeaSelfServiceBootstrap \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "iam:CreatePolicy", "iam:CreatePolicyVersion", "iam:DeletePolicyVersion",
        "iam:GetPolicy", "iam:GetPolicyVersion", "iam:ListPolicyVersions",
        "iam:AttachUserPolicy", "iam:DetachUserPolicy", "iam:ListAttachedUserPolicies"
      ],
      "Resource": "arn:aws:iam::009952409575:policy/MedaeaGitHubDeployPolicy-*"
    }]
  }'
```

This is scoped to only the three `MedaeaGitHubDeployPolicy-*` policies — not full IAM admin.

---

## Policy Files

All three policy JSON files live in `infra-live/iam/`.

| File | Policy name | AWS size limit | Current size |
|---|---|---|---|
| `github-deploy-policy-1.json` | `MedaeaGitHubDeployPolicy-1` | 6,144 chars | ~5,153 chars |
| `github-deploy-policy-2.json` | `MedaeaGitHubDeployPolicy-2` | 6,144 chars | ~5,645 chars |
| `github-deploy-policy-3.json` | `MedaeaGitHubDeployPolicy-3` | 6,144 chars | ~4,246 chars |

---

## Policy 1 — Networking, Edge, WAF

**Statement: VPCAndNetworking**
- `ec2:*` — VPC, subnets, IGW, NAT gateway, route tables, security groups, EIPs, network interfaces
- `ec2:GetSecurityGroupsForVpc` — required by ELBv2 CreateLoadBalancer internally

**Statement: WAFv2**
Full CRUD for WAF WebACLs and logging:
- `wafv2:CreateWebACL`, `wafv2:UpdateWebACL`, `wafv2:DeleteWebACL`, `wafv2:GetWebACL`, `wafv2:ListWebACLs`
- `wafv2:CreateIPSet`, `wafv2:UpdateIPSet`, `wafv2:DeleteIPSet`, `wafv2:GetIPSet`, `wafv2:ListIPSets`
- `wafv2:AssociateWebACL`, `wafv2:DisassociateWebACL`, `wafv2:GetWebACLForResource`
- `wafv2:PutLoggingConfiguration`, `wafv2:GetLoggingConfiguration`, `wafv2:DeleteLoggingConfiguration`
- `wafv2:TagResource`, `wafv2:UntagResource`, `wafv2:ListTagsForResource`

**Statement: ACMAndCloudFront**
- `acm:*` — certificates, validation, tagging
- `cloudfront:*` — distributions, OAC, cache policies, invalidations

**Statement: Route53**
- `route53:*` — hosted zones, resource record sets, health checks

---

## Policy 2 — ELB, ECS, ECR, RDS, ElastiCache, SSM, Secrets

**Statement: ELB**
Full CRUD for Application Load Balancers:
- `elasticloadbalancing:CreateLoadBalancer`, `DeleteLoadBalancer`, `DescribeLoadBalancers`, `DescribeLoadBalancerAttributes`, `ModifyLoadBalancerAttributes`
- `elasticloadbalancing:CreateTargetGroup`, `DeleteTargetGroup`, `DescribeTargetGroups`, `DescribeTargetGroupAttributes`, `ModifyTargetGroup`, `ModifyTargetGroupAttributes`
- `elasticloadbalancing:CreateListener`, `DeleteListener`, `DescribeListeners`, `ModifyListener`, `DescribeListenerAttributes`
- `elasticloadbalancing:CreateRule`, `DeleteRule`, `DescribeRules`, `ModifyRule`
- `elasticloadbalancing:AddTags`, `RemoveTags`, `DescribeTags`, `SetSecurityGroups`, `SetSubnets`

**Statement: ECSAndECR**
- `ecs:*` — clusters, task definitions, services, tasks
- `ecr:*` — repositories, images, lifecycle policies, scanning

**Statement: RDSAndElastiCache**
- `rds:*` — DB instances, subnet groups, parameter groups, snapshots
- `elasticache:*` — replication groups, subnet groups, parameter groups

**Statement: SecretsAndSSM**
- `secretsmanager:*` — create, get, update, delete, restore, rotate secrets
- `ssm:*` — get, put, delete parameters; describe parameter group

---

## Policy 3 — CloudWatch, Logs, SNS, IAM, CodeDeploy, KMS

**Statement: CloudWatchAndLogs**

CloudWatch:
- `cloudwatch:PutMetricAlarm`, `DeleteAlarms`, `DescribeAlarms`, `GetMetricStatistics`, `ListTagsForResource`

CloudWatch Logs (full CRUD including resource policies):
- `logs:CreateLogGroup`, `logs:DeleteLogGroup`, `logs:DescribeLogGroups`, `logs:DescribeLogStreams`
- `logs:PutRetentionPolicy`, `logs:DeleteRetentionPolicy`
- `logs:PutResourcePolicy`, `logs:DescribeResourcePolicies`, `logs:DeleteResourcePolicy`
- `logs:CreateLogDelivery`, `logs:TagResource`, `logs:UntagResource`, `logs:ListTagsForResource`

SNS (full CRUD for alert topics):
- `sns:CreateTopic`, `sns:DeleteTopic`, `sns:GetTopicAttributes`, `sns:SetTopicAttributes`
- `sns:Subscribe`, `sns:Unsubscribe`, `sns:GetSubscriptionAttributes`
- `sns:ListTopics`, `sns:ListSubscriptionsByTopic`, `sns:TagResource`, `sns:ListTagsForResource`

**Statement: IAMForECSAndCodeDeploy**
- `iam:CreateRole`, `iam:DeleteRole`, `iam:GetRole`, `iam:ListRoles`
- `iam:AttachRolePolicy`, `iam:DetachRolePolicy`, `iam:ListAttachedRolePolicies`
- `iam:PassRole` (scoped to ECS task and CodeDeploy roles)
- `iam:CreateInstanceProfile`, `iam:DeleteInstanceProfile`, `iam:AddRoleToInstanceProfile`
- `iam:CreateOpenIDConnectProvider`, `iam:DeleteOpenIDConnectProvider`

**Statement: CodeDeploy**
- `codedeploy:CreateApplication`, `DeleteApplication`, `GetApplication`, `ListApplications`
- `codedeploy:CreateDeploymentGroup`, `DeleteDeploymentGroup`, `GetDeploymentGroup`, `UpdateDeploymentGroup`
- `codedeploy:CreateDeployment`, `GetDeployment`, `ListDeployments`, `StopDeployment`
- `codedeploy:RegisterApplicationRevision`, `GetDeploymentConfig`

**Statement: KMSForEncryption**
Full CRUD for KMS keys used by RDS, ElastiCache, Secrets Manager, S3:
- `kms:CreateKey`, `kms:DescribeKey`, `kms:ListKeys`, `kms:ListAliases`, `kms:ListResourceTags`
- `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey`, `kms:GenerateDataKeyWithoutPlaintext`
- `kms:PutKeyPolicy`, `kms:GetKeyPolicy`, `kms:EnableKeyRotation`, `kms:DisableKeyRotation`, `kms:GetKeyRotationStatus`
- `kms:ScheduleKeyDeletion`, `kms:CancelKeyDeletion`
- `kms:CreateAlias`, `kms:DeleteAlias`, `kms:UpdateAlias`
- `kms:CreateGrant`, `kms:RetireGrant`, `kms:RevokeGrant`, `kms:ListGrants`
- `kms:ReEncryptFrom`, `kms:ReEncryptTo`
- `kms:TagResource`, `kms:UntagResource`

---

## Adding a New Permission

When the pipeline fails with `AccessDenied`:

1. Identify the action from the error message (e.g. `elasticloadbalancing:DescribeListenerAttributes`)
2. Find which policy file contains related actions for that service
3. Add the action to the correct statement in the JSON file
4. Verify the file stays under 6,144 characters
5. Merge the branch — the next pipeline run applies it automatically via `ensure-iam`

**Pattern:** AWS Terraform provider always calls `Describe/Get/Read` after every `Create` — if you add `CreateX`, also add `DescribeX`/`GetX` and `DeleteX` to avoid hitting the same error on the next run.

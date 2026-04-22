# Current Pipeline State

> Single source of truth. Overwritten on each iteration — do not use deployment-history.md for current values.

## IAM Policies (github-deploy-user-medaea-01)

| File | Sid(s) | Size | Limit |
|---|---|---|---|
| `github-deploy-policy-1.json` | NetworkAndEdge | ~4987 chars | 6144 |
| `github-deploy-policy-2.json` | RDSAndElastiCache, SecretsAndSSM | **5585 chars** | 6144 |
| `github-deploy-policy-3.json` | CloudWatchAndLogs, IAMForECSAndCodeDeploy, CodeDeploy, KMSForEncryption | **3840 chars** | 6144 |

### Actions added per phase
| Phase | Policy | Actions added |
|---|---|---|
| 10 | policy-3 KMSForEncryption | CreateKey, TagResource, PutKeyPolicy, EnableKeyRotation, ScheduleKeyDeletion, CreateAlias, DeleteAlias + 8 related |
| 10 | policy-3 IAMForECSAndCodeDeploy | CreateServiceLinkedRole, DeleteServiceLinkedRole, GetServiceLinkedRoleDeletionStatus |
| 11 | policy-2 RDSAndElastiCache | CreateReplicationGroup, DeleteReplicationGroup, DescribeReplicationGroups, ModifyReplicationGroup, IncreaseReplicaCount, DecreaseReplicaCount, DescribeCacheEngineVersions, CreateSnapshot, DeleteSnapshot, DescribeSnapshots + 1 |
| 11 | policy-2 SecretsAndSSM | GetResourcePolicy, PutResourcePolicy, DeleteResourcePolicy, RestoreSecret, RotateSecret, CancelRotateSecret, ListSecretVersionIds |

## Module Defaults (infra-modules @ develop)

| Module | Variable | Current default | Notes |
|---|---|---|---|
| `modules/rds` | `engine_version` | `"15.5"` | 16.1 and 16.2 absent in us-east-1 |
| `modules/rds` | `multi_az` | `false` | Dev only |
| `modules/redis` | `auth_token` | `null` | Optional; transit encryption always on |
| `modules/redis` | `num_cache_clusters` | `2` | Set to 1 in dev live config |
| `modules/s3` | `replication_destination_bucket_arn` | `null` | Replication gated on non-null |

## Checkov Skips in Modules

| Check | Resource | Module | Reason |
|---|---|---|---|
| CKV2_AWS_64 | `aws_kms_key.this` | redis, s3, secrets-manager | Policy explicitly defined via `aws_iam_policy_document.kms` |
| CKV2_AWS_50 | `aws_elasticache_replication_group.this` | redis | Failover conditional on `num_cache_clusters >= 2`; AWS rejects it for single-node. Skip placed **inside** resource block (line-before format not recognised by Checkov) |

## Open PRs — Pending Merge

> Merge infra-modules before infra-live (live config references `ref=develop`).

### infra-modules
| Branch | What it fixes |
|---|---|
| `feature/MEP-52-optional-s3-replication` | S3 replication optional |
| `feature/MEP-52-redis-module-dev-friendly` | Redis: optional auth_token, single-node allowed |
| `feature/MEP-52-module-fixes-batch1` | S3 lifecycle `filter {}`; RDS version default |
| `feature/MEP-52-rds-version-15-5` | RDS version → 15.5 |
| `feature/MEP-52-checkov-redis-failover-skip` | Checkov CKV2_AWS_50 skip on replication group |
| `feature/MEP-52-checkov-skip-inside-block` | Move CKV2_AWS_50 skip inside resource block (Checkov format fix) |

### infra-live
| Branch | What it fixes |
|---|---|
| `feature/MEP-52-split-iam-policy` | Split policy into 3 files |
| `feature/MEP-52-patch-ec2-describe-permissions` | EC2 describe actions |
| `feature/MEP-52-self-managed-iam-permissions` | Self-managed IAM bootstrap |
| `feature/MEP-52-dynamic-iam-in-pipeline` | Pipeline upserts IAM on every run |
| `feature/MEP-52-fix-deploy-pipeline` | Base pipeline fix |
| `feature/MEP-52-patch-kms-and-elasticache-iam` | KMS write + ElastiCache SLR |
| `feature/MEP-52-patch-elasticache-secretsmanager` | ElastiCache replication + SecretsManager resource policy |

## Known Patterns

**AWS Terraform provider calls `Describe*`/`Get*` after every `Create*`** — when a new resource type is added, its read-back action is likely missing from the policy. Add it to the appropriate policy file and re-run.

**Checkov unconditional checks vs. conditional logic** — use `#checkov:skip=<CHECK_ID>:<justification>` when the module handles the requirement conditionally (e.g., failover enabled for num_cache_clusters >= 2). Document the skip in the table above.

## RDS PostgreSQL Availability in us-east-1

| Version | Result |
|---|---|
| 16.2 | ❌ Not found |
| 16.1 | ❌ Not found |
| **15.5** | ✅ Current default |

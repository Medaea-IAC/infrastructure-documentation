# Current Pipeline State

> Updated after each fix iteration. This file reflects the definitive state — do not use deployment-history.md for current values.

## IAM Policies (github-deploy-user-medaea-01)

| File | Sid(s) | Size | Limit |
|---|---|---|---|
| `github-deploy-policy-1.json` | NetworkAndEdge | ~4987 chars | 6144 |
| `github-deploy-policy-2.json` | RDSAndElastiCache, SecretsAndSSM | **5585 chars** | 6144 |
| `github-deploy-policy-3.json` | CloudWatchAndLogs, IAMForECSAndCodeDeploy, CodeDeploy, KMSForEncryption | **3840 chars** | 6144 |

### Key actions added per phase
| Phase | Policy | Actions added |
|---|---|---|
| 10 | policy-3 KMSForEncryption | CreateKey, TagResource, PutKeyPolicy, EnableKeyRotation, ScheduleKeyDeletion, CreateAlias, DeleteAlias + 8 related |
| 10 | policy-3 IAMForECSAndCodeDeploy | CreateServiceLinkedRole, DeleteServiceLinkedRole, GetServiceLinkedRoleDeletionStatus |
| 11 | policy-2 RDSAndElastiCache | CreateReplicationGroup, DeleteReplicationGroup, DescribeReplicationGroups, ModifyReplicationGroup, IncreaseReplicaCount, DecreaseReplicaCount, DescribeCacheEngineVersions, CreateSnapshot, DeleteSnapshot, DescribeSnapshots + 1 more |
| 11 | policy-2 SecretsAndSSM | GetResourcePolicy, PutResourcePolicy, DeleteResourcePolicy, RestoreSecret, RotateSecret, CancelRotateSecret, ListSecretVersionIds |

## Module Defaults (infra-modules @ develop)

| Module | Variable | Current default | Notes |
|---|---|---|---|
| `modules/rds` | `engine_version` | `"15.5"` | 16.x not available in us-east-1 via RDS |
| `modules/rds` | `multi_az` | `false` | Dev only |
| `modules/redis` | `auth_token` | `null` | Optional; transit encryption still on |
| `modules/redis` | `num_cache_clusters` | `2` | Set to 1 in dev live config |
| `modules/s3` | `replication_destination_bucket_arn` | `null` | Replication gated on non-null |

## Open PRs — Pending Merge

> Merge order: infra-modules first, then infra-live (live config references module ref=develop)

### infra-modules
| Branch | What it fixes |
|---|---|
| `feature/MEP-52-optional-s3-replication` | S3 replication optional |
| `feature/MEP-52-redis-module-dev-friendly` | Redis: optional auth_token, single-node allowed |
| `feature/MEP-52-module-fixes-batch1` | S3 lifecycle filter {}; RDS version default |
| `feature/MEP-52-rds-version-15-5` | RDS version → 15.5 |

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

## Known IAM Pattern

The AWS Terraform provider calls `Describe*` and `Get*` after every `Create*`. When a new resource type is added, expect its read-back action to be missing from the policy and appear as AccessDenied on the first apply. Add the action to the appropriate policy and re-run.

## RDS PostgreSQL Availability in us-east-1

| Version tried | Result |
|---|---|
| 16.2 | ❌ Not found |
| 16.1 | ❌ Not found |
| **15.5** | ✅ Used |

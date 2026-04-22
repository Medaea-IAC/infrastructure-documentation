# Current Pipeline State

> Single source of truth. Overwritten on each iteration.

## IAM Policies (github-deploy-user-medaea-01)

| File | Sid(s) | Size | Limit |
|---|---|---|---|
| `github-deploy-policy-1.json` | NetworkAndEdge | ~4987 chars | 6144 |
| `github-deploy-policy-2.json` | RDSAndElastiCache, SecretsAndSSM | **5585 chars** | 6144 |
| `github-deploy-policy-3.json` | CloudWatchAndLogs, IAMForECSAndCodeDeploy, CodeDeploy, KMSForEncryption | **3840 chars** | 6144 |

## Module Defaults (infra-modules @ develop)

| Module | Variable | Current default | Notes |
|---|---|---|---|
| `modules/rds` | `engine_version` | `"16.4"` | 16.1, 16.2, 15.5 absent in us-east-1 |
| `modules/rds` | `multi_az` | `false` | Dev only |
| `modules/redis` | `auth_token` | `null` | Optional; transit encryption always on |
| `modules/redis` | `num_cache_clusters` | `2` | Set to `1` in dev live config |
| `modules/s3` | `replication_destination_bucket_arn` | `null` | Replication gated on non-null |
| `modules/secrets-manager` | `recovery_window_in_days` | `0` (dev) / `30` (prod) | ForceDeleteWithoutRecovery for dev |

## KMS Key Policies — Service Principals

| Module | Service principal added | Actions |
|---|---|---|
| `modules/redis` | `elasticache.amazonaws.com` | Decrypt, GenerateDataKey, CreateGrant, DescribeKey, ReEncrypt*, ListGrants |

## Checkov Skips in Modules

| Check | Resource | Module | Skip placement |
|---|---|---|---|
| CKV2_AWS_64 | `aws_kms_key.this` | redis, s3, secrets-manager | Line before resource |
| CKV2_AWS_50 | `aws_elasticache_replication_group.this` | redis | **Inside** resource block (line-before format not recognised) |

## Open PRs — Pending Merge

> Merge infra-modules before infra-live.

### infra-modules
| Branch | What it fixes |
|---|---|
| `feature/MEP-52-checkov-skip-inside-block` | CKV2_AWS_50 skip inside resource block |
| `feature/MEP-52-rds-kms-secrets-fix` | RDS version 16.4; ElastiCache KMS grant; Secrets recovery window 0 |

### infra-live
| Branch | What it fixes |
|---|---|
| `feature/MEP-52-patch-elasticache-secretsmanager` | ElastiCache replication + SecretsManager resource policy actions |

## RDS PostgreSQL Version History in us-east-1

| Version tried | Result |
|---|---|
| 16.2 | ❌ Not found |
| 16.1 | ❌ Not found |
| 15.5 | ❌ Not found |
| **16.4** | ✅ Current default |

## Known Patterns

- **AWS Terraform provider calls `Describe*`/`Get*` after every `Create*`** — missing read-back actions cause AccessDenied on first apply. Add to the appropriate policy file.
- **ElastiCache KMS** — requires explicit `elasticache.amazonaws.com` service principal in key policy; root account statement alone is insufficient for service principals.
- **Secrets Manager recreation after failed apply** — failed applies with default recovery window leave secrets in scheduled-deletion state. ForceDeleteWithoutRecovery (`recovery_window_in_days = 0`) for non-prod prevents this. Already-scheduled secrets require manual restore (see `docs/manual-steps.md`).

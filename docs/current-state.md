# Current Pipeline State

> Single source of truth. Overwritten on each iteration.

## IAM Policies (github-deploy-user-medaea-01)

| File | Sid(s) | Size | Limit |
|---|---|---|---|
| `github-deploy-policy-1.json` | NetworkAndEdge | ~4987 chars | 6144 |
| `github-deploy-policy-2.json` | RDSAndElastiCache, SecretsAndSSM | 5585 chars | 6144 |
| `github-deploy-policy-3.json` | CloudWatchAndLogs, IAMForECSAndCodeDeploy, CodeDeploy, KMSForEncryption | **4050 chars** | 6144 |

### KMS actions in policy-3 (full list)
`Decrypt, Encrypt, GenerateDataKey, GenerateDataKeyWithoutPlaintext, DescribeKey, ListAliases, ListKeys, ListResourceTags, ListGrants, CreateKey, TagResource, UntagResource, PutKeyPolicy, GetKeyPolicy, EnableKeyRotation, DisableKeyRotation, GetKeyRotationStatus, ScheduleKeyDeletion, CancelKeyDeletion, CreateAlias, DeleteAlias, UpdateAlias, CreateGrant, RetireGrant, RevokeGrant, ReEncryptFrom, ReEncryptTo`

## Module Defaults (infra-modules @ develop)

| Module | Variable | Current default | Notes |
|---|---|---|---|
| `modules/rds` | `engine_version` | `"16.4"` | 16.1, 16.2, 15.5 absent in us-east-1 |
| `modules/rds` | `multi_az` | `false` | Dev only |
| `modules/redis` | `auth_token` | `null` | Optional; transit encryption always on |
| `modules/redis` | `num_cache_clusters` | `2` | Set to `1` in dev live config |
| `modules/s3` | `replication_destination_bucket_arn` | `null` | Replication gated on non-null |
| `modules/secrets-manager` | `recovery_window_in_days` | `0` (dev) / `30` (prod) | ForceDeleteWithoutRecovery for dev |

## KMS Key Policies — Service Principals Added

| Module | Service principal | Actions |
|---|---|---|
| `modules/redis` | `elasticache.amazonaws.com` | Decrypt, GenerateDataKey, CreateGrant, DescribeKey, ReEncrypt*, ListGrants |

## Secrets State — Dev

| Secret name | ARN | Status |
|---|---|---|
| `medaea/dev/django-secret-key` | `...:medaea/dev/django-secret-key-VupIOV` | Imported via `import` block |
| `medaea/dev/jwt-secret` | `...:medaea/dev/jwt-secret-eq0lET` | Imported via `import` block |
| `medaea/dev/openai-api-key` | `...:medaea/dev/openai-api-key-Rq6RxV` | Imported via `import` block |
| `medaea/dev/redis-password` | `...:medaea/dev/redis-password-usfqrU` | Imported via `import` block |

## Open PRs — Pending Merge

> Merge infra-modules before infra-live.

### infra-modules (merge to develop)
| Branch | What it fixes |
|---|---|
| `feature/MEP-52-checkov-skip-inside-block` | CKV2_AWS_50 skip inside resource block |
| `feature/MEP-52-rds-kms-secrets-fix` | RDS 16.4; ElastiCache KMS service grant; Secrets recovery=0 |

### infra-live (merge to develop)
| Branch | What it fixes |
|---|---|
| `feature/MEP-52-kms-grant-and-secrets-import` | `kms:CreateGrant` in policy-3; import blocks for 4 dev secrets |

## Checkov Skips

| Check | Resource | Module | Placement |
|---|---|---|---|
| CKV2_AWS_64 | `aws_kms_key.this` | redis, s3, secrets-manager | Line before resource |
| CKV2_AWS_50 | `aws_elasticache_replication_group.this` | redis | **Inside** resource block |

## RDS PostgreSQL Versions Tried in us-east-1

| Version | Result |
|---|---|
| 16.2 | ❌ | 16.1 | ❌ | 15.5 | ❌ | **16.4** | ✅ |

## Known Patterns

- **KMS + service principal**: Root account statement in key policy covers IAM entities only. Service principals (e.g., `elasticache.amazonaws.com`) need a separate `AllowService` statement. The **deploy user** also needs `kms:CreateGrant` in their IAM policy to create KMS grants for those services.
- **Secrets Manager after failed apply**: Secrets left in scheduled-deletion state → manual `restore-secret`; restored secrets land outside Terraform state → `import` blocks to adopt them.
- **Terraform `import` blocks are idempotent** — safe to leave in permanently once resources are in state.

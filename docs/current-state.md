# Current Pipeline State

> Single source of truth. Overwritten on each iteration. Last updated: Phase 22+.

---

## IAM Policies — github-deploy-user-medaea-01

| File | AWS Policy Name | Size | Limit | Status |
|---|---|---|---|---|
| `github-deploy-policy-1.json` | `MedaeaGitHubDeployPolicy-1` | ~5,153 chars | 6,144 | ✅ OK |
| `github-deploy-policy-2.json` | `MedaeaGitHubDeployPolicy-2` | ~5,645 chars | 6,144 | ✅ OK |
| `github-deploy-policy-3.json` | `MedaeaGitHubDeployPolicy-3` | ~4,246 chars | 6,144 | ✅ OK |

### Complete Action List — Policy 1 (VPCAndNetworking, WAFv2, ACM, CloudFront, Route53)

Key additions vs original:
- `ec2:GetSecurityGroupsForVpc` — needed by ELBv2 CreateLoadBalancer
- `wafv2:PutLoggingConfiguration`, `wafv2:GetLoggingConfiguration`, `wafv2:DeleteLoggingConfiguration`

### Complete Action List — Policy 2 (ELB, ECS, ECR, RDS, ElastiCache, SSM, Secrets)

Key additions vs original:
- `elasticloadbalancing:DescribeListenerAttributes`

### Complete Action List — Policy 3 (CloudWatch, Logs, SNS, IAM, CodeDeploy, KMS)

Key additions vs original:
- `cloudwatch:ListTagsForResource`
- `logs:PutResourcePolicy`, `logs:DescribeResourcePolicies`, `logs:DeleteResourcePolicy`
- `sns:GetSubscriptionAttributes`
- All KMS grant actions: `kms:CreateGrant`, `kms:RetireGrant`, `kms:RevokeGrant`, `kms:ListGrants`, `kms:ReEncryptFrom`, `kms:ReEncryptTo`

---

## Module Defaults (infra-modules @ develop)

| Module | Variable | Current default | Notes |
|---|---|---|---|
| `modules/rds` | `engine_version` | `"16.4"` | 16.1, 16.2, 15.5 absent in us-east-1 |
| `modules/rds` | `multi_az` | `false` | Dev only |
| `modules/redis` | `auth_token` | `null` | Optional; transit encryption always on |
| `modules/redis` | `num_cache_clusters` | `2` | Set to `1` in dev live config |
| `modules/s3` | `replication_destination_bucket_arn` | `null` | Replication gated on non-null |
| `modules/secrets-manager` | `recovery_window_in_days` | `0` (dev) | ForceDeleteWithoutRecovery for dev |
| `modules/cloudfront` | `create_oac_policy` | `true` | Boolean — avoids unknown count at plan time |

---

## ACM Certificate SANs (dns layer)

```hcl
domain_name               = "medaea.net"
subject_alternative_names = [
  "*.medaea.net",        # covers dev.medaea.net, etc.
  "*.ehr.medaea.net",    # covers dev.ehr.medaea.net
  "*.api.medaea.net",    # covers dev.api.medaea.net
]
```

A single `*.medaea.net` wildcard only covers ONE level — `*.ehr.medaea.net` and `*.api.medaea.net` are required for two-level subdomains.

---

## KMS Key Policies — Service Principals

| Module | Service principal | Reason |
|---|---|---|
| `modules/redis` | `elasticache.amazonaws.com` | ElastiCache needs to decrypt data at rest |

Deploy user needs `kms:CreateGrant` in policy-3 to establish grants for service principals.

---

## Secrets State — Dev

| Secret name | Status |
|---|---|
| `medaea/dev/django-secret-key` | Imported via `import {}` block |
| `medaea/dev/jwt-secret` | Imported via `import {}` block |
| `medaea/dev/openai-api-key` | Imported via `import {}` block |
| `medaea/dev/redis-password` | Imported via `import {}` block |

---

## Edge Layer — Key Configurations

| Resource | Configuration |
|---|---|
| Route53 records | `allow_overwrite = true` on all 3 alias records |
| Compute remote state | `defaults = { alb_dns_name = "", alb_zone_id = "" }` |
| Platform `aws_account_id` | Replaced with `data.aws_caller_identity.current.account_id` |
| WAF log group resource policy | `aws_cloudwatch_log_resource_policy` grants `delivery.logs.amazonaws.com` write access |

---

## Compute Layer Variables

| Variable | Source | Value |
|---|---|---|
| `acm_wildcard_cert_arn` | Removed — auto-read from dns remote state | `data.terraform_remote_state.dns.outputs.acm_certificate_arn` |
| `alert_email` | GitHub repo variable `vars.ALERT_EMAIL` | Fallback default: `ops@medaea.net` |

---

## Checkov Skips

| Check | Resource | Module | Placement rule |
|---|---|---|---|
| `CKV2_AWS_64` | `aws_kms_key.this` | redis, s3, secrets-manager | Line before resource block |
| `CKV2_AWS_50` | `aws_elasticache_replication_group.this` | redis | **Inside** resource block (not line-before) |

---

## Destroy Workflows

| Workflow | File | Trigger |
|---|---|---|
| Destroy All (sequential) | `destroy-all.yml` | `workflow_dispatch` — type `destroy-all` to confirm |
| Destroy Single Layer | `destroy.yml` | `workflow_dispatch` — type `destroy` to confirm |

Destroy order: `platform → edge → secrets → compute → data → network → dns`

---

## RDS PostgreSQL Versions Tried in us-east-1

| Version | Result |
|---|---|
| 16.2 | ❌ Not available |
| 16.1 | ❌ Not available |
| 15.5 | ❌ Not available |
| **16.4** | ✅ Working |

---

## Known Patterns

| Pattern | Rule |
|---|---|
| `count` at plan time | Never use resource attribute values in `count`. Add a boolean variable with `default = true/false` |
| Required variables | Use `data.aws_caller_identity` for account IDs; `data.terraform_remote_state` for cross-layer values; always add `default` |
| Remote state missing | Add `defaults {}` block so layers can plan before dependencies are applied |
| `for_each` outputs | Use `values(resource.this)[0]` not `one(resource.this)` |
| Terraform Create + Read | After any `Create*` IAM permission, also add `Describe*/Get*` (Terraform reads state after every create) and `Delete*` (needed for destroy) |
| KMS service principals | Root key policy covers IAM principals only. Service principals (e.g. `elasticache.amazonaws.com`) need a separate statement AND the deploy user needs `kms:CreateGrant` |
| WAF log delivery | `delivery.logs.amazonaws.com` needs a CloudWatch log group **resource policy** (not an IAM user policy) |
| Route53 re-creates | Use `allow_overwrite = true` on alias records to handle records that exist from previous partial applies |
| Checkov skip placement | `CKV2_AWS_50` requires the comment **inside** the resource block |

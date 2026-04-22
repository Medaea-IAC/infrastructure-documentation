# Troubleshooting Guide

> Every error encountered in the Medaea infrastructure pipeline and exactly how it was resolved. Ordered by layer.

---

## How to Read This Guide

Each entry follows the pattern:
- **Error** — exact message or key phrase from the pipeline output
- **Layer** — which Terraform layer failed
- **Root cause** — why it happened
- **Fix** — what was changed to resolve it

---

## dns Layer

### InvalidViewerCertificate — CloudFront distribution creation fails

**Error:**
```
InvalidViewerCertificate: The certificate that is attached to your distribution
doesn't cover the alternate domain name (CNAME) that you're trying to add.
```

**Layer:** edge (triggered by dns misconfiguration)

**Root cause:**
The ACM wildcard cert `*.medaea.net` covers only ONE level of subdomain. `dev.medaea.net` is covered, but `dev.ehr.medaea.net` and `dev.api.medaea.net` are two levels deep and are NOT covered.

**Fix:**
Added `*.ehr.medaea.net` and `*.api.medaea.net` as Subject Alternative Names in `environments/dev/dns/main.tf`:
```hcl
subject_alternative_names = [
  "*.medaea.net",
  "*.ehr.medaea.net",
  "*.api.medaea.net",
]
```
After merging, the dns layer must be re-applied before edge. The ACM module handles DNS validation automatically.

---

## network Layer

_(No errors recorded — network layer applied cleanly.)_

---

## data Layer

### RDS engine version not available in us-east-1

**Error:**
```
InvalidParameterCombination: Cannot find version X.Y for postgres
```

**Layer:** data

**Root cause:** PostgreSQL versions 16.1, 16.2, and 15.5 are not available in `us-east-1` for the chosen instance class.

**Fix:** Set `engine_version = "16.4"` in the RDS module. Verified working in us-east-1.

---

### Checkov CKV2_AWS_50 — ElastiCache replication group skip not recognized

**Error:** Checkov scan fails with CKV2_AWS_50 on the ElastiCache replication group.

**Root cause:** The `#checkov:skip` comment must be placed **inside** the resource block, not on the line before it.

**Fix:**
```hcl
resource "aws_elasticache_replication_group" "this" {
  #checkov:skip=CKV2_AWS_50: AUTH token optional for dev; encryption-in-transit always enabled
  ...
}
```

---

### ElastiCache KMS — service principal not in key policy

**Error:**
```
AuthorizationError: elasticache.amazonaws.com is not authorized to use key
```

**Root cause:** The KMS key policy only had a root account statement, which covers IAM principals — not AWS service principals. `elasticache.amazonaws.com` needs its own explicit statement.

**Fix:** Added a separate `AllowElastiCacheService` statement to the KMS key policy in the redis module. Also added `kms:CreateGrant` to policy-3 so the deploy user can create KMS grants for service principals.

---

### Secrets Manager — secrets in scheduled-deletion state after failed apply

**Error:**
```
ResourceExistsException: A resource with the ID already exists
```

**Root cause:** A previous failed apply left secrets in scheduled-deletion state. Terraform tries to re-create them but AWS sees them as existing.

**Fix:**
1. Manually restore each secret: `aws secretsmanager restore-secret --secret-id <ARN>`
2. Add `import {}` blocks in `secrets/main.tf` to adopt the existing secrets into state

---

## compute Layer

### ec2:GetSecurityGroupsForVpc — AccessDenied on CreateLoadBalancer

**Error:**
```
AccessDenied: User is not authorized to perform: ec2:GetSecurityGroupsForVpc
```

**Layer:** compute

**Root cause:** ELBv2 `CreateLoadBalancer` internally calls `ec2:GetSecurityGroupsForVpc` to validate the security groups. This is an internal AWS call that still requires IAM permission.

**Fix:** Added `ec2:GetSecurityGroupsForVpc` to the `VPCAndNetworking` statement in policy-1.

---

### SNS:GetSubscriptionAttributes — AccessDenied

**Error:**
```
AuthorizationError: User is not authorized to perform: SNS:GetSubscriptionAttributes
```

**Layer:** compute

**Root cause:** Terraform reads back `aws_sns_topic_subscription` state after creating it using `GetSubscriptionAttributes`. This action was missing from the SNS block in policy-3.

**Fix:** Added `sns:GetSubscriptionAttributes` to policy-3 `CloudWatchAndLogs` statement.

---

### cloudwatch:ListTagsForResource — AccessDenied

**Error:**
```
AccessDenied: User is not authorized to perform: cloudwatch:ListTagsForResource
```

**Layer:** compute

**Root cause:** Terraform reads tags on `aws_cloudwatch_metric_alarm` for drift detection. `ListTagsForResource` is separate from the other CloudWatch permissions.

**Fix:** Added `cloudwatch:ListTagsForResource` to policy-3 `CloudWatchAndLogs` statement.

---

### elasticloadbalancing:DescribeListenerAttributes — AccessDenied

**Error:**
```
AccessDenied: User is not authorized to perform: elasticloadbalancing:DescribeListenerAttributes
```

**Layer:** compute

**Root cause:** After creating `aws_lb_listener`, Terraform calls `DescribeListenerAttributes` to read back the resource state. This is separate from `DescribeListeners`.

**Fix:** Added `elasticloadbalancing:DescribeListenerAttributes` to the ELB statement in policy-2.

---

## edge Layer

### No value for required variable: aws_account_id (platform layer)

**Error:**
```
No value for required variable: aws_account_id — has no default value
```

**Layer:** platform

**Root cause:** `aws_account_id` was a required variable with no default, only used to construct SSM parameter ARNs.

**Fix:** Removed the variable. Replaced with `data "aws_caller_identity" "current"` which self-resolves from the AWS credentials in use. No pipeline variable needed.

---

### Unable to find remote state — compute layer

**Error:**
```
Unable to find remote state. No stored state was found for the given workspace
```

**Layer:** edge

**Root cause:** `data.terraform_remote_state.compute` fails hard when the compute state file doesn't exist yet (first run, or compute failed).

**Fix:** Added a `defaults` block to the compute remote state data source in edge/main.tf:
```hcl
defaults = {
  alb_dns_name = ""
  alb_zone_id  = ""
}
```
Edge can now plan even when compute hasn't been applied yet.

---

### Invalid count argument — CloudFront OAC bucket policy

**Error:**
```
Invalid count argument — The "count" value depends on resource attributes
that cannot be determined until apply
```

**Layer:** edge

**Root cause:** The CloudFront module used `count = var.s3_bucket_id != "" ? 1 : 0`. Callers pass `s3_bucket_id = module.website_s3.bucket_id` which is unknown at plan time.

**Fix:** Added a `create_oac_policy` boolean variable (`default = true`) to the CloudFront module. Count is now `var.create_oac_policy ? 1 : 0` — always deterministic at plan time.

---

### Route53 record already exists

**Error:**
```
InvalidChangeBatch: Tried to create resource record set but it already exists
```

**Layer:** edge

**Root cause:** Route53 A records were created by a previous partial apply. Terraform doesn't have them in state and tries to `CREATE` instead of `UPSERT`.

**Fix:** Added `allow_overwrite = true` to all three Route53 alias records (website, ehr, api). This is safe to keep permanently — alias records are idempotent.

---

### WAFLogDestinationPermissionIssueException

**Error:**
```
WAFLogDestinationPermissionIssueException: Unable to deliver logs to the configured destination
```

**Layer:** edge

**Root cause:** WAF's log delivery service (`delivery.logs.amazonaws.com`) is a service principal that needs a **CloudWatch log group resource policy** granting it write access. This is NOT controlled via IAM user policies.

**Fix:** Added `aws_cloudwatch_log_resource_policy` and `data.aws_iam_policy_document` to the WAF module granting `delivery.logs.amazonaws.com` permissions:
- `logs:CreateLogStream`
- `logs:PutLogEvents`

Also added `depends_on = [aws_cloudwatch_log_resource_policy.waf_log_delivery]` to the `aws_wafv2_web_acl_logging_configuration` resource so the policy is always in place first.

---

### wafv2:PutLoggingConfiguration — AccessDenied

**Error:**
```
AccessDenied: User is not authorized to perform: wafv2:PutLoggingConfiguration
```

**Layer:** edge

**Root cause:** Action was missing from the WAFv2 statement in policy-1.

**Fix:** Added `wafv2:PutLoggingConfiguration`, `wafv2:GetLoggingConfiguration`, and `wafv2:DeleteLoggingConfiguration` to policy-1 (complete CRUD set for WAF logging config).

---

### logs:PutResourcePolicy / DescribeResourcePolicies / DeleteResourcePolicy — AccessDenied

**Errors (three separate runs):**
```
AccessDenied: User is not authorized to perform: logs:PutResourcePolicy
AccessDenied: User is not authorized to perform: logs:DescribeResourcePolicies
AccessDenied: User is not authorized to perform: logs:DeleteResourcePolicy
```

**Layer:** edge

**Root cause:** `aws_cloudwatch_log_resource_policy` requires separate IAM permissions from regular log group management. The three actions needed are:
- `PutResourcePolicy` — CREATE/UPDATE the policy
- `DescribeResourcePolicies` — READ back after create
- `DeleteResourcePolicy` — DELETE on destroy/replace

**Fix:** Added all three to the `CloudWatchAndLogs` statement in policy-3.

---

### wafv2:GetLoggingConfiguration — AccessDenied

**Error:**
```
AccessDenied: User is not authorized to perform: wafv2:GetLoggingConfiguration
```

**Layer:** edge

**Root cause:** Terraform reads back `aws_wafv2_web_acl_logging_configuration` after creating it. `GetLoggingConfiguration` is separate from `PutLoggingConfiguration`.

**Fix:** Added `wafv2:GetLoggingConfiguration` and `wafv2:DeleteLoggingConfiguration` to policy-1 (proactively completing the full CRUD set).

---

## platform Layer

_(Covered under edge layer errors above — the `aws_account_id` issue was technically in the platform layer.)_

---

## General Patterns

### Terraform always calls Describe/Get after Create

Whenever a new AWS resource type is added to the Terraform config, expect to need both:
- The `Create*` action
- The corresponding `Describe*` / `Get*` / `List*` action (Terraform reads state after every create)
- The `Delete*` action (needed for destroy and replace operations)

When you see one, add all three proactively.

### count cannot depend on unknown-at-plan-time values

If `count` is computed from a resource attribute (e.g. `module.x.bucket_id`), Terraform fails at plan time with "Invalid count argument" because the value won't be known until apply.

**Fix:** Add a separate boolean variable with a `default` so the count is always deterministic:
```hcl
variable "create_oac_policy" {
  type    = bool
  default = true
}
# count = var.create_oac_policy ? 1 : 0
```

### data.terraform_remote_state fails if state doesn't exist

Remote state data sources fail hard if the referenced state file doesn't exist. Add a `defaults` block for any values consumed from another layer:
```hcl
defaults = {
  alb_dns_name = ""
  alb_zone_id  = ""
}
```

### Required variables without defaults block the plan

Never leave a required Terraform variable with no default and no pipeline wiring:
- For account IDs: use `data "aws_caller_identity" "current"`
- For cross-layer values: use `data.terraform_remote_state.X.outputs.Y`
- For optional config: add a sensible `default`

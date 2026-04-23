# Security, Cybersecurity & ONC Compliance

> Security controls, breach prevention, incident response, and ONC Health IT validation requirements
> as applied to the current Medaea EHR infrastructure on AWS.

---

## 1. Current Security Controls in the Infrastructure

Every layer of the Medaea stack applies security controls by default. The table below maps each control to the Terraform resource that enforces it.

### Encryption

| What | How | Where |
|---|---|---|
| RDS data at rest | `storage_encrypted = true` + AWS-managed KMS key | `modules/rds/main.tf` |
| RDS in transit | `rds.force_ssl = 1` via DB parameter group | `modules/rds/main.tf` |
| ElastiCache at rest | KMS CMK (`aws_kms_key`) | `modules/redis/main.tf` |
| ElastiCache in transit | `transit_encryption_enabled = true` | `modules/redis/main.tf` |
| S3 media bucket at rest | `AES256` server-side encryption | `modules/s3/main.tf` |
| S3 media bucket access | Public access blocked on all four settings | `modules/s3/main.tf` |
| HTTPS to end users | CloudFront enforces HTTPS-only, TLS 1.2 min | `modules/cloudfront/main.tf` |
| ALB to ECS | HTTPS listener on port 443 | `modules/compute/main.tf` |
| Secrets | AWS Secrets Manager (not env vars or SSM plaintext) | `modules/secrets-manager/` |

### Network Isolation

```
Internet → CloudFront → WAFv2 → ALB (public subnet)
                                     │
                              ECS Fargate (private subnet)
                                     │
                    RDS + ElastiCache (private subnet, no public access)
```

- RDS and ElastiCache are in **private subnets** with no internet route
- Security groups restrict RDS to port 5432 (only from ECS SG)
- Security groups restrict Redis to port 6379 (only from ECS SG)
- All egress from ECS goes through NAT Gateway (no direct internet attach)

### Edge Protection

| Control | Service | Config |
|---|---|---|
| DDoS protection | AWS Shield Standard | Automatic on CloudFront + ALB |
| Web Application Firewall | WAFv2 | Managed rules: `AWSManagedRulesCommonRuleSet`, `AWSManagedRulesSQLiRuleSet` |
| Rate limiting | WAFv2 rule | 2,000 req/5 min per IP (configurable) |
| Bot protection | WAFv2 `AWSManagedRulesBotControlRuleSet` | Attached to CloudFront |
| HTTPS only | CloudFront viewer policy | `redirect-to-https` enforced |
| HSTS | CloudFront response headers policy | `Strict-Transport-Security: max-age=31536000` |

### Access Control

| What | Control |
|---|---|
| AWS API access | Single deploy IAM user, least-privilege split across 3 policies |
| ECS task access | Per-task IAM roles — no wildcard `*` resource on sensitive actions |
| Secrets access | ECS task role grants `secretsmanager:GetSecretValue` only on its own secrets |
| No hardcoded credentials | All secrets via Secrets Manager; no env vars with plaintext passwords |
| State bucket access | S3 bucket policy restricts access to deploy user + versioning enabled |

### Audit & Logging

| Log | Destination | Retention |
|---|---|---|
| ALB access logs | S3 | 90 days |
| CloudFront access logs | S3 | 90 days |
| WAFv2 logs | CloudWatch log group `/aws/waf/medaea-dev` | 90 days |
| RDS logs (PostgreSQL + upgrade) | CloudWatch log group | 30 days |
| ECS task logs | CloudWatch log group `/ecs/medaea-dev-*` | 30 days |
| VPC Flow Logs | CloudWatch | 30 days |

---

## 2. HIPAA Security Rule Compliance

The Medaea EHR handles **Protected Health Information (PHI)**. The HIPAA Security Rule (45 CFR Part 164) requires these safeguards:

### Administrative Safeguards

| Requirement | Implementation |
|---|---|
| Security Officer designation | Assign a named Security Officer responsible for HIPAA compliance |
| Risk Analysis | Annual risk assessment covering AWS infrastructure, application code, and third-party integrations |
| Workforce training | All engineers and clinical staff complete HIPAA training annually |
| Access management | User access reviewed quarterly; offboarding removes AWS IAM + GitHub access within 24 hours |
| Audit controls | CloudWatch + WAF logs retained and reviewed for anomalous access patterns |
| Contingency plan | Disaster recovery: RDS automated backups (7-day retention), S3 versioning, multi-region failover plan |

### Physical Safeguards

AWS manages physical access controls for data centers (SOC 2 Type II, ISO 27001 certified). No on-premise components.

### Technical Safeguards

| Requirement | Implementation |
|---|---|
| Access control | IAM roles per ECS task; Secrets Manager for credentials |
| Audit controls | All API calls logged to CloudTrail; CloudWatch for app-level events |
| Integrity | RDS checksums + encrypted storage; S3 integrity checks |
| Person authentication | MFA enforced on all AWS Console logins (IAM policy + SCP) |
| Transmission security | TLS 1.2+ enforced at CloudFront and ALB; `rds.force_ssl=1` |

### Business Associate Agreements (BAA)

AWS provides a BAA covering all AWS services used in this stack. Sign the AWS BAA before storing any PHI:
- AWS Console → Account → Agreements → Business Associate Addendum

---

## 3. ONC Health IT Certification — §170.315 Criteria

The Office of the National Coordinator (ONC) certifies EHR systems under the **21st Century Cures Act**. The relevant certification criteria for the Medaea EHR:

### §170.315(d) — Privacy & Security Criteria (Required for Certification)

| Criterion | Requirement | Medaea Implementation |
|---|---|---|
| **(d)(1)** Authentication, access control | Role-based access control; unique user ID per provider | Application-level RBAC + Cognito/Auth (app layer) |
| **(d)(2)** Auditable events and audit log status | Immutable audit log for all PHI access/modify/delete | CloudTrail (infrastructure) + application audit log (app layer) |
| **(d)(3)** Audit report(s) | Generate audit reports for specified time periods | CloudWatch Insights queries + Athena on S3 logs |
| **(d)(4)** Amendments | Patient ability to request record amendments | Application feature (not infrastructure) |
| **(d)(5)** Automatic log-off | Session timeout after inactivity | Application-level (JWT expiry, frontend idle timeout) |
| **(d)(6)** Emergency access | Break-glass access procedure documented | Documented in runbooks; separate emergency IAM role |
| **(d)(7)** End-user device encryption | Enforce HTTPS to all devices | CloudFront HTTPS-only enforced |
| **(d)(8)** Integrity | Verify PHI has not been altered in transit or at rest | TLS in transit; KMS + checksums at rest |
| **(d)(9)** Trusted connection | TLS for all external connections | ALB + CloudFront enforce TLS 1.2+ |
| **(d)(10)** Auditing actions on health information | Log all create/read/update/delete on patient records | Application audit log (PostgreSQL triggers or app-level) |
| **(d)(12)** Encrypt authentication credentials | Passwords hashed (bcrypt/argon2); never stored plaintext | Application layer (Django password hashing) |
| **(d)(13)** Multi-factor authentication | MFA for all providers accessing PHI | AWS Cognito MFA (app layer) or TOTP |

### §170.315(g) — Patient Access & Interoperability

| Criterion | Requirement | Medaea Implementation |
|---|---|---|
| **(g)(6)** Consolidated CDA creation | Export patient summaries as C-CDA | Application feature |
| **(g)(7)** Application access — patient selection | SMART on FHIR authorization | API layer (FastAPI + FHIR R4) |
| **(g)(8)** Application access — data category | FHIR R4 API for USCDI data categories | `dev.api.medaea.net` (ECS FastAPI service) |
| **(g)(9)** Application access — all data | Patient-facing API with SMART scopes | Application layer |
| **(g)(10)** Standardized API — patient and population services | FHIR R4 compliant, HL7 bulk export | Application layer |

### USCDI v3 Data Requirements

The infrastructure must support storage and retrieval of USCDI v3 data classes:

| USCDI Category | Storage Location |
|---|---|
| Patient demographics | PostgreSQL (`medaea_dev` database) |
| Clinical notes | PostgreSQL + S3 (large documents) |
| Laboratory results | PostgreSQL |
| Medications | PostgreSQL |
| Immunizations | PostgreSQL |
| Diagnostic imaging reports | S3 (`medaea-dev-media` bucket) |
| Vital signs | PostgreSQL |
| Care team members | PostgreSQL |

---

## 4. Cybersecurity Threat Landscape & Mitigations

### Threat Matrix — MITRE ATT&CK for Cloud

| Threat | Risk | Current Mitigation | Gap / Recommendation |
|---|---|---|---|
| **SQL Injection** | Critical | WAFv2 SQLi managed rule set | Add app-level parameterized queries (Django ORM) |
| **Credential theft** | Critical | Secrets Manager (no plaintext); IAM role per task | Enable IAM Access Analyzer; rotate keys quarterly |
| **Ransomware / data exfiltration** | High | VPC private subnets; no direct DB internet access | Enable GuardDuty; S3 Object Lock for critical data |
| **Insider threat** | High | Least-privilege IAM; CloudTrail all API calls | Enable CloudTrail data events on S3 + RDS |
| **DDoS** | High | Shield Standard + WAFv2 rate limiting | Consider Shield Advanced for SLA-backed protection |
| **SSRF (Server-Side Request Forgery)** | High | ECS metadata endpoint blocked via task IAM role | Add IMDSv2 enforcement on EC2/ECS metadata |
| **Container escape** | Medium | Fargate (no host access; AWS-managed hypervisor) | Fargate eliminates container escape to host |
| **Malicious Docker image** | Medium | ECR private repos; image tag immutability | Enable ECR image scanning (Amazon Inspector) |
| **Man-in-the-middle** | Medium | TLS 1.2+ end-to-end; HSTS enabled | Pin TLS certificates at ALB; monitor cert expiry |
| **Secrets in code/logs** | Medium | Secrets Manager; no env var plaintext | Scan repos with git-secrets or Gitleaks in CI |
| **Unpatched dependencies** | Medium | Auto minor version upgrade on RDS | Enable Dependabot/Renovate for application deps |
| **WAF bypass** | Low | AWS Managed Rules + custom rate limits | Log all WAF ALLOW decisions; review anomalies monthly |

### Recommended Additional Controls (Not Yet Implemented)

```
Priority 1 — Enable immediately (low cost, high impact):
  □ AWS GuardDuty          — threat detection on CloudTrail/VPC/DNS logs
  □ AWS Security Hub       — aggregated security posture score
  □ Amazon Inspector       — ECR image vulnerability scanning
  □ CloudTrail data events — log S3 GetObject + RDS queries

Priority 2 — Enable before production:
  □ AWS Config             — compliance rules for resource configuration drift
  □ AWS Macie              — PHI/PII detection in S3 buckets
  □ Shield Advanced        — DDoS SLA + response team (if >$3k/month infra)
  □ VPC Flow Logs to SIEM  — ship logs to Splunk/Datadog for alerting

Priority 3 — Ongoing:
  □ Quarterly penetration test (required for many HIPAA auditors)
  □ Annual third-party security audit
  □ SOC 2 Type II certification (required by enterprise health system customers)
```

---

## 5. Breach Detection & Incident Response Plan

### Detection Sources

| Source | What it catches | Where to check |
|---|---|---|
| GuardDuty findings | Compromised credentials, unusual API calls, crypto mining, port scanning | AWS Console → GuardDuty → Findings |
| WAFv2 blocked requests | SQL injection, XSS, rate-limit violations | CloudWatch → `/aws/waf/medaea-dev` |
| CloudTrail anomalies | Unexpected IAM actions, new user creation, policy changes | CloudTrail → Event history |
| RDS slow query log | Unusually large data dumps (possible exfiltration) | CloudWatch → `/aws/rds/instance/medaea-dev-ehr-dev/postgresql` |
| ALB 4xx/5xx spike | Brute force, scanning, application errors | CloudWatch → `AWS/ApplicationELB` metrics |
| VPC Flow Logs | Unexpected outbound connections from private subnets | CloudWatch Insights on VPC flow log group |

### Incident Response Runbook

**Step 1 — Contain**
```bash
# Immediately revoke the compromised deploy user key
aws iam update-access-key \
  --user-name github-deploy-user-medaea-01 \
  --access-key-id <KEY_ID> \
  --status Inactive

# Block a specific IP at WAF (emergency)
aws wafv2 create-ip-set \
  --name "emergency-block" \
  --scope CLOUDFRONT \
  --ip-address-version IPV4 \
  --addresses "1.2.3.4/32"
```

**Step 2 — Assess scope**
```bash
# Check all API calls made by the compromised principal in last 24h
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=github-deploy-user-medaea-01 \
  --start-time $(date -d '24 hours ago' --iso-8601=seconds) \
  --query 'Events[*].{Time:EventTime,Event:EventName,Resource:Resources[0].ResourceName}'

# Check for new IAM users or policy changes
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser \
  --start-time $(date -d '24 hours ago' --iso-8601=seconds)
```

**Step 3 — Notify**

| Timeline | Action |
|---|---|
| 0–1 hour | Internal security team notified; incident ticket created |
| 1–24 hours | Determine if PHI was accessed — check RDS and S3 logs |
| 24–48 hours | If PHI breached: HIPAA Breach Notification Rule triggered |
| 60 days | Written notice to affected patients required (HIPAA 45 CFR §164.404) |
| Same day if >500 patients | HHS notification required + prominent media notice |

**Step 4 — Recover**
```bash
# Rotate all secrets after a breach
aws secretsmanager rotate-secret \
  --secret-id medaea/dev/django-secret-key

# Issue new deploy user key
aws iam create-access-key --user-name github-deploy-user-medaea-01
# Update GitHub Actions secrets: AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY

# Review and re-apply IAM policies (ensure no escalation occurred)
bash scripts/apply-iam-policies.sh
```

**Step 5 — Post-incident**
- Root cause analysis document within 5 business days
- Update threat model and controls
- Notify cyber insurance carrier
- Preserve all CloudTrail/WAF/Flow logs for forensics (do not delete)

---

## 6. Security Checklist — Before Going to Production

```
Infrastructure:
  ✓ RDS encryption at rest + force_ssl
  ✓ ElastiCache encryption at rest + in transit
  ✓ S3 public access blocked
  ✓ WAFv2 + managed rule sets on CloudFront
  ✓ Private subnets for RDS + ElastiCache
  ✓ Secrets Manager (no plaintext credentials)
  □ GuardDuty enabled
  □ Security Hub enabled
  □ CloudTrail data events on S3 + RDS
  □ Amazon Inspector on ECR repos
  □ RDS deletion_protection = true (already false for dev — flip for prod)

Application:
  □ FHIR R4 API with SMART on FHIR scopes
  □ Application-level audit log for all PHI access
  □ MFA enforced for all provider logins
  □ Session timeout (≤ 30 min inactivity per ONC §170.315(d)(5))
  □ Role-based access control (provider vs. patient vs. admin)
  □ PHI de-identified before any analytics/reporting export

Compliance:
  □ AWS BAA signed
  □ ONC §170.315 self-certification testing complete
  □ Annual risk analysis documented
  □ Penetration test completed
  □ HIPAA policies and procedures documented
  □ Workforce HIPAA training completed
  □ Business Associate Agreements with all vendors
```

---

## 7. Key Contacts & Escalation

| Role | Responsibility |
|---|---|
| Security Officer | HIPAA compliance, breach decisions, annual risk analysis |
| DevOps Lead | AWS access control, incident containment, key rotation |
| Legal / Compliance | HHS notifications, patient notifications, cyber insurance |
| AWS Support | Infrastructure-level incidents (open P1 case immediately for breach) |

AWS Enterprise Support P1 phone: `+1-800-558-1834`  
HHS Breach Portal: `https://ocrportal.hhs.gov/ocr/breach/wizard.jsf`

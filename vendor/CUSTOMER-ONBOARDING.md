# Vendor Customer Onboarding Playbook

> Internal playbook for onboarding new CMP customers.
> Covers the complete journey from pre-deployment assessment through post-deployment support.

---

## Table of Contents

1. [Pre-Deployment Checklist](#pre-deployment-checklist)
2. [Customer Environment Assessment](#customer-environment-assessment)
3. [Choosing Deployment Model](#choosing-deployment-model)
4. [License Provisioning Workflow](#license-provisioning-workflow)
5. [Registry Credential Provisioning](#registry-credential-provisioning)
6. [Handoff to Customer](#handoff-to-customer)
7. [Post-Deployment Vendor Responsibilities](#post-deployment-vendor-responsibilities)
8. [Customer Upgrade Notification Process](#customer-upgrade-notification-process)

---

## Pre-Deployment Checklist

Complete all items before initiating customer deployment:

### Contract and Administrative

- [ ] Customer contract signed and recorded in CRM
- [ ] Customer ID assigned (format: `cust_<identifier>`)
- [ ] License tier confirmed (trial / standard / enterprise)
- [ ] Support tier confirmed (basic / standard / premium)
- [ ] Deployment model agreed upon (SaaS / hybrid / on-premises / air-gapped)
- [ ] Primary technical contact identified
- [ ] Escalation contacts documented
- [ ] Communication channel established (email, Slack, Teams)

### Technical Prerequisites

- [ ] Customer environment assessment completed (see next section)
- [ ] Infrastructure requirements validated
- [ ] Network connectivity requirements confirmed
- [ ] Security review completed (for enterprise/air-gapped)
- [ ] Data residency requirements documented (if applicable)
- [ ] Backup and disaster recovery requirements documented

### Vendor Preparation

- [ ] License file generated and validated
- [ ] Registry credentials provisioned (for online deployments)
- [ ] Air-gapped package prepared (for air-gapped deployments)
- [ ] Customer-specific configuration template prepared
- [ ] Installation documentation customized (if needed)
- [ ] Onboarding kickoff meeting scheduled

---

## Customer Environment Assessment

### Minimum Infrastructure Requirements

| Resource | Minimum | Recommended | Notes |
|----------|---------|-------------|-------|
| CPU | 4 cores | 8+ cores | Across all containers |
| RAM | 8 GB | 16+ GB | Backend and worker are memory-intensive |
| Disk | 50 GB | 100+ GB | SSD recommended for database performance |
| Docker | 24.0+ | Latest stable | Docker Compose V2 required |
| OS | Linux (x86_64) | Ubuntu 22.04 LTS / RHEL 8+ | Windows/macOS not supported for production |

### Network Requirements

| Requirement | Port | Direction | Purpose |
|-------------|------|-----------|---------|
| HTTPS (frontend) | 443 | Inbound | User access to CMP UI |
| HTTP (backend API) | 8001 | Internal | Backend API (behind reverse proxy) |
| Redis | 6379 | Internal | Session cache, task queue |
| DynamoDB | 8000 | Internal | Database (local) or AWS endpoint |
| Registry access | 443 | Outbound | Pull container images (online only) |
| Vendor verification | 443 | Outbound | License verification (SaaS/hybrid only) |
| SMTP | 587/465 | Outbound | Email notifications (optional) |

### Assessment Questionnaire

Send this to the customer's technical team before deployment:

```markdown
## CMP Deployment Environment Assessment

1. **Operating System**: What OS and version will host CMP?
2. **Docker Version**: What Docker Engine version is installed? (`docker --version`)
3. **Docker Compose**: Is Docker Compose V2 available? (`docker compose version`)
4. **Available Resources**: CPU cores, RAM, and disk space allocated for CMP?
5. **Network Topology**: 
   - Is outbound internet access available?
   - Are there proxy servers for outbound traffic?
   - What ports are available for inbound traffic?
6. **DNS**: Will a custom domain be configured? (e.g., cmp.customer.com)
7. **TLS Certificates**: Will the customer provide TLS certificates or use Let's Encrypt?
8. **Existing Infrastructure**:
   - Is there an existing reverse proxy (nginx, HAProxy, ALB)?
   - Is there an existing Redis instance to use?
   - Is AWS DynamoDB available, or will DynamoDB Local be used?
9. **Authentication**: 
   - Is SSO integration required? (SAML/OIDC provider details)
   - What identity provider is in use?
10. **Backup Strategy**: What backup infrastructure is available?
11. **Monitoring**: What monitoring tools are in use? (Prometheus, Datadog, etc.)
12. **Security Requirements**:
    - Are there specific compliance requirements? (SOC2, HIPAA, FedRAMP)
    - Is a security review required before deployment?
    - Are there restrictions on container capabilities?
13. **Air-Gapped** (if applicable):
    - How will packages be transferred? (USB, secure courier, approved transfer)
    - Is there a local container registry?
    - How are updates currently applied to other systems?
```

### Assessment Validation

After receiving responses, validate:

| Check | Pass Criteria | Action if Fail |
|-------|--------------|----------------|
| Docker version | >= 24.0 | Advise upgrade before proceeding |
| Available RAM | >= 8 GB | Recommend resource increase |
| Disk space | >= 50 GB free | Recommend expansion |
| Outbound access (online) | Can reach <account_id>.dkr.ecr.<region>.amazonaws.com:443 | Switch to air-gapped or configure proxy |
| x86_64 architecture | `uname -m` returns `x86_64` | Not supported on ARM — escalate |

---

## Choosing Deployment Model

Use this decision tree to recommend the appropriate deployment model:

```
                    ┌─────────────────────────┐
                    │ Does the customer need  │
                    │ full internet isolation? │
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                   YES                        NO
                    │                         │
              ┌─────┴─────┐         ┌────────┴────────┐
              │ AIR-GAPPED │         │ Who manages the │
              └───────────┘         │ infrastructure? │
                                    └────────┬────────┘
                                             │
                              ┌──────────────┼──────────────┐
                              │              │              │
                           VENDOR        SHARED        CUSTOMER
                              │              │              │
                        ┌─────┴─────┐  ┌────┴────┐  ┌─────┴──────┐
                        │   SAAS    │  │ HYBRID  │  │ON-PREMISES │
                        └───────────┘  └─────────┘  └────────────┘
```

### Model Comparison

| Aspect | SaaS | Hybrid | On-Premises | Air-Gapped |
|--------|------|--------|-------------|------------|
| Infrastructure | Vendor-managed | Customer infra, vendor monitoring | Customer-managed | Customer-managed |
| Updates | Automatic | Coordinated | Manual (customer-initiated) | Manual (physical media) |
| License verification | Online (periodic) | Configurable | Optional | Offline only |
| Outbound network | Required | Required | Optional | None |
| Vendor access | Full | Monitoring only | None (support on request) | None |
| Typical customer | Startups, SMBs | Mid-market | Large enterprise | Government, banking |
| Support model | Proactive | Shared responsibility | Reactive | Reactive + scheduled |

### Recommendation Criteria

**Recommend SaaS when:**
- Customer wants zero operational overhead
- No data residency restrictions
- Comfortable with vendor-managed infrastructure

**Recommend Hybrid when:**
- Customer wants control over infrastructure
- Vendor monitoring is acceptable
- Regular update cadence is desired

**Recommend On-Premises when:**
- Customer requires full infrastructure control
- Data must stay within customer network
- Customer has DevOps/platform team capacity

**Recommend Air-Gapped when:**
- Regulatory requirement for network isolation (FedRAMP, classified environments)
- No outbound internet access permitted
- Physical security requirements for all software delivery

---

## License Provisioning Workflow

### Workflow Diagram

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Gather     │───▶│   Generate   │───▶│   Validate   │───▶│   Deliver    │
│   Info       │    │   License    │    │   License    │    │   License    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### Step 1: Gather Customer Information

From the signed contract and technical assessment, collect:

| Field | Source | Example |
|-------|--------|---------|
| Customer name | Contract | "Acme Corp" |
| Customer ID | CRM (auto-generated) | "cust_acme_789" |
| License type | Contract tier | "enterprise" |
| Deployment model | Technical assessment | "on-premises" |
| Max users | Contract | 500 |
| Max resources | Contract | 10000 |
| Features | Tier + add-ons | All enterprise features |
| Duration | Contract term | 365 days |
| Support tier | Contract | "premium" |
| Version range | Current release cycle | "8.0.0" to "9.99.99" |

### Step 2: Generate the License

```bash
# Use the license generation tool
python scripts/license/generate_license.py \
  --customer-name "Acme Corp" \
  --customer-id "cust_acme_789" \
  --type enterprise \
  --deployment-model on-premises \
  --max-users 500 \
  --max-resources 10000 \
  --features catalog,workflows,cost_management,policy_governance,ai_assistant,sso,multi_tenancy,budgets,scheduled_jobs \
  --duration-days 365 \
  --support-tier premium \
  --min-version "8.0.0" \
  --max-version "9.99.99" \
  --output license_cust_acme_789.json
```

### Step 3: Validate the License

```bash
# Verify the generated license is valid
python scripts/license/validate_license.py license_cust_acme_789.json

# Expected output:
# ✓ JSON structure valid
# ✓ All required fields present
# ✓ Field values within valid domains
# ✓ Signature verification passed
# ✓ Expiry date is in the future
# ✓ Version range is valid
# License is ready for delivery.
```

### Step 4: Deliver the License

| Deployment Model | Delivery Method |
|-----------------|-----------------|
| SaaS | Pre-installed by vendor during provisioning |
| Hybrid | Encrypted email or secure portal |
| On-Premises | Encrypted email or secure portal |
| Air-Gapped | Included in air-gapped package (physical media) |

### Step 5: Record in CRM

Document in the customer record:
- License ID
- Issue date
- Expiry date
- Delivery method and date
- Confirmation of receipt from customer

---

## Registry Credential Provisioning

### For Online Deployments (SaaS, Hybrid, On-Premises)

#### Step 1: Generate Credentials

```bash
python scripts/registry/generate_credentials.py \
  --customer-id "cust_acme_789" \
  --customer-name "Acme Corp" \
  --scope "autonimbus/cmp/*:8.*,autonimbus/cmp/*:9.*" \
  --expires-days 365
```

Output:
```
Registry credentials generated:
  Username: cust_acme_789
  Password: <generated-token>
  Registry: <account_id>.dkr.ecr.<region>.amazonaws.com
  Scope: autonimbus/cmp/*:8.*, autonimbus/cmp/*:9.*
  Expires: 2026-01-15
```

#### Step 2: Verify Access

```bash
# Authenticate with ECR using AWS CLI
aws ecr get-login-password --region <region> \
  | docker login --username AWS --password-stdin \
    <account_id>.dkr.ecr.<region>.amazonaws.com

# Verify pull access
docker pull <account_id>.dkr.ecr.<region>.amazonaws.com/autonimbus/cmp/cmp-backend:8.10.0-a1b2c3d4
```

#### Step 3: Deliver Credentials

Deliver via secure channel (separate from the license file):
- Encrypted email with IAM credentials in separate message
- Secure portal with time-limited download link
- Direct secure messaging (Signal, encrypted Slack)

Include instructions:
```bash
# Customer configures in their .env.production:
ECR_REGISTRY=<account_id>.dkr.ecr.<region>.amazonaws.com/autonimbus/cmp
AWS_ACCESS_KEY_ID=<provided-access-key>
AWS_SECRET_ACCESS_KEY=<provided-secret-key>
AWS_REGION=<region>

# Authenticate before pulling:
aws ecr get-login-password --region <region> \
  | docker login --username AWS --password-stdin \
    <account_id>.dkr.ecr.<region>.amazonaws.com
```

### For Air-Gapped Deployments

No registry credentials needed — images are delivered as tar archives in the air-gapped package.

---

## Handoff to Customer

### What to Deliver

| Item | Online Deployment | Air-Gapped Deployment |
|------|-------------------|----------------------|
| License file (`license.json`) | ✓ | ✓ (in package) |
| Registry credentials | ✓ | ✗ |
| Configuration template (`.env.template.<model>`) | ✓ | ✓ (in package) |
| Installation guide | ✓ (link to docs) | ✓ (in package) |
| Air-gapped package | ✗ | ✓ |
| Manifest verification public key | ✗ | ✓ (out-of-band) |
| Support contact information | ✓ | ✓ |
| Upgrade notification preferences | ✓ (collect) | ✓ (collect) |

### Handoff Meeting Agenda

1. **Confirm receipt** of all deliverables
2. **Walk through** the installation guide
3. **Review** the configuration template and required settings
4. **Explain** the health endpoints and how to verify successful installation
5. **Demonstrate** the upgrade process (dry-run if possible)
6. **Confirm** support channel and escalation path
7. **Set expectations** for first health check (vendor-initiated for hybrid/SaaS)
8. **Schedule** follow-up check-in (1 week post-deployment)

### Verification Steps (Post-Installation)

Ask the customer to confirm:

```bash
# 1. Health endpoint responds
curl -s https://cmp.customer.com/health | jq .
# Expected: {"status": "healthy", "services": {...}}

# 2. Version endpoint responds
curl -s https://cmp.customer.com/version | jq .
# Expected: {"version": "8.10.0", "build_hash": "a1b2c3d4", ...}

# 3. Login works
# Customer should be able to log in with initial admin credentials

# 4. License status is valid
# Admin panel shows valid license with correct tier and features
```

---

## Post-Deployment Vendor Responsibilities

### By Deployment Model

| Responsibility | SaaS | Hybrid | On-Premises | Air-Gapped |
|---------------|------|--------|-------------|------------|
| Infrastructure monitoring | ✓ | ✓ | ✗ | ✗ |
| Application monitoring | ✓ | ✓ | ✗ | ✗ |
| Automatic updates | ✓ | ✗ | ✗ | ✗ |
| Update notifications | ✓ | ✓ | ✓ | ✓ (per schedule) |
| Security patches | Immediate | Notify + assist | Notify | Notify + package |
| License renewal reminders | ✓ | ✓ | ✓ | ✓ |
| Health check reviews | Daily | Weekly | On request | On request |
| Backup verification | ✓ | Shared | ✗ | ✗ |

### Support Tier SLAs

| Metric | Basic | Standard | Premium |
|--------|-------|----------|---------|
| Response time (critical) | 24 hours | 4 hours | 1 hour |
| Response time (high) | 48 hours | 8 hours | 4 hours |
| Response time (medium) | 5 business days | 2 business days | 1 business day |
| Availability | Business hours | Extended hours | 24/7 |
| Dedicated contact | ✗ | ✗ | ✓ |
| Proactive monitoring | ✗ | ✗ | ✓ |
| Quarterly reviews | ✗ | ✗ | ✓ |

### Ongoing Vendor Tasks

1. **License expiry monitoring**: Track all customer license expiry dates; initiate renewal workflow at 90/60/30 days
2. **Security advisories**: Monitor CVE databases for vulnerabilities affecting CMP dependencies; notify affected customers per support tier
3. **Version tracking**: Maintain record of each customer's installed version for targeted communications
4. **Credential rotation**: Proactively rotate registry credentials annually (or on request)
5. **Health monitoring** (SaaS/hybrid): Review health endpoint data for anomalies

---

## Customer Upgrade Notification Process

### Notification Triggers

| Trigger | Urgency | Notification Timeline |
|---------|---------|----------------------|
| New minor/patch release | Normal | Within 1 week of release |
| New major release | Important | Within 2 weeks, with migration guide |
| Critical security patch | Urgent | Within 1 hour of release |
| License approaching expiry | Important | 90/60/30 days before |
| End-of-support for current version | Important | 6 months before EOL |

### Notification Template

```markdown
Subject: CMP Update Available — v{version}

Hi {customer_contact},

A new version of CMP is available: **v{version}**

**Release Type:** {Minor Feature Release / Security Patch / Major Upgrade}

**What's New:**
- {Summary of changes from release notes}

**Action Required:**
- {For patches: "We recommend upgrading within {timeframe}"}
- {For major: "Please review the migration guide and schedule an upgrade window"}

**Upgrade Instructions:**
- Online: Run `./scripts/upgrade.sh --target-version {version}`
- Air-gapped: Contact us to receive the updated package

**Compatibility:**
- Minimum current version required: v{min_version}
- {If major jump needed: "Intermediate upgrade to v{intermediate} required first"}

**Support:**
If you need assistance with the upgrade, contact us at {support_channel}.

Best regards,
Autonimbus CMP Team
```

### Notification Channels by Deployment Model

| Model | Primary Channel | Secondary Channel |
|-------|----------------|-------------------|
| SaaS | In-app notification + email | Slack/Teams webhook |
| Hybrid | Email | Slack/Teams webhook |
| On-Premises | Email | Phone (critical only) |
| Air-Gapped | Email to designated contact | Phone (critical only) |

### Air-Gapped Customer Update Process

For air-gapped customers, the update process is:

1. **Notify** the customer contact that a new version is available
2. **Schedule** a delivery window
3. **Generate** the air-gapped package with the new version
4. **Include** the customer's current license (if still valid) or a renewed license
5. **Deliver** via approved secure media
6. **Provide** upgrade instructions and release notes
7. **Offer** remote support session (if customer environment allows temporary connectivity)
8. **Follow up** to confirm successful upgrade

### Version End-of-Life Policy

| Version Age | Status | Support Level |
|-------------|--------|--------------|
| Current (N) | Active | Full support |
| Previous (N-1) | Maintained | Security patches only |
| N-2 | End of Life | No patches, upgrade required |

Notify customers on N-2 versions:
- 6 months before EOL: Initial notification
- 3 months before EOL: Reminder with upgrade path
- 1 month before EOL: Final warning
- At EOL: Support limited to upgrade assistance only

# Vendor Troubleshooting and Support Guide

> Internal guide for the Autonimbus support team.
> Covers diagnosing customer issues, interpreting system responses, and resolution procedures.

---

## Table of Contents

1. [Common Customer-Reported Issues](#common-customer-reported-issues)
2. [Interpreting Health Endpoint Responses](#interpreting-health-endpoint-responses)
3. [Diagnosing License Validation Failures](#diagnosing-license-validation-failures)
4. [Assisting with Failed Upgrades](#assisting-with-failed-upgrades)
5. [Generating Hotfix Releases](#generating-hotfix-releases)
6. [Escalation Procedures](#escalation-procedures)
7. [Credential Rotation Requests](#credential-rotation-requests)
8. [Air-Gapped Customer Support](#air-gapped-customer-support)

---

## Common Customer-Reported Issues

### Issue: Application Won't Start

**Symptoms**: Customer reports the application fails to start after deployment or restart.

**Diagnostic Steps**:

1. Ask for container logs:
   ```bash
   docker compose -f docker-compose.production.yml logs cmp-backend --tail 50
   ```

2. Check for startup error categories:

| Log Message Pattern | Root Cause | Resolution |
|--------------------|-----------:|------------|
| `License file not found` | Missing license file | Verify file path and volume mount |
| `License signature validation failed` | Corrupted or tampered license | Re-generate and re-deliver license |
| `License expired on...` | License past expiry date | Generate renewed license |
| `does not match configured DEPLOYMENT_MODEL` | Environment mismatch | Fix license `environment` field or `DEPLOYMENT_MODEL` env var |
| `Missing required configuration` | Missing env vars | Check `.env` file for required keys |
| `DEPLOYMENT_MODEL...Valid values` | Invalid deployment model | Set to: saas, hybrid, on-premises, or air-gapped |
| `Migration scripts not available` | Missing migration files | Verify migrations directory is mounted |
| `Rollback script missing for migration` | Incomplete migration set | Provide missing rollback scripts |
| `Secret source...missing or unreadable` | Docker secrets not mounted | Verify secret file paths |

3. If none of the above, check resource constraints:
   ```bash
   docker stats --no-stream
   ```

### Issue: HTTP 403 on Feature Access

**Symptoms**: Users get "This feature requires a license upgrade" when accessing certain features.

**Diagnostic Steps**:

1. Check the license status endpoint:
   ```bash
   curl -s https://cmp.customer.com/api/v1/license/status | jq .features_enabled
   ```

2. Verify the feature key is in the license's `features` list
3. Verify the admin feature toggle is enabled:
   ```bash
   curl -s -H "Authorization: Bearer <admin-token>" \
     https://cmp.customer.com/api/v1/settings/feature-toggles/me | jq .
   ```

**Resolution**: Either add the feature to the license (re-sign and deliver) or enable the admin toggle.

### Issue: "User Limit Exceeded" or "Resource Limit Exceeded"

**Symptoms**: Operations fail with HTTP 403 indicating limit exceeded.

**Diagnostic Steps**:

1. Check current usage vs limits:
   ```bash
   curl -s https://cmp.customer.com/api/v1/license/status | jq '{
     current_users, max_users, current_resources, max_resources
   }'
   ```

**Resolution**:
- If legitimate growth: Generate new license with increased limits
- If unexpected: Help customer audit active users/resources and clean up unused ones

### Issue: Services Degraded or Unhealthy

**Symptoms**: Customer reports slow performance or partial functionality loss.

**Diagnostic Steps**:

1. Check health endpoint:
   ```bash
   curl -s https://cmp.customer.com/health | jq .
   ```

2. Identify which services are unhealthy (see next section for interpretation)

3. Check container resource usage:
   ```bash
   docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
   ```

### Issue: Cannot Pull Images from Registry

**Symptoms**: `docker compose pull` fails with authentication errors.

**Diagnostic Steps**:

1. Ask customer to test credentials:
   ```bash
   docker login registry.autonimbus.com -u <username> -p <password>
   ```

2. Check error message:
   - "invalid credentials" → Credentials are wrong or revoked
   - "expired credentials" → Credentials have expired, need rotation
   - "access denied" → Credentials don't have scope for requested tag

**Resolution**: See [Credential Rotation Requests](#credential-rotation-requests).

### Issue: Slow Application Performance

**Symptoms**: Pages load slowly, API responses take > 5 seconds.

**Diagnostic Steps**:

1. Check health response time:
   ```bash
   curl -s -w "\nTotal time: %{time_total}s\n" https://cmp.customer.com/health
   ```

2. Check if Redis is healthy (session/cache layer):
   ```bash
   curl -s https://cmp.customer.com/health | jq .services.redis
   ```

3. Check container resource limits vs usage:
   ```bash
   docker inspect cmp-backend --format '{{.HostConfig.Memory}}'
   docker stats cmp-backend --no-stream
   ```

**Resolution**:
- If Redis unhealthy: Restart Redis container, check memory limits
- If CPU/memory constrained: Recommend increasing resource limits in docker-compose
- If database slow: Check DynamoDB provisioned capacity or local instance resources

---

## Interpreting Health Endpoint Responses

### Endpoint: `GET /health`

```json
{
  "status": "healthy | degraded | unhealthy",
  "services": {
    "backend": "healthy | degraded | unhealthy",
    "frontend": "healthy | degraded | unhealthy",
    "worker": "healthy | degraded | unhealthy",
    "ai_service": "healthy | degraded | unhealthy",
    "redis": "healthy | degraded | unhealthy",
    "database": "healthy | degraded | unhealthy"
  },
  "timestamp": "2025-01-15T10:30:00Z",
  "response_time_ms": 45
}
```

### Aggregate Status Logic

| Aggregate Status | Meaning | Condition |
|-----------------|---------|-----------|
| `healthy` | All services operational | All services report healthy |
| `degraded` | Partial functionality loss | At least one non-critical service (frontend, worker, ai_service) is unhealthy, but all critical services are healthy |
| `unhealthy` | Major functionality loss | At least one critical service (backend, database, redis) is unhealthy |

### Service Classification

| Service | Classification | Impact if Unhealthy |
|---------|---------------|-------------------|
| `backend` | **Critical** | API completely unavailable |
| `database` | **Critical** | No data access, all operations fail |
| `redis` | **Critical** | No sessions, no task queue, auth degraded |
| `frontend` | Non-critical | UI unavailable, API still works |
| `worker` | Non-critical | Background jobs don't execute |
| `ai_service` | Non-critical | AI features unavailable |

### Failure Types

When a service is unhealthy, check the detailed health results for failure type:

| Failure Type | Meaning | Likely Cause |
|-------------|---------|--------------|
| `timeout` | Service didn't respond within 3 seconds | Service overloaded, resource exhaustion |
| `connection_refused` | TCP connection rejected | Service not running, wrong port |
| `error_response` | Service responded with error | Application error, misconfiguration |

### Endpoint: `GET /version`

```json
{
  "version": "8.10.0",
  "build_hash": "a1b2c3d4",
  "build_timestamp": "2025-01-15T08:00:00Z",
  "database_schema_version": "003"
}
```

Use this to confirm:
- The customer is running the expected version
- The build hash matches the release (detect unauthorized modifications)
- The database schema is at the expected migration version

### Endpoint: `GET /health/upgrade-status`

```json
{
  "current_version": "8.10.0",
  "latest_available_version": "8.11.0",
  "pending_migrations": [],
  "is_up_to_date": false
}
```

Use this to check:
- Whether the customer needs an upgrade
- If there are pending migrations that haven't been applied
- Whether the system is in a consistent state

---

## Diagnosing License Validation Failures

### Step 1: Identify the Error

Ask the customer for the backend container logs at startup:

```bash
docker compose -f docker-compose.production.yml logs cmp-backend 2>&1 | grep -i "license"
```

### Step 2: Diagnose by Error Type

#### "License file not found at: /app/license/license.json"

**Cause**: File doesn't exist at the expected path.

**Diagnosis**:
```bash
# Check if the file exists in the container
docker compose exec cmp-backend ls -la /app/license/

# Check the volume mount in docker-compose.production.yml
grep -A5 "license" docker-compose.production.yml
```

**Resolution**:
- Verify the volume mount maps the host license file to `/app/license/license.json`
- If using a custom path, verify `LICENSE_FILE_PATH` environment variable is set correctly
- Verify file permissions (readable by the container's non-root user)

#### "License signature validation failed — file may be tampered"

**Cause**: The file content doesn't match the cryptographic signature.

**Diagnosis**:
```bash
# Get the file hash to compare with what was delivered
sha256sum /path/to/license.json

# Check file size matches expected
ls -la /path/to/license.json

# Check for encoding issues (BOM, line endings)
file /path/to/license.json
hexdump -C /path/to/license.json | head -1
```

**Common causes**:
- File was edited after signing (even whitespace changes break the signature)
- File was corrupted during transfer
- Wrong license file (from a different customer or old version)
- BOM (byte order mark) added by Windows editors
- Line ending conversion (CRLF vs LF)

**Resolution**:
1. Re-deliver the original signed license file
2. Instruct customer to NOT edit the file
3. Use binary-safe transfer methods (base64 encoding if needed)

#### "License expired on {date}"

**Cause**: System clock is past the license expiry date.

**Diagnosis**:
```bash
# Check system time
date -u

# Check license expiry
cat /path/to/license.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['expires_at'])"
```

**Resolution**:
- If clock is wrong: Fix system time (NTP sync)
- If genuinely expired: Generate renewed license and deliver

#### "License environment type '{x}' does not match configured DEPLOYMENT_MODEL '{y}'"

**Cause**: Mismatch between license and environment configuration.

**Diagnosis**:
```bash
# Check configured deployment model
docker compose exec cmp-backend env | grep DEPLOYMENT_MODEL

# Check license environment field
cat /path/to/license.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['environment'], d['deployment_model'])"
```

**Resolution**:
- Either fix the `DEPLOYMENT_MODEL` environment variable to match the license
- Or generate a new license with the correct `environment` and `deployment_model` fields

#### "Application version {x} is outside the licensed range [{min}, {max}]"

**Cause**: Customer upgraded to a version not covered by their license.

**Diagnosis**:
```bash
# Check application version
cat /app/VERSION  # or check /version endpoint

# Check license version range
cat /path/to/license.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['permitted_versions'])"
```

**Resolution**:
- Generate new license with expanded `permitted_versions.max`
- Or instruct customer to use a version within their licensed range

### Step 3: Verify Fix

After applying the fix, have the customer restart and check:
```bash
docker compose -f docker-compose.production.yml restart cmp-backend
docker compose -f docker-compose.production.yml logs cmp-backend --tail 20 | grep -i "license"
# Should see: "License validated successfully: customer=..."
```

---

## Assisting with Failed Upgrades

### Step 1: Gather Information

Ask the customer for:

1. **Upgrade audit log**:
   ```bash
   cat /var/log/cmp/upgrades/upgrade_*.json | python3 -m json.tool
   ```

2. **Current state**:
   ```bash
   curl -s https://cmp.customer.com/version
   curl -s https://cmp.customer.com/health
   ```

3. **Container status**:
   ```bash
   docker compose -f docker-compose.production.yml ps
   ```

### Step 2: Identify the Failing Step

The audit log shows each step's outcome:

```json
[
  {"timestamp": "2025-01-15T10:00:00Z", "step": "backup", "outcome": "success", "duration_seconds": 45.2},
  {"timestamp": "2025-01-15T10:00:45Z", "step": "pull_images", "outcome": "success", "duration_seconds": 120.5},
  {"timestamp": "2025-01-15T10:02:45Z", "step": "run_migrations", "outcome": "failure", "duration_seconds": 12.3, "details": "Migration 004 failed: ..."}
]
```

### Step 3: Resolution by Failing Step

#### Backup Step Failed

**Cause**: Cannot create backup (disk full, permissions, AWS CLI issue).

**Resolution**:
- Check disk space: `df -h /var/lib/cmp/backups`
- Check AWS CLI configuration for DynamoDB export
- Verify backup directory permissions
- The upgrade hasn't started yet — safe to retry after fixing the issue

#### Pull Images Failed

**Cause**: Registry authentication, network, or image not found.

**Resolution**:
- Verify registry credentials: `docker login registry.autonimbus.com`
- Check network connectivity to registry
- Verify the target version exists in the registry
- For air-gapped: Verify image tar files are present and not corrupted

#### Migrations Failed

**Cause**: Database migration script error.

**Resolution**:
1. Check which migration failed (from audit log details)
2. The migration engine automatically rolls back partial changes
3. Check if the database is at the last good version:
   ```bash
   curl -s https://cmp.customer.com/version | jq .database_schema_version
   ```
4. If a migration bug: Generate a hotfix with corrected migration script
5. If data issue: Help customer clean up data that violates new constraints

#### Restart Services Failed

**Cause**: Containers won't start with new images.

**Resolution**:
1. Check container logs for startup errors
2. Common causes: configuration incompatibility with new version, port conflicts
3. Guide customer through rollback:
   ```bash
   ./scripts/upgrade.sh --rollback
   ```

#### Health Check Failed

**Cause**: Services started but aren't healthy within 120 seconds.

**Resolution**:
1. Check which services are unhealthy
2. Give services more time (some need longer for initial startup)
3. Check resource constraints (new version may need more memory)
4. If a specific service won't become healthy, check its logs

### Step 4: Rollback Guidance

If the upgrade cannot be completed:

```bash
# If upgrade.sh was used with --auto-rollback, it may have already rolled back
# Check current state first:
curl -s https://cmp.customer.com/version

# Manual rollback (if auto-rollback didn't trigger):
./scripts/upgrade.sh --rollback

# What rollback restores:
# - Database contents from pre-upgrade backup
# - Previous container images
# - Previous configuration files
```

### Step 5: Post-Rollback Verification

```bash
# Verify system is back to previous state
curl -s https://cmp.customer.com/health | jq .status
# Expected: "healthy"

curl -s https://cmp.customer.com/version | jq .version
# Expected: previous version number
```

---

## Generating Hotfix Releases

### When to Generate a Hotfix

- Critical security vulnerability affecting customer deployments
- Data corruption or loss bug
- Complete feature breakage with no workaround
- Regression introduced in the latest release

### Hotfix Process

#### 1. Create Hotfix Branch

```bash
# Branch from the affected release tag
git checkout -b hotfix/8.10.1 v8.10.0

# Apply the minimal fix
# ... make changes ...

git add .
git commit -m "fix: [CRITICAL] Description of the fix"
```

#### 2. Test the Fix

```bash
# Run unit tests
cd backend && python -m pytest tests/ -x

# Run property tests
cd backend && python -m pytest tests/properties/ -x

# If possible, reproduce the issue and verify the fix
```

#### 3. Merge and Tag

```bash
# Merge to main via expedited PR (requires 1 reviewer for hotfixes)
git checkout main
git merge hotfix/8.10.1

# Tag the hotfix
git tag -a v8.10.1 -m "Hotfix: Critical fix for [issue description]"
git push origin v8.10.1
```

#### 4. Monitor Pipeline

- Watch the release pipeline in GitHub Actions
- For extreme emergencies, use manual dispatch with `skip_scan: true`
- Document the justification for skipping the scan

#### 5. Deliver to Affected Customers

| Deployment Model | Delivery |
|-----------------|----------|
| SaaS | Auto-deployed by vendor |
| Hybrid | Notify + provide upgrade command |
| On-Premises | Notify + provide upgrade instructions |
| Air-Gapped | Generate air-gapped package + deliver via secure media |

#### 6. Customer Communication

```markdown
Subject: [CRITICAL] CMP Security Patch v8.10.1 — Immediate Action Required

A critical security vulnerability has been identified and patched.

**Severity:** Critical
**Affected Versions:** v8.10.0
**Fixed Version:** v8.10.1

**Impact:** [Brief description of the vulnerability impact]

**Action Required:** Upgrade to v8.10.1 immediately.

**Upgrade Command:**
./scripts/upgrade.sh --target-version 8.10.1

Please confirm completion of the upgrade by replying to this message.
```

---

## Escalation Procedures

### Severity Classification

| Severity | Definition | Examples |
|----------|-----------|----------|
| **Critical (P1)** | Complete system outage, data loss, security breach | Application won't start, database corruption, active exploit |
| **High (P2)** | Major feature broken, significant performance degradation | Upgrade failed with no rollback, authentication broken |
| **Medium (P3)** | Feature partially broken, workaround available | Single feature 403 error, slow queries |
| **Low (P4)** | Minor issue, cosmetic, documentation | UI glitch, unclear error message |

### Escalation Path

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  L1 Support     │───▶│  L2 Engineering │───▶│  L3 Architecture│
│  (First response│    │  (Deep diagnosis│    │  (Design-level  │
│   + known fixes)│    │   + code fixes) │    │   decisions)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Escalation Triggers

| From | To | Trigger |
|------|-----|---------|
| L1 → L2 | After 30 min without resolution for P1/P2 | Issue requires code-level investigation |
| L1 → L2 | After 4 hours without resolution for P3 | Known fixes don't apply |
| L2 → L3 | Issue requires architecture change | Design decision needed |
| L2 → L3 | Hotfix needed | Code change required for release |
| Any → Management | Customer escalation | Customer requests management involvement |

### Escalation Template

```markdown
## Escalation Report

**Customer:** {name} ({customer_id})
**Severity:** P{1-4}
**Reported:** {timestamp}
**Escalated:** {timestamp}

**Issue Summary:**
{One paragraph description}

**Impact:**
{What functionality is affected, how many users}

**Steps Taken:**
1. {What was tried}
2. {What was tried}

**Current State:**
{System status, health endpoint output}

**Hypothesis:**
{Best guess at root cause}

**Requested Action:**
{What you need from the next level}
```

---

## Credential Rotation Requests

### When Customers Request Rotation

Common reasons:
- Scheduled rotation (security policy compliance)
- Suspected credential compromise
- Personnel change (employee with access left)
- Audit requirement

### Rotation Procedure

#### 1. Generate New Credentials

```bash
python scripts/registry/rotate_credentials.py \
  --customer-id "cust_acme_789" \
  --overlap-hours 24 \
  --reason "Scheduled annual rotation"
```

#### 2. Verify Overlap Period

Both old and new credentials are valid for 24 hours:

```bash
# Verify old credentials still work
docker login registry.autonimbus.com -u cust_acme_789 -p <old-token>

# Verify new credentials work
docker login registry.autonimbus.com -u cust_acme_789 -p <new-token>
```

#### 3. Deliver New Credentials

Send via secure channel (same method as initial provisioning):
- Encrypted email
- Secure portal
- Direct secure messaging

Include instructions:
```markdown
Your registry credentials have been rotated.

**New credentials:**
- Username: cust_acme_789
- Password: <new-token>

**Important:** Your old credentials will remain valid for 24 hours.
Please update your configuration within this window:

1. Update your .env file:
   REGISTRY_PASSWORD=<new-token>

2. Verify access:
   docker login registry.autonimbus.com -u cust_acme_789 -p <new-token>

3. No restart required — credentials are used only during image pulls.
```

#### 4. Confirm Rotation Complete

After 24 hours:
- Old credentials are automatically invalidated
- Verify customer has confirmed the update
- If no confirmation: Contact customer before old credentials expire

### Emergency Credential Revocation

For suspected compromise:

```bash
# Immediate revocation (no overlap period)
python scripts/registry/revoke_credentials.py \
  --customer-id "cust_acme_789" \
  --immediate \
  --reason "Suspected credential compromise"

# Generate new credentials immediately
python scripts/registry/generate_credentials.py \
  --customer-id "cust_acme_789" \
  --scope "autonimbus/cmp/*:8.*,autonimbus/cmp/*:9.*" \
  --expires-days 365
```

Notify customer immediately via phone + email that:
1. Old credentials have been revoked
2. New credentials are being delivered
3. They should audit their systems for unauthorized access

---

## Air-Gapped Customer Support

### Challenges

- No remote access to customer systems
- No real-time log streaming
- Communication delays for troubleshooting
- Package delivery requires physical logistics

### Support Workflow

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Customer   │───▶│   Vendor     │───▶│   Vendor     │───▶│   Customer   │
│   Reports    │    │   Analyzes   │    │   Prepares   │    │   Applies    │
│   Issue      │    │   Remotely   │    │   Fix Package│    │   Fix        │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### Information Gathering (Remote)

Since we can't access the system, ask the customer to provide:

```bash
# 1. System state snapshot
curl -s localhost/health > health_snapshot.json
curl -s localhost/version > version_snapshot.json

# 2. Container status
docker compose -f docker-compose.production.yml ps > container_status.txt

# 3. Recent logs (last 200 lines per service)
docker compose -f docker-compose.production.yml logs --tail 200 > all_logs.txt

# 4. Resource usage
docker stats --no-stream > resource_usage.txt

# 5. Disk space
df -h > disk_usage.txt

# 6. Upgrade audit logs (if upgrade-related)
cat /var/log/cmp/upgrades/*.json > upgrade_audit.txt

# 7. License status (if license-related)
cat /app/license/license.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
del d['signature']  # Don't send signature
print(json.dumps(d, indent=2))
" > license_info.json
```

Ask the customer to send these files via their approved secure communication channel.

### Providing Fixes

#### For Configuration Issues

Provide step-by-step instructions:
```markdown
## Fix Instructions

1. Stop the application:
   docker compose -f docker-compose.production.yml down

2. Edit the configuration:
   vi /opt/cmp/config/.env
   # Change: SETTING_X=old_value
   # To:     SETTING_X=new_value

3. Start the application:
   docker compose -f docker-compose.production.yml up -d

4. Verify:
   curl -s localhost/health | python3 -m json.tool
```

#### For Software Fixes (Hotfix Package)

1. Generate a hotfix air-gapped package
2. Include only the changed images (not the full package if bandwidth is a concern)
3. Include a targeted upgrade script
4. Deliver via approved secure media

```bash
# Minimal hotfix package structure
cmp-hotfix-v8.10.1/
├── images/
│   └── cmp-backend-8.10.1-<hash>.tar.gz  (only the fixed service)
├── scripts/
│   └── apply-hotfix.sh
├── MANIFEST.sha256
├── MANIFEST.sha256.sig
└── HOTFIX-NOTES.md
```

#### For License Issues

Generate a corrected license file and deliver via approved channel. Include verification instructions:

```bash
# Replace the license file
cp /path/to/new/license.json /app/license/license.json

# The application will hot-reload within 60 seconds
# Or restart for immediate effect:
docker compose -f docker-compose.production.yml restart cmp-backend

# Verify
docker compose -f docker-compose.production.yml logs cmp-backend --tail 10 | grep -i "license"
```

### Scheduled Support Sessions

For air-gapped customers with premium support:

1. **Monthly check-in**: Review health snapshots, discuss upcoming needs
2. **Quarterly review**: Version assessment, upgrade planning, capacity review
3. **Annual renewal**: License renewal, credential rotation, version upgrade planning

### Documentation for Air-Gapped Customers

Ensure all documentation is included in the air-gapped package:
- Installation guide (`INSTALL-air-gapped.md`)
- Upgrade guide (`UPGRADE.md`)
- Troubleshooting guide (customer-facing version)
- Configuration reference
- Health endpoint reference

The customer should be able to resolve common issues without vendor contact.

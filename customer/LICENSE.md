# CMP License Guide

This guide explains how CMP licensing works, how to activate and manage your license, monitor usage against limits, and resolve common license issues.

---

## Table of Contents

1. [Understanding Your License](#understanding-your-license)
2. [License File Placement](#license-file-placement)
3. [Checking License Status](#checking-license-status)
4. [Feature Availability](#feature-availability)
5. [User and Resource Limit Monitoring](#user-and-resource-limit-monitoring)
6. [License Renewal](#license-renewal)
7. [Hot-Reload: Updating License Without Restart](#hot-reload-updating-license-without-restart)
8. [License Error Messages](#license-error-messages)
9. [Offline vs Online License Validation](#offline-vs-online-license-validation)
10. [Contacting Vendor for License Issues](#contacting-vendor-for-license-issues)

---

## Understanding Your License

Your CMP license is a cryptographically signed JSON file (`license.json`) that defines what you're entitled to use. It contains:

### License Components

| Field | Description |
|-------|-------------|
| **License Type** | `trial`, `standard`, or `enterprise` — determines feature tier |
| **Customer** | Your organization name and customer ID |
| **Expiry Date** | When the license expires (application won't start after this date) |
| **Features** | List of CMP features you're licensed to use |
| **User Limit** | Maximum number of users allowed in the system |
| **Resource Limit** | Maximum number of managed cloud resources |
| **Cloud Providers** | Which cloud providers are enabled and how many accounts per provider |
| **Deployment Model** | Must match your `DEPLOYMENT_MODEL` setting |
| **Permitted Versions** | Range of CMP versions this license covers |
| **Support Tier** | `basic`, `standard`, or `premium` |
| **Vendor Verification** | Whether online license verification is enabled |

### License Types

| Type | Typical Use | Duration | Features |
|------|-------------|----------|----------|
| **Trial** | Evaluation | 30 days | Limited feature set |
| **Standard** | Small/medium teams | Annual | Core features |
| **Enterprise** | Large organizations | Annual | All features |

### Security

Your license is signed with RSA 4096-bit cryptography. This means:

- The license cannot be modified without invalidating the signature
- CMP verifies the signature at every startup
- No network connection is needed for validation (the public key is embedded in the application)

---

## License File Placement

### Default Location

Place your license file at:

```
/opt/cmp/license/license.json
```

The Docker Compose configuration mounts this directory into the container at `/app/license/`.

### Custom Location

To use a different path, set the `LICENSE_FILE_PATH` environment variable in your `.env`:

```bash
LICENSE_FILE_PATH=/app/license/my-license.json
```

And ensure the corresponding host path is mounted in `docker-compose.yml`:

```yaml
volumes:
  - /path/to/your/license:/app/license:ro
```

### File Permissions

- The license file must be readable by the container user (UID 1000 by default)
- Recommended permissions: `chmod 644 license.json`
- The file is read-only from the application's perspective

### Verifying File Placement

```bash
# Check the file exists and is readable
ls -la /opt/cmp/license/license.json

# Verify it's valid JSON
cat /opt/cmp/license/license.json | jq .license_id
```

---

## Checking License Status

### Via API Endpoint

```bash
curl -s http://localhost:8001/api/v1/license/status | jq .
```

Response:
```json
{
  "valid": true,
  "license_type": "enterprise",
  "customer_name": "Acme Corp",
  "expires_at": "2026-01-15T00:00:00Z",
  "days_remaining": 342,
  "features_enabled": [
    "catalog",
    "workflows",
    "scheduled_jobs",
    "terraform",
    "cost_management",
    "reports",
    "policy_governance",
    "multi_tenancy",
    "budgets",
    "event_automation",
    "webhooks",
    "cloud_accounts",
    "sso",
    "ai_assistant"
  ],
  "current_users": 47,
  "max_users": 500,
  "current_resources": 1250,
  "max_resources": 10000,
  "cloud_providers": {
    "aws": 10,
    "azure": 10,
    "gcp": 10
  },
  "deployment_model": "on-premises",
  "support_tier": "premium"
}
```

### Via CMP Web Interface

Navigate to **Settings → License** in the CMP dashboard to view:

- License type and customer information
- Expiry date and days remaining
- Feature availability
- Current usage vs limits (users and resources)

### Via Health Endpoint

The version endpoint includes basic license info:

```bash
curl -s http://localhost:8001/version | jq .
```

---

## Feature Availability

CMP uses a two-layer feature gating system:

### How It Works

```
Feature Available = Licensed Feature AND Admin Toggle Enabled
```

1. **License layer:** Your license defines which features you're entitled to use
2. **Admin toggle layer:** Administrators can additionally disable licensed features

A feature is accessible only when **both** conditions are true.

### Checking Available Features

```bash
# Requires authentication
curl -s -H "Authorization: Bearer <token>" \
  http://localhost:8001/api/v1/settings/feature-toggles/me | jq .
```

Features not in your license will always appear as disabled, regardless of admin toggle settings.

### What Happens When Accessing an Unlicensed Feature

If a user attempts to access a feature not included in the license:

- **API:** Returns HTTP 403 with message: `"This feature requires a license upgrade. Contact your administrator."`
- **UI:** The feature is hidden from navigation (the frontend receives it as disabled)

### Feature Key Reference

| Feature Key | Description | Standard | Enterprise |
|-------------|-------------|----------|------------|
| `catalog` | Self-service resource catalog | ✓ | ✓ |
| `workflows` | Workflow automation engine | ✓ | ✓ |
| `scheduled_jobs` | Scheduled automation tasks | ✓ | ✓ |
| `terraform` | Terraform provisioning, workspaces, drift, state backend | — | ✓ |
| `cost_management` | Cost tracking and optimization | ✓ | ✓ |
| `reports` | Reports and analytics dashboards | ✓ | ✓ |
| `policy_governance` | Policy engine and compliance | — | ✓ |
| `budgets` | Budget management and alerts | ✓ | ✓ |
| `event_automation` | Event-driven automation + event log | — | ✓ |
| `webhooks` | Inbound webhook endpoints | ✓ | ✓ |
| `cloud_accounts` | Cloud account management (AWS/Azure/GCP) | ✓ | ✓ |
| `multi_tenancy` | Multi-tenant isolation | — | ✓ |
| `sso` | Single sign-on (OIDC/SAML) | — | ✓ |
| `ai_assistant` | AI-powered operations assistant | — | ✓ |

### Cloud Provider Limits

Your license also controls which cloud providers you can connect and how many accounts per provider:

```json
"cloud_providers": {
  "aws": 2,
  "azure": 2,
  "gcp": 1
}
```

- A value of `0` means the provider is disabled (you cannot add accounts for it)
- A value > 0 is the maximum number of accounts you can add for that provider
- Existing accounts are not removed if limits are reduced — only new account creation is blocked

---

## User and Resource Limit Monitoring

Your license defines maximum limits for users and managed resources.

### Checking Current Usage

```bash
curl -s http://localhost:8001/api/v1/license/status | jq '{
  current_users: .current_users,
  max_users: .max_users,
  current_resources: .current_resources,
  max_resources: .max_resources
}'
```

### What Happens at the Limit

When you reach a limit:

- **User limit:** New user creation is blocked. Existing users continue to work normally.
  - Error: `"User limit exceeded. Licensed maximum: 500 users. Contact your administrator for a license upgrade."`
- **Resource limit:** New resource provisioning is blocked. Existing resources continue to be managed.
  - Error: `"Resource limit exceeded. Licensed maximum: 10000 resources. Contact your administrator for a license upgrade."`
- **Cloud provider account limit:** New account creation for that provider is blocked. Existing accounts continue to work.
  - Error: `"Account limit for 'aws' reached: current count (2) has reached the licensed maximum (2). Contact Autonimbus to upgrade."`
- **Cloud provider disabled (limit = 0):** Cannot add accounts for that provider at all.
  - Error: `"Cloud provider 'gcp' is not enabled in your license. Contact Autonimbus to upgrade."`

### Proactive Monitoring

Set up alerts when approaching limits:

- **80% threshold:** Plan for license upgrade
- **90% threshold:** Urgent — contact Autonimbus for limit increase
- **100%:** Operations blocked for new users/resources

### Reducing Usage

To free up capacity:

- **Users:** Deactivate or remove unused user accounts
- **Resources:** Remove resources no longer under management

---

## License Renewal

### When to Renew

- Your license `expires_at` date is approaching (check `days_remaining` in status)
- You need additional users or resources beyond current limits
- You want to upgrade to a higher tier for more features

### Renewal Process

1. **Contact Autonimbus** at least 30 days before expiry
2. **Receive new license file** via secure delivery
3. **Replace the license file** on your server:
   ```bash
   cp /path/to/new-license.json /opt/cmp/license/license.json
   ```
4. **Wait for hot-reload** (up to 60 seconds) or restart the application
5. **Verify** the new license is active:
   ```bash
   curl -s http://localhost:8001/api/v1/license/status | jq '{expires_at, days_remaining}'
   ```

### What Happens When a License Expires

- The application **will not start** if the license is expired
- If the application is already running when the license expires, it continues to run until the next restart
- At next restart, you'll see: `"License expired on 2025-01-15. Please contact Autonimbus for renewal."`

### Grace Period

There is no grace period. Once expired, the application will not start. Plan renewals well in advance.

---

## Hot-Reload: Updating License Without Restart

CMP monitors the license file for changes and automatically applies updates within 60 seconds.

### How It Works

1. CMP watches the license file path for modifications
2. When a change is detected, the new file is read and validated
3. If valid, the new license is applied immediately
4. If invalid, the previous license remains in effect and an error is logged

### Use Cases for Hot-Reload

- **Feature additions:** Vendor adds new features to your license
- **Limit increases:** User or resource limits increased
- **Renewal:** New expiry date applied without downtime

### Procedure

```bash
# Replace the license file
cp /path/to/updated-license.json /opt/cmp/license/license.json

# Wait up to 60 seconds, then verify
sleep 60
curl -s http://localhost:8001/api/v1/license/status | jq .
```

### Limitations

- If the new license file has an invalid signature, it is rejected (old license stays active)
- If the new license has a different deployment model, it is rejected (requires restart with matching DEPLOYMENT_MODEL)
- Hot-reload cannot recover from an expired license that prevented startup

---

## License Error Messages

### Startup Errors (Application Won't Start)

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `"License file not found at /app/license/license.json"` | File missing or wrong path | Place license file at configured path |
| `"License file is not readable"` | Permission issue | `chmod 644 license.json` |
| `"License signature validation failed"` | File corrupted or tampered | Request new license from Autonimbus |
| `"License expired on YYYY-MM-DD. Please contact Autonimbus for renewal."` | Past expiry date | Obtain renewed license |
| `"License malformed: missing required field 'license_id'"` | Incomplete or corrupted file | Request new license |
| `"Deployment model mismatch: license='on-premises', configured='air-gapped'"` | DEPLOYMENT_MODEL doesn't match license | Fix `.env` or request correct license |
| `"CMP version 9.5.0 not permitted by license (allowed: 8.0.0 - 8.99.99)"` | Version outside licensed range | Upgrade license or use permitted version |

### Runtime Errors (Application Running)

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `"User limit exceeded. Licensed maximum: N users."` | Too many active users | Remove inactive users or upgrade license |
| `"Resource limit exceeded. Licensed maximum: N resources."` | Too many managed resources | Remove unmanaged resources or upgrade |
| `"This feature requires a license upgrade."` | Feature not in license | Upgrade to higher tier |

---

## Offline vs Online License Validation

### Offline Validation (Default)

All license validation is performed locally without any network calls:

- RSA signature verified using embedded public key
- Expiry checked against system clock
- Limits checked against local database counts
- No data sent to Autonimbus servers

This is the default for `on-premises` and `air-gapped` deployment models.

### Online Validation (Optional)

When enabled (typically for `saas` and `hybrid` models), CMP periodically verifies the license with Autonimbus:

- Verification occurs every 24 hours
- Connection timeout: 30 seconds
- Retry attempts: 3
- **If the vendor server is unreachable, CMP continues operating normally**

Online verification provides:
- Real-time license revocation capability
- Usage reporting to vendor
- Proactive renewal notifications

### Disabling Online Verification

For `air-gapped` deployments, all outbound calls are automatically disabled. For `on-premises`, online verification is disabled by default.

If your license has `vendor_verification.enabled: true` but you want to disable it, contact Autonimbus to issue a license with verification disabled.

---

## Contacting Vendor for License Issues

### When to Contact Autonimbus

- License is approaching expiry (within 30 days)
- You need to increase user or resource limits
- You want to add features (upgrade tier)
- License file appears corrupted (signature validation fails)
- Deployment model change required
- Version range needs updating for an upgrade

### Information to Provide

When contacting support, include:

1. **Customer ID** (from your license: `jq .customer.id license.json`)
2. **License ID** (from your license: `jq .license_id license.json`)
3. **Current CMP version** (`curl -s http://localhost:8001/version | jq .version`)
4. **Error message** (exact text from logs or API response)
5. **Deployment model** (your `DEPLOYMENT_MODEL` value)

### Contact Methods

- **Email:** support@autonimbus.com
- **Portal:** https://support.autonimbus.com (for online deployments)
- **Emergency:** Contact your designated support representative (premium tier)

### For Air-Gapped Environments

Since you cannot access online support portals:

1. Export the error from logs: `docker compose logs cmp-backend | grep -i license > license-issue.log`
2. Transfer the log file via your secure file exchange process
3. Contact your Autonimbus representative through your established communication channel

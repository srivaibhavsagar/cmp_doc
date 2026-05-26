# Vendor License Management Guide

> Internal guide for managing CMP customer licenses.
> Covers key generation, license creation, modification, tiers, and troubleshooting.

---

## Table of Contents

1. [RSA Key Pair Management](#rsa-key-pair-management)
2. [License File Schema Reference](#license-file-schema-reference)
3. [Creating a New Customer License](#creating-a-new-customer-license)
4. [Modifying an Existing License](#modifying-an-existing-license)
5. [License Type Tiers](#license-type-tiers)
6. [Feature Key Reference](#feature-key-reference)
7. [Deployment Model Constraints](#deployment-model-constraints)
8. [License Expiry and Renewal](#license-expiry-and-renewal)
9. [Troubleshooting License Validation Errors](#troubleshooting-license-validation-errors)

---

## RSA Key Pair Management

### Key Specifications

| Property | Value |
|----------|-------|
| Algorithm | RSA |
| Key Size | 4096 bits |
| Signature Scheme | RSA-PSS with SHA-256 |
| Salt Length | PSS.MAX_LENGTH |
| Public Key Format | PEM (embedded in application binary) |
| Private Key Format | PEM (stored in secure vault) |

### Generating a New Key Pair

Only generate a new key pair when:
- Initial setup of the licensing system
- Key compromise (emergency rotation)
- Scheduled key rotation (every 3–5 years)

```bash
# Generate RSA 4096-bit private key
openssl genpkey -algorithm RSA -out vendor_private_key.pem \
  -pkeyopt rsa_keygen_bits:4096

# Extract the public key
openssl rsa -in vendor_private_key.pem -pubout -out vendor_public_key.pem

# Verify key size
openssl rsa -in vendor_private_key.pem -text -noout | head -1
# Expected output: Private-Key: (4096 bit, 2 primes)
```

### Secure Storage

| Key | Storage Location | Access |
|-----|-----------------|--------|
| Private Key | Hardware Security Module (HSM) or encrypted vault (AWS KMS, HashiCorp Vault) | Release engineering team only (2-person rule) |
| Public Key | Embedded in application binary (`license_validator.py` → `EMBEDDED_PUBLIC_KEY_PEM`) | Read-only, compiled into binary |
| Backup | Encrypted offline backup in physically secured location | Emergency access only |

### Key Rotation Procedure

1. Generate new key pair
2. Update `EMBEDDED_PUBLIC_KEY_PEM` in `backend/app/services/license_validator.py`
3. Build and release a new version with the updated public key
4. Re-sign ALL active customer licenses with the new private key
5. Distribute updated licenses to all customers
6. Customers must upgrade to the new version before the re-signed licenses take effect
7. Retire the old private key after all customers have upgraded

**Warning**: Key rotation requires coordinated customer upgrades. Plan at least 90 days for the transition.

---

## License File Schema Reference

### Complete Schema

```json
{
  "version": "1.0",
  "license_id": "string (required, unique identifier)",
  "customer": {
    "name": "string (required, customer display name)",
    "id": "string (required, unique customer identifier)"
  },
  "type": "string (required, enum: trial | standard | enterprise)",
  "environment": "string (required, enum: saas | hybrid | on-premises | air-gapped)",
  "issued_at": "string (required, ISO 8601 datetime with timezone)",
  "expires_at": "string (required, ISO 8601 datetime with timezone)",
  "limits": {
    "max_users": "integer (required, >= 1)",
    "max_managed_resources": "integer (required, >= 1)"
  },
  "cloud_providers": {
    "aws": "integer (required, >= 0, 0 = disabled)",
    "azure": "integer (required, >= 0, 0 = disabled)",
    "gcp": "integer (required, >= 0, 0 = disabled)"
  },
  "features": ["string[] (required, list of enabled feature keys)"],
  "permitted_versions": {
    "min": "string (required, semver: MAJOR.MINOR.PATCH)",
    "max": "string (required, semver: MAJOR.MINOR.PATCH)"
  },
  "support_tier": "string (required, enum: basic | standard | premium)",
  "deployment_model": "string (required, enum: saas | hybrid | on-premises | air-gapped)",
  "vendor_verification": {
    "enabled": "boolean (required)",
    "endpoint": "string (optional, URL for vendor verification)"
  },
  "signature": "string (required, base64-encoded RSA-PSS signature)"
}
```

### Field Validation Rules

| Field | Type | Constraints |
|-------|------|-------------|
| `version` | string | Must be `"1.0"` (current schema version) |
| `license_id` | string | Non-empty, unique across all licenses |
| `customer.name` | string | Non-empty |
| `customer.id` | string | Non-empty, unique per customer |
| `type` | enum | One of: `trial`, `standard`, `enterprise` |
| `environment` | enum | One of: `saas`, `hybrid`, `on-premises`, `air-gapped` |
| `issued_at` | datetime | Valid ISO 8601 with timezone, must be before `expires_at` |
| `expires_at` | datetime | Valid ISO 8601 with timezone, must be after `issued_at` |
| `limits.max_users` | integer | >= 1 |
| `limits.max_managed_resources` | integer | >= 1 |
| `cloud_providers.aws` | integer | >= 0 (0 = AWS disabled) |
| `cloud_providers.azure` | integer | >= 0 (0 = Azure disabled) |
| `cloud_providers.gcp` | integer | >= 0 (0 = GCP disabled) |
| `features` | string[] | Non-empty list of valid feature keys |
| `permitted_versions.min` | string | Valid semver (MAJOR.MINOR.PATCH) |
| `permitted_versions.max` | string | Valid semver, must be >= `min` |
| `support_tier` | enum | One of: `basic`, `standard`, `premium` |
| `deployment_model` | enum | One of: `saas`, `hybrid`, `on-premises`, `air-gapped` |
| `vendor_verification.enabled` | boolean | Required |
| `vendor_verification.endpoint` | string | Valid URL if `enabled` is true |
| `signature` | string | Valid base64-encoded RSA-PSS signature |

### Critical Constraint

The `environment` field and `deployment_model` field **must match** the customer's configured `DEPLOYMENT_MODEL` environment variable. If they don't match, the application will refuse to start.

---

## Creating a New Customer License

### Prerequisites

- Customer contract signed and recorded in CRM
- Customer ID assigned (format: `cust_<identifier>`)
- License ID generated (format: `lic_<customer>_<year>_<sequence>`)
- Vendor private key accessible

### Step-by-Step Procedure

#### 1. Gather Customer Information

| Information | Source | Example |
|-------------|--------|---------|
| Customer name | Contract | "Acme Corp" |
| Customer ID | CRM system | "cust_acme_789" |
| License tier | Contract | "enterprise" |
| Deployment model | Technical assessment | "on-premises" |
| User count | Contract | 500 |
| Resource limit | Contract | 10000 |
| Features | Contract/tier | See tier table below |
| Duration | Contract | 12 months |
| Support tier | Contract | "premium" |
| Version range | Current release | "8.0.0" to "9.99.99" |

#### 2. Create the License Payload

```python
#!/usr/bin/env python3
"""generate_license.py — Create and sign a CMP customer license."""

import json
import base64
import sys
from datetime import datetime, timezone, timedelta
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

def generate_license(
    customer_name: str,
    customer_id: str,
    license_type: str,
    deployment_model: str,
    max_users: int,
    max_resources: int,
    features: list[str],
    duration_days: int,
    support_tier: str,
    min_version: str,
    max_version: str,
    vendor_verification: bool = False,
    private_key_path: str = "/secure/vendor_private_key.pem",
) -> dict:
    """Generate a signed license file."""
    
    now = datetime.now(timezone.utc)
    
    # Build the license payload
    payload = {
        "version": "1.0",
        "license_id": f"lic_{customer_id.replace('cust_', '')}_{now.year}_{now.strftime('%m%d')}",
        "customer": {
            "name": customer_name,
            "id": customer_id,
        },
        "type": license_type,
        "environment": deployment_model,
        "issued_at": now.isoformat(),
        "expires_at": (now + timedelta(days=duration_days)).isoformat(),
        "limits": {
            "max_users": max_users,
            "max_managed_resources": max_resources,
        },
        "features": features,
        "permitted_versions": {
            "min": min_version,
            "max": max_version,
        },
        "support_tier": support_tier,
        "deployment_model": deployment_model,
        "vendor_verification": {
            "enabled": vendor_verification,
            "endpoint": "https://license.autonimbus.com/verify" if vendor_verification else None,
        },
    }
    
    # Sign the payload
    with open(private_key_path, "rb") as f:
        private_key = serialization.load_pem_private_key(f.read(), password=None)
    
    payload_bytes = json.dumps(
        payload, sort_keys=True, separators=(",", ":"), default=str
    ).encode("utf-8")
    
    signature = private_key.sign(
        payload_bytes,
        padding.PSS(
            mgf=padding.MGF1(hashes.SHA256()),
            salt_length=padding.PSS.MAX_LENGTH,
        ),
        hashes.SHA256(),
    )
    
    payload["signature"] = base64.b64encode(signature).decode("utf-8")
    return payload


if __name__ == "__main__":
    # Example: Generate an enterprise license for Acme Corp
    license_data = generate_license(
        customer_name="Acme Corp",
        customer_id="cust_acme_789",
        license_type="enterprise",
        deployment_model="on-premises",
        max_users=500,
        max_resources=10000,
        features=[
            "catalog",
            "workflows",
            "scheduled_jobs",
            "terraform",
            "cost_management",
            "reports",
            "policy_governance",
            "budgets",
            "event_automation",
            "webhooks",
            "cloud_accounts",
            "multi_tenancy",
            "sso",
            "ai_assistant",
        ],
        duration_days=365,
        support_tier="premium",
        min_version="8.0.0",
        max_version="9.99.99",
    )
    
    output_file = f"license_{license_data['customer']['id']}.json"
    with open(output_file, "w") as f:
        json.dump(license_data, f, indent=2)
    
    print(f"License generated: {output_file}")
    print(f"  License ID: {license_data['license_id']}")
    print(f"  Customer:   {license_data['customer']['name']}")
    print(f"  Type:       {license_data['type']}")
    print(f"  Expires:    {license_data['expires_at']}")
    print(f"  Features:   {len(license_data['features'])} enabled")
```

#### 3. Validate Before Delivery

```bash
# Verify the license can be parsed and signature is valid
python scripts/license/validate_license.py license_cust_acme_789.json
```

#### 4. Deliver to Customer

- Use secure file transfer (encrypted email, SFTP, secure portal)
- Include installation instructions (place at `/app/license/license.json`)
- Record delivery in CRM with timestamp and delivery method

---

## Modifying an Existing License

### Common Modifications

| Change | Fields Affected | Notes |
|--------|----------------|-------|
| Add features | `features` | Append new feature keys |
| Increase user limit | `limits.max_users` | Update to new value |
| Increase resource limit | `limits.max_managed_resources` | Update to new value |
| Change cloud provider limits | `cloud_providers.aws/azure/gcp` | Set new account limits per provider |
| Enable a cloud provider | `cloud_providers.<provider>` | Set from 0 to desired limit |
| Extend expiry | `expires_at` | Set new expiry date |
| Upgrade tier | `type`, `features`, `limits`, `cloud_providers` | Update all tier-related fields |
| Change deployment model | `environment`, `deployment_model`, `vendor_verification` | Both fields must match |
| Expand version range | `permitted_versions.max` | Extend to cover new major version |

### Procedure

1. Load the existing license payload (without signature)
2. Modify the required fields
3. Re-sign with the vendor private key (generates new signature)
4. Deliver the updated license to the customer

```python
# Modify and re-sign an existing license
import json

# Load existing license
with open("license_cust_acme_789.json", "r") as f:
    license_data = json.load(f)

# Remove old signature
del license_data["signature"]

# Make modifications
license_data["limits"]["max_users"] = 1000  # Increased from 500
license_data["features"].append("advanced_analytics")  # New feature
license_data["expires_at"] = "2027-01-15T00:00:00+00:00"  # Extended

# Re-sign (same process as initial generation)
# ... (use the signing code from above)
```

### Hot-Reload

The application detects license file changes within **60 seconds** via SHA-256 hash comparison. Customers can update their license file without restarting the application — new features and limits take effect automatically.

---

## License Type Tiers

### Trial

| Property | Value |
|----------|-------|
| Duration | 30 days |
| Max Users | 10 |
| Max Resources | 100 |
| Cloud Providers | AWS: 1, Azure: 0, GCP: 0 |
| Support Tier | basic |
| Vendor Verification | enabled |

**Included Features:**
- `catalog`
- `workflows`
- `cost_management`
- `cloud_accounts`
- `notifications`
- `infrastructure`

### Standard

| Property | Value |
|----------|-------|
| Duration | 12 months (renewable) |
| Max Users | 50–200 |
| Max Resources | 1,000–5,000 |
| Cloud Providers | AWS: 3, Azure: 2, GCP: 2 |
| Support Tier | standard |
| Vendor Verification | per deployment model |

**Included Features:**
- `catalog`
- `workflows`
- `scheduled_jobs`
- `terraform`
- `cost_management`
- `reports`
- `policy_governance`
- `budgets`
- `event_automation`
- `webhooks`
- `cloud_accounts`
- `notifications`
- `infrastructure`
- `logging_monitoring`

### Enterprise

| Property | Value |
|----------|-------|
| Duration | 12–36 months (renewable) |
| Max Users | 100–unlimited |
| Max Resources | 5,000–unlimited |
| Cloud Providers | AWS: 10, Azure: 10, GCP: 10 (or custom) |
| Support Tier | premium |
| Vendor Verification | per deployment model |

**Included Features (all):**
- `catalog`
- `workflows`
- `scheduled_jobs`
- `terraform`
- `cost_management`
- `reports`
- `policy_governance`
- `budgets`
- `event_automation`
- `webhooks`
- `cloud_accounts`
- `multi_tenancy`
- `sso`
- `ai_assistant`
- `notifications`
- `infrastructure`
- `logging_monitoring`

---

## Feature Key Reference

| Feature Key | CMP Capability | Minimum Tier |
|-------------|---------------|--------------|
| `catalog` | Self-service catalog for provisioning resources | Trial |
| `workflows` | Workflow automation and orchestration (tasks, flows, executions) | Trial |
| `scheduled_jobs` | Scheduled/recurring automation jobs | Standard |
| `terraform` | Terraform templates, workspaces, drift detection, state backend | Standard |
| `cost_management` | Cost tracking, models, cloud pricing, analytics | Trial |
| `reports` | Reports and analytics dashboards | Standard |
| `policy_governance` | Policy engine, quotas, approval workflows | Standard |
| `budgets` | Budget management and alerts | Standard |
| `event_automation` | Event-driven automation rules + event log | Standard |
| `webhooks` | Inbound webhook endpoints | Standard |
| `cloud_accounts` | Cloud account management (AWS/Azure/GCP credential CRUD), resources | Trial |
| `multi_tenancy` | Multi-tenant isolation and management | Enterprise |
| `sso` | Single Sign-On integration (OIDC/SAML) | Enterprise |
| `ai_assistant` | AI-powered assistant (Gemini integration) | Enterprise |
| `notifications` | User notification center (in-app, email, Slack, Teams) | Starter |
| `infrastructure` | Infrastructure management (resource actions, shared variables) | Starter |
| `logging_monitoring` | Platform logging and monitoring dashboards | Standard |

### Feature Gating Behavior

A feature is accessible to end users only when **both** conditions are met:
1. The feature key is present in the license's `features` list
2. The admin-configured feature toggle for that key is enabled

If either condition is false:
- The feature toggle endpoint reports the feature as disabled
- The frontend hides the feature from navigation
- API requests to the feature return HTTP 403 with message: "This feature requires a license upgrade"

---

## Deployment Model Constraints

### Matching Rules

The license `environment` field **must exactly match** the application's `DEPLOYMENT_MODEL` environment variable:

| License `environment` | Required `DEPLOYMENT_MODEL` | Behavior |
|----------------------|---------------------------|----------|
| `saas` | `saas` | Vendor verification enabled, auto-update notifications |
| `hybrid` | `hybrid` | Vendor verification per config, shared responsibility |
| `on-premises` | `on-premises` | Vendor verification optional, customer-managed |
| `air-gapped` | `air-gapped` | All outbound calls disabled, fully offline |

**Mismatch = Startup Failure**: If the license environment doesn't match `DEPLOYMENT_MODEL`, the application logs the mismatch and exits with code 1.

### Deployment-Specific License Settings

| Deployment Model | `vendor_verification.enabled` | Outbound Calls | Update Checks |
|-----------------|------------------------------|----------------|---------------|
| SaaS | `true` (required) | Allowed | Automatic |
| Hybrid | Configurable | Allowed | Configurable |
| On-Premises | Configurable | Allowed | Manual |
| Air-Gapped | `false` (required) | Blocked | Disabled |

### Air-Gapped Specific Rules

For air-gapped licenses:
- `vendor_verification.enabled` must be `false`
- `vendor_verification.endpoint` is ignored
- No telemetry, update checks, or outbound network calls
- License validation is entirely offline
- Version updates require physical media delivery

---

## License Expiry and Renewal

### Expiry Behavior

When a license expires (`expires_at` timestamp passes):
- **Running application**: Continues operating until next restart
- **Application restart**: Refuses to start, logs expiry date and renewal instructions
- **Grace period**: None — expiry is enforced immediately on startup

### Renewal Workflow

1. **90 days before expiry**: Send renewal reminder to customer (per support tier)
2. **60 days before expiry**: Second reminder with renewal quote
3. **30 days before expiry**: Final reminder, escalate to account manager
4. **On renewal**: Generate new license with updated `expires_at`, deliver to customer
5. **Post-renewal**: Customer places new license file; hot-reload applies within 60 seconds

### Renewal License Generation

```python
# Renew: extend expiry by contract duration
license_data["issued_at"] = datetime.now(timezone.utc).isoformat()
license_data["expires_at"] = (datetime.now(timezone.utc) + timedelta(days=365)).isoformat()
# Re-sign and deliver
```

### Tracking Expiry

Maintain a dashboard of all active licenses with:
- Customer name and ID
- Current expiry date
- Days remaining
- Support tier (determines notification schedule)
- Deployment model
- Last verification timestamp (for SaaS/hybrid)

---

## Troubleshooting License Validation Errors

### Error: "License file not found"

```
License file not found at: /app/license/license.json
```

**Cause**: The license file doesn't exist at the expected path.

**Resolution**:
1. Verify the file exists: `ls -la /app/license/license.json`
2. Check the `LICENSE_FILE_PATH` environment variable if using a custom path
3. Verify file permissions (must be readable by the application user)
4. For Docker deployments, verify the volume mount is correct

### Error: "License signature validation failed"

```
License signature validation failed — file may be tampered
```

**Cause**: The license file content doesn't match its cryptographic signature.

**Resolution**:
1. Verify the license file hasn't been manually edited after signing
2. Verify the file wasn't corrupted during transfer (check file size, encoding)
3. Verify the application binary contains the correct public key
4. Re-generate and re-sign the license if the file was modified
5. Check for BOM (byte order mark) or encoding issues — file must be UTF-8

### Error: "License expired"

```
License expired on 2025-01-15T00:00:00+00:00. Please contact Autonimbus to renew your license.
```

**Cause**: The current system time is at or past the license's `expires_at` timestamp.

**Resolution**:
1. Verify the system clock is correct (`date -u`)
2. Generate a renewed license with an updated `expires_at`
3. Deliver to customer; hot-reload applies within 60 seconds

### Error: "License environment type does not match DEPLOYMENT_MODEL"

```
License environment type 'on-premises' does not match configured DEPLOYMENT_MODEL 'saas'. Startup rejected due to environment mismatch.
```

**Cause**: The license was generated for a different deployment model than what's configured.

**Resolution**:
1. Verify the customer's `DEPLOYMENT_MODEL` environment variable
2. Generate a new license with the correct `environment` and `deployment_model` fields
3. Both fields must match the customer's configured `DEPLOYMENT_MODEL`

### Error: "Application version outside licensed range"

```
Application version 10.0.0 is outside the licensed range [8.0.0, 9.99.99]. Please upgrade your license.
```

**Cause**: The customer upgraded to a version not covered by their license.

**Resolution**:
1. Generate a new license with expanded `permitted_versions.max`
2. Or instruct the customer to downgrade to a version within their licensed range

### Error: "License file contains malformed JSON"

```
License file contains malformed JSON: Expecting ',' delimiter: line 15 column 3
```

**Cause**: The license file is not valid JSON.

**Resolution**:
1. Validate the JSON: `python -m json.tool license.json`
2. Check for trailing commas, missing quotes, or encoding issues
3. Re-generate the license from the tooling (don't hand-edit)

### Error: "License file contains invalid data"

```
License file contains invalid data: 1 validation error for LicenseFile
limits -> max_users: ensure this value is greater than or equal to 1
```

**Cause**: A field value doesn't meet the Pydantic validation constraints.

**Resolution**:
1. Check the specific field mentioned in the error
2. Refer to the schema reference above for valid values
3. Re-generate with corrected values

### Error: "User limit exceeded" / "Resource limit exceeded"

```
User limit exceeded: current count (501) exceeds licensed maximum (500)
```

**Cause**: Runtime enforcement — the customer has exceeded their licensed limits.

**Resolution**:
1. This is a runtime error (HTTP 403), not a startup error
2. Customer must either reduce usage or upgrade their license limits
3. Generate a new license with increased `max_users` or `max_managed_resources`

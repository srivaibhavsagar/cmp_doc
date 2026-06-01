# Vendor Release Operations Guide

> Internal operations guide for the Autonimbus release engineering team.
> Covers the complete release lifecycle from tagging through customer delivery.

---

## Table of Contents

1. [Version Numbering Strategy](#version-numbering-strategy)
2. [Triggering a Release Build](#triggering-a-release-build)
3. [Release Pipeline Stages](#release-pipeline-stages)
4. [Generating Customer License Files](#generating-customer-license-files)
5. [Signing License Files](#signing-license-files)
6. [Provisioning Customer Registry Credentials](#provisioning-customer-registry-credentials)
7. [Revoking Customer Access](#revoking-customer-access)
8. [Creating Air-Gapped Packages](#creating-air-gapped-packages)
9. [Release Checklist](#release-checklist)
10. [Hotfix Releases](#hotfix-releases)

---

## Version Numbering Strategy

CMP follows strict [Semantic Versioning](https://semver.org/) (SemVer):

```
MAJOR.MINOR.PATCH
  │      │     └── Bug fixes, security patches (backward-compatible)
  │      └──────── New features, non-breaking changes
  └─────────────── Breaking changes, major architecture shifts
```

### Rules

| Scenario | Version Bump | Example |
|----------|-------------|---------|
| Bug fix or security patch | PATCH | 8.10.0 → 8.10.1 |
| New feature (backward-compatible) | MINOR | 8.10.1 → 8.11.0 |
| Breaking change or major rework | MAJOR | 8.11.0 → 9.0.0 |
| Emergency hotfix | PATCH | 8.10.0 → 8.10.1 |

### Constraints

- Customers can only upgrade **one major version at a time** (e.g., 8.x → 9.x is allowed; 8.x → 10.x is rejected by the upgrade orchestrator).
- The license file's `permitted_versions` field controls which versions a customer can run. Always set `max` to cover the next major version (e.g., `"max": "9.99.99"` for an enterprise license).
- Image tags follow the format: `<version>-<8-char-commit-hash>` (e.g., `8.10.0-a1b2c3d4`).

### Git Tag Format

```
v<MAJOR>.<MINOR>.<PATCH>
```

Examples: `v8.10.0`, `v9.0.0`, `v8.10.1`

---

## Triggering a Release Build

The release pipeline (`.github/workflows/release.yml`) is triggered in two ways:

### Method 1: Git Tag Push (Standard Release)

```bash
# Ensure you're on the release branch with all changes merged
git checkout main
git pull origin main

# Create an annotated tag
git tag -a v8.10.0 -m "Release v8.10.0: Feature X, Fix Y"

# Push the tag to trigger the pipeline
git push origin v8.10.0
```

The pipeline triggers on tags matching the pattern `v[0-9]+.[0-9]+.[0-9]+`.

### Method 2: Manual Workflow Dispatch (Emergency/Hotfix)

1. Navigate to **Actions** → **CMP Release — Enterprise Package Build**
2. Click **Run workflow**
3. Fill in parameters:
   - **version**: The release version (e.g., `8.10.1`)
   - **skip_scan**: Set to `true` only for emergency hotfixes (skips Trivy vulnerability scan)
4. Click **Run workflow**

### Important Notes

- The release pipeline is **completely separate** from `deploy-dev.yml`, `deploy-azure-dev.yml`, `deploy.yml`, and `dev-ops.yml`.
- The pipeline verifies that existing workflow files are unmodified in the release commit.
- Never push tags from feature branches — always tag from `main` after all PRs are merged.

---

## Release Pipeline Stages

The pipeline executes the following stages in order:

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Prepare   │───▶│ Verify Workflows │───▶│ Compile Backend │
└─────────────┘    └──────────────────┘    └─────────────────┘
                                                     │
                   ┌──────────────────┐              │
                   │  Build Frontend  │◀─────────────┘
                   └──────────────────┘
                            │
                   ┌──────────────────┐
                   │  Build Images    │ (matrix: backend, frontend, worker, ai)
                   └──────────────────┘
                            │
                   ┌──────────────────┐
                   │  Scan Images     │ (Trivy vulnerability scan)
                   └──────────────────┘
                            │
                   ┌──────────────────┐
                   │ Publish Images   │ (push to <account_id>.dkr.ecr.<region>.amazonaws.com)
                   └──────────────────┘
                            │
                   ┌──────────────────┐
                   │ Assemble Package │ (release archive + signing)
                   └──────────────────┘
```

### Stage Details

| Stage | Purpose | Failure Behavior |
|-------|---------|-----------------|
| **Prepare** | Determine version and commit metadata | Pipeline halts |
| **Verify Workflows** | Confirm protected workflow files are unmodified | Pipeline halts with error listing modified files |
| **Compile Backend** | Nuitka compilation → standalone Linux x86_64 binary | Reports failing module name, exits non-zero |
| **Build Frontend** | Vite production build with obfuscation, no source maps | Reports failing file/module, exits non-zero |
| **Build Images** | Multi-stage Docker builds for all 4 services | Reports image name and error |
| **Scan Images** | Trivy scan for critical/high CVEs | Reports affected image, CVE IDs, affected packages |
| **Publish Images** | Push to `<account_id>.dkr.ecr.<region>.amazonaws.com` | Reports push failure |
| **Assemble Package** | Generate manifest, sign, create release archive | Reports missing artifacts or signing failure |

### Pipeline Secrets Required

| Secret | Purpose |
|--------|---------|
| `REGISTRY_USERNAME` | Private registry authentication |
| `REGISTRY_PASSWORD` | Private registry authentication |
| `CODE_SIGNING_PRIVATE_KEY` | RSA key for signing release manifests |

---

## Generating Customer License Files

### Prerequisites

- Access to the vendor RSA 4096-bit private key (stored in secure vault)
- Customer information (name, ID, contract details)
- Python 3.12+ with `cryptography` library installed

### License File Schema

```json
{
  "version": "1.0",
  "license_id": "lic_<unique-id>",
  "customer": {
    "name": "Customer Name",
    "id": "cust_<unique-id>"
  },
  "type": "enterprise",
  "environment": "on-premises",
  "issued_at": "2025-01-15T00:00:00Z",
  "expires_at": "2026-01-15T00:00:00Z",
  "limits": {
    "max_users": 500,
    "max_managed_resources": 10000
  },
  "features": [
    "catalog",
    "workflows",
    "cost_management",
    "policy_governance",
    "ai_assistant",
    "sso",
    "multi_tenancy",
    "budgets",
    "scheduled_jobs"
  ],
  "permitted_versions": {
    "min": "8.0.0",
    "max": "9.99.99"
  },
  "support_tier": "premium",
  "deployment_model": "on-premises",
  "vendor_verification": {
    "enabled": false,
    "endpoint": "https://license.autonimbus.com/verify"
  }
}
```

### Step-by-Step: Generate a License

```bash
# 1. Create the license payload (all fields except signature)
cat > /tmp/license_payload.json << 'EOF'
{
  "version": "1.0",
  "license_id": "lic_acme_2025_001",
  "customer": {
    "name": "Acme Corp",
    "id": "cust_acme_789"
  },
  "type": "enterprise",
  "environment": "on-premises",
  "issued_at": "2025-01-15T00:00:00Z",
  "expires_at": "2026-01-15T00:00:00Z",
  "limits": {
    "max_users": 500,
    "max_managed_resources": 10000
  },
  "features": [
    "catalog",
    "workflows",
    "cost_management",
    "policy_governance",
    "ai_assistant",
    "sso",
    "multi_tenancy",
    "budgets",
    "scheduled_jobs"
  ],
  "permitted_versions": {
    "min": "8.0.0",
    "max": "9.99.99"
  },
  "support_tier": "premium",
  "deployment_model": "on-premises",
  "vendor_verification": {
    "enabled": false,
    "endpoint": "https://license.autonimbus.com/verify"
  }
}
EOF
```

```python
# 2. Sign the license using the vendor private key
import json
import base64
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

# Load the vendor private key
with open("/path/to/vendor_private_key.pem", "rb") as f:
    private_key = serialization.load_pem_private_key(f.read(), password=None)

# Load the license payload
with open("/tmp/license_payload.json", "r") as f:
    payload = json.load(f)

# Serialize payload deterministically (sorted keys, no whitespace)
payload_bytes = json.dumps(payload, sort_keys=True, separators=(",", ":")).encode("utf-8")

# Sign with RSA-PSS (SHA-256)
signature = private_key.sign(
    payload_bytes,
    padding.PSS(
        mgf=padding.MGF1(hashes.SHA256()),
        salt_length=padding.PSS.MAX_LENGTH,
    ),
    hashes.SHA256(),
)

# Add signature to payload
payload["signature"] = base64.b64encode(signature).decode("utf-8")

# Write the final license file
with open("license.json", "w") as f:
    json.dump(payload, f, indent=2)

print(f"License generated: license.json")
print(f"  Customer: {payload['customer']['name']}")
print(f"  Expires:  {payload['expires_at']}")
print(f"  Type:     {payload['type']}")
```

### Delivery

- Deliver the `license.json` file to the customer via secure channel (encrypted email, secure file transfer).
- The customer places it at `/app/license/license.json` (or the path configured via `LICENSE_FILE_PATH` environment variable).

---

## Signing License Files

### Signature Algorithm

- **Algorithm**: RSA-PSS with SHA-256
- **Key Size**: 4096 bits
- **Salt Length**: Maximum (PSS.MAX_LENGTH)
- **Payload**: All license fields except `signature`, serialized as sorted JSON with no whitespace (`separators=(",", ":")`)

### Verification

The application binary embeds the RSA public key and verifies the signature at startup. Any modification to the license file after signing will cause signature verification to fail, preventing the application from starting.

---

## Provisioning Customer Registry Credentials

### Generate Credentials

```bash
# Generate unique pull credentials for a customer
# Credentials are scoped to the customer's licensed image tags

# Example: Generate credentials for Acme Corp (licensed for v8.x and v9.x)
python scripts/registry/generate_credentials.py \
  --customer-id "cust_acme_789" \
  --customer-name "Acme Corp" \
  --scope "autonimbus/cmp/*:8.*,autonimbus/cmp/*:9.*" \
  --expires-days 365
```

### Credential Scoping

Credentials are scoped to specific image tags based on the customer's `permitted_versions`:

| License `permitted_versions` | Registry Scope |
|------------------------------|---------------|
| `min: 8.0.0, max: 8.99.99` | `autonimbus/cmp/*:8.*` |
| `min: 8.0.0, max: 9.99.99` | `autonimbus/cmp/*:8.*,autonimbus/cmp/*:9.*` |

### Credential Rotation

When rotating credentials:

1. Generate new credentials with the same scope
2. Both old and new credentials remain valid for a **24-hour overlap period**
3. Notify the customer to update their credentials
4. After 24 hours, the old credentials are automatically invalidated

```bash
# Rotate credentials with 24-hour overlap
python scripts/registry/rotate_credentials.py \
  --customer-id "cust_acme_789" \
  --overlap-hours 24
```

### Delivery

Deliver credentials to the customer via secure channel. The customer configures them in their environment:

```bash
# Customer's .env file
REGISTRY_USERNAME=cust_acme_789
REGISTRY_PASSWORD=<generated-token>
```

---

## Revoking Customer Access

When a customer's license is revoked or expires:

### Step 1: Revoke Registry Credentials

```bash
# Invalidate all pull credentials for the customer
python scripts/registry/revoke_credentials.py \
  --customer-id "cust_acme_789" \
  --reason "License expired"
```

Credentials are invalidated within **24 hours** of revocation. The customer will receive descriptive error messages indicating expired credentials on their next pull attempt.

### Step 2: Revoke License

The license itself enforces expiry — once `expires_at` passes, the application will refuse to start. No additional action is needed for license-level revocation beyond letting it expire.

For immediate revocation (before natural expiry):
1. Do NOT issue a renewed license
2. Revoke registry credentials (prevents pulling new versions)
3. The running instance continues until restart (at which point the expired/revoked license blocks startup)

### Step 3: Document the Revocation

Record the revocation in the customer management system with:
- Customer ID and name
- Revocation date and reason
- Credentials invalidated
- Expected impact timeline

---

## Creating Air-Gapped Packages

For customers in restricted environments (government, banking, healthcare) that cannot access the internet:

### Generate Air-Gapped Package

```bash
# Run the air-gapped packaging script after a successful release build
./scripts/build/package-airgapped.sh \
  --version 8.10.0 \
  --commit-hash a1b2c3d4 \
  --output-dir /tmp/airgapped-release
```

### Package Contents

```
cmp-release-v8.10.0/
├── docker-compose.production.yml
├── images/
│   ├── cmp-frontend-8.10.0-a1b2c3d4.tar.gz
│   ├── cmp-backend-8.10.0-a1b2c3d4.tar.gz
│   ├── cmp-worker-8.10.0-a1b2c3d4.tar.gz
│   ├── cmp-ai-8.10.0-a1b2c3d4.tar.gz
│   ├── redis-7-alpine.tar.gz
│   └── dynamodb-local.tar.gz
├── config/
│   ├── .env.template
│   ├── .env.template.air-gapped
│   └── encryption-key-gen.sh
├── migrations/
├── scripts/
│   ├── install.sh
│   ├── upgrade.sh
│   ├── verify-checksums.sh
│   ├── health-check.sh
│   └── backup.sh
├── docs/
│   ├── INSTALL-air-gapped.md
│   ├── UPGRADE.md
│   └── RELEASE-NOTES.md
├── license/
│   └── license.json  (customer-specific, added before delivery)
├── MANIFEST.sha256
└── MANIFEST.sha256.sig
```

### Delivery Process

1. Generate the air-gapped package from the release pipeline artifacts
2. Add the customer-specific `license.json` to the `license/` directory
3. Regenerate the SHA-256 manifest to include the license file
4. Re-sign the manifest with the vendor code-signing key
5. Transfer via approved secure media (encrypted USB, secure courier, approved file transfer)
6. Provide the manifest verification public key separately (out-of-band)

### Customer Installation

The customer runs:
```bash
# Verify package integrity
./scripts/verify-checksums.sh

# Install (loads images into local Docker daemon)
./scripts/install.sh
```

---

## Release Checklist

### Pre-Release

- [ ] All PRs for this release are merged to `main`
- [ ] `VERSION` file updated to the new version number
- [ ] All unit tests pass on `main`
- [ ] All property-based tests pass on `main`
- [ ] CHANGELOG or release notes drafted
- [ ] License `permitted_versions.max` covers this release for all active customers
- [ ] No critical/high CVEs in dependency audit (`pip audit`, `npm audit`)
- [ ] Database migration scripts written and tested (if schema changes)
- [ ] Rollback scripts written for all new migrations
- [ ] Breaking changes documented (if major version bump)

### Release Execution

- [ ] Create annotated git tag: `git tag -a v<version> -m "<message>"`
- [ ] Push tag: `git push origin v<version>`
- [ ] Monitor pipeline execution in GitHub Actions
- [ ] Verify all pipeline stages pass (compile, build, scan, publish, assemble)
- [ ] Verify images are available in `<account_id>.dkr.ecr.<region>.amazonaws.com`
- [ ] Download and inspect the release package artifact

### Post-Release

- [ ] Verify release package completeness (all required files present)
- [ ] Verify `MANIFEST.sha256` contains entries for all files
- [ ] Verify `MANIFEST.sha256.sig` is valid (signed correctly)
- [ ] Run `verify-checksums.sh` against the package
- [ ] Test installation from the package in a clean environment
- [ ] Test upgrade from previous version in a staging environment
- [ ] Generate air-gapped packages for air-gapped customers
- [ ] Notify customers of the new release (per support tier SLA)
- [ ] Update internal release tracking documentation

---

## Hotfix Releases

For critical issues requiring immediate patching:

### Process

1. Create a hotfix branch from the release tag:
   ```bash
   git checkout -b hotfix/8.10.1 v8.10.0
   ```

2. Apply the minimal fix and commit

3. Merge to `main` (via PR with expedited review)

4. Tag the hotfix release:
   ```bash
   git tag -a v8.10.1 -m "Hotfix: Critical security patch for CVE-XXXX"
   git push origin v8.10.1
   ```

5. If the vulnerability scan must be skipped (extreme emergency only):
   - Use manual workflow dispatch with `skip_scan: true`
   - Document the justification
   - Schedule a follow-up scan within 24 hours

### Hotfix Notification

Notify affected customers immediately via their configured support channel (email, Slack, PagerDuty) based on severity:

| Severity | Notification Timeline | Channel |
|----------|----------------------|---------|
| Critical (security) | Within 1 hour | Direct call + email |
| High (data loss risk) | Within 4 hours | Email + ticket |
| Medium (functionality) | Within 24 hours | Email |

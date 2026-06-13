# CMP Enterprise — Step-by-Step Operations Guide

> **For:** Vaibhav (Vendor/Developer perspective)
> **Covers:** Installation for all deployment models, version upgrades, license management, and day-to-day operations

---

## Table of Contents

1. [Prerequisites (All Models)](#prerequisites-all-models)
2. [Installation — On-Premises](#installation--on-premises)
3. [Installation — Air-Gapped](#installation--air-gapped)
4. [Installation — Hybrid](#installation--hybrid)
5. [Installation — SaaS](#installation--saas)
6. [Upgrading CMP Version](#upgrading-cmp-version)
7. [License Operations](#license-operations)
8. [Rollback Procedures](#rollback-procedures)
9. [Day-to-Day Operations](#day-to-day-operations)
10. [Releasing a New Version (Vendor)](#releasing-a-new-version-vendor)

---

## Prerequisites (All Models)

### System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 8+ cores |
| RAM | 8 GB | 16+ GB |
| Disk | 50 GB SSD | 100+ GB SSD |
| Docker Engine | 24.0+ | Latest stable |
| Docker Compose | v2.20+ | Latest stable |
| OS | Linux x86_64 | Ubuntu 22.04 LTS / RHEL 8+ |

### What You Need Before Starting

| Item | Where to Get It |
|------|-----------------|
| License file (`license.json`) | You generate it (see [License Operations](#license-operations)) |
| Registry credentials | You generate them (for online deployments) |
| Air-gapped package | You build it from the release pipeline |
| RSA private key | Your secure vault (for signing licenses) |
| Code-signing key | GitHub Secrets (for release pipeline) |

---

## Installation — On-Premises

> Customer has internet access but manages their own infrastructure.

### Step 1: Generate the Customer License

```bash
cd cloud_management_platform/backend

python3 -c "
import json, base64
from datetime import datetime, timezone, timedelta
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

# Load your vendor private key
with open('/path/to/vendor_private_key.pem', 'rb') as f:
    private_key = serialization.load_pem_private_key(f.read(), password=None)

# Build the license
payload = {
    'version': '1.0',
    'license_id': 'lic_customer_2026_001',
    'customer': {'name': 'Customer Name', 'id': 'cust_customer_001'},
    'type': 'enterprise',
    'environment': 'on-premises',
    'issued_at': datetime.now(timezone.utc).isoformat(),
    'expires_at': (datetime.now(timezone.utc) + timedelta(days=365)).isoformat(),
    'limits': {'max_users': 500, 'max_managed_resources': 10000},
    'features': ['catalog','workflows','cost_management','policy_governance','ai_assistant','sso','multi_tenancy','budgets','scheduled_jobs'],
    'permitted_versions': {'min': '1.0.0', 'max': '9.99.99'},
    'support_tier': 'premium',
    'deployment_model': 'on-premises',
    'vendor_verification': {'enabled': False, 'endpoint': None}
}

# Sign it
payload_bytes = json.dumps(payload, sort_keys=True, separators=(',',':')).encode()
signature = private_key.sign(
    payload_bytes,
    padding.PSS(mgf=padding.MGF1(hashes.SHA256()), salt_length=padding.PSS.MAX_LENGTH),
    hashes.SHA256()
)
payload['signature'] = base64.b64encode(signature).decode()

# Save
with open('license.json', 'w') as f:
    json.dump(payload, f, indent=2)
print('License generated: license.json')
"
```

### Step 2: Generate Registry Credentials

```bash
python3 -c "
from app.services.registry_manager import RegistryManager
rm = RegistryManager()
cred = rm.generate_credential('cust_customer_001', ['1.0.0', 'latest'])
print(f'Username: {cred.username}')
print(f'Password: {cred.password_plain}')
print(f'Registry: <account_id>.dkr.ecr.<region>.amazonaws.com')
"
```

### Step 3: Send to Customer

Deliver securely:
- `license.json`
- Registry username + password (separate message)
- `.env.template.on-premises` (from `config/`)
- Installation instructions (below)

### Step 4: Customer Installation Steps

```bash
# 1. Create installation directory
mkdir -p /opt/cmp/{license,config,logs,backups}
cd /opt/cmp

# 2. Login to private registry
docker login <account_id>.dkr.ecr.<region>.amazonaws.com \
  --username <provided-username> \
  --password <provided-password>

# 3. Place the docker-compose file
# (provided by vendor or pulled from registry)
cp /path/to/docker-compose.production.yml ./docker-compose.yml

# 4. Configure environment
cp /path/to/.env.template.on-premises ./.env

# Edit .env — fill in required values:
#   SECRET_KEY=<generate with: openssl rand -hex 32>
#   ENCRYPTION_KEY=<generate with: python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())">
#   DYNAMODB_ENDPOINT=http://database:8000
#   AWS_REGION=us-east-1
#   DEPLOYMENT_MODEL=on-premises

# 5. Place license file
cp /path/to/license.json /opt/cmp/license/license.json

# 6. Pull images
docker compose --profile local pull

# 7. Start services
docker compose --profile local up -d

# 8. Wait for startup (30-60 seconds)
sleep 60

# 9. Verify
curl -s http://localhost:8000/health | python3 -m json.tool
curl -s -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/health/version | python3 -m json.tool
```

**Expected health response:**
```json
{
  "status": "healthy",
  "services": {
    "backend": "healthy",
    "frontend": "healthy",
    "worker": "healthy",
    "ai_service": "healthy",
    "redis": "healthy",
    "database": "healthy"
  }
}
```

---

## Installation — Air-Gapped

> Customer has NO internet access. Everything delivered via physical media.

### Step 1: Generate License (same as on-premises but with `air-gapped`)

```bash
# Same as above but change these fields:
#   'environment': 'air-gapped',
#   'deployment_model': 'air-gapped',
#   'vendor_verification': {'enabled': False, 'endpoint': None}
```

### Step 2: Build the Air-Gapped Package

```bash
cd cloud_management_platform

# First, ensure images are built (run the release pipeline or build locally)
# Then package for air-gapped delivery:
./scripts/build/package-airgapped.sh \
  --version 1.0.0 \
  --output-dir ./dist
```

This creates: `dist/cmp-airgapped-v1.0.0/` containing all images, scripts, configs, and docs.

### Step 3: Add Customer License to Package

```bash
cp license.json dist/cmp-airgapped-v1.0.0/license/license.json

# Regenerate checksums (since we added a file)
cd dist/cmp-airgapped-v1.0.0
find . -type f -not -name "MANIFEST.sha256" | sort | while read f; do
    shasum -a 256 "$f"
done > MANIFEST.sha256
cd ../..
```

### Step 4: Create Distributable Archive

```bash
cd dist
tar -czf cmp-airgapped-v1.0.0.tar.gz cmp-airgapped-v1.0.0/
```

### Step 5: Deliver to Customer

Transfer `cmp-airgapped-v1.0.0.tar.gz` via:
- Encrypted USB drive
- Secure file transfer to DMZ
- Physical courier

### Step 6: Customer Installation Steps

```bash
# 1. Transfer package to server
cp /media/usb/cmp-airgapped-v1.0.0.tar.gz /opt/
cd /opt
tar -xzf cmp-airgapped-v1.0.0.tar.gz
cd cmp-airgapped-v1.0.0

# 2. Verify package integrity
./scripts/verify-checksums.sh
# Must show: "All checksums verified successfully!"
# If it fails — DO NOT PROCEED. Package may be corrupted.

# 3. Load container images
./scripts/install.sh
# Loads images in dependency order: redis → dynamodb → backend → worker → ai → frontend

# 4. Set up installation directory
mkdir -p /opt/cmp/{license,config,logs,backups}
cp docker-compose.yml /opt/cmp/docker-compose.yml
cp config/.env.template.air-gapped /opt/cmp/.env

# 5. Generate security keys
openssl rand -hex 32  # → copy to SECRET_KEY in .env
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# → copy to ENCRYPTION_KEY in .env

# 6. Edit .env
vi /opt/cmp/.env
# Set: SECRET_KEY, ENCRYPTION_KEY, DEPLOYMENT_MODEL=air-gapped

# 7. Place license
cp license/license.json /opt/cmp/license/license.json

# 8. Start
cd /opt/cmp
docker compose --profile local up -d

# 9. Verify (wait 60 seconds first)
sleep 60
curl -s http://localhost:8000/health | python3 -m json.tool
```

---

## Installation — Hybrid

> Customer manages infrastructure, vendor has monitoring access.

### Step 1: Generate License

Same as on-premises but:
```python
'environment': 'hybrid',
'deployment_model': 'hybrid',
'vendor_verification': {'enabled': True, 'endpoint': 'https://license.autonimbus.com/verify'}
```

### Step 2: Generate Registry Credentials (same as on-premises)

### Step 3: Customer Installation

Same as on-premises installation, but use `.env.template.hybrid`:

```bash
cp /path/to/.env.template.hybrid /opt/cmp/.env
# Key differences in hybrid template:
#   DEPLOYMENT_MODEL=hybrid
#   LICENSE_VERIFY_ONLINE=true
#   VENDOR_MONITORING_ENABLED=true
#   AUTO_UPDATE_NOTIFICATIONS=true
```

### Step 4: Verify Vendor Monitoring Access

After installation, confirm the health endpoint is accessible from vendor network:
```bash
curl -s https://cmp.customer.com/health
curl -s -H "Authorization: Bearer <admin_token>" \
  https://cmp.customer.com/health/version
```

---

## Installation — SaaS

> Vendor manages everything. Customer just uses the application.

### Step 1: Generate License

```python
'environment': 'saas',
'deployment_model': 'saas',
'vendor_verification': {'enabled': True, 'endpoint': 'https://license.autonimbus.com/verify'}
```

### Step 2: Deploy on Vendor Infrastructure

```bash
# Use .env.template.saas
cp config/.env.template.saas /opt/cmp/.env

# Key settings:
#   DEPLOYMENT_MODEL=saas
#   LICENSE_VERIFY_ONLINE=true
#   TELEMETRY_ENABLED=true
#   AUTO_UPDATE_NOTIFICATIONS=true

# Deploy
cd /opt/cmp
docker compose --profile local up -d
```

### Step 3: Provide Customer Access

- Set up DNS (e.g., `customer-name.cmp.autonimbus.com`)
- Configure TLS certificate
- Create initial admin user (see below)
- Send login credentials to customer

#### Bootstrap Initial Admin User

The bootstrap-admin endpoint requires a `BOOTSTRAP_SECRET` environment variable for security.

```bash
# 1. Set the BOOTSTRAP_SECRET in .env.production (generate a secure value)
BOOTSTRAP_SECRET=$(python3 -c "import secrets; print(secrets.token_urlsafe(32))")
echo "BOOTSTRAP_SECRET=$BOOTSTRAP_SECRET" >> /opt/cmp/.env.production

# 2. Restart the backend to pick up the new env var
docker compose restart backend

# 3. Register the admin user first
curl -X POST https://cmp.customer.com/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "email": "admin@customer.com", "password": "SecurePassword123!"}'

# 4. Promote to admin using the bootstrap endpoint
curl -X POST https://cmp.customer.com/api/v1/auth/bootstrap-admin \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "tenant_id": "default", "bootstrap_secret": "'$BOOTSTRAP_SECRET'"}'

# 5. Remove BOOTSTRAP_SECRET from .env after initial setup (recommended)
# This permanently disables the bootstrap endpoint
sed -i '/BOOTSTRAP_SECRET/d' /opt/cmp/.env.production
docker compose restart backend
```

> **Security note:** The bootstrap endpoint is disabled (returns 403) unless `BOOTSTRAP_SECRET` is set. After the initial admin is created, remove the secret from the environment to prevent future use.

---

## Upgrading CMP Version

### Online Upgrade (On-Premises / Hybrid / SaaS)

```bash
# 1. Check current version (requires admin auth)
curl -s -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/health/version | python3 -m json.tool

# 2. Dry run (shows what will happen, no changes)
cd /opt/cmp
./scripts/upgrade.sh --target-version 1.1.0 --dry-run

# 3. Execute upgrade (with auto-rollback on failure)
./scripts/upgrade.sh --target-version 1.1.0 --auto-rollback

# 4. Verify
curl -s -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/health/version | python3 -m json.tool
curl -s http://localhost:8000/health | python3 -m json.tool
```

**What happens during upgrade:**
```
Step 1: BACKUP        → Database + config + compose backed up
Step 2: PULL IMAGES   → New version images downloaded
Step 3: MIGRATIONS    → Database schema updated
Step 4: RESTART       → Services stopped and started with new images
Step 5: HEALTH CHECK  → All services verified healthy
```

If any step fails → automatic rollback restores everything.

### Air-Gapped Upgrade

```bash
# 1. Receive new package from vendor
cp /media/usb/cmp-airgapped-v1.1.0.tar.gz /opt/
cd /opt
tar -xzf cmp-airgapped-v1.1.0.tar.gz
cd cmp-airgapped-v1.1.0

# 2. Verify integrity
./scripts/verify-checksums.sh

# 3. Load new images
./scripts/install.sh

# 4. Run upgrade
cd /opt/cmp
./scripts/upgrade.sh --target-version 1.1.0 --auto-rollback

# 5. Verify
curl -s -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/health/version | python3 -m json.tool
```

### Version Compatibility Rules

| Current | Target | Allowed? | Notes |
|---------|--------|----------|-------|
| 1.0.0 | 1.1.0 | ✓ | Minor upgrade |
| 1.0.0 | 1.5.0 | ✓ | Minor upgrade (any jump within same major) |
| 1.5.0 | 2.0.0 | ✓ | Major +1 |
| 1.5.0 | 3.0.0 | ✗ | Major +2 (must go 1.x → 2.x → 3.x) |
| 2.0.0 | 1.5.0 | ✗ | Downgrade not allowed |

---

## License Operations

### Generate a New License

```bash
cd cloud_management_platform/backend

python3 << 'EOF'
import json, base64
from datetime import datetime, timezone, timedelta
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

# Load private key
with open('/path/to/vendor_private_key.pem', 'rb') as f:
    private_key = serialization.load_pem_private_key(f.read(), password=None)

payload = {
    "version": "1.0",
    "license_id": "lic_CUSTOMER_2026_001",
    "customer": {"name": "CUSTOMER NAME", "id": "cust_CUSTOMER_001"},
    "type": "enterprise",           # trial | standard | enterprise
    "environment": "on-premises",   # saas | hybrid | on-premises | air-gapped
    "issued_at": datetime.now(timezone.utc).isoformat(),
    "expires_at": (datetime.now(timezone.utc) + timedelta(days=365)).isoformat(),
    "limits": {"max_users": 500, "max_managed_resources": 10000},
    "features": [
        "catalog", "workflows", "cost_management", "policy_governance",
        "ai_assistant", "sso", "multi_tenancy", "budgets", "scheduled_jobs"
    ],
    "permitted_versions": {"min": "1.0.0", "max": "9.99.99"},
    "support_tier": "premium",      # basic | standard | premium
    "deployment_model": "on-premises",
    "vendor_verification": {"enabled": False, "endpoint": None}
}

# Sign
payload_bytes = json.dumps(payload, sort_keys=True, separators=(',',':')).encode()
signature = private_key.sign(
    payload_bytes,
    padding.PSS(mgf=padding.MGF1(hashes.SHA256()), salt_length=padding.PSS.MAX_LENGTH),
    hashes.SHA256()
)
payload['signature'] = base64.b64encode(signature).decode()

with open('license.json', 'w') as f:
    json.dump(payload, f, indent=2)
print("✓ License generated: license.json")
print(f"  Customer: {payload['customer']['name']}")
print(f"  Type: {payload['type']}")
print(f"  Expires: {payload['expires_at']}")
print(f"  Features: {len(payload['features'])}")
EOF
```

### Renew a License (Extend Expiry)

```bash
python3 << 'EOF'
import json, base64
from datetime import datetime, timezone, timedelta
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

# Load existing license
with open('license.json') as f:
    payload = json.load(f)

# Remove old signature
del payload['signature']

# Update expiry
payload['issued_at'] = datetime.now(timezone.utc).isoformat()
payload['expires_at'] = (datetime.now(timezone.utc) + timedelta(days=365)).isoformat()

# Re-sign
with open('/path/to/vendor_private_key.pem', 'rb') as f:
    private_key = serialization.load_pem_private_key(f.read(), password=None)

payload_bytes = json.dumps(payload, sort_keys=True, separators=(',',':')).encode()
signature = private_key.sign(
    payload_bytes,
    padding.PSS(mgf=padding.MGF1(hashes.SHA256()), salt_length=padding.PSS.MAX_LENGTH),
    hashes.SHA256()
)
payload['signature'] = base64.b64encode(signature).decode()

with open('license_renewed.json', 'w') as f:
    json.dump(payload, f, indent=2)
print(f"✓ License renewed until: {payload['expires_at']}")
EOF
```

### Upgrade License (Add Features / Increase Limits)

Same as renewal, but also modify:
```python
payload['limits']['max_users'] = 1000          # Increase users
payload['features'].append('new_feature')       # Add feature
payload['type'] = 'enterprise'                  # Upgrade tier
```

### Apply License to Running System (Hot-Reload)

```bash
# Just replace the file — CMP detects changes within 60 seconds
cp license_renewed.json /opt/cmp/license/license.json

# Verify (wait 60 seconds)
sleep 60
curl -s http://localhost:8000/api/v1/license/status | python3 -m json.tool
```

No restart needed! The license validator watches the file and re-validates automatically.

### Revoke a Customer License

Simply don't renew it. Once `expires_at` passes:
- Running app continues until next restart
- On restart, app refuses to start with: "License expired"

For immediate revocation (online deployments):
```bash
# Revoke registry credentials (prevents pulling new images)
python3 -c "
from app.services.registry_manager import RegistryManager
rm = RegistryManager()
rm.revoke_customer_credentials('cust_customer_001')
print('✓ Credentials revoked')
"
```

---

## Rollback Procedures

### Automatic Rollback (During Upgrade)

If you used `--auto-rollback`, it happens automatically on failure. Nothing to do.

### Manual Rollback

```bash
cd /opt/cmp

# Option 1: Use the upgrade script
./scripts/upgrade.sh --rollback

# Option 2: Manual steps
docker compose down

# Restore from backup (backups are in /opt/cmp/backups/)
ls backups/  # Find the latest backup

# Restore database
# (backup script handles this)

# Restore config
cp backups/<backup-id>/config/.env ./.env

# Restore compose
cp backups/<backup-id>/docker-compose.yml ./docker-compose.yml

# Start with old version
docker compose up -d

# Verify
curl -s -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/health/version | python3 -m json.tool
```

---

## Day-to-Day Operations

### Check System Health

```bash
# Quick health check (public)
curl -s http://localhost:8000/health | python3 -m json.tool

# Version info (requires admin auth)
curl -s -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/health/version | python3 -m json.tool

# Upgrade status (requires admin auth)
curl -s -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/health/upgrade-status | python3 -m json.tool

# License status (requires auth token)
curl -s -H "Authorization: Bearer <token>" \
  http://localhost:8000/api/v1/license/status | python3 -m json.tool
```

### View Logs

```bash
# All services
docker compose logs --tail 50

# Specific service
docker compose logs cmp-backend --tail 100

# Follow in real-time
docker compose logs -f cmp-backend

# Filter for errors
docker compose logs cmp-backend | grep -i error
```

### Restart a Service

```bash
# Single service
docker compose restart cmp-backend

# All services
docker compose --profile local down && docker compose --profile local up -d
```

### Check Resource Usage

```bash
docker stats --no-stream
```

### Create a Backup

```bash
./scripts/backup.sh --output /opt/cmp/backups/
```

### Rotate Secrets

```bash
# 1. Generate new secret
NEW_SECRET=$(openssl rand -hex 32)

# 2. Update .env
sed -i "s/^SECRET_KEY=.*/SECRET_KEY=$NEW_SECRET/" /opt/cmp/.env

# 3. CMP detects the change within 60 seconds (no restart needed)
# Or restart immediately:
docker compose restart cmp-backend
```

### Monitor License Expiry

```bash
curl -s -H "Authorization: Bearer <token>" \
  http://localhost:8000/api/v1/license/status | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Days remaining: {d[\"days_remaining\"]}')"
```

---

## Releasing a New Version (Vendor)

### Step 1: Prepare the Release

```bash
cd cloud_management_platform

# Update VERSION file
echo "1.1.0" > backend/VERSION

# Commit
git add backend/VERSION
git commit -m "release: v1.1.0"
git push origin main
```

### Step 2: Tag and Trigger Pipeline

```bash
# Create annotated tag
git tag -a v1.1.0 -m "Release v1.1.0: Feature X, Fix Y"

# Push tag (triggers the release pipeline)
git push origin v1.1.0
```

### Step 3: Monitor Pipeline

Go to GitHub Actions → "CMP Release — Enterprise Package Build" and watch:
1. ✓ Prepare (version metadata)
2. ✓ Verify workflows (existing files untouched)
3. ✓ Compile backend (Nuitka → binary)
4. ✓ Build frontend (Vite → obfuscated bundle)
5. ✓ Build images (multi-stage Docker)
6. ✓ Scan images (Trivy — no critical/high CVEs)
7. ✓ Publish images (push to <account_id>.dkr.ecr.<region>.amazonaws.com)
8. ✓ Assemble package (signed release archive)

### Step 4: Deliver to Customers

**Online customers (on-premises/hybrid):**
```
Email: "CMP v1.1.0 is available. Run: ./scripts/upgrade.sh --target-version 1.1.0"
```

**Air-gapped customers:**
```bash
# Build air-gapped package from pipeline artifacts
./scripts/build/package-airgapped.sh --version 1.1.0

# Add customer license
cp customer_license.json dist/cmp-airgapped-v1.1.0/license/license.json

# Create archive and deliver via secure media
tar -czf cmp-airgapped-v1.1.0.tar.gz -C dist cmp-airgapped-v1.1.0
```

### Step 5: Emergency Hotfix

```bash
# Branch from the release tag
git checkout -b hotfix/1.0.1 v1.0.0

# Fix the issue
# ... make changes ...
git commit -m "fix: critical security patch"

# Merge to main
git checkout main && git merge hotfix/1.0.1

# Tag and push
git tag -a v1.0.1 -m "Hotfix: Critical security patch"
git push origin v1.0.1

# For extreme emergencies, use manual dispatch with skip_scan:
# GitHub Actions → Run workflow → version: 1.0.1, skip_scan: true
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Check health | `curl -s localhost:8000/health \| python3 -m json.tool` |
| Check version | `curl -s -H "Auth: Bearer <token>" localhost:8000/health/version \| python3 -m json.tool` |
| Check license | `curl -s -H "Auth..." localhost:8000/api/v1/license/status` |
| Start services | `docker compose --profile local up -d` |
| Stop services | `docker compose down` |
| View logs | `docker compose logs --tail 100 <service>` |
| Upgrade (online) | `./scripts/upgrade.sh --target-version X.Y.Z --auto-rollback` |
| Upgrade (dry run) | `./scripts/upgrade.sh --target-version X.Y.Z --dry-run` |
| Rollback | `./scripts/upgrade.sh --rollback` |
| Backup | `./scripts/backup.sh --output /opt/cmp/backups/` |
| Verify package | `./scripts/verify-checksums.sh` |
| Load images | `./scripts/install.sh` |
| Generate license | See [License Operations](#license-operations) |
| Renew license | Replace file → auto-detected in 60s |
| Release new version | `git tag -a vX.Y.Z && git push origin vX.Y.Z` |
| Build air-gapped | `./scripts/build/package-airgapped.sh --version X.Y.Z` |

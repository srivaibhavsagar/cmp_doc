# CMP — Step-by-Step Customer Shipping Guide

> Quick reference for shipping CMP to a customer from sale to live deployment.
> For full detail on each step, refer to the linked source documents.

---

## Documents to Follow (in order)

| Step | Document | Purpose |
|------|----------|---------|
| 1 | `cmp_doc/vendor/RELEASE-OPERATIONS.md` | Build and publish Docker images |
| 2 | `cmp_doc/vendor/LICENSE-MANAGEMENT.md` | Generate a signed license per customer |
| 3 | `cmp_doc/vendor/CUSTOMER-ONBOARDING.md` | Full onboarding playbook per sale |
| 4 | `cmp_doc/customer/INSTALLATION.md` | Hand to customer — install guide |
| 5 | `cmp_doc/customer/LICENSE.md` | Hand to customer — license placement guide |
| 6 | `cmp_doc/customer/CONFIGURATION.md` | Hand to customer — `.env.production` setup |
| 7 | `cmp_doc/customer/AIR-GAPPED-OPERATIONS.md` | Air-gapped customers only |

---

## Your Side — Per Release

### Step 1: Build and Publish Docker Images

Follow `RELEASE-OPERATIONS.md`.

```bash
# Tag the release on main
git checkout main && git pull origin main
git tag -a v<VERSION> -m "Release v<VERSION>"
git push origin v<VERSION>
```

The GitHub Actions pipeline (`release.yml`) automatically:
- Compiles backend with Nuitka → native binary (no Python source exposed)
- Builds frontend with Vite (no source maps)
- Builds all 4 Docker images (`cmp-backend`, `cmp-frontend`, `cmp-worker`, `cmp-ai`)
- Scans for CVEs with Trivy
- Pushes to `<account_id>.dkr.ecr.<region>.amazonaws.com`
- Assembles and signs a release package with `MANIFEST.sha256`

**Never ship source code or the dev `docker-compose.yml` to customers.**

---

## Your Side — Per Customer Sale

### Step 2: Generate a Signed License

Follow `LICENSE-MANAGEMENT.md`. Use `vendor/sign_onprem_license.py`.

```bash
python3 vendor/sign_onprem_license.py \
  --private-key vendor/vendor_private_key.pem \
  --customer "Acme Corp" \
  --customer-id "cust_acme_001" \
  --license-type enterprise \
  --deployment-model on-premises \
  --max-users 50 \
  --max-resources 500 \
  --aws-accounts 5 \
  --azure-accounts 3 \
  --gcp-accounts 2 \
  --features tier \
  --expires-days 365 \
  --support-tier premium \
  --min-version "0.0.1" \
  --max-version "99.99.99" \
  --output ./license_cust_acme_001.json
```

**License tiers at a glance:**

| Tier | Users | Resources | Cloud Accounts | Features |
|------|-------|-----------|----------------|----------|
| `starter` | ≤ 10 | ≤ 100 | AWS: 1 | catalog, workflows, cloud_accounts, notifications, infrastructure |
| `professional` | 50–200 | 1k–5k | AWS: 3, Azure: 2, GCP: 2 | All except SSO, AI, multi-tenancy |
| `enterprise` | 100+ | 5k+ | AWS: 10, Azure: 10, GCP: 10 | All features |

### Step 3: Provision AWS ECR Access

Provide the customer with IAM credentials (or cross-account ECR access) scoped to pull from your ECR repository. Deliver via secure channel (separate from the license file).

```bash
# Customer configures in their .env.production:
ECR_REGISTRY=<account_id>.dkr.ecr.<region>.amazonaws.com/autonimbus/cmp
AWS_ACCESS_KEY_ID=<provided-access-key>
AWS_SECRET_ACCESS_KEY=<provided-secret-key>
AWS_REGION=<region>

# Customer authenticates before pulling images:
aws ecr get-login-password --region <region> \
  | docker login --username AWS --password-stdin \
    <account_id>.dkr.ecr.<region>.amazonaws.com
```

### Step 4: Prepare the Customer Handoff Package

| Item | Online Deployment | Air-Gapped |
|------|-------------------|------------|
| `license.json` (signed) | ✓ | ✓ (in package) |
| AWS ECR pull credentials | ✓ | ✗ |
| `docker-compose.production.yml` | ✓ | ✓ (in package) |
| `.env.template.<model>` from `config/` | ✓ | ✓ (in package) |
| `INSTALLATION.md` | ✓ (link) | ✓ (in package) |
| Air-gapped image tar archives | ✗ | ✓ |

---

## Customer's Side — What They Do

Follow `customer/INSTALLATION.md`, `customer/LICENSE.md`, `customer/CONFIGURATION.md`.

### Step 5: Customer Sets Up Configuration

```bash
# Copy the env template
cp .env.template.on-premises .env.production

# Fill in required values:
# SECRET_KEY, ENCRYPTION_KEY, DYNAMODB_ENDPOINT, REDIS_URL, etc.
# DEPLOYMENT_MODEL is already set in the template — do not remove it
```

### Step 6: Customer Places the License File

```bash
# Default path expected by the production compose
/app/config/license.json

# Or configure a custom path in .env.production:
LICENSE_FILE_PATH=/path/to/license.json
```

### Step 7: Customer Starts the Application

```bash
# With bundled Redis + DynamoDB (single server)
CMP_VERSION=<version> docker compose --profile local \
  -f docker-compose.production.yml up -d

# With managed AWS services (production-grade)
CMP_VERSION=<version> docker compose \
  -f docker-compose.production.yml up -d
```

### Step 8: Verify Successful Deployment

Ask the customer to confirm these all return expected responses:

```bash
# 1. Health check
curl -s https://cmp.customer.com/health | jq .
# Expected: {"status": "healthy", ...}

# 2. Version info (requires admin auth token)
curl -s -H "Authorization: Bearer <admin_token>" \
  https://cmp.customer.com/health/version | jq .
# Expected: {"version": "<version>", ...}

# 3. Login works in the browser
# 4. Admin panel → System Status shows a valid license
```

---

## What Protects the License from Being Bypassed

| Attack | Protection |
|--------|-----------|
| Remove `DEPLOYMENT_MODEL` from compose | Compiled binary defaults to `on-premises` — license still enforced |
| Edit `license_validator.py` | Impossible — customers receive a Nuitka-compiled binary, no Python source |
| Patch the binary | SHA-256 entrypoint check kills the container at startup and every 60s |
| Forge a `license.json` | RSA-4096 signature check rejects it — only your private key can sign |
| Replace the binary entirely | SHA-256 checksum mismatch — container halts immediately |

---

## Ongoing Vendor Responsibilities

### License Renewals

Track expiry for all customers and initiate renewals proactively:

| Days to Expiry | Action |
|----------------|--------|
| 90 days | Send renewal reminder |
| 60 days | Send renewal quote |
| 30 days | Final warning — escalate to account manager |
| On renewal | Re-sign license, deliver — hot-reload applies within 60s (no restart needed) |

### License Revocation

```bash
# 1. Revoke ECR access (remove customer's IAM policy or cross-account role)
# Customer can no longer pull new images from your ECR repository

# 2. Do not issue a renewed license.
# The running instance continues until next restart,
# at which point the expired license blocks startup.
```

### Security Patches

| Severity | Notify Customer Within |
|----------|----------------------|
| Critical (security CVE) | 1 hour |
| High (data loss risk) | 4 hours |
| Medium (functionality) | 24 hours |

---

## Key Files — Quick Reference

| File | Purpose |
|------|---------|
| `vendor/sign_onprem_license.py` | Sign a license for on-premises / hybrid / air-gapped customers |
| `vendor/generate_saas_license.py` | Auto-generate your own SaaS license (used in CI/CD) |
| `vendor/vendor_private_key.pem` | **SECRET** — never commit, never ship to customers |
| `vendor/vendor_public_key.pem` | Embedded in the compiled binary — customers cannot change it |
| `config/.env.template.on-premises` | Config template to give on-premises customers |
| `config/.env.template.saas` | Config template for SaaS deployments |
| `config/.env.template.air-gapped` | Config template for air-gapped customers |
| `docker-compose.production.yml` | The only compose file customers should receive |
| `license/license.json` | Your own SaaS license (auto-generated in CI/CD) |

# CMP Installation Guide

This guide walks you through installing the Cloud Management Platform (CMP) in your environment. It covers system requirements, installation methods (online and air-gapped), initial configuration, license activation, and post-installation verification.

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Pre-Installation Checklist](#pre-installation-checklist)
3. [Online Installation (Registry Pull)](#online-installation-registry-pull)
4. [Air-Gapped Installation](#air-gapped-installation)
5. [Initial Configuration Walkthrough](#initial-configuration-walkthrough)
6. [First-Time License Activation](#first-time-license-activation)
7. [Post-Installation Verification](#post-installation-verification)
8. [Deployment-Model-Specific Sections](#deployment-model-specific-sections)
9. [Common Installation Errors](#common-installation-errors)

---

## System Requirements

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores | 8+ cores |
| RAM | 8 GB | 16+ GB |
| Disk (application) | 40 GB SSD | 100+ GB SSD |
| Disk (data/backups) | 50 GB | 200+ GB |
| Network | 100 Mbps | 1 Gbps |

### Software Requirements

| Software | Version | Notes |
|----------|---------|-------|
| Docker Engine | 24.0+ | Required for all deployment models |
| Docker Compose | v2.20+ | Plugin or standalone |
| Operating System | Linux (x86_64) | Ubuntu 22.04+, RHEL 8+, Amazon Linux 2023 |
| Kernel | 5.10+ | Required for security features |

### Network Requirements

| Port | Service | Direction | Notes |
|------|---------|-----------|-------|
| 443 | HTTPS (Frontend) | Inbound | User-facing web interface |
| 8001 | Backend API | Internal | Backend service (not exposed externally) |
| 6379 | Redis | Internal | Cache and message broker |
| 8000 | DynamoDB | Internal | Database (or AWS endpoint) |

> **Note:** For online installations, outbound HTTPS (port 443) access to `<account_id>.dkr.ecr.<region>.amazonaws.com` is required during image pull.

---

## Pre-Installation Checklist

Before starting installation, verify the following:

- [ ] Docker Engine 24.0+ installed and running (`docker --version`)
- [ ] Docker Compose v2.20+ available (`docker compose version`)
- [ ] Sufficient disk space on target volumes
- [ ] Required ports are available and not blocked by firewall
- [ ] DNS configured for your CMP hostname (e.g., `cmp.yourcompany.com`)
- [ ] TLS certificate ready for your domain (or plan to use Let's Encrypt)
- [ ] License file received from Autonimbus (`license.json`)
- [ ] Registry credentials received (for online installation)
- [ ] Dedicated service account for running CMP containers
- [ ] Backup storage location identified
- [ ] Maintenance window scheduled for initial deployment

---

## Online Installation (Registry Pull)

Use this method when your environment has outbound internet access to AWS ECR.

### Step 1: Authenticate with the Container Registry

```bash
# Authenticate Docker to your AWS ECR registry using the AWS CLI
aws ecr get-login-password --region <region> \
  | docker login --username AWS --password-stdin \
    <account_id>.dkr.ecr.<region>.amazonaws.com
```

> Your `<account_id>`, `<region>`, and the AWS credentials (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) are provided by Autonimbus in your onboarding package.

### Step 2: Create the Installation Directory

```bash
mkdir -p /opt/cmp
cd /opt/cmp
```

### Step 3: Place Configuration Files

Copy the provided `docker-compose.production.yml` and configuration template to your installation directory:

```bash
cp /path/to/delivery/docker-compose.production.yml ./docker-compose.yml
cp /path/to/delivery/config/.env.template ./.env
```

### Step 4: Configure Environment

Edit the `.env` file with your environment-specific values (see [Initial Configuration Walkthrough](#initial-configuration-walkthrough) below).

### Step 5: Pull Container Images

```bash
docker compose pull
```

This pulls all CMP service images from the private registry.

### Step 6: Place License File

```bash
mkdir -p /opt/cmp/license
cp /path/to/license.json /opt/cmp/license/license.json
```

### Step 7: Start Services

```bash
docker compose up -d
```

### Step 8: Verify Installation

```bash
# Check all containers are running
docker compose ps

# Check health endpoint
curl -s http://localhost:8001/health | jq .
```

---

## Air-Gapped Installation

Use this method for environments without internet access (government, banking, healthcare).

### Step 1: Receive and Transfer the Package

Your Autonimbus representative will deliver the air-gapped package via secure media (encrypted USB, secure file transfer, etc.). Transfer the package to your target server.

```bash
# Example: copy from USB
cp /media/usb/cmp-release-v*.tar.gz /opt/
cd /opt
tar -xzf cmp-release-v*.tar.gz
cd cmp-release-v*
```

### Step 2: Verify Package Integrity

**This step is critical for security.** Verify that the package has not been tampered with:

```bash
./scripts/verify-checksums.sh
```

Expected output:
```
Verifying package integrity...
✓ cmp-frontend-8.10.0-a1b2c3d4.tar.gz  OK
✓ cmp-backend-8.10.0-a1b2c3d4.tar.gz   OK
✓ cmp-worker-8.10.0-a1b2c3d4.tar.gz    OK
✓ cmp-ai-8.10.0-a1b2c3d4.tar.gz        OK
✓ redis-7-alpine.tar.gz                  OK
✓ dynamodb-local.tar.gz                  OK
✓ docker-compose.production.yml          OK
All files verified successfully.
```

If any file fails verification, **do not proceed**. Contact Autonimbus support immediately.

### Step 3: Load Container Images

```bash
./scripts/install.sh
```

This script loads all container images into the local Docker daemon in the correct dependency order. If any image fails to load, the script halts and reports the error.

### Step 4: Set Up Installation Directory

```bash
mkdir -p /opt/cmp
cp docker-compose.production.yml /opt/cmp/docker-compose.yml
cp config/.env.template.air-gapped /opt/cmp/.env
```

### Step 5: Configure Environment

Edit `/opt/cmp/.env` with your environment-specific values. See [Initial Configuration Walkthrough](#initial-configuration-walkthrough).

### Step 6: Place License File

```bash
mkdir -p /opt/cmp/license
cp license/license.json /opt/cmp/license/license.json
```

### Step 7: Start Services

```bash
cd /opt/cmp
docker compose up -d
```

### Step 8: Verify Installation

```bash
curl -s http://localhost:8001/health | jq .
curl -s http://localhost:8001/version | jq .
```

---

## Initial Configuration Walkthrough

The `.env` file controls all CMP configuration. Below is a walkthrough of the key settings.

### Required Settings (Application Will Not Start Without These)

```bash
# Security keys — generate unique values for your installation
SECRET_KEY=your-secret-key-minimum-32-characters-long
ENCRYPTION_KEY=your-fernet-encryption-key-base64-encoded

# Database
DYNAMODB_ENDPOINT=http://database:8000
AWS_REGION=us-east-1

# Deployment model — must match your license
DEPLOYMENT_MODEL=on-premises   # Options: saas, hybrid, on-premises, air-gapped
```

### Generate Security Keys

```bash
# Generate SECRET_KEY (random 64-character string)
openssl rand -hex 32

# Generate ENCRYPTION_KEY (Fernet-compatible base64 key)
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### Optional but Recommended Settings

```bash
# Frontend URL (for CORS and redirects)
FRONTEND_URL=https://cmp.yourcompany.com

# Redis (for caching and background tasks)
REDIS_URL=redis://redis:6379/0

# SMTP (for email notifications)
SMTP_HOST=smtp.yourcompany.com
SMTP_PORT=587
SMTP_USE_TLS=true
SMTP_USERNAME=cmp-notifications@yourcompany.com
SMTP_PASSWORD=your-smtp-password

# SSO/SAML (if using single sign-on)
SSO_ENABLED=true
SSO_ENTITY_ID=https://cmp.yourcompany.com
SSO_METADATA_URL=https://idp.yourcompany.com/metadata
```

### Data Seeding

```bash
# Control whether sample data is seeded on first startup
# When true (default), demo workflows, tasks, flows, and catalog items are created
# when the database is empty. Users and groups are always seeded regardless.
# Set to false for production deployments where you want a clean platform.
SEED_SAMPLE_DATA=true   # Options: true, false
```

> **Note:** This setting only takes effect on first startup when the database tables are empty. Changing it after initial seeding has no effect. Users and groups (admin, developer, user, readonly) are always created regardless of this setting.

### Encrypting Sensitive Values

For enhanced security, sensitive values can be encrypted at rest. See the [Configuration Reference](CONFIGURATION.md) for details on Fernet encryption.

---

## First-Time License Activation

### Step 1: Place the License File

Your license file (`license.json`) was provided by Autonimbus. Place it in the configured license directory:

```bash
cp license.json /opt/cmp/license/license.json
```

### Step 2: Verify License Path in Configuration

Ensure your `.env` file points to the license:

```bash
LICENSE_FILE_PATH=/app/license/license.json
```

The Docker Compose file mounts `/opt/cmp/license` to `/app/license` inside the container.

### Step 3: Start or Restart the Application

```bash
docker compose up -d
```

### Step 4: Verify License Status

```bash
curl -s http://localhost:8001/api/v1/license/status | jq .
```

Expected response:
```json
{
  "valid": true,
  "license_type": "enterprise",
  "customer_name": "Your Company",
  "expires_at": "2026-01-15T00:00:00Z",
  "days_remaining": 365,
  "features_enabled": ["catalog", "workflows", "cost_management", ...],
  "current_users": 0,
  "max_users": 500,
  "current_resources": 0,
  "max_resources": 10000,
  "deployment_model": "on-premises",
  "support_tier": "premium"
}
```

### License Activation Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "License file not found" | File missing or wrong path | Verify `LICENSE_FILE_PATH` and file exists |
| "Signature validation failed" | Tampered or corrupted file | Request a new license from Autonimbus |
| "License expired" | Past expiry date | Contact Autonimbus for renewal |
| "Deployment model mismatch" | License env ≠ DEPLOYMENT_MODEL | Ensure `.env` DEPLOYMENT_MODEL matches license |

---

## Post-Installation Verification

After installation, run through these checks to confirm everything is working:

### 1. Health Check

```bash
curl -s http://localhost:8001/health | jq .
```

All services should report `"healthy"`:
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

### 2. Version Check

```bash
curl -s http://localhost:8001/version | jq .
```

### 3. Frontend Access

Open your browser and navigate to your configured frontend URL (e.g., `https://cmp.yourcompany.com`). You should see the CMP login page.

### 4. First Login

Log in with the default admin credentials provided in your delivery package. **Change the default password immediately after first login.**

### 5. License Verification

Navigate to **Settings → License** in the CMP UI to confirm your license is active and features are available.

---

## Deployment-Model-Specific Sections

### On-Premises Deployment

For on-premises deployments, you manage the full infrastructure:

- Use `.env.template.on-premises` as your starting configuration
- Set `DEPLOYMENT_MODEL=on-premises`
- License validation is performed offline (no outbound calls)
- Upgrades are manual — you control when to apply new versions
- You are responsible for backups, monitoring, and maintenance

### Air-Gapped Deployment

For air-gapped environments with no internet access:

- Use `.env.template.air-gapped` as your starting configuration
- Set `DEPLOYMENT_MODEL=air-gapped`
- All outbound network calls are disabled (no license verification, no update checks, no telemetry)
- All software updates delivered via physical media or secure file transfer
- See [Air-Gapped Operations Guide](AIR-GAPPED-OPERATIONS.md) for ongoing operations

### Hybrid Deployment

For hybrid deployments with shared vendor/customer responsibility:

- Use `.env.template.hybrid` as your starting configuration
- Set `DEPLOYMENT_MODEL=hybrid`
- Vendor may have monitoring access (configure as agreed in your contract)
- Upgrades may be vendor-assisted

---

## Common Installation Errors

### "Permission denied" when starting containers

**Cause:** Docker daemon not accessible by current user.

**Solution:**
```bash
# Add your user to the docker group
sudo usermod -aG docker $USER
# Log out and back in, then retry
```

### "Port already in use"

**Cause:** Another service is using a required port.

**Solution:**
```bash
# Find what's using the port
sudo lsof -i :443
# Either stop the conflicting service or change CMP's port in docker-compose.yml
```

### "No space left on device"

**Cause:** Insufficient disk space for container images or data.

**Solution:**
```bash
# Check disk usage
df -h
# Clean unused Docker resources
docker system prune -a
```

### "License file not found" at startup

**Cause:** License file not mounted correctly into the container.

**Solution:**
- Verify the file exists at the host path (e.g., `/opt/cmp/license/license.json`)
- Verify the volume mount in `docker-compose.yml` maps to the correct container path
- Check file permissions (must be readable by the container user)

### "Required configuration missing: SECRET_KEY, ENCRYPTION_KEY"

**Cause:** Required environment variables not set.

**Solution:**
- Ensure your `.env` file contains all required keys
- Verify the `.env` file is in the same directory as `docker-compose.yml`
- Generate keys using the commands in [Initial Configuration Walkthrough](#initial-configuration-walkthrough)

### "Deployment model mismatch"

**Cause:** The `DEPLOYMENT_MODEL` environment variable doesn't match the `deployment_model` field in your license file.

**Solution:**
- Check your license file: `cat license/license.json | jq .deployment_model`
- Set `DEPLOYMENT_MODEL` in `.env` to match exactly

### Container exits immediately after starting

**Cause:** Configuration or license validation failure.

**Solution:**
```bash
# Check container logs for the specific error
docker compose logs cmp-backend --tail 50
```

Common causes: missing license, invalid encryption key format, unreachable database endpoint.

### "Connection refused" on health endpoint

**Cause:** Backend service hasn't finished starting yet.

**Solution:**
- Wait up to 60 seconds for the backend to initialize
- Check logs: `docker compose logs cmp-backend -f`
- Verify the database is accessible from the backend container

---

## Next Steps

After successful installation:

1. Review the [Configuration Reference](CONFIGURATION.md) for all available settings
2. Read the [License Guide](LICENSE.md) to understand your entitlements
3. Set up monitoring using the [Troubleshooting Guide](TROUBLESHOOTING.md)
4. Plan your upgrade strategy with the [Upgrade Guide](UPGRADE.md)

---

## Getting Help

If you encounter issues not covered here:

1. Check the [Troubleshooting Guide](TROUBLESHOOTING.md)
2. Collect diagnostic information: `docker compose logs > cmp-diagnostics.log`
3. Contact Autonimbus support with your customer ID and diagnostic logs

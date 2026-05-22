# CMP Air-Gapped Operations Guide

This guide covers all operational procedures for running CMP in an air-gapped environment — a fully isolated network with no outbound internet connectivity.

---

## Table of Contents

1. [What Air-Gapped Means for CMP](#what-air-gapped-means-for-cmp)
2. [Receiving and Verifying Packages](#receiving-and-verifying-packages)
3. [Installation in Air-Gapped Environment](#installation-in-air-gapped-environment)
4. [Upgrading in Air-Gapped Environment](#upgrading-in-air-gapped-environment)
5. [License Management Without Internet](#license-management-without-internet)
6. [Backup and Restore](#backup-and-restore)
7. [Monitoring Without External Services](#monitoring-without-external-services)
8. [Requesting Support](#requesting-support)

---

## What Air-Gapped Means for CMP

When CMP is deployed in air-gapped mode (`DEPLOYMENT_MODEL=air-gapped`), the following behaviors are enforced:

### What is Disabled

| Feature | Behavior in Air-Gapped Mode |
|---------|----------------------------|
| License verification | No outbound calls to vendor server |
| Update checks | No automatic update notifications |
| Telemetry | No usage data sent anywhere |
| Registry pulls | Cannot pull images from registry.autonimbus.com |
| External webhooks | Slack, Teams, PagerDuty notifications disabled unless on internal network |
| AI service (cloud) | Disabled unless using on-premises AI model |

### What Continues to Work

| Feature | How It Works |
|---------|-------------|
| License validation | Offline RSA signature verification (no network needed) |
| All CMP features | Fully functional within licensed scope |
| Health monitoring | Internal health checks between services |
| Background tasks | Celery workers process tasks locally |
| Database operations | Local DynamoDB instance |
| User authentication | JWT-based, fully local |

### Key Principle

**Everything operates locally.** No CMP process will attempt outbound network connections. If any component accidentally tries to reach an external endpoint, it will fail silently without affecting operations.

---

## Receiving and Verifying Packages

### How Packages Are Delivered

Autonimbus delivers air-gapped packages through secure channels agreed upon during onboarding:

- Encrypted USB drives
- Secure file transfer (SFTP to a DMZ)
- Physical media with chain-of-custody documentation
- Secure courier service

### Package Contents

Each air-gapped release package contains:

```
cmp-release-v8.10.0/
├── images/                          # Container images as tar.gz
│   ├── cmp-frontend-8.10.0-a1b2c3d4.tar.gz
│   ├── cmp-backend-8.10.0-a1b2c3d4.tar.gz
│   ├── cmp-worker-8.10.0-a1b2c3d4.tar.gz
│   ├── cmp-ai-8.10.0-a1b2c3d4.tar.gz
│   ├── redis-7-alpine.tar.gz
│   └── dynamodb-local.tar.gz
├── config/                          # Configuration templates
│   ├── .env.template.air-gapped
│   └── encryption-key-gen.sh
├── migrations/                      # Database migration scripts
├── scripts/                         # Operational scripts
│   ├── install.sh
│   ├── upgrade.sh
│   ├── verify-checksums.sh
│   ├── health-check.sh
│   └── backup.sh
├── docs/                            # Documentation
│   ├── INSTALL.md
│   ├── UPGRADE.md
│   └── RELEASE-NOTES.md
├── license/
│   └── license.json.template
├── docker-compose.production.yml
├── MANIFEST.sha256                  # File checksums
└── MANIFEST.sha256.sig              # Signed manifest
```

### Verifying Package Integrity

**Always verify before installing or upgrading.** This confirms the package hasn't been tampered with during delivery.

#### Step 1: Verify the Manifest Signature

The manifest is signed with Autonimbus's code-signing key. Verify the signature using the public key provided during onboarding:

```bash
# Verify manifest signature (requires the Autonimbus public key)
openssl dgst -sha256 -verify autonimbus-public.pem \
  -signature MANIFEST.sha256.sig MANIFEST.sha256
```

Expected output: `Verified OK`

If verification fails: **Stop. Do not proceed.** The package may have been tampered with. Contact Autonimbus through your secure channel.

#### Step 2: Verify File Checksums

```bash
./scripts/verify-checksums.sh
```

Expected output:
```
Verifying package integrity...
✓ images/cmp-frontend-8.10.0-a1b2c3d4.tar.gz    OK
✓ images/cmp-backend-8.10.0-a1b2c3d4.tar.gz     OK
✓ images/cmp-worker-8.10.0-a1b2c3d4.tar.gz      OK
✓ images/cmp-ai-8.10.0-a1b2c3d4.tar.gz          OK
✓ images/redis-7-alpine.tar.gz                    OK
✓ images/dynamodb-local.tar.gz                    OK
✓ docker-compose.production.yml                   OK
✓ scripts/install.sh                              OK
✓ scripts/upgrade.sh                              OK
All 15 files verified successfully.
```

If any file fails:
```
✗ images/cmp-backend-8.10.0-a1b2c3d4.tar.gz    FAILED
  Expected: a1b2c3d4e5f6...
  Got:      9z8y7x6w5v4u...

VERIFICATION FAILED: 1 file(s) did not match expected checksums.
Do NOT proceed with installation.
```

**If verification fails:** Do not install. Request a new package from Autonimbus.

---

## Installation in Air-Gapped Environment

### Prerequisites

- Docker Engine 24.0+ installed (installed from local packages, not internet)
- Docker Compose v2.20+ available
- Package verified (see above)
- License file received from Autonimbus

### Step-by-Step Installation

#### 1. Extract the Package

```bash
cd /opt
tar -xzf cmp-release-v8.10.0.tar.gz
cd cmp-release-v8.10.0
```

#### 2. Verify Integrity

```bash
./scripts/verify-checksums.sh
```

#### 3. Load Container Images

```bash
./scripts/install.sh
```

This loads all images into the local Docker daemon in dependency order:
1. Database (DynamoDB local)
2. Redis
3. Backend
4. Worker
5. AI service
6. Frontend

If any image fails to load, the script stops and reports the error. Previously loaded images remain intact.

#### 4. Create Installation Directory

```bash
mkdir -p /opt/cmp/{license,config,logs,backups,data}
cp docker-compose.production.yml /opt/cmp/docker-compose.yml
cp config/.env.template.air-gapped /opt/cmp/.env
```

#### 5. Generate Security Keys

```bash
# Generate SECRET_KEY
openssl rand -hex 32
# Copy output to .env as SECRET_KEY value

# Generate ENCRYPTION_KEY
./config/encryption-key-gen.sh
# Copy output to .env as ENCRYPTION_KEY value
```

#### 6. Configure Environment

Edit `/opt/cmp/.env`:

```bash
# Required settings
SECRET_KEY=<generated-value>
ENCRYPTION_KEY=<generated-value>
DYNAMODB_ENDPOINT=http://database:8000
AWS_REGION=us-east-1
DEPLOYMENT_MODEL=air-gapped

# Your organization's settings
FRONTEND_URL=https://cmp.internal.company.com
```

#### 7. Place License File

```bash
cp /path/to/your/license.json /opt/cmp/license/license.json
```

#### 8. Start Services

```bash
cd /opt/cmp
docker compose up -d
```

#### 9. Verify Installation

```bash
# Wait 30-60 seconds for startup
sleep 60

# Check health
curl -s http://localhost:8001/health | jq .

# Check version
curl -s http://localhost:8001/version | jq .

# Check license
curl -s http://localhost:8001/api/v1/license/status | jq .valid
```

---

## Upgrading in Air-Gapped Environment

### Overview

Air-gapped upgrades follow the same logical process as online upgrades, but images come from the delivered package instead of a registry.

### Step-by-Step Upgrade

#### 1. Receive New Version Package

Obtain the upgrade package from Autonimbus via your secure delivery channel.

#### 2. Transfer to Server

```bash
cp /media/secure/cmp-release-v8.11.0.tar.gz /opt/
cd /opt
tar -xzf cmp-release-v8.11.0.tar.gz
cd cmp-release-v8.11.0
```

#### 3. Verify Package Integrity

```bash
# Verify signature
openssl dgst -sha256 -verify /opt/cmp/keys/autonimbus-public.pem \
  -signature MANIFEST.sha256.sig MANIFEST.sha256

# Verify checksums
./scripts/verify-checksums.sh
```

**Do not proceed if verification fails.**

#### 4. Review Release Notes

```bash
cat docs/RELEASE-NOTES.md
```

Check for:
- Breaking changes
- New required configuration
- Migration notes
- Known issues

#### 5. Run Pre-Upgrade Check (Dry Run)

```bash
cd /opt/cmp
/opt/cmp-release-v8.11.0/scripts/upgrade.sh --target-version 8.11.0 --dry-run
```

#### 6. Load New Images

```bash
cd /opt/cmp-release-v8.11.0
./scripts/install.sh
```

#### 7. Execute Upgrade

```bash
cd /opt/cmp
/opt/cmp-release-v8.11.0/scripts/upgrade.sh --target-version 8.11.0 --auto-rollback
```

#### 8. Verify Upgrade

```bash
curl -s http://localhost:8001/version | jq .version
# Should show: "8.11.0"

curl -s http://localhost:8001/health | jq .status
# Should show: "healthy"
```

#### 9. Clean Up Old Package (Optional)

After confirming the upgrade is successful and stable (wait at least 24 hours):

```bash
rm -rf /opt/cmp-release-v8.10.0*
```

Keep the current version package as a rollback reference.

---

## License Management Without Internet

### How Offline Licensing Works

In air-gapped mode:
- License validation uses RSA signature verification with an embedded public key
- No network calls are made to any vendor server
- The license is validated at startup and monitored for file changes

### Activating a License

1. Receive the license file from Autonimbus (via secure delivery)
2. Place it at `/opt/cmp/license/license.json`
3. Start or restart CMP

### Renewing a License

When your license approaches expiry:

1. **Contact Autonimbus** through your established secure channel (30+ days before expiry)
2. **Receive new license file** via secure delivery
3. **Replace the license file:**
   ```bash
   cp /path/to/new-license.json /opt/cmp/license/license.json
   ```
4. **Wait for hot-reload** (up to 60 seconds) — no restart needed
5. **Verify:**
   ```bash
   curl -s http://localhost:8001/api/v1/license/status | jq '{expires_at, days_remaining}'
   ```

### Monitoring License Expiry

Set up a local cron job to alert you before expiry:

```bash
# Add to crontab: check license daily
0 9 * * * curl -s http://localhost:8001/api/v1/license/status | \
  python3 -c "import sys,json; d=json.load(sys.stdin); \
  print(f'WARNING: License expires in {d[\"days_remaining\"]} days') \
  if d['days_remaining'] < 30 else None"
```

### License Limits in Air-Gapped Mode

User and resource limits are enforced locally:
- Current counts are tracked in the local database
- No external verification of counts
- Limits are enforced in real-time at the API level

---

## Backup and Restore

### Why Backups Are Critical in Air-Gapped Environments

Without cloud storage or external backup services, you must manage backups entirely within your local infrastructure.

### Creating Backups

```bash
cd /opt/cmp
./scripts/backup.sh --output /opt/cmp/backups/
```

This creates a timestamped backup including:
- Database contents (full export)
- Configuration files (`.env`, encrypted configs)
- Docker Compose definitions
- License file

### Backup Schedule

Recommended backup schedule:

| Frequency | What | Retention |
|-----------|------|-----------|
| Daily | Database + config | 7 days |
| Weekly | Full backup (all components) | 4 weeks |
| Pre-upgrade | Full backup | Until next successful upgrade |

### Automating Backups

```bash
# Add to crontab: daily backup at 2 AM
0 2 * * * cd /opt/cmp && ./scripts/backup.sh --output /opt/cmp/backups/ 2>&1 | logger -t cmp-backup
```

### Restoring from Backup

```bash
cd /opt/cmp

# Stop services
docker compose down

# Restore from backup
./scripts/backup.sh --restore --source /opt/cmp/backups/backup-20250115-020000.tar.gz

# Start services
docker compose up -d

# Verify
curl -s http://localhost:8001/health | jq .
```

### Off-Site Backup Storage

For disaster recovery, store backup copies on separate physical media:

1. Create the backup
2. Copy to external storage (encrypted USB, tape, separate server)
3. Store in a physically separate location
4. Test restore from off-site media periodically

### Backup Encryption

For sensitive environments, encrypt backups before storing:

```bash
# Encrypt backup
gpg --symmetric --cipher-algo AES256 backup-20250115-020000.tar.gz

# Decrypt when needed
gpg --decrypt backup-20250115-020000.tar.gz.gpg > backup-20250115-020000.tar.gz
```

---

## Monitoring Without External Services

### Internal Health Monitoring

Since external monitoring services (Datadog, New Relic, etc.) are unavailable, use local monitoring:

#### Health Check Script

```bash
#!/bin/bash
# /opt/cmp/scripts/monitor.sh — run via cron every 5 minutes

HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/health)
STATUS=$(curl -s http://localhost:8001/health | python3 -c "import sys,json; print(json.load(sys.stdin).get('status','unknown'))")

if [ "$HEALTH" != "200" ] || [ "$STATUS" != "healthy" ]; then
    echo "$(date -Iseconds) ALERT: CMP health check failed. HTTP=$HEALTH Status=$STATUS" >> /opt/cmp/logs/alerts.log
    # Add your local alerting mechanism here (email via internal SMTP, syslog, etc.)
fi
```

#### Cron Setup

```bash
# Check health every 5 minutes
*/5 * * * * /opt/cmp/scripts/monitor.sh

# Check disk space daily
0 8 * * * df -h / | awk 'NR==2{if($5+0 > 80) print "DISK WARNING: "$5" used"}' >> /opt/cmp/logs/alerts.log

# Check container restarts daily
0 8 * * * docker ps --format '{{.Names}} {{.Status}}' | grep -i restart >> /opt/cmp/logs/alerts.log
```

### Resource Monitoring

```bash
# Create a resource snapshot (run periodically)
echo "=== $(date -Iseconds) ===" >> /opt/cmp/logs/resources.log
docker stats --no-stream >> /opt/cmp/logs/resources.log
df -h >> /opt/cmp/logs/resources.log
free -h >> /opt/cmp/logs/resources.log
```

### Log Aggregation

Without external log services, aggregate logs locally:

```bash
# Collect all service logs into a single file (rotate daily)
docker compose logs --no-color > /opt/cmp/logs/all-services-$(date +%Y%m%d).log
```

### Alerting Options for Air-Gapped Environments

| Method | Setup |
|--------|-------|
| Internal SMTP | Configure CMP to send alerts via internal mail server |
| Syslog | Forward alerts to internal syslog server |
| File-based | Write alerts to monitored log file |
| SNMP | Send SNMP traps to internal monitoring system |
| Local dashboard | Use Prometheus + Grafana deployed locally |

### Setting Up Local Prometheus + Grafana (Optional)

If you want visual dashboards without external services:

1. Include Prometheus and Grafana in your Docker Compose setup
2. Configure Prometheus to scrape the CMP health endpoint
3. Create Grafana dashboards for service health, resource usage, and response times
4. All data stays local — no external connectivity needed

---

## Requesting Support

### Challenges of Air-Gapped Support

Without internet access, traditional support channels (email, portals, remote access) are unavailable. Support follows a structured offline process.

### Support Process

#### 1. Collect Diagnostics

```bash
cd /opt/cmp

# Run the diagnostic collection script
./scripts/collect-diagnostics.sh
# Or manually:
mkdir -p /tmp/cmp-diag
docker compose ps > /tmp/cmp-diag/status.txt
docker compose logs --tail 500 > /tmp/cmp-diag/logs.txt
curl -s http://localhost:8001/health > /tmp/cmp-diag/health.json
curl -s http://localhost:8001/version > /tmp/cmp-diag/version.json
docker stats --no-stream > /tmp/cmp-diag/resources.txt
cat /opt/cmp/logs/upgrade-audit.log > /tmp/cmp-diag/upgrade-audit.log 2>/dev/null
tar -czf /tmp/cmp-diagnostics.tar.gz -C /tmp cmp-diag
```

#### 2. Prepare Support Request

Create a text file describing the issue:

```
Customer ID: cust_xyz789
License ID: lic_abc123
CMP Version: 8.10.0
Deployment Model: air-gapped
Date/Time of Issue: 2025-01-15 10:30 UTC

Issue Description:
[Describe what happened, what you expected, and what you observed]

Steps Taken:
[List troubleshooting steps you've already tried]

Impact:
[Describe the business impact — who is affected and how]
```

#### 3. Transfer Diagnostics

Use your established secure file exchange process:
- Encrypted USB to DMZ transfer point
- Secure file drop
- Physical courier (for non-urgent issues)

#### 4. Receive Response

Autonimbus will analyze the diagnostics and provide:
- Root cause analysis
- Resolution steps (as documentation)
- If needed: a hotfix package delivered via secure channel

### Emergency Support

For critical issues (system completely down):

1. Use your emergency contact procedure (established during onboarding)
2. This may involve a phone call to your designated support engineer
3. They will guide you through resolution steps verbally
4. Any required files (patches, configs) will be delivered via secure channel

### Requesting Software Updates

To request a new version or hotfix:

1. Contact your Autonimbus representative through your secure channel
2. Specify the target version or describe the issue requiring a hotfix
3. Autonimbus prepares the package and delivers via your agreed method
4. Follow the [Upgrading](#upgrading-in-air-gapped-environment) procedure

### Maintaining the Autonimbus Public Key

The public key used for package verification should be:
- Received during initial onboarding via a verified channel
- Stored securely on the CMP server: `/opt/cmp/keys/autonimbus-public.pem`
- Verified via key fingerprint if a new key is ever delivered
- Never modified or replaced without verification from Autonimbus

---

## Operational Checklist

### Daily

- [ ] Verify health endpoint returns `"healthy"`
- [ ] Check disk space (> 20% free)
- [ ] Review alert log for any warnings

### Weekly

- [ ] Review container resource usage trends
- [ ] Verify backup completed successfully
- [ ] Check license expiry (`days_remaining` > 30)
- [ ] Review application logs for recurring errors

### Monthly

- [ ] Test backup restore procedure
- [ ] Review and rotate log files
- [ ] Check for available updates from Autonimbus
- [ ] Verify all containers are running expected versions

### Quarterly

- [ ] Full disaster recovery test
- [ ] Review resource capacity planning
- [ ] Audit user accounts (remove inactive)
- [ ] Review security configurations

---

## Quick Reference

| Task | Command |
|------|---------|
| Check health | `curl -s http://localhost:8001/health \| jq .` |
| Check version | `curl -s http://localhost:8001/version \| jq .` |
| Check license | `curl -s http://localhost:8001/api/v1/license/status \| jq .` |
| View logs | `docker compose logs --tail 100 <service>` |
| Restart service | `docker compose restart <service>` |
| Restart all | `docker compose down && docker compose up -d` |
| Create backup | `./scripts/backup.sh --output /opt/cmp/backups/` |
| Restore backup | `./scripts/backup.sh --restore --source <path>` |
| Container status | `docker compose ps` |
| Resource usage | `docker stats --no-stream` |
| Disk space | `df -h` |

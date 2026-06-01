# CMP Upgrade Guide

This guide covers the complete upgrade process for the Cloud Management Platform (CMP), including pre-upgrade preparation, execution, monitoring, rollback procedures, and troubleshooting.

---

## Table of Contents

1. [Pre-Upgrade Checklist](#pre-upgrade-checklist)
2. [Online Upgrade Procedure](#online-upgrade-procedure)
3. [Air-Gapped Upgrade Procedure](#air-gapped-upgrade-procedure)
4. [Upgrade Script Usage](#upgrade-script-usage)
5. [What Happens During an Upgrade](#what-happens-during-an-upgrade)
6. [Monitoring Upgrade Progress](#monitoring-upgrade-progress)
7. [Rollback Procedure](#rollback-procedure)
8. [Version Compatibility Rules](#version-compatibility-rules)
9. [Post-Upgrade Verification](#post-upgrade-verification)
10. [Troubleshooting Failed Upgrades](#troubleshooting-failed-upgrades)

---

## Pre-Upgrade Checklist

Complete all items before starting an upgrade:

- [ ] **Verify current version:** `curl -s http://localhost:8001/version | jq .version`
- [ ] **Check target version compatibility:** Target must be within 1 major version of current (see [Version Compatibility Rules](#version-compatibility-rules))
- [ ] **Review release notes:** Read the release notes for the target version for breaking changes
- [ ] **Verify disk space:** At least 10 GB free for backup and new images (`df -h`)
- [ ] **Verify backup storage:** Backup destination has sufficient space
- [ ] **Test existing backup:** Confirm your last backup is restorable
- [ ] **Schedule maintenance window:** Notify users of expected downtime (typically 5–15 minutes)
- [ ] **Verify license validity:** Ensure license covers the target version (`curl -s http://localhost:8001/api/v1/license/status | jq .`)
- [ ] **Check health status:** All services should be healthy before upgrading (`curl -s http://localhost:8001/health`)
- [ ] **Document current state:** Record current version, config, and any customizations
- [ ] **Have rollback plan ready:** Know how to rollback if the upgrade fails

---

## Online Upgrade Procedure

Use this method when your environment has internet access to AWS ECR.

### Step 1: Re-authenticate with ECR

```bash
# ECR tokens expire every 12 hours — re-authenticate before upgrading
aws ecr get-login-password --region <region> \
  | docker login --username AWS --password-stdin \
    <account_id>.dkr.ecr.<region>.amazonaws.com
```

```bash
cd /opt/cmp
./scripts/upgrade.sh --target-version 8.11.0 --dry-run
```

The `--dry-run` flag validates compatibility and shows what will happen without making changes.

### Step 2: Execute the Upgrade

```bash
./scripts/upgrade.sh --target-version 8.11.0
```

Or with automatic rollback on failure:

```bash
./scripts/upgrade.sh --target-version 8.11.0 --auto-rollback
```

### Step 3: Monitor Progress

The upgrade script outputs progress in real-time. You can also monitor via:

```bash
# Watch the audit log
tail -f /opt/cmp/logs/upgrade-audit.log

# Check health endpoint
curl -s http://localhost:8001/health | jq .
```

### Step 4: Verify Success

```bash
curl -s http://localhost:8001/version | jq .
curl -s http://localhost:8001/health | jq .
```

---

## Air-Gapped Upgrade Procedure

Use this method for environments without internet access.

### Step 1: Receive the Upgrade Package

Obtain the new version package from Autonimbus via secure delivery (encrypted USB, secure file transfer).

```bash
# Transfer to server
cp /media/usb/cmp-release-v8.11.0.tar.gz /opt/
cd /opt
tar -xzf cmp-release-v8.11.0.tar.gz
cd cmp-release-v8.11.0
```

### Step 2: Verify Package Integrity

```bash
./scripts/verify-checksums.sh
```

**Do not proceed if verification fails.** Contact Autonimbus support.

### Step 3: Run Pre-Upgrade Checks

```bash
./scripts/upgrade.sh --target-version 8.11.0 --dry-run
```

### Step 4: Load New Container Images

```bash
./scripts/install.sh
```

This loads the new version images into the local Docker daemon.

### Step 5: Execute the Upgrade

```bash
cd /opt/cmp
./scripts/upgrade.sh --target-version 8.11.0 --auto-rollback
```

### Step 6: Verify Success

```bash
curl -s http://localhost:8001/version | jq .
curl -s http://localhost:8001/health | jq .
```

---

## Upgrade Script Usage

The `upgrade.sh` script is the primary tool for managing upgrades.

### Syntax

```bash
./scripts/upgrade.sh [OPTIONS]
```

### Options

| Flag | Description | Required |
|------|-------------|----------|
| `--target-version <version>` | The version to upgrade to (e.g., `8.11.0`) | Yes |
| `--dry-run` | Validate compatibility and show plan without executing | No |
| `--auto-rollback` | Automatically rollback if any step fails (no operator prompt) | No |

### Examples

```bash
# Check what would happen (no changes made)
./scripts/upgrade.sh --target-version 8.11.0 --dry-run

# Upgrade with manual rollback confirmation on failure
./scripts/upgrade.sh --target-version 8.11.0

# Upgrade with automatic rollback on failure
./scripts/upgrade.sh --target-version 8.11.0 --auto-rollback
```

### Dry Run Output

```
=== CMP Upgrade Dry Run ===
Current version: 8.10.0
Target version:  8.11.0
Compatibility:   ✓ OK (same major version)

Planned steps:
  1. Create backup (database, config, compose definitions)
  2. Pull new images (cmp-frontend:8.11.0, cmp-backend:8.11.0, ...)
  3. Run database migrations (2 pending)
  4. Restart services
  5. Validate health (all services)

Estimated downtime: 5-10 minutes
No changes were made (dry run).
```

---

## What Happens During an Upgrade

The upgrade orchestrator executes a defined sequence of steps. Each step has a maximum timeout of 600 seconds (10 minutes).

### Upgrade Sequence

```
┌─────────────────────────────────────────────────────────┐
│ 1. BACKUP                                                │
│    • Database contents exported                          │
│    • Configuration files (.env, encrypted configs) saved │
│    • Docker Compose definitions saved                    │
│    • Backup verified as restorable                       │
├─────────────────────────────────────────────────────────┤
│ 2. PULL NEW IMAGES                                       │
│    • Download new version container images                │
│    • (Air-gapped: images already loaded locally)         │
├─────────────────────────────────────────────────────────┤
│ 3. RUN DATABASE MIGRATIONS                               │
│    • Pending migrations applied in order                 │
│    • Each migration is atomic (all-or-nothing)           │
│    • Distributed lock prevents concurrent migrations     │
├─────────────────────────────────────────────────────────┤
│ 4. RESTART SERVICES                                      │
│    • Stop current containers                             │
│    • Start new version containers                        │
├─────────────────────────────────────────────────────────┤
│ 5. HEALTH VALIDATION                                     │
│    • Check all services (120s timeout, 3 retries each)   │
│    • Backend, frontend, worker, AI, Redis, database      │
│    • All must report "healthy" for success               │
└─────────────────────────────────────────────────────────┘
```

### Step Failure Behavior

If any step fails:

1. The upgrade halts immediately
2. The failing step name and error are displayed
3. **With `--auto-rollback`:** Automatic rollback begins
4. **Without `--auto-rollback`:** You are prompted to confirm rollback

---

## Monitoring Upgrade Progress

### Audit Log

All upgrade actions are logged with timestamps to the audit log:

```bash
tail -f /opt/cmp/logs/upgrade-audit.log
```

Example audit log entries:
```
2025-01-15T10:30:00Z | backup          | success | 45.2s | Database and config backed up
2025-01-15T10:30:45Z | pull_images     | success | 120.5s | All images pulled successfully
2025-01-15T10:32:46Z | run_migrations  | success | 8.3s | 2 migrations applied
2025-01-15T10:32:54Z | restart_services| success | 15.1s | All services restarted
2025-01-15T10:33:09Z | health_check    | success | 12.0s | All services healthy
```

### Health Endpoint

During and after upgrade, monitor the health endpoint:

```bash
watch -n 5 'curl -s http://localhost:8001/health | jq .'
```

### Upgrade Status Endpoint

Check upgrade status programmatically:

```bash
curl -s http://localhost:8001/health/upgrade-status | jq .
```

---

## Rollback Procedure

### When to Rollback

Consider rolling back if:

- The upgrade script reports a step failure
- Health checks fail after upgrade
- Application behavior is degraded after upgrade
- Users report critical issues with the new version

### Automatic Rollback

If you used `--auto-rollback`, rollback happens automatically when any step fails. No action is required.

### Manual Rollback

If the upgrade failed without `--auto-rollback`, you'll be prompted:

```
Upgrade failed at step: run_migrations
Error: Migration 003 timed out after 300 seconds

Would you like to rollback to the previous version? [y/N]: y
```

If you need to rollback manually after the script has exited:

```bash
# Restore from the most recent backup
./scripts/upgrade.sh --rollback

# Or restore from a specific backup
./scripts/upgrade.sh --rollback --backup-id 20250115-103000
```

### What Gets Rolled Back

| Component | Rollback Action |
|-----------|----------------|
| Database | Restored from pre-upgrade backup |
| Container images | Reverted to previous version tags |
| Configuration | Previous `.env` and encrypted configs restored |
| Docker Compose | Previous service definitions restored |

### Verifying Rollback Success

After rollback:

```bash
# Confirm version is back to previous
curl -s http://localhost:8001/version | jq .version

# Confirm all services are healthy
curl -s http://localhost:8001/health | jq .

# Confirm data integrity
# (Check key records in the application UI)
```

See the [Rollback Guide](ROLLBACK.md) for detailed rollback procedures and partial failure scenarios.

---

## Version Compatibility Rules

CMP enforces strict version compatibility to ensure safe upgrades:

### Rules

1. **Maximum one major version jump:** You can upgrade from `8.x.x` to `9.x.x`, but NOT from `8.x.x` to `10.x.x`
2. **No downgrades:** You cannot upgrade to a version lower than your current version
3. **Minor and patch versions:** No restrictions within the same major version (e.g., `8.5.0` → `8.11.0` is fine)
4. **License coverage:** Your license must permit the target version (check `permitted_versions` in your license)

### Intermediate Upgrade Paths

If you need to jump multiple major versions, upgrade sequentially:

```
Current: 7.5.0
Target:  9.2.0

Required path:
  7.5.0 → 8.0.0 (or latest 8.x) → 9.2.0
```

### Version Check

```bash
# Check if upgrade is compatible
./scripts/upgrade.sh --target-version 9.0.0 --dry-run
```

If incompatible, you'll see:
```
ERROR: Version jump too large.
Current version: 7.5.0
Target version:  9.0.0
Maximum allowed: 8.x.x

Please upgrade to version 8.x first, then upgrade to 9.x.
```

---

## Post-Upgrade Verification

After a successful upgrade, verify the following:

### 1. Version Confirmation

```bash
curl -s http://localhost:8001/version | jq .
```

Confirm the version matches your target.

### 2. Health Check

```bash
curl -s http://localhost:8001/health | jq .
```

All services should report `"healthy"`.

### 3. License Status

```bash
curl -s http://localhost:8001/api/v1/license/status | jq .
```

Confirm the license is still valid for the new version.

### 4. Application Functionality

- Log in to the CMP web interface
- Verify key workflows (resource provisioning, catalog access, etc.)
- Check that all licensed features are accessible

### 5. Database Schema

```bash
curl -s http://localhost:8001/version | jq .database_schema_version
```

Confirm the schema version matches the expected version for the new release.

---

## Troubleshooting Failed Upgrades

### "Version jump too large"

**Cause:** Target version is more than 1 major version ahead.

**Solution:** Upgrade to an intermediate version first. See [Version Compatibility Rules](#version-compatibility-rules).

### "Backup verification failed"

**Cause:** The pre-upgrade backup could not be verified as restorable.

**Solution:**
- Check disk space: `df -h`
- Check backup directory permissions
- Verify database is accessible
- Retry the upgrade after resolving the issue

### "Migration failed: timeout"

**Cause:** A database migration took longer than 5 minutes.

**Solution:**
- Check database performance and load
- Ensure no other processes are locking the database
- Contact Autonimbus support with the migration version number

### "Health check failed after upgrade"

**Cause:** One or more services didn't become healthy after restart.

**Solution:**
```bash
# Check which service is unhealthy
curl -s http://localhost:8001/health | jq .services

# Check logs for the unhealthy service
docker compose logs cmp-backend --tail 100
docker compose logs cmp-worker --tail 100
```

Common causes:
- New configuration required by the new version
- License doesn't cover the new version
- Resource limits too low for new version requirements

### "Image pull failed"

**Cause:** Cannot download new container images.

**Solution:**
- Re-authenticate with ECR: `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com`
- Check network connectivity to `<account_id>.dkr.ecr.<region>.amazonaws.com`
- Verify your license permits the target version
- For air-gapped: ensure images were loaded with `install.sh`

### Upgrade stuck / no progress

**Cause:** A step is taking longer than expected.

**Solution:**
- Check the audit log: `tail -f /opt/cmp/logs/upgrade-audit.log`
- Each step has a 600-second timeout — it will eventually fail and prompt for rollback
- If truly stuck, you can safely Ctrl+C and run rollback manually

---

## Getting Help

If you encounter upgrade issues not covered here:

1. Collect the upgrade audit log: `cat /opt/cmp/logs/upgrade-audit.log`
2. Collect service logs: `docker compose logs > upgrade-diagnostics.log`
3. Note your current and target versions
4. Contact Autonimbus support with your customer ID and logs

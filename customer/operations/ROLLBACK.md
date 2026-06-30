# CMP Rollback Guide

This guide covers when and how to rollback a CMP upgrade, what gets restored, and how to verify a successful rollback.

---

## Table of Contents

1. [When to Rollback](#when-to-rollback)
2. [Automatic Rollback](#automatic-rollback)
3. [Manual Rollback Procedure](#manual-rollback-procedure)
4. [What Gets Rolled Back](#what-gets-rolled-back)
5. [Verifying Rollback Success](#verifying-rollback-success)
6. [Partial Failure Scenarios](#partial-failure-scenarios)
7. [Contacting Vendor Support](#contacting-vendor-support)

---

## When to Rollback

Consider performing a rollback in the following situations:

### Immediate Rollback (During Upgrade)

- The upgrade script reports a step failure
- Database migration fails or times out
- New container images fail to start
- Health checks fail after service restart

### Post-Upgrade Rollback

- Application is running but critical functionality is broken
- Performance is significantly degraded compared to previous version
- Data integrity issues discovered after upgrade
- Users report widespread errors or inability to work
- A critical security issue is discovered in the new version

### When NOT to Rollback

- Minor UI changes that users need time to adjust to
- A single non-critical feature has a bug (report to vendor instead)
- Performance is slightly different but within acceptable range
- The issue can be resolved with a configuration change

---

## Automatic Rollback

If you used the `--auto-rollback` flag during upgrade, rollback happens automatically when any step fails.

### How It Works

```bash
# Upgrade with automatic rollback
./scripts/upgrade.sh --target-version 8.11.0 --auto-rollback
```

When a failure occurs:

1. The failing step is logged with error details
2. Rollback begins immediately (no operator prompt)
3. Database is restored from the pre-upgrade backup
4. Container images are reverted to the previous version
5. Configuration files are restored
6. Services are restarted with the previous version
7. Health checks confirm the rollback succeeded

### Automatic Rollback Output

```
=== CMP Upgrade ===
[10:30:00] backup          ✓ success (45.2s)
[10:30:45] pull_images     ✓ success (120.5s)
[10:32:46] run_migrations  ✗ FAILED (300.0s) - Migration 003 timed out

Auto-rollback enabled. Beginning rollback...
[10:37:46] restore_database  ✓ success (30.1s)
[10:38:16] revert_images     ✓ success (5.2s)
[10:38:21] restore_config    ✓ success (2.1s)
[10:38:23] restart_services  ✓ success (15.3s)
[10:38:39] health_check      ✓ success (8.5s)

Rollback complete. System restored to version 8.10.0.
```

---

## Manual Rollback Procedure

Use manual rollback when:
- You didn't use `--auto-rollback` and the upgrade failed
- You need to rollback after the upgrade script has already exited
- You discover issues after the upgrade appeared successful

### Step 1: Stop Current Services

```bash
cd /opt/cmp
docker compose down
```

### Step 2: Identify the Backup

Backups are stored in `/opt/cmp/backups/` with timestamp-based naming:

```bash
ls -la /opt/cmp/backups/
# Example output:
# drwxr-xr-x  upgrade-20250115-103000/
# drwxr-xr-x  upgrade-20250110-140000/
```

Use the most recent backup (created just before the failed upgrade):

```bash
BACKUP_DIR=/opt/cmp/backups/upgrade-20250115-103000
```

### Step 3: Restore Database

```bash
# Restore database from backup
./scripts/backup.sh --restore --source $BACKUP_DIR/database-backup.tar.gz
```

### Step 4: Restore Configuration

```bash
# Restore .env and encrypted config files
cp $BACKUP_DIR/config/.env /opt/cmp/.env
cp $BACKUP_DIR/config/encrypted.env /opt/cmp/config/encrypted.env 2>/dev/null
```

### Step 5: Restore Docker Compose Definition

```bash
# Restore the previous docker-compose.yml (with previous image tags)
cp $BACKUP_DIR/docker-compose.yml /opt/cmp/docker-compose.yml
```

### Step 6: Start Services with Previous Version

```bash
docker compose up -d
```

### Step 7: Verify Rollback

```bash
# Check version is back to previous
curl -s http://localhost:8001/version | jq .version

# Check all services are healthy
curl -s http://localhost:8001/health | jq .
```

### Using the Upgrade Script for Rollback

The upgrade script also supports explicit rollback:

```bash
# Rollback using the most recent backup
./scripts/upgrade.sh --rollback

# Rollback using a specific backup
./scripts/upgrade.sh --rollback --backup-id upgrade-20250115-103000
```

---

## What Gets Rolled Back

### Components Restored During Rollback

| Component | What's Restored | How |
|-----------|----------------|-----|
| **Database** | All data as of pre-upgrade snapshot | Full restore from backup |
| **Container Images** | Previous version image tags | Docker Compose reverted to old tags |
| **Configuration (.env)** | Previous environment settings | File restored from backup |
| **Encrypted Config** | Previous encrypted values | File restored from backup |
| **Docker Compose** | Previous service definitions | File restored from backup |
| **Database Schema** | Previous schema version | Restored with database backup |

### What is NOT Rolled Back

| Component | Reason |
|-----------|--------|
| **Audit logs** | Preserved for compliance (append-only) |
| **Upgrade audit log** | Records the failed upgrade attempt |
| **Docker images on disk** | New images remain cached (not harmful) |
| **External system changes** | Changes made to cloud resources during the upgrade window |

### Data Implications

- **Data created during the upgrade window is lost.** If users were active between the upgrade start and rollback, their actions during that period are not preserved.
- **Schedule upgrades during maintenance windows** to minimize data loss risk.

---

## Verifying Rollback Success

After rollback, verify the system is fully operational:

### 1. Version Check

```bash
curl -s http://localhost:8001/version | jq .
```

Confirm the version matches the pre-upgrade version.

### 2. Health Check

```bash
curl -s http://localhost:8001/health | jq .
```

All services should report `"healthy"`.

### 3. Database Schema Version

```bash
curl -s http://localhost:8001/version | jq .database_schema_version
```

Should match the schema version from before the upgrade.

### 4. License Status

```bash
curl -s http://localhost:8001/api/v1/license/status | jq .valid
```

Should return `true`.

### 5. Application Functionality

- Log in to the CMP web interface
- Verify key workflows function correctly
- Check that data from before the upgrade is intact
- Verify all licensed features are accessible

### 6. Container Status

```bash
docker compose ps
```

All containers should be running with the previous version image tags.

---

## Partial Failure Scenarios

In rare cases, a rollback may not fully complete, or the system may be in an inconsistent state.

### Scenario: Migration Rolled Back but Services Running New Version

**Symptoms:** Database is at old schema version, but containers are running new image tags.

**Cause:** Rollback restored the database but failed to restart services with old images.

**Solution:**
```bash
# Manually restore docker-compose.yml from backup
cp /opt/cmp/backups/upgrade-<timestamp>/docker-compose.yml /opt/cmp/docker-compose.yml

# Restart with old images
docker compose down
docker compose up -d

# Verify
curl -s http://localhost:8001/version | jq .
```

### Scenario: Database Restore Failed

**Symptoms:** Rollback script reports database restore failure.

**Cause:** Backup file corrupted, insufficient disk space, or database service unavailable.

**Solution:**
1. Check disk space: `df -h`
2. Verify database service is running: `docker compose ps database`
3. Try restoring manually:
   ```bash
   docker compose up -d database
   # Wait for database to be ready
   ./scripts/backup.sh --restore --source /opt/cmp/backups/upgrade-<timestamp>/database-backup.tar.gz
   ```
4. If restore still fails, contact Autonimbus support immediately

### Scenario: Services Won't Start After Rollback

**Symptoms:** Containers exit immediately after rollback.

**Cause:** Configuration mismatch or corrupted restore.

**Solution:**
```bash
# Check container logs
docker compose logs cmp-backend --tail 50

# Common fixes:
# 1. Verify .env was restored correctly
cat /opt/cmp/.env | grep -E "^(SECRET_KEY|ENCRYPTION_KEY|DYNAMODB_ENDPOINT|AWS_REGION)"

# 2. Verify license file is still in place
ls -la /opt/cmp/license/license.json

# 3. Verify database is accessible
docker compose logs database --tail 20
```

### Scenario: Rollback Succeeded but Data is Missing

**Symptoms:** Application works but recent data (created before upgrade) is missing.

**Cause:** The backup was taken before the most recent data was written.

**Solution:**
- This is expected behavior — the backup represents a point-in-time snapshot
- Data created between the backup and the upgrade start may be lost
- This is why maintenance windows with no user activity are recommended

---

## Contacting Vendor Support

### When to Contact Support After Rollback

- Rollback failed and the system is in an inconsistent state
- You've rolled back but the same issue occurs on retry
- You need help understanding why the upgrade failed
- Database restore failed and you need recovery assistance
- You need a hotfix for the issue that caused the rollback

### Information to Provide

When contacting Autonimbus support after a rollback:

1. **Customer ID and License ID**
2. **Current version** (after rollback)
3. **Target version** (that failed)
4. **Upgrade audit log:**
   ```bash
   cat /opt/cmp/logs/upgrade-audit.log
   ```
5. **Service logs from the failure:**
   ```bash
   docker compose logs > rollback-diagnostics.log
   ```
6. **Health status:**
   ```bash
   curl -s http://localhost:8001/health > health-status.json
   ```
7. **Description of what happened** (timeline of events)

### For Air-Gapped Environments

1. Collect all diagnostic files listed above
2. Transfer via your secure file exchange process
3. Contact your Autonimbus representative through your established channel

### Severity Classification

| Severity | Situation | Expected Response |
|----------|-----------|-------------------|
| **Critical** | System down, rollback failed, data at risk | Immediate (premium), 4 hours (standard) |
| **High** | Rollback succeeded but upgrade cannot proceed | 4 hours (premium), 8 hours (standard) |
| **Medium** | Rollback succeeded, need guidance for retry | 8 hours (premium), 24 hours (standard) |
| **Low** | Informational, planning next upgrade attempt | 24 hours (premium), 48 hours (standard) |

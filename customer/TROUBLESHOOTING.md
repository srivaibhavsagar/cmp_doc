# CMP Troubleshooting Guide

This guide helps you diagnose and resolve common issues with the Cloud Management Platform. It covers health endpoint interpretation, startup failures, service-specific troubleshooting, and when to contact vendor support.

---

## Table of Contents

1. [Health Endpoint Interpretation](#health-endpoint-interpretation)
2. [Common Startup Failures](#common-startup-failures)
3. [Service-Specific Troubleshooting](#service-specific-troubleshooting)
4. [Log Locations and Reading Logs](#log-locations-and-reading-logs)
5. [Container Health and Resource Monitoring](#container-health-and-resource-monitoring)
6. [Network Connectivity Issues](#network-connectivity-issues)
7. [Performance Degradation Diagnosis](#performance-degradation-diagnosis)
8. [When and How to Contact Vendor Support](#when-and-how-to-contact-vendor-support)
9. [Collecting Diagnostic Information](#collecting-diagnostic-information)

---

## Health Endpoint Interpretation

CMP exposes health endpoints that don't require authentication, making them ideal for monitoring.

### Health Endpoint

```bash
curl -s http://localhost:8001/health | jq .
```

### Health States

| State | Meaning | Action Required |
|-------|---------|-----------------|
| `"healthy"` | All services operational | None |
| `"degraded"` | Non-critical service(s) unhealthy | Investigate, but system is functional |
| `"unhealthy"` | Critical service(s) unreachable | Immediate action required |

### Critical vs Non-Critical Services

| Service | Critical | Impact if Down |
|---------|----------|----------------|
| `backend` | Yes | API completely unavailable |
| `database` | Yes | All data operations fail |
| `redis` | Yes | Caching, sessions, and background tasks affected |
| `frontend` | No | Web UI unavailable, API still works |
| `worker` | No | Background tasks delayed |
| `ai_service` | No | AI features unavailable |

### Example Responses

**Healthy:**
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
  },
  "timestamp": "2025-01-15T10:30:00Z",
  "response_time_ms": 45
}
```

**Degraded (non-critical service down):**
```json
{
  "status": "degraded",
  "services": {
    "backend": "healthy",
    "frontend": "healthy",
    "worker": "unhealthy",
    "ai_service": "healthy",
    "redis": "healthy",
    "database": "healthy"
  },
  "timestamp": "2025-01-15T10:30:00Z",
  "response_time_ms": 3012
}
```

**Unhealthy (critical service down):**
```json
{
  "status": "unhealthy",
  "services": {
    "backend": "healthy",
    "frontend": "healthy",
    "worker": "healthy",
    "ai_service": "healthy",
    "redis": "unhealthy",
    "database": "healthy"
  },
  "timestamp": "2025-01-15T10:30:00Z",
  "response_time_ms": 3500
}
```

### Service Check Failure Details

When a service check fails, the health endpoint includes failure details:

```json
{
  "services": {
    "redis": {
      "status": "unhealthy",
      "failure_type": "connection_refused",
      "last_healthy": "2025-01-15T10:25:00Z"
    }
  }
}
```

Failure types:
- `timeout` — Service didn't respond within 3 seconds
- `connection_refused` — Service is not accepting connections
- `error_response` — Service responded with an error

---

## Common Startup Failures

### License Errors

#### "License file not found"

```
FATAL: License file not found at /app/license/license.json
```

**Cause:** License file missing or volume mount incorrect.

**Solution:**
```bash
# Verify file exists on host
ls -la /opt/cmp/license/license.json

# Verify volume mount in docker-compose.yml
grep -A2 "license" docker-compose.yml

# Fix: ensure the mount maps correctly
# volumes:
#   - /opt/cmp/license:/app/license:ro
```

#### "License signature validation failed"

```
FATAL: License signature validation failed. The license file may be corrupted or tampered with.
```

**Cause:** License file was modified, corrupted during transfer, or is not a valid Autonimbus license.

**Solution:**
- Request a new license file from Autonimbus
- Verify the file wasn't modified: compare SHA-256 hash with the one provided by vendor
- Ensure the file was transferred in binary mode (not text mode which can alter line endings)

#### "License expired"

```
FATAL: License expired on 2025-01-15. Please contact Autonimbus for renewal.
```

**Solution:** Contact Autonimbus for a renewed license. See [License Guide](LICENSE.md#license-renewal).

### Configuration Errors

#### "Required configuration missing"

```
FATAL: Required configuration missing: SECRET_KEY, ENCRYPTION_KEY
Application cannot start. Please set the above values in your .env file or environment.
```

**Solution:**
```bash
# Check your .env file has all required keys
grep -E "^(SECRET_KEY|ENCRYPTION_KEY|DYNAMODB_ENDPOINT|AWS_REGION)" /opt/cmp/.env

# Generate missing keys
openssl rand -hex 32  # for SECRET_KEY
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"  # for ENCRYPTION_KEY
```

#### "Invalid DEPLOYMENT_MODEL"

```
FATAL: Invalid DEPLOYMENT_MODEL value: "production"
Valid values: saas, hybrid, on-premises, air-gapped
```

**Solution:** Set `DEPLOYMENT_MODEL` in your `.env` to one of the valid values.

#### "Deployment model mismatch"

```
FATAL: Deployment model mismatch: license='on-premises', configured='air-gapped'
```

**Solution:** Either change `DEPLOYMENT_MODEL` in `.env` to match your license, or request a license for your deployment model.

#### "SMTP_USE_TLS enabled but SMTP_HOST not configured"

```
FATAL: Configuration validation failed: SMTP_USE_TLS enabled without SMTP_HOST configured
```

**Solution:** Either set `SMTP_HOST` or disable `SMTP_USE_TLS=false`.

### Migration Errors

#### "Migration scripts not available"

```
FATAL: Database schema version mismatch.
Current schema: 002
Expected schema: 005
Required migration scripts (003, 004, 005) not found.
Please ensure migration scripts are included in the deployment package.
```

**Solution:**
- Verify the migration scripts are present in the container
- This usually indicates an incomplete deployment package
- Contact Autonimbus support

#### "Migration failed"

```
FATAL: Migration 003 failed: DynamoDB table creation timed out
Database left at schema version 002. Application cannot start.
```

**Solution:**
- Check database connectivity and performance
- Check for resource constraints on the database
- See [Upgrade Guide - Troubleshooting](UPGRADE.md#troubleshooting-failed-upgrades)

---

## Service-Specific Troubleshooting

### Backend (cmp-backend)

**Container won't start:**
```bash
docker compose logs cmp-backend --tail 50
```

Common causes:
- License validation failure (see License Errors above)
- Configuration validation failure (see Configuration Errors above)
- Database unreachable
- Port conflict

**Backend starts but returns 500 errors:**
```bash
# Check for runtime errors
docker compose logs cmp-backend --tail 100 | grep -i error

# Check database connectivity
docker compose exec cmp-backend curl -s http://database:8000
```

### Frontend (cmp-frontend)

**Blank page or loading forever:**
- Check browser developer console for JavaScript errors
- Verify `FRONTEND_URL` matches the URL you're accessing
- Check CORS settings (`ALLOWED_ORIGINS`)

**"Connection refused" in browser:**
```bash
# Verify frontend container is running
docker compose ps cmp-frontend

# Check nginx logs
docker compose logs cmp-frontend --tail 50
```

### Worker (cmp-worker)

**Background tasks not executing:**
```bash
# Check worker is running
docker compose ps cmp-worker

# Check worker logs
docker compose logs cmp-worker --tail 50

# Verify Redis connectivity (worker depends on Redis)
docker compose exec cmp-worker redis-cli -h redis ping
```

### AI Service (cmp-ai)

**AI features not working:**
```bash
# Check AI service is running
docker compose ps cmp-ai

# Check for API key issues
docker compose logs cmp-ai --tail 50 | grep -i "api\|key\|auth"
```

Common causes:
- `GEMINI_API_KEY` not configured or invalid
- AI feature not licensed (check license status)
- AI feature disabled by admin toggle

### Redis

**Redis connection failures:**
```bash
# Check Redis is running
docker compose ps redis

# Test connectivity
docker compose exec redis redis-cli ping
# Expected: PONG

# Check memory usage
docker compose exec redis redis-cli info memory | grep used_memory_human
```

**Redis out of memory:**
```bash
# Check memory limit
docker compose exec redis redis-cli config get maxmemory

# Clear expired keys
docker compose exec redis redis-cli dbsize
```

### Database (DynamoDB)

**Database connection failures:**
```bash
# Check database container
docker compose ps database

# Test endpoint
curl -s http://localhost:8000

# Check database logs
docker compose logs database --tail 50
```

**Slow queries:**
- Check disk I/O: `iostat -x 1 5`
- Check container resource usage: `docker stats database`
- Consider increasing memory allocation

---

## Log Locations and Reading Logs

### Container Logs

All CMP services log to stdout/stderr, accessible via Docker:

```bash
# All services
docker compose logs

# Specific service
docker compose logs cmp-backend

# Follow logs in real-time
docker compose logs -f cmp-backend

# Last N lines
docker compose logs --tail 100 cmp-backend

# Logs since a specific time
docker compose logs --since "2025-01-15T10:00:00" cmp-backend
```

### Log Levels

CMP uses structured logging with these levels:

| Level | Meaning | When to Look |
|-------|---------|--------------|
| `ERROR` | Something failed | Always investigate |
| `WARNING` | Potential issue | Review periodically |
| `INFO` | Normal operations | General monitoring |
| `DEBUG` | Detailed diagnostics | Only when troubleshooting |

### Filtering Logs

```bash
# Find errors only
docker compose logs cmp-backend | grep -i error

# Find license-related messages
docker compose logs cmp-backend | grep -i license

# Find configuration messages
docker compose logs cmp-backend | grep -i "config\|loaded from"

# Find health state transitions
docker compose logs cmp-backend | grep -i "healthy\|degraded\|unhealthy"
```

### Upgrade Audit Log

The upgrade audit log is stored at `/opt/cmp/logs/upgrade-audit.log`:

```bash
cat /opt/cmp/logs/upgrade-audit.log
```

### Log Rotation

Container logs are managed by Docker's logging driver. To prevent disk exhaustion, configure log rotation in your Docker daemon settings (`/etc/docker/daemon.json`):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

---

## Container Health and Resource Monitoring

### Container Status

```bash
# Overview of all containers
docker compose ps

# Detailed status with health
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Resource Usage

```bash
# Real-time resource usage
docker stats

# Snapshot of resource usage
docker stats --no-stream
```

### Key Metrics to Monitor

| Metric | Warning Threshold | Critical Threshold |
|--------|-------------------|-------------------|
| CPU usage | > 80% sustained | > 95% sustained |
| Memory usage | > 80% of limit | > 95% of limit |
| Disk usage | > 80% | > 90% |
| Container restarts | > 2 in 5 minutes | > 5 in 5 minutes |

### Checking Container Restarts

```bash
# Check restart count
docker inspect --format='{{.RestartCount}}' cmp-backend

# Check last restart time
docker inspect --format='{{.State.StartedAt}}' cmp-backend
```

### Disk Space

```bash
# Overall disk usage
df -h

# Docker-specific disk usage
docker system df

# Clean unused resources (careful in production)
docker system prune --volumes
```

---

## Network Connectivity Issues

### Internal Service Communication

CMP services communicate over a Docker network. If services can't reach each other:

```bash
# Verify network exists
docker network ls | grep cmp

# Check which containers are on the network
docker network inspect cmp_default

# Test connectivity between containers
docker compose exec cmp-backend ping -c 3 redis
docker compose exec cmp-backend ping -c 3 database
```

### DNS Resolution

```bash
# Test DNS resolution inside container
docker compose exec cmp-backend nslookup redis
docker compose exec cmp-backend nslookup database
```

### Port Conflicts

```bash
# Check what's using a port
sudo lsof -i :443
sudo lsof -i :8001
sudo lsof -i :6379

# Check Docker port mappings
docker compose port cmp-frontend 80
```

### Firewall Issues

```bash
# Check if firewall is blocking
sudo iptables -L -n | grep -E "(443|8001|6379)"

# For UFW
sudo ufw status

# For firewalld
sudo firewall-cmd --list-all
```

### External Connectivity (Non-Air-Gapped)

```bash
# Test registry access
curl -s https://registry.autonimbus.com/v2/ -o /dev/null -w "%{http_code}"

# Test DNS resolution
nslookup registry.autonimbus.com

# Test with proxy (if applicable)
curl -x http://proxy:8080 https://registry.autonimbus.com/v2/
```

---

## Performance Degradation Diagnosis

### Symptoms and First Steps

| Symptom | First Check |
|---------|-------------|
| Slow page loads | Frontend container resources, network latency |
| Slow API responses | Backend container CPU/memory, database performance |
| Background tasks delayed | Worker container, Redis queue depth |
| Timeouts | Network connectivity, resource limits |

### Backend Performance

```bash
# Check backend response time
time curl -s http://localhost:8001/health > /dev/null

# Check backend container resources
docker stats cmp-backend --no-stream

# Check for high CPU
docker compose exec cmp-backend top -bn1 | head -20
```

### Database Performance

```bash
# Check database container resources
docker stats database --no-stream

# Check disk I/O
iostat -x 1 5

# Check database size
docker compose exec database du -sh /data
```

### Redis Performance

```bash
# Check Redis latency
docker compose exec redis redis-cli --latency

# Check Redis memory
docker compose exec redis redis-cli info memory

# Check queue depth (pending background tasks)
docker compose exec redis redis-cli llen celery
```

### Resource Limit Adjustments

If a service is consistently hitting resource limits, increase them in your `.env`:

```bash
# Example: increase backend memory
BACKEND_MEMORY_LIMIT=4g
BACKEND_CPU_LIMIT=4.0

# Apply changes
docker compose up -d cmp-backend
```

---

## When and How to Contact Vendor Support

### Contact Autonimbus Support When:

- System is down and you cannot resolve it within 30 minutes
- Data integrity issues are suspected
- Upgrade or rollback has failed
- License issues cannot be resolved with documentation
- Security incident suspected (tamper detection alerts)
- Performance issues persist after resource adjustments

### Before Contacting Support

1. Check this troubleshooting guide for your specific issue
2. Collect diagnostic information (see next section)
3. Note the timeline of events (when did the issue start?)
4. Document any recent changes (upgrades, config changes, infrastructure changes)

### Support Channels

| Channel | Use For | Availability |
|---------|---------|--------------|
| Email: support@autonimbus.com | Non-urgent issues | Business hours |
| Support Portal | Ticket tracking | 24/7 (online deployments) |
| Emergency Contact | System down, data at risk | 24/7 (premium tier) |

### Severity Levels

| Severity | Definition | Response Time (Premium) | Response Time (Standard) |
|----------|------------|------------------------|-------------------------|
| P1 - Critical | System down, no workaround | 1 hour | 4 hours |
| P2 - High | Major feature broken | 4 hours | 8 hours |
| P3 - Medium | Feature degraded, workaround exists | 8 hours | 24 hours |
| P4 - Low | Minor issue, question | 24 hours | 48 hours |

---

## Collecting Diagnostic Information

When contacting support, collect the following information to speed up resolution.

### Quick Diagnostic Script

```bash
#!/bin/bash
# Save as: collect-diagnostics.sh
DIAG_DIR="/tmp/cmp-diagnostics-$(date +%Y%m%d-%H%M%S)"
mkdir -p $DIAG_DIR

echo "Collecting CMP diagnostics..."

# System info
echo "=== System Info ===" > $DIAG_DIR/system-info.txt
uname -a >> $DIAG_DIR/system-info.txt
docker --version >> $DIAG_DIR/system-info.txt
docker compose version >> $DIAG_DIR/system-info.txt
df -h >> $DIAG_DIR/system-info.txt
free -h >> $DIAG_DIR/system-info.txt

# Container status
docker compose ps > $DIAG_DIR/container-status.txt 2>&1
docker stats --no-stream > $DIAG_DIR/container-resources.txt 2>&1

# Health and version
curl -s http://localhost:8001/health > $DIAG_DIR/health.json 2>&1
curl -s http://localhost:8001/version > $DIAG_DIR/version.json 2>&1

# Service logs (last 500 lines each)
docker compose logs --tail 500 cmp-backend > $DIAG_DIR/backend.log 2>&1
docker compose logs --tail 500 cmp-frontend > $DIAG_DIR/frontend.log 2>&1
docker compose logs --tail 500 cmp-worker > $DIAG_DIR/worker.log 2>&1
docker compose logs --tail 500 cmp-ai > $DIAG_DIR/ai.log 2>&1
docker compose logs --tail 500 redis > $DIAG_DIR/redis.log 2>&1
docker compose logs --tail 500 database > $DIAG_DIR/database.log 2>&1

# Upgrade audit log (if exists)
cp /opt/cmp/logs/upgrade-audit.log $DIAG_DIR/ 2>/dev/null

# Package it up
tar -czf $DIAG_DIR.tar.gz -C /tmp $(basename $DIAG_DIR)
echo "Diagnostics collected: $DIAG_DIR.tar.gz"
```

### What to Include in Support Tickets

1. **Customer ID** and **License ID**
2. **CMP Version** (from `/version` endpoint)
3. **Deployment Model** (saas, hybrid, on-premises, air-gapped)
4. **Issue Description** (what happened, when, what you expected)
5. **Steps to Reproduce** (if applicable)
6. **Diagnostic Archive** (from the script above)
7. **Recent Changes** (upgrades, config changes, infrastructure changes)

### Sensitive Information

The diagnostic script does **not** collect:
- Passwords or secret keys
- License file contents
- Encrypted configuration values
- User data

If support needs additional information, they will request it specifically.

### For Air-Gapped Environments

Since you cannot upload diagnostics online:

1. Run the diagnostic collection script
2. Transfer the archive via your secure file exchange process
3. Contact your Autonimbus representative through your established channel
4. Reference your customer ID and a brief description of the issue

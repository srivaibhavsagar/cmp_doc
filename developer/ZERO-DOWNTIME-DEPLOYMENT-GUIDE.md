# Zero-Downtime Deployment Guide

A comprehensive guide to achieving zero-downtime upgrades for the Cloud Management Platform (CMP) across customer-based deployments. Covers rolling deployments, health checks, graceful shutdown, database compatibility, frontend atomic swaps, and multi-customer rollout orchestration.

---

## Overview

Zero-downtime deployment means users experience no interruption — no error pages, no dropped requests, no broken UI — while the platform is upgraded to a new version. This is critical for customer-based deployments where each customer has their own CMP instance and expects continuous availability.

### Key Principles

| Principle | What It Means |
|-----------|---------------|
| Start before stop | New version starts and becomes healthy before old version is terminated |
| Graceful drain | Old version finishes in-flight requests before shutting down |
| Backward compatibility | New and old versions coexist briefly — both must work with the same data |
| Atomic frontend swap | Users on old version keep working; new page loads get new version |
| Staged rollout | Deploy to low-risk customers first, then expand to all |
| Automatic rollback | If health checks fail, revert to previous version without manual intervention |

---

## Architecture Requirements

### Minimum Infrastructure for Zero Downtime

```
                    ┌───────────────────────┐
                    │   Reverse Proxy/LB    │
                    │   (Traefik / Nginx /  │
                    │    ALB / Caddy)       │
                    └───────────┬───────────┘
                    ┌───────────┴───────────┐
                    ▼                       ▼
    ┌───────────────────────┐  ┌───────────────────────┐
    │  Container (v1.2.0)   │  │  Container (v1.3.0)   │
    │  [draining/stopping]  │  │  [starting/healthy]   │
    │                       │  │                       │
    │  cmp-backend          │  │  cmp-backend          │
    │  cmp-frontend         │  │  cmp-frontend         │
    │  cmp-worker           │  │  cmp-worker           │
    └───────────────────────┘  └───────────────────────┘
                    │                       │
                    ▼                       ▼
    ┌─────────────────────────────────────────────────┐
    │            Shared State (unchanged)              │
    │  DynamoDB  │  Redis  │  S3 (if applicable)      │
    └─────────────────────────────────────────────────┘
```

For a single-node deployment (common in customer-based setups), the reverse proxy runs on the same host and routes between old and new container sets during the transition window.

---

## Strategy 1: Rolling Container Replacement (Docker Compose)

This is the primary strategy for customer-based deployments running Docker Compose on a single host or small cluster.

### Docker Compose Configuration

```yaml
# docker-compose.yml
services:
  backend:
    image: ${REGISTRY}/cmp-backend:${VERSION}
    deploy:
      replicas: 2
      update_config:
        parallelism: 1          # upgrade one container at a time
        delay: 15s              # wait 15s between container upgrades
        order: start-first      # new container starts BEFORE old one stops
        failure_action: rollback
      rollback_config:
        parallelism: 1
        order: stop-first
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/api/v1/health/ready"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    stop_grace_period: 30s      # time allowed for graceful shutdown

  frontend:
    image: ${REGISTRY}/cmp-frontend:${VERSION}
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 10s

  worker:
    image: ${REGISTRY}/cmp-worker:${VERSION}
    deploy:
      replicas: 1
      update_config:
        order: start-first
        failure_action: rollback
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
    stop_grace_period: 60s      # workers need more time to finish tasks
```

### How `start-first` Works

```
Timeline:
─────────────────────────────────────────────────────────────────────

t=0s    [v1.2.0 ■■■■■■■■■■ serving traffic]

t=1s    [v1.2.0 ■■■■■■■■■■ serving traffic]
        [v1.3.0 ░░░ starting...]

t=15s   [v1.2.0 ■■■■■■■■■■ serving traffic]
        [v1.3.0 ░░░░░░░░ health check pending...]

t=25s   [v1.2.0 ■■■■■■■■■■ serving traffic]
        [v1.3.0 ■■■■■■■■■■ healthy ✓ → receiving traffic]

t=26s   [v1.2.0 ░░░░ draining... (finishing in-flight requests)]
        [v1.3.0 ■■■■■■■■■■ serving all traffic]

t=56s   [v1.2.0 stopped ✗]
        [v1.3.0 ■■■■■■■■■■ serving all traffic]

Zero requests dropped. Zero errors served.
```

---

## Strategy 2: Health Check Endpoints

The load balancer/orchestrator needs to know when a container is ready to receive traffic and when it should be removed from rotation.

### Implementation

```python
# backend/app/api/v1/endpoints/health.py
from fastapi import APIRouter, Response
from app.core.config import settings
from app.core.dynamodb import get_dynamodb_client
import redis.asyncio as aioredis
import asyncio

router = APIRouter(tags=["Health"])


@router.get("/health")
async def liveness_check():
    """
    Liveness probe — is the process running?
    Returns 200 if the application is alive.
    Used by orchestrators to decide whether to restart the container.
    """
    return {
        "status": "alive",
        "version": settings.VERSION,
    }


@router.get("/health/ready")
async def readiness_check(response: Response):
    """
    Readiness probe — is the application ready to serve traffic?
    Returns 200 only when all critical dependencies are reachable.
    Used by load balancers to decide whether to route traffic here.
    """
    checks = {}

    # Check DynamoDB connectivity
    try:
        client = get_dynamodb_client()
        loop = asyncio.get_event_loop()
        await loop.run_in_executor(None, client.describe_table, TableName=settings.DYNAMODB_TABLE)
        checks["dynamodb"] = "healthy"
    except Exception as e:
        checks["dynamodb"] = f"unhealthy: {str(e)}"

    # Check Redis connectivity (non-fatal per hard rules — Redis is best-effort)
    try:
        r = aioredis.from_url(settings.REDIS_URL)
        await r.ping()
        checks["redis"] = "healthy"
        await r.close()
    except Exception:
        checks["redis"] = "degraded (non-fatal)"

    # DynamoDB must be healthy for readiness
    if "unhealthy" in checks.get("dynamodb", ""):
        response.status_code = 503
        return {"ready": False, "checks": checks}

    return {
        "ready": True,
        "version": settings.VERSION,
        "checks": checks,
    }


@router.get("/health/version")
async def version_info():
    """
    Returns current version info. Useful for verifying deployment succeeded.
    """
    return {
        "version": settings.VERSION,
        "environment": settings.ENVIRONMENT,
    }
```

### Register the Health Router

```python
# backend/app/api/v1/api.py
from app.api.v1.endpoints import health

api_router.include_router(health.router, prefix="/health")
```

### Health Check Decision Matrix

| Probe | Purpose | Failure Action | Check Frequency |
|-------|---------|----------------|-----------------|
| `/health` (liveness) | Is the process running? | Restart the container | Every 10s |
| `/health/ready` (readiness) | Can it serve traffic? | Remove from load balancer | Every 10s |
| `/health/version` (admin auth required) | What version is running? | Informational only | On-demand |

---

## Strategy 3: Graceful Shutdown

When a container receives a stop signal (SIGTERM), it must finish in-flight requests before exiting.

### Backend Graceful Shutdown

```python
# backend/app/main.py
import signal
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI

# Track in-flight request count
_in_flight = 0
_shutting_down = False


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan — startup and shutdown hooks."""
    # Startup
    await create_tables()
    register_event_handlers()
    yield
    # Shutdown — graceful drain
    global _shutting_down
    _shutting_down = True
    
    # Wait for in-flight requests to complete (max 25s)
    drain_deadline = asyncio.get_event_loop().time() + 25
    while _in_flight > 0 and asyncio.get_event_loop().time() < drain_deadline:
        await asyncio.sleep(0.5)
    
    # Cleanup resources
    await close_redis_connections()
    await flush_event_bus()


app = FastAPI(lifespan=lifespan)


@app.middleware("http")
async def track_in_flight(request, call_next):
    """Track active requests for graceful shutdown."""
    global _in_flight
    _in_flight += 1
    try:
        response = await call_next(request)
        return response
    finally:
        _in_flight -= 1
```

### Uvicorn Configuration

```bash
# Run with graceful shutdown timeout
uvicorn app.main:app \
    --host 0.0.0.0 \
    --port 8001 \
    --workers 4 \
    --timeout-graceful-shutdown 30
```

This tells Uvicorn: after receiving SIGTERM, wait up to 30 seconds for in-flight requests to complete before forcefully exiting.

### Worker Graceful Shutdown

Background workers need extra care — they may be mid-execution on a long-running task:

```python
# backend/app/worker.py
import signal
import asyncio

_shutdown_requested = False

def handle_shutdown(signum, frame):
    global _shutdown_requested
    _shutdown_requested = True
    logger.info("Shutdown requested — finishing current task...")

signal.signal(signal.SIGTERM, handle_shutdown)

async def process_tasks():
    while not _shutdown_requested:
        task = await get_next_task()
        if task:
            await execute_task(task)  # completes fully before checking shutdown flag
        else:
            await asyncio.sleep(1)
    
    logger.info("Worker shutdown complete — no tasks abandoned")
```

---

## Strategy 4: Database Compatibility (Expand-Contract Pattern)

During a rolling deployment, both old and new versions run simultaneously. The database schema (DynamoDB items) must be compatible with both.

### The Expand-Contract Pattern

```
Version 1.2.0 (current):
  Item: { PK, SK, name, status, owner_id }

Version 1.3.0 (new — adds "priority" field, renames "owner_id" → "assigned_to"):

  Step 1 — EXPAND (deploy v1.3.0-a):
    - Writes BOTH "owner_id" AND "assigned_to" (same value)
    - Writes "priority" (new field, defaults to "medium")
    - Reads "assigned_to" with fallback to "owner_id"
    
  Step 2 — MIGRATE (background task):
    - Backfill all existing items: copy owner_id → assigned_to, set priority
    
  Step 3 — CONTRACT (deploy v1.3.0-b, after migration completes):
    - Stop writing "owner_id"
    - Read only "assigned_to"
    - Remove fallback logic
```

### Rules for DynamoDB Schema Changes

| Change Type | Safe for Zero Downtime? | Approach |
|-------------|------------------------|----------|
| Add new optional field | ✅ Yes | New version writes it; old version ignores it |
| Add new required field | ⚠️ Careful | Add as optional first, backfill, then enforce |
| Rename a field | ❌ Not directly | Use expand-contract (write both → migrate → drop old) |
| Remove a field | ❌ Not directly | Stop reading first → deploy → stop writing → deploy |
| Change field type | ❌ Not directly | Expand-contract with new field name |
| Add new GSI | ✅ Yes | DynamoDB builds GSI in background; queries start working when ready |
| Change key schema | ❌ Never | Requires new table + data migration |

### Implementation Example

```python
# models/resource.py — v1.3.0 (expand phase)
class ResourceInDB(BaseModel):
    # ... existing fields ...
    owner_id: Optional[str] = None        # deprecated — kept for backward compat
    assigned_to: Optional[str] = None     # new canonical field
    priority: Optional[str] = "medium"    # new field

    @property
    def effective_assignee(self) -> Optional[str]:
        """Read assigned_to with fallback to owner_id during migration."""
        return self.assigned_to or self.owner_id


# crud/resource.py — v1.3.0 (expand phase)
async def create_resource(tenant_id: str, data: ResourceCreate) -> dict:
    item = {
        # ... existing fields ...
        "assigned_to": data.assigned_to,
        "owner_id": data.assigned_to,     # write both during expand phase
        "priority": data.priority or "medium",
    }
    item = _sanitize_for_dynamodb(item)
    # ... DynamoDB put_item ...
```

---

## Strategy 5: Frontend Atomic Swap

React frontend deployments are inherently zero-downtime when done correctly, because they're static assets.

### How It Works

```
Before deploy:
  /dist/index.html          → references chunk-abc123.js, style-def456.css
  /dist/chunk-abc123.js     → v1.2.0 application code
  /dist/style-def456.css    → v1.2.0 styles

After deploy:
  /dist/index.html          → references chunk-xyz789.js, style-uvw012.css  [UPDATED]
  /dist/chunk-xyz789.js     → v1.3.0 application code                      [NEW]
  /dist/style-uvw012.css    → v1.3.0 styles                                [NEW]
  /dist/chunk-abc123.js     → v1.2.0 application code                      [STILL EXISTS]
  /dist/style-def456.css    → v1.2.0 styles                                [STILL EXISTS]
```

**Why this is zero-downtime:**
- Users already on the page continue using old chunks (still served)
- New page loads get the new `index.html` → new chunks
- No broken imports because Vite hashes every chunk filename uniquely

### Nginx Configuration for Frontend

```nginx
# nginx.conf for cmp-frontend container
server {
    listen 3000;
    root /usr/share/nginx/html;

    # index.html — never cache (so users get new version on refresh)
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";
    }

    # Hashed assets — cache forever (filename changes = new content)
    location /assets/ {
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Vite Build Configuration

```typescript
// vite.config.ts — ensure content-hash in filenames
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        // Content-hash ensures unique filenames per build
        entryFileNames: 'assets/[name]-[hash].js',
        chunkFileNames: 'assets/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash].[ext]',
      },
    },
  },
});
```

### Keeping Old Assets Available

During the transition window (while some users still have old tabs open), old assets must remain accessible. Two approaches:

**Approach A: Multi-stage Docker build (recommended for customer deployments)**

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
# Copy ONLY the new build — old assets naturally expire from browser cache
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

**Approach B: Asset persistence volume (for longer transition windows)**

```yaml
# docker-compose.yml
services:
  frontend:
    volumes:
      - frontend_assets:/usr/share/nginx/html/assets
    # Assets accumulate across deploys; old hashed files remain available
```

---

## Strategy 6: Reverse Proxy Configuration

The reverse proxy is the traffic director during zero-downtime deploys. It routes requests only to healthy containers.

### Traefik (Recommended for Docker Compose)

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/certs:/certs

  backend:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=PathPrefix(`/api`)"
      - "traefik.http.services.backend.loadbalancer.server.port=8001"
      - "traefik.http.services.backend.loadbalancer.healthcheck.path=/api/v1/health/ready"
      - "traefik.http.services.backend.loadbalancer.healthcheck.interval=5s"
      - "traefik.http.services.backend.loadbalancer.healthcheck.timeout=3s"

  frontend:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=PathPrefix(`/`)"
      - "traefik.http.services.frontend.loadbalancer.server.port=3000"
      - "traefik.http.services.frontend.loadbalancer.healthcheck.path=/"
      - "traefik.http.services.frontend.loadbalancer.healthcheck.interval=5s"
```

### Nginx (Alternative)

```nginx
# nginx-proxy.conf
upstream backend {
    server backend_1:8001;
    server backend_2:8001 backup;  # used during rolling deploy
}

upstream frontend {
    server frontend_1:3000;
    server frontend_2:3000 backup;
}

server {
    listen 443 ssl;
    
    location /api/ {
        proxy_pass http://backend;
        proxy_next_upstream error timeout http_502 http_503;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
    
    location / {
        proxy_pass http://frontend;
        proxy_next_upstream error timeout http_502;
    }
}
```

---

## Strategy 7: Multi-Customer Staged Rollout

For customer-based deployments where each customer has their own CMP instance, use a phased approach to minimize blast radius.

### Rollout Phases

```
Phase 0: Build & Test
├── Build new Docker images (tagged with version)
├── Run automated test suite against new images
├── Push to container registry
└── Generate deployment manifest

Phase 1: Internal Validation (Day 0)
├── Deploy to internal/staging instance
├── Run smoke tests (login, CRUD, cloud operations)
├── Run integration tests (automation triggers, event system)
└── Validate for 2-4 hours under real usage

Phase 2: Canary Customers (Day 0-1)
├── Select 1-2 low-risk customers (internal teams, dev environments)
├── Deploy with zero-downtime rolling update
├── Monitor error rates, latency, and logs for 30-60 minutes
└── If issues detected → rollback canary, investigate

Phase 3: Broad Rollout (Day 1-2)
├── Deploy to remaining customers in batches of 5-10
├── 15-minute pause between batches
├── Monitor aggregate metrics across all instances
└── If error rate spikes → halt rollout, rollback affected batch

Phase 4: Verification (Day 2-3)
├── Confirm all instances on new version (/health/version)
├── Monitor for 24-48 hours
└── Mark release as stable
```

### Deployment Script (Per Customer)

```bash
#!/bin/bash
# deploy-customer.sh — zero-downtime deploy for a single customer instance
set -euo pipefail

CUSTOMER_ID="$1"
NEW_VERSION="$2"
COMPOSE_DIR="/opt/cmp/customers/${CUSTOMER_ID}"
HEALTH_URL="http://localhost:8001/api/v1/health/ready"

echo "═══════════════════════════════════════════════"
echo "  Deploying CMP ${NEW_VERSION} to ${CUSTOMER_ID}"
echo "═══════════════════════════════════════════════"

cd "${COMPOSE_DIR}"

# Step 1: Pull new images
echo "[1/5] Pulling new images..."
VERSION="${NEW_VERSION}" docker compose pull

# Step 2: Rolling update (start-first ensures zero downtime)
echo "[2/5] Starting rolling update..."
VERSION="${NEW_VERSION}" docker compose up -d --no-recreate --remove-orphans

# Step 3: Wait for new containers to be healthy
echo "[3/5] Waiting for health checks..."
MAX_WAIT=120
ELAPSED=0
until curl -sf "${HEALTH_URL}" | grep -q '"ready": true'; do
    sleep 5
    ELAPSED=$((ELAPSED + 5))
    if [ $ELAPSED -ge $MAX_WAIT ]; then
        echo "ERROR: Health check failed after ${MAX_WAIT}s — rolling back"
        VERSION="${OLD_VERSION}" docker compose up -d
        exit 1
    fi
done

# Step 4: Verify version
echo "[4/5] Verifying deployment..."
DEPLOYED_VERSION=$(curl -sf "http://localhost:8001/api/v1/health/version" | jq -r '.version')
if [ "${DEPLOYED_VERSION}" != "${NEW_VERSION}" ]; then
    echo "ERROR: Version mismatch (expected ${NEW_VERSION}, got ${DEPLOYED_VERSION})"
    exit 1
fi

# Step 5: Clean up old images
echo "[5/5] Cleaning up old images..."
docker image prune -f --filter "until=48h"

echo "✓ Successfully deployed ${NEW_VERSION} to ${CUSTOMER_ID}"
```

### Batch Orchestration Script

```bash
#!/bin/bash
# deploy-all-customers.sh — staged rollout across all customer instances
set -euo pipefail

NEW_VERSION="$1"
BATCH_SIZE=5
PAUSE_BETWEEN_BATCHES=900  # 15 minutes

# Customer lists by priority
CANARY_CUSTOMERS=("internal-staging" "dev-team-1")
ALL_CUSTOMERS=($(ls /opt/cmp/customers/ | grep -v "internal-staging\|dev-team-1"))

echo "Starting staged rollout of v${NEW_VERSION}"
echo "Canary: ${#CANARY_CUSTOMERS[@]} customers"
echo "Broad:  ${#ALL_CUSTOMERS[@]} customers (batches of ${BATCH_SIZE})"

# Phase 1: Canary
echo ""
echo "══════ PHASE 1: CANARY ══════"
for customer in "${CANARY_CUSTOMERS[@]}"; do
    ./deploy-customer.sh "${customer}" "${NEW_VERSION}"
done

echo "Canary deployed. Monitoring for 30 minutes..."
sleep 1800

# Check canary health
for customer in "${CANARY_CUSTOMERS[@]}"; do
    if ! curl -sf "http://${customer}:8001/api/v1/health/ready" | grep -q '"ready": true'; then
        echo "ABORT: Canary ${customer} is unhealthy. Halting rollout."
        exit 1
    fi
done
echo "Canary healthy ✓ — proceeding to broad rollout"

# Phase 2: Broad rollout in batches
echo ""
echo "══════ PHASE 2: BROAD ROLLOUT ══════"
TOTAL=${#ALL_CUSTOMERS[@]}
for ((i=0; i<TOTAL; i+=BATCH_SIZE)); do
    BATCH=("${ALL_CUSTOMERS[@]:i:BATCH_SIZE}")
    BATCH_NUM=$(( (i / BATCH_SIZE) + 1 ))
    echo ""
    echo "── Batch ${BATCH_NUM} (${#BATCH[@]} customers) ──"
    
    for customer in "${BATCH[@]}"; do
        ./deploy-customer.sh "${customer}" "${NEW_VERSION}" &
    done
    wait  # wait for all parallel deploys in this batch
    
    if [ $((i + BATCH_SIZE)) -lt $TOTAL ]; then
        echo "Batch complete. Pausing ${PAUSE_BETWEEN_BATCHES}s before next batch..."
        sleep "${PAUSE_BETWEEN_BATCHES}"
    fi
done

echo ""
echo "═══════════════════════════════════════"
echo "  Rollout complete: v${NEW_VERSION}"
echo "  Customers upgraded: $((${#CANARY_CUSTOMERS[@]} + TOTAL))"
echo "═══════════════════════════════════════"
```

---

## Strategy 8: Automatic Rollback

If a deployment fails health checks, the system should automatically revert without human intervention.

### Docker Compose Rollback

```yaml
# docker-compose.yml
services:
  backend:
    deploy:
      update_config:
        failure_action: rollback    # automatic rollback on health check failure
        max_failure_ratio: 0.0      # zero tolerance — any failure triggers rollback
        monitor: 30s                # monitor for 30s after deploy before declaring success
```

### Manual Rollback Command

```bash
# Instant rollback to previous version
VERSION="1.2.0" docker compose up -d

# Or using Docker Compose rollback (if using Docker Swarm mode)
docker service rollback cmp_backend
```

### Rollback Decision Criteria

| Signal | Action |
|--------|--------|
| `/health/ready` returns 503 for > 30s | Automatic rollback |
| Error rate > 5% in first 5 minutes | Automatic rollback |
| p95 latency > 2x baseline | Alert + manual decision |
| Any 500 errors on critical paths (auth, resource CRUD) | Automatic rollback |
| Background task queue growing (not processing) | Alert + manual decision |

---

## Strategy 9: JWT and Auth Compatibility

JWT tokens must remain valid across version upgrades. A user logged in on v1.2.0 must not be forced to re-authenticate when their next request hits v1.3.0.

### What Ensures Compatibility

| Factor | Status | Why |
|--------|--------|-----|
| Same `JWT_SECRET` across versions | ✅ Required | Both versions sign/verify with same key |
| Same `HS256` algorithm | ✅ Required | Token format unchanged |
| Same token payload structure | ✅ Required | `sub`, `tenant_id`, `jti` must not change |
| Token blacklist in shared Redis | ✅ Required | Both versions check same blacklist |
| New claims in token | ⚠️ Careful | Add as optional; old version ignores unknown claims |

### Rules for Auth Changes Across Versions

- **Never change the JWT signing key during a rolling deploy** — old tokens become invalid instantly.
- **Never change the token payload structure** without a migration period.
- **If adding new claims**: old version must tolerate unknown claims (it already does — JWT libraries ignore extra fields).
- **If rotating the signing key**: use a key-rotation strategy (accept both old and new key for a transition period).

```python
# Example: dual-key verification during key rotation
def verify_token(token: str) -> dict:
    """Try current key first, then fall back to previous key."""
    for key in [settings.JWT_SECRET, settings.JWT_SECRET_PREVIOUS]:
        try:
            return jwt.decode(token, key, algorithms=["HS256"])
        except JWTError:
            continue
    raise HTTPException(status_code=401, detail="Invalid token")
```

---

## Strategy 10: Event System Compatibility

The in-process EventBus must handle events from both old and new version formats during the transition window.

### Rules

1. **New event types**: Add handlers in the new version; old version won't emit them — no conflict.
2. **Modified event payloads**: Always add fields as optional; never remove fields in the same release.
3. **Event handler changes**: New handlers must tolerate old-format payloads (missing fields = use defaults).

```python
# services/event_handlers.py — v1.3.0
async def handle_resource_created(event: CmpEvent):
    """Handle resource creation — tolerates v1.2.0 event format."""
    metadata = event.metadata or {}
    
    # New field in v1.3.0 — may not exist in events emitted by v1.2.0 containers
    priority = metadata.get("priority", "medium")  # safe default
    source_version = metadata.get("source_version", "unknown")
    
    # Process event regardless of version
    await notify_subscribers(event, priority=priority)
```

---

## Deployment Checklist

Use this checklist for every version upgrade:

### Pre-Deployment

- [ ] New version passes all automated tests
- [ ] Database changes follow expand-contract pattern (no breaking schema changes)
- [ ] New Pydantic model fields are `Optional` with defaults
- [ ] JWT payload structure unchanged (or backward-compatible)
- [ ] Event payloads backward-compatible (additive only)
- [ ] Health check endpoints functional in new version
- [ ] Docker images built, tagged, and pushed to registry
- [ ] Rollback version identified and tested
- [ ] Deployment script tested against staging

### During Deployment

- [ ] Deploy to canary customers first
- [ ] Verify `/health/ready` returns 200 on new containers
- [ ] Verify `/health/version` shows expected version
- [ ] Monitor error rates during transition
- [ ] Confirm no duplicate event processing (check worker logs)

### Post-Deployment

- [ ] All instances reporting new version
- [ ] Error rates at or below pre-deploy baseline
- [ ] No increase in p95 latency
- [ ] Background tasks processing normally (queue depth stable)
- [ ] Customer-facing features verified (login, CRUD, cloud operations)
- [ ] Old Docker images cleaned up after 48h stability window

---

## Troubleshooting

### New Container Fails Health Check

```
Symptom: Container starts but /health/ready returns 503
```

**Diagnosis:**
1. Check container logs: `docker logs cmp-backend-2`
2. Verify DynamoDB connectivity from inside the container
3. Check if startup migrations are blocking readiness
4. Verify environment variables are correct in new image

**Resolution:** The `failure_action: rollback` setting will automatically revert. Fix the issue and redeploy.

### Users See Stale Frontend After Deploy

```
Symptom: Users report seeing old UI even after deploy
```

**Diagnosis:** Browser has cached `index.html`.

**Resolution:** Verify Nginx serves `index.html` with `Cache-Control: no-cache`. Users can force-refresh (Ctrl+Shift+R). This is cosmetic — functionality still works.

### Duplicate Background Task Execution

```
Symptom: Scheduled tasks (drift detection, policy scans) run multiple times
```

**Diagnosis:** Multiple containers running the worker service simultaneously during transition.

**Resolution:**
- Ensure only one worker replica (`replicas: 1` in compose)
- Or implement Redis-based distributed locking:

```python
async def run_with_lock(task_name: str, task_fn, ttl: int = 300):
    lock = await redis.set(f"cmp:lock:{task_name}", "1", nx=True, ex=ttl)
    if lock:
        try:
            await task_fn()
        finally:
            await redis.delete(f"cmp:lock:{task_name}")
```

### Token Rejected After Deploy

```
Symptom: Users get 401 errors immediately after deploy
```

**Diagnosis:** JWT_SECRET changed between versions, or token blacklist Redis lost data.

**Resolution:**
- Verify `JWT_SECRET` is identical in old and new `.env`
- Check Redis connectivity from new containers
- If Redis was restarted: tokens will work (blacklist is security improvement, not requirement per hard rules)

---

## Version Compatibility Matrix

When planning upgrades, verify compatibility between adjacent versions:

| Component | Must Match Between Versions? | Notes |
|-----------|------------------------------|-------|
| JWT_SECRET | ✅ Yes | Tokens signed by old version must validate on new |
| ENCRYPTION_KEY (Fernet) | ✅ Yes | Credentials encrypted by old version must decrypt on new |
| DynamoDB table structure | ✅ Backward-compatible | New fields optional; no field removal |
| Redis key format | ✅ Backward-compatible | New keys fine; don't change existing key patterns |
| API response format | ⚠️ Additive only | New fields OK; don't remove or rename existing fields |
| Frontend API calls | ⚠️ Must work against old backend | During rollover, new frontend may hit old backend |

---

## Summary

Zero-downtime deployment for CMP customer instances relies on six pillars:

1. **Start-first container replacement** — new version healthy before old stops
2. **Health check gating** — load balancer only routes to ready containers
3. **Graceful shutdown** — in-flight requests complete before container exits
4. **Backward-compatible data** — expand-contract for schema changes
5. **Atomic frontend swap** — hashed assets coexist, `index.html` is the only thing that changes
6. **Staged rollout** — canary → batch → full, with automatic rollback at each stage

Implementing these patterns ensures that upgrading from v1.2.0 to v1.3.0 (or any version) is invisible to end users — no errors, no re-authentication, no downtime.

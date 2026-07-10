# CMP Infrastructure Sizing Guide

Recommended instance types, storage, and architecture for deploying the Cloud Management Platform across AWS, Azure, and GCP — based on real stress testing data.

---

## Quick Reference

| Concurrent Users | Deployment | vCPUs | RAM | Workers | Storage | Estimated Monthly Cost |
|-----------------|------------|-------|-----|---------|---------|----------------------|
| 1–15 | Single node | 2 | 4 GB | 2 | 30 GB | $30–50 |
| 15–50 | Single node | 4 | 8 GB | 4 | 50 GB | $60–120 |
| 50–100 | Single node (large) | 8 | 16 GB | 8 | 80 GB | $150–250 |
| 100–200 | Multi-node (2 nodes) | 4 each | 8 GB each | 4 each | 50 GB each | $200–400 |
| 200–500 | Multi-node (3 nodes) | 4 each | 8 GB each | 4 each | 50 GB each | $300–600 |
| 500+ | Multi-node (4+ nodes) | 8 each | 16 GB each | 8 each | 80 GB each | $600+ |

---

## Stress Test Baseline (measured)

These numbers come from actual Locust stress tests against the CMP backend:

| Config | Max Users | Median Latency | P95 Latency | Error Rate |
|--------|-----------|---------------|-------------|------------|
| 1 worker, 5 threads, 2 vCPU | 15 | 170ms | 710ms | <2% |
| 1 worker, 5 threads, 2 vCPU (39 users) | 39 | 1,800ms | 4,000ms | 21% (502s) |
| 2 workers, 20 threads, 2 vCPU (projected) | 40–50 | 400–600ms | 1,200ms | <3% |
| 4 workers, 20 threads, 4 vCPU (projected) | 60–80 | 200–400ms | 800ms | <2% |

**Key formula:**
```
Max comfortable users ≈ vCPUs × workers × 10–15
```

---

## Instance Type Recommendations

### AWS (EC2)

| Tier | Users | Instance Type | vCPUs | RAM | EBS Volume | Use Case |
|------|-------|---------------|-------|-----|------------|----------|
| Dev/Test | 1–5 | `t3.small` | 2 | 2 GB | 20 GB gp3 | Local testing, demos |
| Starter | 5–15 | `t3.medium` | 2 | 4 GB | 30 GB gp3 | Small team, POC |
| Standard | 15–50 | `t3.large` | 2 | 8 GB | 50 GB gp3 | Production, single team |
| Growth | 50–100 | `t3.xlarge` | 4 | 16 GB | 80 GB gp3 | Multiple teams, heavy usage |
| Enterprise (single) | 100–150 | `m6i.xlarge` | 4 | 16 GB | 100 GB gp3 | Sustained workloads (no burst) |
| Enterprise (multi-node) | 100–500 | 2–4× `t3.large` | 2 each | 8 GB each | 50 GB gp3 each | HA + scale |

**Notes:**
- `t3` instances are burstable — great for CMP since load is spiky (login bursts, sync intervals)
- Use `m6i` for sustained 24/7 load where burst credits would deplete
- EBS gp3: 3,000 IOPS baseline is sufficient; CMP is not disk-intensive
- Consider `t3a` (AMD) for ~10% cost savings with same performance

### Azure (Virtual Machines)

| Tier | Users | VM Size | vCPUs | RAM | Disk | Use Case |
|------|-------|---------|-------|-----|------|----------|
| Dev/Test | 1–5 | `Standard_B2s` | 2 | 4 GB | 30 GB P4 SSD | Local testing, demos |
| Starter | 5–15 | `Standard_B2ms` | 2 | 8 GB | 30 GB P6 SSD | Small team, POC |
| Standard | 15–50 | `Standard_B4ms` | 4 | 16 GB | 64 GB P6 SSD | Production, single team |
| Growth | 50–100 | `Standard_D4s_v5` | 4 | 16 GB | 128 GB P10 SSD | Multiple teams |
| Enterprise (single) | 100–150 | `Standard_D4as_v5` | 4 | 16 GB | 128 GB P10 SSD | Sustained workloads |
| Enterprise (multi-node) | 100–500 | 2–4× `Standard_B4ms` | 4 each | 16 GB each | 64 GB each | HA + scale |

**Notes:**
- `B-series` = burstable (same concept as AWS t3)
- `D-series` = general purpose, predictable performance
- Azure Managed Disks: P6 (64 GB) offers 240 IOPS — sufficient for CMP
- Use Availability Sets or Zones for multi-node HA

### GCP (Compute Engine)

| Tier | Users | Machine Type | vCPUs | RAM | Disk | Use Case |
|------|-------|-------------|-------|-----|------|----------|
| Dev/Test | 1–5 | `e2-small` | 2 | 2 GB | 20 GB pd-balanced | Local testing, demos |
| Starter | 5–15 | `e2-medium` | 2 | 4 GB | 30 GB pd-balanced | Small team, POC |
| Standard | 15–50 | `e2-standard-4` | 4 | 16 GB | 50 GB pd-ssd | Production, single team |
| Growth | 50–100 | `n2-standard-4` | 4 | 16 GB | 80 GB pd-ssd | Multiple teams |
| Enterprise (single) | 100–150 | `n2-standard-4` | 4 | 16 GB | 100 GB pd-ssd | Sustained workloads |
| Enterprise (multi-node) | 100–500 | 2–4× `e2-standard-4` | 4 each | 16 GB each | 50 GB pd-ssd each | HA + scale |

**Notes:**
- `e2` = cost-optimized (best value for CMP workloads)
- `n2` = balanced performance (better for sustained CPU)
- `pd-balanced` is fine for dev; use `pd-ssd` for production (higher IOPS)
- Cloud Run is also viable: set min instances = 2, max = 10, memory = 2 GB per instance

---

## Storage Breakdown

| Component | Size Needed | Notes |
|-----------|-------------|-------|
| OS + Docker | 10–15 GB | Base system |
| Backend container image | 1–2 GB | Python + dependencies |
| Frontend container image | 0.5 GB | Node + build artifacts |
| DynamoDB Local data | 5–20 GB | Grows with resources/events (production uses AWS DynamoDB — no local storage) |
| Redis data | 0.5–2 GB | Token blacklist + cache |
| Logs | 5–10 GB | Rotate with logrotate |
| Buffer | 10–20 GB | Room for updates, temp files |

**Recommended total disk:**
- Dev: 30 GB
- Production (single node): 50–80 GB
- Production (per node in multi-node): 50 GB (DynamoDB is external)

---

## Architecture Patterns

### Single Node (1–100 users)

```
┌─────────────────────────────────────────────┐
│  VM (t3.large / B4ms / e2-standard-4)       │
│                                              │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │  Frontend   │  │  Backend (uvicorn)   │  │
│  │  (Nginx/    │  │  - 4 workers         │  │
│  │   Caddy)    │  │  - 20 threads each   │  │
│  │  :3000      │  │  :8000               │  │
│  └─────────────┘  └──────────────────────┘  │
│                                              │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │  Redis      │  │  DynamoDB Local      │  │
│  │  :6379      │  │  (or AWS DynamoDB)   │  │
│  └─────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────┘
```

**Configuration:**
```yaml
# docker-compose.yml
backend:
  environment:
    - WEB_CONCURRENCY=4          # Match vCPUs
    - CMP_THREAD_POOL_SIZE=20    # DynamoDB concurrency
    - REDIS_URL=redis://redis:6379/0
```

### Multi-Node (100–500 users)

```
                    ┌────────────────┐
                    │  Load Balancer │
                    │  (ALB / Azure  │
                    │   LB / GCP LB)│
                    └────────┬───────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────┴─────┐     ┌─────┴─────┐     ┌─────┴─────┐
    │  Node 1   │     │  Node 2   │     │  Node 3   │
    │  4 workers│     │  4 workers│     │  4 workers│
    │  Backend  │     │  Backend  │     │  Backend  │
    │  Frontend │     │  Frontend │     │  Frontend │
    └───────────┘     └───────────┘     └───────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────┴─────┐ ┌─────┴─────┐ ┌─────┴─────┐
        │  Redis    │ │ DynamoDB  │ │ (Optional) │
        │ (shared)  │ │ (AWS)     │ │ S3/GCS     │
        └───────────┘ └───────────┘ └───────────┘
```

**Key requirements for multi-node:**
- All nodes share the same Redis instance (event bus + cache + locks)
- All nodes share the same DynamoDB (or use AWS DynamoDB)
- Scheduler leader lock ensures only 1 node runs background jobs
- Event bus pub/sub ensures events propagate to all nodes
- Load balancer health check: `GET /health` → 200

**Configuration (per node):**
```yaml
environment:
  - WEB_CONCURRENCY=4
  - CMP_THREAD_POOL_SIZE=20
  - REDIS_URL=redis://shared-redis-host:6379/0
  - DYNAMODB_ENDPOINT=https://dynamodb.us-east-1.amazonaws.com  # Real DynamoDB
```

---

## Scaling Decision Tree

```
How many concurrent users?
│
├── < 15 users
│   └── t3.medium (2 vCPU, 4 GB) — single node, 2 workers
│
├── 15–50 users
│   └── t3.large (2 vCPU, 8 GB) or t3.xlarge (4 vCPU, 16 GB) — single node
│
├── 50–100 users
│   ├── Need HA? → 2× t3.large behind ALB
│   └── Cost-sensitive? → 1× t3.xlarge, 4 workers
│
├── 100–200 users
│   └── 2–3× t3.large behind ALB (mandatory for this scale)
│
└── 200+ users
    └── 3–4× t3.xlarge behind ALB + consider read replicas
```

---

## Redis Sizing

| Users | Redis Memory | Instance |
|-------|-------------|----------|
| 1–50 | 50–200 MB | Co-located (Docker) or ElastiCache `cache.t3.micro` |
| 50–200 | 200–500 MB | ElastiCache `cache.t3.small` / Azure Cache Basic C0 |
| 200–500 | 500 MB–1 GB | ElastiCache `cache.t3.medium` / Azure Cache Standard C1 |

**What's stored in Redis:**
- Token blacklist (logout): ~100 bytes × active sessions
- Rate limit counters: ~50 bytes × endpoints × IPs
- Shared cache (dashboard stats, cost summary): ~10–50 KB per tenant
- Event bus pub/sub: ephemeral (no storage)
- Scheduler leader lock: ~100 bytes

---

## DynamoDB Capacity

| Users | Read Capacity | Write Capacity | Mode |
|-------|--------------|----------------|------|
| 1–50 | On-Demand | On-Demand | Pay-per-request (recommended) |
| 50–200 | On-Demand | On-Demand | Pay-per-request |
| 200+ | Provisioned 100 RCU / 50 WCU | Provisioned | Cost optimization with auto-scaling |

**Table sizes (estimated):**
- Users table: <1 MB (grows slowly)
- Resources cache: 1–50 MB (depends on cloud accounts — 10K resources ≈ 20 MB)
- Executions: 5–100 MB (grows with provisioning activity)
- Events: 10–200 MB (grows fast — configure TTL/retention)
- All other tables: <5 MB each

---

## Network & Load Balancer

| Provider | Service | Configuration |
|----------|---------|---------------|
| AWS | ALB | Target group with health check `/health`, sticky sessions OFF (JWT is stateless) |
| Azure | Azure Load Balancer (Standard) | Backend pool, health probe `/health` |
| GCP | Cloud Load Balancing (HTTP/S) | Backend service with health check `/health` |

**Health check settings:**
- Path: `/health`
- Interval: 10s
- Threshold: 3 consecutive failures
- Timeout: 5s

**SSL/TLS:**
- Terminate at load balancer (Let's Encrypt or ACM/Azure Cert/GCP-managed)
- Backend communication over HTTP (within VPC)

---

## Cost Comparison (monthly estimates, single node production)

| Tier | AWS | Azure | GCP |
|------|-----|-------|-----|
| Starter (15 users) | ~$35 (t3.medium + 30GB gp3) | ~$40 (B2ms + P6) | ~$30 (e2-medium + pd-balanced) |
| Standard (50 users) | ~$75 (t3.large + 50GB gp3) | ~$85 (B4ms + P6) | ~$70 (e2-standard-4 + pd-ssd) |
| Growth (100 users) | ~$150 (t3.xlarge + 80GB gp3) | ~$170 (D4s_v5 + P10) | ~$140 (n2-standard-4 + pd-ssd) |

**Add for managed services (production):**
- DynamoDB On-Demand: ~$5–50/month (usage-based)
- ElastiCache/Redis: ~$15–50/month (cache.t3.micro–small)
- Load Balancer: ~$20–30/month (if multi-node)
- Domain + SSL: $0–12/year

---

## Environment Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `WEB_CONCURRENCY` | 2 | Uvicorn worker count (set to vCPU count) |
| `CMP_THREAD_POOL_SIZE` | 20 | Async thread pool size per worker |
| `REDIS_URL` | redis://redis:6379/0 | Shared Redis for cache, locks, events |
| `CMP_NODE_ID` | (auto-detected) | Node identifier for logs |
| `DYNAMODB_ENDPOINT` | (AWS default) | DynamoDB endpoint URL |

---

## Monitoring Checklist

Before scaling, monitor these metrics:

| Metric | Warning | Scale Up When |
|--------|---------|---------------|
| CPU (avg) | >70% sustained | >80% for 5+ minutes |
| Memory | >75% | >85% |
| Response P95 | >1s | >2s sustained |
| Error rate (5xx) | >1% | >5% |
| Event bus queue size | >5,000 | >8,000 |
| Redis memory | >75% of limit | >85% |
| DynamoDB throttles | Any | >10/minute sustained |

---

## Migration Path

```
Phase 1: Current (single VM, 2 vCPU)
   → Deploy with WEB_CONCURRENCY=2, CMP_THREAD_POOL_SIZE=20
   → Handles 30–50 users comfortably

Phase 2: Vertical scale (upgrade VM to 4 vCPU)
   → Set WEB_CONCURRENCY=4
   → Handles 60–100 users
   → Zero architecture changes

Phase 3: Horizontal scale (add nodes)
   → Move Redis to managed service (ElastiCache/Azure Cache/Memorystore)
   → Move DynamoDB to AWS managed (if not already)
   → Add load balancer
   → Deploy 2–3 nodes
   → Handles 100–300 users
   → Multi-node code already in place (event bus, scheduler lock, shared cache)

Phase 4: Enterprise scale
   → 4+ nodes behind ALB
   → DynamoDB with provisioned capacity + auto-scaling
   → Redis cluster mode
   → CDN for frontend static assets
   → Handles 500+ users
```

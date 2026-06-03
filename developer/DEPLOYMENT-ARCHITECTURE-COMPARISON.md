# CMP Deployment Architecture Comparison

A detailed comparison of deployment topologies for the Cloud Management Platform (CMP) — from single EC2 to multi-EC2 load-balanced clusters — covering architecture, performance characteristics, failure modes, cost, and recommendations.

---

## Target Workload

| Metric | Value |
|--------|-------|
| Registered users | ~2,000 |
| Managed cloud resources | 10,000+ |
| Peak concurrent sessions (estimate) | 150–300 |
| Background operations (automations, drift checks, policy scans) | Continuous |
| AI/Gemini requests | Bursty, 2–30s per call |

---

## Architecture Options

### Option A: Single EC2 — Multi-Container (Current)

All services run on one EC2 instance via Docker Compose.

```
┌──────────────────────────────────────────────────────────────────┐
│                     EC2 (m5.xlarge / 4 vCPU, 16GB)               │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ cmp-frontend │  │ cmp-backend  │  │  cmp-worker  │           │
│  │  (Nginx)     │  │ (FastAPI x4  │  │ (Background  │           │
│  │  Port 3000   │  │  workers)    │  │  tasks)      │           │
│  │              │  │  Port 8000   │  │  Port 8002   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   cmp-ai     │  │    Redis     │  │  DynamoDB    │           │
│  │  (AI/Gemini) │  │  (Cache +    │  │   Local      │           │
│  │  Port 8001   │  │   Queue)     │  │  Port 8000   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Option B: Multi-EC2 — Load Balanced

Multiple EC2 instances behind an ALB, with managed AWS services for state.

```
                        ┌───────────────┐
                        │     ALB       │
                        │  (HTTPS:443)  │
                        └───────┬───────┘
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│   EC2 Instance 1    │ │   EC2 Instance 2    │ │   EC2 Instance 3    │
│                     │ │                     │ │                     │
│ ┌────────────────┐  │ │ ┌────────────────┐  │ │ ┌────────────────┐  │
│ │ cmp-frontend   │  │ │ │ cmp-frontend   │  │ │ │ cmp-frontend   │  │
│ │ cmp-backend    │  │ │ │ cmp-backend    │  │ │ │ cmp-backend    │  │
│ │ cmp-worker     │  │ │ │ cmp-worker     │  │ │ │ cmp-worker     │  │
│ │ cmp-ai         │  │ │ │ cmp-ai         │  │ │ │ cmp-ai         │  │
│ └────────────────┘  │ │ └────────────────┘  │ │ └────────────────┘  │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
                    │           │           │
                    ▼           ▼           ▼
         ┌──────────────────────────────────────────┐
         │          Shared Managed Services          │
         │                                          │
         │  ┌─────────────┐    ┌─────────────────┐ │
         │  │ ElastiCache │    │   DynamoDB      │ │
         │  │  (Redis)    │    │  (Managed)      │ │
         │  └─────────────┘    └─────────────────┘ │
         └──────────────────────────────────────────┘
```

### Option C: Multi-EC2 — Role-Separated (Recommended for Scale)

Dedicated EC2 instances per service role, with managed backing services.

```
                        ┌───────────────┐
                        │     ALB       │
                        │  (HTTPS:443)  │
                        └───────┬───────┘
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
    ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐
    │  API Tier (x2)  │  │ AI Tier (x1) │  │Worker Tier   │
    │                 │  │              │  │   (x1-2)     │
    │ cmp-frontend    │  │ cmp-ai       │  │ cmp-worker   │
    │ cmp-backend     │  │              │  │              │
    │                 │  │              │  │              │
    └─────────────────┘  └──────────────┘  └──────────────┘
               │                │                │
               ▼                ▼                ▼
    ┌───────────────────────────────────────────────────┐
    │              Shared Managed Services               │
    │                                                   │
    │  ┌──────────────┐  ┌──────────────┐              │
    │  │ ElastiCache  │  │   DynamoDB   │              │
    │  │ (Redis       │  │  (On-demand) │              │
    │  │  Cluster)    │  │              │              │
    │  └──────────────┘  └──────────────┘              │
    └───────────────────────────────────────────────────┘
```

---

## Detailed Comparison

### Performance & Scalability

| Factor | Option A: Single EC2 | Option B: Multi-EC2 (Cloned) | Option C: Multi-EC2 (Role-Separated) |
|--------|---------------------|------------------------------|--------------------------------------|
| Max concurrent users | ~300 (constrained by single host) | ~1000+ (scales horizontally) | ~2000+ (optimal resource allocation) |
| API response time (p95) | <200ms at low load; degrades under heavy background work | Consistent <200ms (load distributed) | Consistent <100ms (API not competing with worker/AI) |
| Background task throughput | Limited — shares CPU/memory with API | Better — more total compute | Best — dedicated resources for workers |
| AI request handling | Can block API workers under load | Distributed but still co-located | Isolated — AI can OOM without affecting API |
| Horizontal scaling | Not possible | Add EC2 instances (all roles scale together) | Scale each tier independently based on bottleneck |
| Resource efficiency | High utilization but contention risk | Moderate — every node runs all services even if only API is overloaded | High — add resources only where needed |

### Availability & Failure Modes

| Failure Scenario | Option A | Option B | Option C |
|------------------|----------|----------|----------|
| EC2 instance failure | **Total outage** — all services down | Partial — ALB routes to healthy instances | Partial — only affected tier is down, others continue |
| Backend OOM / crash | All services on same host may be affected | Isolated to one node; others serve traffic | Only API tier affected; workers + AI continue |
| Worker memory leak | Starves backend/AI of memory on same host | Affects one node; others are fine | Only worker tier affected; API stays responsive |
| AI service hangs (Gemini timeout) | Ties up shared resources | Affects one node | Zero impact on API/worker tiers |
| Redis failure | Graceful degradation (auth still works, token blacklist is best-effort) | Same — all nodes fail gracefully | Same |
| DynamoDB throttling | All services impacted equally | Same | Can prioritize API reads over worker writes via separate connections |
| Deployment failure (bad binary) | **Total outage** during deploy | Rolling deploy — ALB drains before switching | Rolling per-tier — can deploy API without touching workers |
| Network partition | N/A (single host) | Split-brain possible if nodes can't reach shared services | Same, but smaller blast radius per tier |

### Operational Complexity

| Factor | Option A | Option B | Option C |
|--------|----------|----------|----------|
| Setup complexity | Low — one docker-compose up | Medium — ALB + ASG + shared services | Medium-High — multiple ASGs + target groups |
| Deployment | Simple — pull and restart | Rolling via ASG or CodeDeploy | Per-tier rolling deploy |
| Monitoring | One host to monitor | Multiple hosts, same metrics per node | Different metrics per tier (API latency vs worker queue depth) |
| Debugging | Easy — all logs on one host | Distributed logs — need CloudWatch aggregation | Same, plus need to correlate across tiers |
| Configuration | One `.env.production` | Same config replicated to all nodes | Different configs per tier (worker doesn't need PORT 8000, AI needs GEMINI_API_KEY) |
| Secrets management | One host to secure | Multiple hosts — use AWS Secrets Manager or Parameter Store | Same |
| SSL/TLS | Self-managed or Let's Encrypt on the instance | ALB handles TLS termination (ACM cert) | Same |

### Cost Estimate (AWS, us-east-1)

| Component | Option A | Option B (3 nodes) | Option C |
|-----------|----------|--------------------|-----------| 
| EC2 compute | 1× m5.xlarge (~$140/mo) | 3× m5.large (~$225/mo) | 2× t3.large (API) + 1× t3.medium (worker) + 1× t3.medium (AI) ≈ $200/mo |
| ALB | Not needed ($0) | 1× ALB (~$22/mo + LCU) | 1× ALB (~$22/mo + LCU) |
| ElastiCache Redis | Not needed (local) | cache.t3.small (~$25/mo) | cache.t3.small (~$25/mo) |
| DynamoDB | Not needed (local) or On-Demand (~$25-50/mo) | On-Demand (~$25-50/mo) | On-Demand (~$25-50/mo) |
| Data transfer | Minimal | Inter-AZ ($0.01/GB) | Same |
| **Total estimate** | **~$140–190/mo** | **~$300–350/mo** | **~$270–320/mo** |

*Costs are approximate. Reserved instances reduce EC2 by ~30-40%.*

### Multi-EC2 Considerations (Current Architecture)

Running the current codebase (single backend binary, in-process EventBus) across multiple EC2s behind a load balancer introduces specific challenges:

#### What Works Immediately

- **Stateless API requests** — DynamoDB is external, so any instance can serve any request
- **Auth** — JWT validation is stateless (each node can verify tokens independently)
- **Frontend** — Static bundle, no session state, works perfectly behind ALB

#### What Breaks or Needs Attention

| Issue | Why It's a Problem | Solution |
|-------|-------------------|----------|
| Token blacklist (Redis) | Each node needs to check the same blacklist | Point all nodes to shared ElastiCache (already supported via `REDIS_URL` env var) |
| EventBus (in-process) | Events emitted on Node 1 are invisible to Node 2 | Move to shared Redis pub/sub or proper task queue (ARQ/Celery) |
| Active user tracking | Currently per-process; multi-node = undercounting | Already uses Redis when `REDIS_URL` is set — works out of the box |
| WebSocket/SSE (if any) | Sticky sessions needed or pub/sub fan-out | ALB sticky sessions or Redis-backed socket adapter |
| Background automations | Every node runs its own scheduler = duplicate executions | Need distributed lock (Redis SETNX) or elect one node as scheduler |
| Rate limiting | In-memory counters are per-instance | Move to Redis-based rate limiter (sliding window in shared Redis) |
| Health checks | ALB needs `/health` endpoint | Already exists |
| DynamoDB Local | Can't share across nodes | **Must** use managed DynamoDB — no local option for multi-node |

#### Critical: Duplicate Event Processing

The biggest risk with Option B (identical clones behind ALB) is that **every node's EventBus will process its own events independently**. If a user triggers an automation:
- The request hits Node 1 → Node 1's EventBus fires handlers
- Handlers that create new resources or send notifications run only on Node 1
- If those handlers also emit events, only Node 1 sees them

This is fine for request-scoped events, but **scheduled/cron-style automations** (drift detection, policy scans) will run on ALL nodes simultaneously, causing duplicate executions.

**Fix:** Introduce a distributed lock before scheduled operations:
```python
# Example: Redis-based distributed lock for scheduled tasks
async def run_if_leader(task_name: str, ttl: int = 60):
    lock_key = f"cmp:leader:{task_name}"
    acquired = await redis.set(lock_key, instance_id, nx=True, ex=ttl)
    return acquired
```

---

## Recommendations by Growth Stage

### Now (MVP / Early Production) → Option A

- 2000 users with moderate concurrent usage
- Single EC2 is sufficient and simplest to operate
- Use managed DynamoDB (not local) even in single-EC2 for durability
- Use ElastiCache Redis for token blacklist durability across restarts

**Instance:** `m5.xlarge` (4 vCPU, 16GB) or `m5.2xlarge` if budget allows headroom.

### Growth Phase (2000+ daily active users, SLA requirements) → Option B

- Add a second EC2 behind ALB for redundancy (not performance)
- Primary goal: **eliminate single point of failure**
- Fix the EventBus / distributed processing issues first
- Implement Redis-based rate limiting and distributed locks

**Instance:** 2–3× `m5.large` (2 vCPU, 8GB each) behind ALB.

### Scale Phase (5000+ users, enterprise SLAs, complex automations) → Option C

- Separate tiers when you observe specific bottlenecks
- Typically: AI tier first (Gemini calls are memory-heavy and slow)
- Then worker tier (automations/drift detection become continuous at scale)
- API tier scales with user count; worker tier scales with resource count

**Instance:** Mix of `t3.large` (API), `t3.medium` (worker), `c5.large` (AI — compute-heavy).

---

## Migration Path: A → B → C

### A → B (Add redundancy)

1. Move DynamoDB to managed AWS DynamoDB (if not already)
2. Move Redis to ElastiCache
3. Add distributed locking for scheduled tasks
4. Set up ALB with health check target `/health`
5. Launch second EC2 with same Docker Compose config
6. Register both instances in ALB target group
7. Configure ALB sticky sessions (optional, for SSE connections)

### B → C (Separate tiers)

1. Implement proper task queue (ARQ with Redis broker)
2. Create dedicated `app/worker.py` entry point for task consumers
3. Create dedicated `app/ai_service.py` with only AI routes
4. Create separate ALB target groups per tier
5. Route `/api/v1/ai/*` to AI tier target group (path-based routing)
6. Workers don't need ALB exposure — they pull from Redis queue
7. Scale each Auto Scaling Group independently

---

## Decision Matrix

| If you need... | Go with |
|----------------|---------|
| Simplicity, low cost, fast time-to-production | **Option A** |
| Zero-downtime deployments | **Option B** (minimum 2 nodes) |
| High availability (survive instance failure) | **Option B** |
| Independent scaling of API vs background work | **Option C** |
| Cost-efficient scaling (only scale what's bottlenecked) | **Option C** |
| Enterprise SLA (99.9%+) | **Option C** + Multi-AZ |
| Air-gapped / on-premises customer deployment | **Option A** (simplest to ship) |

---

## Our Recommendation: Option B Hybrid (Best Balance)

After evaluating all options against our actual workload (2000 users, 10K+ resources, multi-cloud operations), **the recommended approach is a hybrid of Option A and B** — specifically:

**2 EC2 instances behind an ALB, with managed backing services and smart role allocation.**

### Why Not Option A (Single EC2)?

- Single point of failure. One instance going down = total outage.
- Zero-downtime deploys are impossible — you must restart containers on the same host.
- No headroom for traffic spikes (marketing demo, onboarding batch of users, incident response where everyone logs in).
- A runaway automation or Gemini call can degrade the entire platform.

### Why Not Option C (Full Role Separation)?

- Over-engineered for 2000 users and 10K resources.
- Requires rewriting the backend into 3 separate entry points (not ready today).
- Higher operational complexity — 3 Auto Scaling Groups, multiple target groups, per-tier deployments.
- Cost increase doesn't justify the benefit until you're past 5000+ daily active users.
- The current in-process EventBus works fine at this scale; a full task queue migration is a large effort.

### The Recommended Architecture: Option B Hybrid

```
                    ┌───────────────────────┐
                    │     Route 53 (DNS)    │
                    └───────────┬───────────┘
                                ▼
                    ┌───────────────────────┐
                    │    ALB (HTTPS:443)    │
                    │  ACM TLS Certificate  │
                    │  Health check: /health│
                    └───────────┬───────────┘
                    ┌───────────┴───────────┐
                    ▼                       ▼
    ┌───────────────────────┐  ┌───────────────────────┐
    │   EC2 #1 (PRIMARY)    │  │   EC2 #2 (SECONDARY)  │
    │   m5.xlarge (4c/16G)  │  │   m5.large (2c/8G)    │
    │                       │  │                       │
    │ ┌───────────────────┐ │  │ ┌───────────────────┐ │
    │ │ cmp-frontend      │ │  │ │ cmp-frontend      │ │
    │ │ cmp-backend       │ │  │ │ cmp-backend       │ │
    │ │ cmp-worker ★      │ │  │ │ cmp-ai            │ │
    │ │ cmp-ai            │ │  │ │                   │ │
    │ └───────────────────┘ │  │ └───────────────────┘ │
    └───────────────────────┘  └───────────────────────┘
                    │                       │
                    ▼                       ▼
    ┌─────────────────────────────────────────────────┐
    │            Managed AWS Services                  │
    │                                                 │
    │  ┌──────────────┐    ┌────────────────────┐    │
    │  │ ElastiCache  │    │    DynamoDB        │    │
    │  │ Redis (r6g.  │    │   (On-Demand +     │    │
    │  │  small)      │    │    DAX if needed)  │    │
    │  └──────────────┘    └────────────────────┘    │
    │                                                 │
    └─────────────────────────────────────────────────┘
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| Worker runs on **Node 1 only** | Avoids duplicate scheduled task execution without needing distributed locks immediately. Node 1 is the "leader" for background work. |
| AI runs on **both nodes** | Distributes Gemini load; each request is independent, no duplication risk. |
| ALB routes all API traffic to both | Standard round-robin; both nodes handle user requests. |
| Node 1 is larger (`m5.xlarge`) | It carries the worker + full stack. Node 2 is smaller since it only handles API + AI. |
| Managed Redis + DynamoDB | Eliminates data loss on instance failure. Required for multi-node shared state. |

### What To Implement Before Going Multi-Node

These are small, targeted changes (not a full rewrite):

#### 1. Redis-based Rate Limiting (replace in-memory counters)

```python
# core/rate_limit.py — already uses Redis if REDIS_URL is set
# Verify it doesn't fall back to per-process in-memory counters
# All rate limit state must go through shared Redis
```

#### 2. Leader Election for Scheduled Tasks

Only one node should run periodic automations (drift detection, policy scans, scheduled reports). Since we designate Node 1 as the worker node, the simplest approach:

```python
# Environment variable on Node 1 only:
# CMP_ROLE=primary
#
# In the scheduler startup:
if os.getenv("CMP_ROLE") != "primary":
    logger.info("Not primary node — skipping scheduled task registration")
    return
```

For a more robust approach (automatic failover if Node 1 dies):

```python
import redis.asyncio as aioredis

async def acquire_leader_lock(redis: aioredis.Redis, task: str, ttl: int = 120) -> bool:
    """Try to become the leader for a scheduled task. Returns True if acquired."""
    instance_id = os.getenv("HOSTNAME", "unknown")
    return await redis.set(f"cmp:leader:{task}", instance_id, nx=True, ex=ttl)

async def run_scheduled_task(task_name: str, task_fn):
    """Run a scheduled task only if this node holds the leader lock."""
    if await acquire_leader_lock(redis, task_name):
        await task_fn()
    else:
        logger.debug(f"Skipping {task_name} — another node is leader")
```

#### 3. EventBus Cross-Node Awareness

The in-process EventBus is fine for request-scoped events (user creates a resource → emit event → handler runs on same node). No change needed for this.

For broadcast events (system-wide notifications, config changes), add Redis pub/sub:

```python
# Only needed if you have events that ALL nodes must react to
# Example: feature toggle change, license update, tenant config change
async def broadcast_event(event: CmpEvent):
    """Publish event to Redis channel for all nodes to receive."""
    await redis.publish("cmp:events:broadcast", event.json())
```

This is optional for now — most events are request-scoped and don't need cross-node delivery.

#### 4. Session/Auth Considerations

Already handled:
- JWT is stateless — any node can verify.
- Token blacklist uses Redis (shared across nodes).
- Active user tracking uses Redis when `REDIS_URL` is set.

No changes needed.

### Deployment Strategy

```
Rolling Deploy (zero downtime):

1. Pull new images on Node 2
2. ALB: drain Node 2 (stop sending new connections, wait for in-flight to complete)
3. Restart containers on Node 2
4. ALB: Node 2 health check passes → re-register
5. Repeat for Node 1
6. Total downtime: 0
```

### Cost Breakdown (Recommended Setup)

| Component | Spec | Monthly Cost |
|-----------|------|-------------|
| EC2 #1 (primary) | m5.xlarge (4 vCPU, 16GB), On-Demand | ~$140 |
| EC2 #2 (secondary) | m5.large (2 vCPU, 8GB), On-Demand | ~$75 |
| ALB | Application Load Balancer + LCU | ~$25 |
| ElastiCache Redis | cache.r6g.small (single-node) | ~$45 |
| DynamoDB | On-Demand (pay per request) | ~$30–60 |
| Route 53 | Hosted zone + queries | ~$1 |
| ACM Certificate | Free (for ALB) | $0 |
| CloudWatch | Basic monitoring + log groups | ~$15 |
| **Total** | | **~$330–360/mo** |

With Reserved Instances (1-year, no upfront) for EC2: **~$240–270/mo**.

### When to Evolve Beyond This

Move to Option C (role-separated) when you observe **any** of these:

| Signal | What it means | Action |
|--------|--------------|--------|
| API p95 latency > 500ms during automation runs | Worker is starving API of CPU | Separate worker to its own instance |
| AI requests timing out > 10% of the time | Gemini calls need more memory/CPU | Dedicated AI instance with `c5.xlarge` |
| User base exceeds 5000 daily active | API tier needs horizontal scaling | Add more API nodes, separate worker |
| Automation queue depth growing (events processing slower than emitted) | Need dedicated worker capacity | Implement ARQ task queue + dedicated worker fleet |
| Compliance requirement for service isolation | Security/audit mandates | Full tier separation |

---

## Summary

For **2000 users / 10K resources** today:

**Go with Option B Hybrid (2 EC2 nodes behind ALB with managed backing services).**

- It eliminates single point of failure.
- It enables zero-downtime deployments.
- It's operationally simple (same Docker Compose on both nodes, minus the worker on Node 2).
- It costs ~$330/mo — a reasonable cost for a production platform managing 10K cloud resources.
- It requires minimal code changes (leader lock for scheduled tasks + verify Redis-based rate limiting).
- It gives you 12+ months of runway before needing to evolve further.

The migration from Option A (current single EC2) to this setup is straightforward:
1. Provision managed Redis + DynamoDB (~1 day)
2. Add leader election for scheduled tasks (~2 hours of code)
3. Set up ALB + second EC2 (~1 day)
4. Configure rolling deploy pipeline (~half day)

**Total effort: ~3 days to go from single-EC2 to production-grade HA.**

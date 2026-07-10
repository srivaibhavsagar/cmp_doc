# Distributed Lock & Leader Election — Developer Guide

## Overview

The distributed lock service (`backend/app/services/distributed_lock.py`) provides leader election and mutual exclusion for multi-node CMP deployments. It ensures that operations which must not run concurrently across nodes (e.g., scheduled background jobs) are coordinated safely via Redis.

When Redis is unavailable, the service falls back to **single-node mode** — all operations are allowed. This maintains CMP's design principle that Redis unavailability must never break core functionality.

---

## Module Location

```
backend/app/
├── services/
│   └── distributed_lock.py    # Leader election & distributed locking
```

---

## When to Use

Use the distributed lock when:

- A background job must run on **exactly one node** (e.g., the scheduler)
- An operation would cause data corruption or duplication if executed concurrently across multiple instances
- You need leader election for any singleton process in a multi-worker deployment

Do **not** use it for:

- Request-level concurrency (use DynamoDB conditional writes or optimistic locking instead)
- Short-lived locks within a single request (use `asyncio.Lock` locally)

---

## Public API

### `acquire_leader_lock(lock_name, ttl_seconds=60) → bool`

Attempts to acquire a named leader lock. Returns `True` if:
- This node successfully acquired the lock (Redis `SET NX`)
- This node already holds the lock (re-entrant — refreshes TTL)
- Redis is unavailable (single-node fallback)

Returns `False` only if another node holds the lock.

```python
from app.services.distributed_lock import acquire_leader_lock

if await acquire_leader_lock("scheduler", ttl_seconds=60):
    # This node is the leader — run the scheduled jobs
    await run_scheduler_loop()
```

### `renew_leader_lock(lock_name, ttl_seconds=60) → bool`

Refreshes the TTL on a lock this node already holds. Call this periodically (at less than `ttl_seconds` intervals) to prevent the lock from expiring while the leader is still active.

Returns `False` if the lock was lost (another node took over).

```python
from app.services.distributed_lock import renew_leader_lock

# Inside a long-running loop
while running:
    if not await renew_leader_lock("scheduler", ttl_seconds=60):
        log.warning("Lost leader lock — stopping scheduler on this node")
        break
    await do_work()
    await asyncio.sleep(30)  # Renew well before the 60s TTL
```

### `release_leader_lock(lock_name) → None`

Explicitly releases a lock (only if this node holds it). Called during graceful shutdown.

```python
from app.services.distributed_lock import release_leader_lock

# On shutdown
await release_leader_lock("scheduler")
```

### `is_leader(lock_name) → bool`

Read-only check — returns whether this node currently holds the named lock. Does not acquire or modify the lock.

```python
from app.services.distributed_lock import is_leader

if await is_leader("scheduler"):
    # Perform leader-only maintenance
    ...
```

---

## How It Works

### Redis Key Schema

```
cmp:leader:{lock_name}
```

The value stored is the node's unique `NODE_ID`, resolved automatically based on the deployment environment (see [Node ID Resolution](#node-id-resolution) below).

### Lock Lifecycle

1. **Acquire**: `SET cmp:leader:{name} {NODE_ID} NX EX {ttl}` — only succeeds if key doesn't exist
2. **Renew**: Check holder matches `NODE_ID`, then `EXPIRE` to refresh TTL
3. **Release**: Check holder matches `NODE_ID`, then `DEL`
4. **Auto-expire**: If a node crashes without releasing, Redis TTL expires the key so another node can acquire

### Fail-Open Design

All Redis errors return `True` (allow the operation). This ensures:
- Single-node deployments work without Redis
- Redis network blips don't halt background processing
- Worst case: brief overlap of two leaders (acceptable for CMP's idempotent jobs)

---

## Configuration

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `REDIS_URL` | *(none)* | Redis connection string. If unset, distributed locking is disabled (single-node mode). |
| `CMP_NODE_ID` | *(auto-detected)* | Explicit override for the node identifier. If unset, the service auto-detects from the runtime environment (see below). |
| `CMP_THREAD_POOL_SIZE` | `20` | Max threads in the async executor pool. All DynamoDB and Redis operations (including lock acquire/renew) run through this pool via `run_in_executor`. Increase if lock operations are queuing behind DynamoDB calls under high concurrency. |

### Node ID Resolution

The `NODE_ID` is resolved at process startup using the following priority chain. The first match wins:

| Priority | Source | Format | When it applies |
|----------|--------|--------|-----------------|
| 1 | `CMP_NODE_ID` env var | Exact value provided | Explicit override (any environment) |
| 2 | EC2 Instance Metadata (IMDSv2) | `ec2-<instance-id>` | AWS EC2 instances (1s timeout) |
| 3 | Azure IMDS (`vmId`) | `azure-<vm-id-prefix>` | Azure VMs (1s timeout) |
| 4 | `ECS_CONTAINER_METADATA_URI_V4` | `ecs-<task-id-prefix>` | AWS ECS / Fargate tasks |
| 5 | `K_REVISION` env var | `gcp-<revision-name>` | GCP Cloud Run instances |
| 6 | Hostname (12-char hex) | `docker-<container-id>` | Docker containers (hostname = short container ID) |
| 7 | `hostname-PID` | `<hostname>-<pid>` | VMs, bare-metal, local dev |

**Examples:**
- EC2 instance: `ec2-i-0a1b2c3d4e5f67890`
- Azure VM: `azure-a1b2c3d4e5f6`
- ECS task: `ecs-a1b2c3d4e5f6`
- Cloud Run: `gcp-my-service-00042-zxq`
- Docker: `docker-3f2a1b9c8d7e`
- Local dev: `Vaibhavs-MacBook-14325`

The EC2 and Azure checks use their respective Instance Metadata Services with a 1-second timeout, so they won't add startup latency on non-cloud environments. This makes log output and Redis key values immediately identifiable in multi-node deployments without requiring manual `CMP_NODE_ID` configuration.

---

## Integration with the Scheduler

The primary consumer is `services/scheduler.py`. At startup, the scheduler calls `acquire_leader_lock("scheduler")` to ensure only one node runs background loops (resource sync, lifecycle checks, cron jobs, etc.).

The renewal pattern:
```python
# Simplified — actual implementation in scheduler.py
async def scheduler_main():
    if not await acquire_leader_lock("scheduler", ttl_seconds=90):
        log.info("Another node is the scheduler leader — skipping")
        return

    while True:
        # Renew every 30s (well within 90s TTL)
        if not await renew_leader_lock("scheduler", ttl_seconds=90):
            log.warning("Lost scheduler leadership")
            break

        await run_all_scheduled_jobs()
        await asyncio.sleep(30)
```

---

## Adding a New Locked Operation

To protect a new operation with distributed locking:

```python
from app.services.distributed_lock import acquire_leader_lock, release_leader_lock

LOCK_NAME = "my-operation"

async def my_singleton_operation():
    if not await acquire_leader_lock(LOCK_NAME, ttl_seconds=120):
        return  # Another node is handling this

    try:
        await do_work()
    finally:
        await release_leader_lock(LOCK_NAME)
```

For long-running loops, use the renew pattern shown above instead of acquire/release.

---

## Testing

When testing code that uses distributed locks:

- **Unit tests**: The lock returns `True` when Redis is unavailable (no `REDIS_URL`), so tests work without Redis by default.
- **Integration tests**: Set `REDIS_URL` to a test Redis instance to verify multi-node behavior.
- **Mock pattern**:

```python
from unittest.mock import AsyncMock, patch

@patch("app.services.distributed_lock.acquire_leader_lock", new_callable=AsyncMock, return_value=True)
async def test_my_operation_as_leader(mock_lock):
    await my_singleton_operation()
    mock_lock.assert_called_once_with("my-operation", ttl_seconds=120)

@patch("app.services.distributed_lock.acquire_leader_lock", new_callable=AsyncMock, return_value=False)
async def test_my_operation_not_leader(mock_lock):
    await my_singleton_operation()
    # Verify no work was done
```

---

## Key Design Decisions

1. **Fail-open** — Redis outages never block operations. Brief leader overlap is acceptable because CMP's scheduled jobs are idempotent.
2. **TTL-based expiry** — No manual cleanup needed if a node crashes. The lock self-heals after `ttl_seconds`.
3. **Re-entrant** — Calling `acquire_leader_lock` when you already hold it simply refreshes the TTL (no deadlock risk).
4. **No Redlock** — Single Redis instance is sufficient for CMP's consistency requirements. If stronger guarantees are needed in the future, consider upgrading to Redlock with multiple Redis nodes.
5. **Lazy Redis init** — The Redis connection is created on first use, not at import time. This avoids startup failures when Redis is not configured.

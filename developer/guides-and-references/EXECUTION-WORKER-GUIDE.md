# Execution Worker — Developer Guide

## Overview

The execution worker (`backend/app/services/execution_worker.py`) decouples long-running provisioning flows from the API process. Instead of running executions as in-process background tasks (which die on API restarts), executions are placed onto a Redis queue and consumed by a separate worker process.

This ensures:

- **Deploy safety** — API redeploys don't kill in-flight executions
- **Reliability** — Long-running tasks (30+ minutes) complete even if the API restarts
- **Scalability** — Multiple worker processes can run for horizontal scaling
- **Single delivery** — Redis BRPOP guarantees each job is consumed exactly once

---

## Architecture

```
┌─────────────────┐         ┌───────────────────────┐         ┌──────────────────┐
│   API Endpoint  │──LPUSH──│  Redis Queue           │──BRPOP──│  Execution Worker│
│  POST /executions│         │  cmp:execution_queue  │         │  (separate proc) │
└─────────────────┘         └───────────────────────┘         └────────┬─────────┘
                                                                       │
                                                               ┌───────▼────────┐
                                                               │ execute_flow() │
                                                               │ (orchestrator) │
                                                               └───────┬────────┘
                                                                       │
                                                               ┌───────▼────────┐
                                                               │   DynamoDB     │
                                                               │   (results)    │
                                                               └────────────────┘
```

**Queue key:** `cmp:execution_queue`  
**Protocol:** Redis List — LPUSH (enqueue) / BRPOP (dequeue)

---

## Module Location

```
backend/app/services/
└── execution_worker.py     # Worker process + enqueue helper
```

---

## Public API

### `enqueue_execution()`

Called by the API endpoint to push a job onto the Redis queue.

```python
from app.services.execution_worker import enqueue_execution

success = await enqueue_execution(
    execution_id="exec-123",
    flow_id="flow-456",
    form_data={"app_name": "my-app", "region": "us-east-1"},
    credential={"access_key": "...", "secret_key": "..."},
    tenant_id="default",
    catalog_info={"catalog_id": "cat-789", "name": "EC2 Instance"},
    user_details={"user_id": "user-1", "email": "user@example.com"},
    audit_id="audit-abc",
    resume_from_step=None,  # or int for resuming failed executions
)

if not success:
    # Redis unavailable — fall back to BackgroundTask (graceful degradation)
    background_tasks.add_task(execute_flow, ...)
```

**Returns:** `True` if enqueued successfully, `False` if Redis is unavailable.

**Fallback behavior:** When Redis is down, the caller should fall back to the legacy `BackgroundTasks.add_task()` approach. This matches CMP's hard rule that Redis unavailability must not break functionality.

### `start_worker()`

Main worker loop. Call within an async context to start consuming jobs.

```python
from app.services.execution_worker import start_worker

await start_worker()
```

---

## Running the Worker

### As a standalone process

```bash
cd backend
python -m app.services.execution_worker
```

### As a Docker container (recommended for production)

```yaml
# docker-compose.yml
services:
  worker:
    image: ${REGISTRY}/cmp-backend:${VERSION}
    command: ["python", "-m", "app.services.execution_worker"]
    env_file: .env
    environment:
      - REDIS_URL=redis://redis:6379/0
      - WORKER_MAX_CONCURRENT=3
      - CMP_THREAD_POOL_SIZE=20
    depends_on:
      redis:
        condition: service_healthy
    stop_grace_period: 300s   # 5 minutes — let in-flight executions finish
    networks:
      - default
      - cmp-task-net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp/cmp_tasks:/tmp/cmp_tasks
```

---

## Configuration

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `REDIS_URL` | (required) | Redis connection URL |
| `WORKER_MAX_CONCURRENT` | `3` | Maximum parallel executions per worker process |
| `CMP_THREAD_POOL_SIZE` | `20` | Thread pool for DynamoDB boto3 calls |

---

## Concurrency Model

The worker uses an `asyncio.Semaphore` to limit concurrent executions:

- Each job from the queue runs as an `asyncio.Task`
- The semaphore caps at `WORKER_MAX_CONCURRENT` (default 3)
- Additional jobs wait for a semaphore slot before starting
- This prevents memory/CPU exhaustion on single-worker deployments

For horizontal scaling, run multiple worker containers — Redis BRPOP ensures each job goes to exactly one worker.

---

## Graceful Shutdown

The worker handles `SIGTERM` and `SIGINT` gracefully:

1. Stops polling the queue for new jobs
2. Waits for all in-flight executions to complete (no timeout — provisioning must finish)
3. Exits cleanly

This is critical for zero-downtime deployments. Set `stop_grace_period: 300s` (or longer) in Docker Compose to allow in-flight executions to drain before the container is force-killed.

---

## Job Payload Format

Jobs are serialized as JSON on the Redis queue:

```json
{
  "execution_id": "exec-123",
  "flow_id": "flow-456",
  "form_data": {"app_name": "my-app", "region": "us-east-1"},
  "credential": {"access_key": "...", "secret_key": "..."},
  "tenant_id": "default",
  "catalog_info": {"catalog_id": "cat-789", "name": "EC2 Instance"},
  "user_details": {"user_id": "user-1", "email": "user@example.com"},
  "audit_id": "audit-abc",
  "resume_from_step": null
}
```

---

## Error Handling

- If a job fails during `execute_flow()`, the error is logged but the worker continues processing other jobs.
- The execution record in DynamoDB is updated to `FAILED` status by the orchestrator.
- Invalid JSON payloads on the queue are logged and discarded (dead-letter behavior).
- Redis connection errors trigger a 5-second backoff before retrying.

---

## Monitoring

**Healthy worker indicators:**
- Worker process is running and connected to Redis
- Queue depth (`LLEN cmp:execution_queue`) stays near zero under normal load
- Executions transition from `PENDING` → `RUNNING` → `SUCCESS/FAILED` in DynamoDB

**Alerts to configure:**
- Queue depth > 10 for > 5 minutes (worker may be down or overloaded)
- Worker process not running (systemd/Docker health check)
- Execution stuck in `RUNNING` for > 30 minutes without step_log updates

---

## Relationship to Existing Components

| Component | Role |
|-----------|------|
| `api/v1/endpoints/executions.py` | Calls `enqueue_execution()` instead of `background_tasks.add_task()` |
| `services/orchestrator.py` | Contains `execute_flow()` — the actual execution logic (unchanged) |
| `crud/execution.py` | DynamoDB CRUD for execution records (unchanged) |
| `execution_worker.py` | Bridges the queue to the orchestrator |

The worker does not duplicate orchestrator logic — it simply provides a reliable transport layer between the API and the flow execution engine.

---

## Migration from BackgroundTasks

Previously, executions were launched via `BackgroundTasks.add_task(execute_flow, ...)`. The new pattern:

1. API calls `enqueue_execution()` first
2. If it returns `True` → done (worker picks it up)
3. If it returns `False` (Redis down) → fall back to `BackgroundTasks.add_task()` as before

This makes the migration non-breaking and maintains the Redis-is-best-effort principle.

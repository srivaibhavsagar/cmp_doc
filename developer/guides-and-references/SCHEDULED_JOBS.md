# CMP Scheduled Jobs & Auto-Refresh Reference

Complete inventory of all background jobs, scheduled tasks, and auto-refresh mechanisms in the Cloud Management Platform.

---

## Backend Scheduler (Background Loops)

All backend jobs run as asyncio tasks spawned at application startup via `start_scheduler()` in `backend/app/services/scheduler.py`. Each loop runs indefinitely with a configurable interval or at a fixed daily time.

| # | Job Name | Interval | First Run Delay | Service | Description |
|---|----------|----------|-----------------|---------|-------------|
| 0 | **Leader Renewal** | Every **30 seconds** | Immediately | `distributed_lock.renew_leader_lock` | Renews the scheduler leader lock in Redis. Only the node holding the lock executes jobs 1–8. Logs `NODE_ID` at startup for traceability. If renewal fails, the node yields leadership. |
| 1 | **Resource Sync** | Every **15 minutes** | 30s after startup | `sync_service.sync_all_credentials` | Fetches live resources from all connected cloud accounts (AWS, Azure, GCP) and populates the `resource_cache` table. Discovers new resources and removes stale ones. |
| 2 | **Lifecycle Check** | Every **1 hour** | 40s after startup | `lifecycle_manager.check_expiring_resources` | Marks inventory items past their `expire_date` as EXPIRED. Sends 7-day, 3-day, and 1-day advance warning notifications to resource owners. Detects orphan cloud resources. |
| 3 | **Cron Scheduler** | Every **1 minute** | 50s after startup | `cron_scheduler.run_due_scheduled_jobs` | Evaluates cron expressions on all active Scheduled Jobs and triggers catalog executions when a job is due. |
| 4 | **Budget Sync** | Every **10 minutes** | 45s after startup | `budget_sync.sync_all_tenants` | Computes live projected costs from active inventory resources (cloud hourly rates + CMP charges) and updates each budget's `current_spend`. Evaluates alert thresholds and emits events when crossed. |
| 5 | **Lease Expiry** | Daily at **02:00 UTC** | 60s after startup | `lease_scheduler.process_expired_leases` | Destroys resources whose `destroy_date` has arrived. Routes Terraform-provisioned resources to `TerraformEngine.destroy()`. Sends 3-day and 1-day warnings before destruction. Creates audit log entries. Batch limit: 500 records per run. |
| 6 | **Approval SLA** | Every **5 minutes** | 55s after startup | `approval_sla.run_approval_sla_checks` | Enforces SLA deadlines on pending approvals. At 50% elapsed: sends reminder + escalates. At 100% elapsed: marks as TIMED_OUT, notifies all parties, emits event. |
| 7 | **Report Delivery** | Every **1 minute** | 65s after startup | `report_scheduler.run_due_scheduled_reports` | Evaluates scheduled report cron expressions and delivers reports via email (PDF/CSV attachments) when due. |
| 8 | **Cost Snapshot** | Daily at **03:00 UTC** | 70s after startup | `cost_snapshot.run_daily_cost_snapshot` | Snapshots live cloud costs for all active resources. Fetches real-time pricing from cloud APIs, calculates 24-hour cost per resource, writes to `cost_ledger` table for cost analytics. |

### Event-Driven Snapshots

In addition to scheduled background loops, certain cost operations are triggered by user or system actions rather than a timer:

| Trigger | Service | Description |
|---------|---------|-------------|
| **Resource Termination** | `cost_snapshot.snapshot_resource_on_delete` | When a resource is destroyed (manually or via lease expiry), captures a final termination snapshot with the resource's total lifetime cost. Calculates elapsed hours since provisioning × hourly rate and writes a `termination` type entry to the `cost_ledger` table. Ensures cost analytics reflect full spend for deleted resources even between daily snapshots. |

### Leader Election (Multi-Node)

In multi-node deployments, only **one node** should run background scheduler loops to prevent duplicate job executions. The scheduler uses the distributed lock service (`services/distributed_lock.py`) to elect a single leader.

- **Dedicated task:** `_leader_renewal_loop` is the first background task spawned by `start_scheduler()`. It acquires and continuously renews the leader lock.
- Lock name: `"scheduler"`
- TTL: 90 seconds (renewed every 30s)
- Fallback: If Redis is unavailable, all nodes assume leadership (single-node mode)
- Recovery: If the leader crashes, another node acquires the lock after TTL expiry
- **Observability:** At startup, each node logs its `NODE_ID` (from `distributed_lock.NODE_ID`) alongside all job intervals. Use this to identify which node currently holds leadership in aggregated logs.

See [DISTRIBUTED-LOCK-GUIDE.md](./DISTRIBUTED-LOCK-GUIDE.md) for full API reference and usage patterns.

### Configuration Constants

```python
SYNC_INTERVAL_SECONDS = 900           # 15 minutes
LIFECYCLE_INTERVAL_SECONDS = 3600     # 1 hour
CRON_INTERVAL_SECONDS = 60            # 1 minute
BUDGET_SYNC_INTERVAL_SECONDS = 600    # 10 minutes
APPROVAL_SLA_INTERVAL_SECONDS = 300   # 5 minutes
REPORT_DELIVERY_INTERVAL_SECONDS = 60 # 1 minute
LEASE_RUN_HOUR_UTC = 2                # 02:00 UTC
COST_SNAPSHOT_RUN_HOUR_UTC = 3        # 03:00 UTC
INITIAL_STARTUP_DELAY_SECONDS = 30    # Wait for DB tables/seeds
```

---

## Backend In-Memory Caches

| Cache | TTL | Location | Purpose |
|-------|-----|----------|---------|
| **Cost Analytics Summary** | 60 seconds | `services/cost_analytics.py` | Caches aggregated cost summary results to avoid repeated DynamoDB scans |
| **Cloud Pricing** | 1 hour (3600s) | `services/cloud_pricing.py` | Caches hourly rate lookups from AWS/Azure/GCP pricing APIs |
| **SSO State** | 5 minutes (300s) | `services/sso_service.py` (Redis) | Stores OAuth state parameter during SSO login flow |
| **Migration Lock** | 10 minutes (600s) | `services/migration_engine.py` (Redis) | Prevents concurrent schema migrations |
| **Distributed Leader Lock** | 60–90 seconds | `services/distributed_lock.py` (Redis) | Leader election for scheduler and singleton operations across nodes |
| **Token Blacklist** | Token remaining lifetime | `core/token_blacklist.py` (Redis) | Tracks revoked JWT tokens until natural expiry |

---

## Auth Token Lifetimes

| Token | Expiry |
|-------|--------|
| Access Token (JWT) | 30 minutes |
| Password Reset Token | 15 minutes |

---

## Frontend Auto-Refresh Intervals

These are client-side polling timers that automatically re-fetch data from the backend.

| # | Page / Component | Interval | Condition | Description |
|---|-----------------|----------|-----------|-------------|
| 1 | **Live Cost Widget** | Every **60 seconds** | Always (while mounted) | Refreshes live cost projections for active resources |
| 2 | **Resources (list)** | Every **60 seconds** | Always (while mounted) | Reloads resource inventory list |
| 3 | **Resource Detail** | Every **60 seconds** | When auto-refresh toggle is ON (default: on) | Refreshes individual resource details and cloud state |
| 4 | **System Status** | Every **30 seconds** | Always (while mounted) | Reloads system health checks and service status |
| 5 | **Executions (list)** | Every **30 seconds** | Always (inventory lookup) | Refreshes inventory lookup for execution display |
| 6 | **Executions (list)** | Every **1 second** | When any execution is running/pending/awaiting_approval | Polls execution list for real-time status updates |
| 7 | **Execution Detail** | Every **1 second** | When execution is running/pending/awaiting_approval | Polls single execution for step-level progress |
| 8 | **Order Detail** | Every **5 seconds** | When order status is provisioning/approved | Polls order and execution progress during provisioning |
| 9 | **Terraform Workspaces** | Every **5 seconds** | When any workspace is locked or in transitional state (initializing/updating/destroying) | Polls workspace list for state transitions |
| 10 | **Logging & Monitoring** | Every **15 seconds** | When auto-refresh checkbox is checked (default: off) | Refreshes container status list |
| 11 | **Approvals (SLA timer)** | Every **1 second** | Always (countdown ticker) | Updates SLA countdown display in real-time |
| 12 | **Approvals (timeout check)** | Every **15 seconds** | When approval is expired and pending | Checks if server has marked approval as timed_out |

---

## Drift Detection (Per-Workspace)

Drift detection is not a global scheduler loop — it's configured per Terraform workspace.

| Setting | Value |
|---------|-------|
| Minimum interval | 1 hour |
| Maximum interval | 168 hours (7 days) |
| Trigger | `DriftDetectionService.run_scheduled_checks()` evaluates all workspaces with a configured schedule |

Workspaces set `drift_detection_interval_hours` via the API, and the service checks if enough time has elapsed since the last run.

---

## Summary by Frequency

| Frequency | Jobs |
|-----------|------|
| **Every 1 second** | Execution detail polling, execution list polling (active only), approval SLA countdown |
| **Every 5 seconds** | Order detail polling (provisioning), Terraform workspace polling (transitional) |
| **Every 15 seconds** | Container monitoring (opt-in), approval timeout check |
| **Every 30 seconds** | System status refresh, executions inventory lookup, **leader renewal (Redis lock)** |
| **Every 1 minute** | Cron scheduler, report delivery |
| **Every 5 minutes** | Approval SLA enforcement |
| **Every 10 minutes** | Budget spend sync |
| **Every 15 minutes** | Resource sync (cloud discovery) |
| **Every 60 seconds** | Live cost widget, resources list, resource detail, cost analytics cache |
| **Every 1 hour** | Lifecycle check, cloud pricing cache refresh |
| **Daily at 02:00 UTC** | Lease expiry processing |
| **Daily at 03:00 UTC** | Cost snapshot (live costs → ledger) |
| **Per-workspace (1-168h)** | Terraform drift detection |

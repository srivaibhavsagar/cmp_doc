# DynamoDB TTL Planning Guide

Reference document for all DynamoDB tables in the Cloud Management Platform and their TTL (Time-to-Live) strategy.

## Current State

- **Total tables:** 46
- **TTL already enabled:** `in_app_notifications` (attribute: `ttl`, 7-day expiry)
- **Billing mode:** All tables use `PAY_PER_REQUEST`

---

## Complete Table Inventory

### Core / Auth (No TTL — Permanent Data)

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 1 | `users` | PK (HASH only) | User accounts, roles, authentication |
| 2 | `credentials` | PK + SK, GSI1, GSI2 | Encrypted cloud provider credentials |
| 3 | `groups` | PK + SK | User groups, role-based membership |
| 4 | `tenants` | PK + SK | Multi-tenant organization records |
| 5 | `settings` | `id` (HASH only) | Feature toggles, platform config |
| 6 | `sso_config` | PK + SK | SSO providers, group mappings, API tokens |

### Catalog & Provisioning (No TTL — Definition Data)

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 7 | `catalogs` | PK + SK | Service catalog definitions |
| 8 | `catalog_components` | PK + SK | Reusable catalog components |
| 9 | `tasks` | PK + SK | Task definitions (Python, Bash, TS, Go, REST, SQL) |
| 10 | `workflows` | PK + SK | Workflow definitions (ordered task steps) |
| 11 | `workflow_versions` | PK + SK | Workflow version history |
| 12 | `flows` | PK + SK | Catalog → workflow linking |
| 13 | `cost_models` | PK + SK | Cost estimation rules |
| 14 | `resource_action_definitions` | PK + SK | Global resource action definitions |

### Execution & Runtime (TTL Candidates)

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 15 | `executions` | PK + SK, GSI-UserCost, GSI-CatalogCost, GSI-ProviderCost | Execution records, logs, cost tracking |
| 16 | `task_runs` | PK + SK | Individual task execution run records |
| 17 | `shared_context` | PK + SK | Shared runtime variables for tasks |
| 18 | `approvals` | PK + SK | Approval workflow records |

### Shopping Cart & Orders

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 19 | `carts` | PK + SK | Cart items (pre-checkout) |
| 20 | `orders` | PK + SK, GSI-UserOrders | Completed order records |

### Cost & Analytics

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 21 | `cost_ledger` | PK + SK | Daily live cost snapshots from cloud providers |
| 22 | `budgets` | PK + SK | Cloud spend budget rules and alert thresholds |

### Reporting & Audit

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 23 | `user_login_logs` | PK + SK, GSI1 | Login/logout event logs |
| 24 | `audit_logs` | PK + SK, GSI1, GSI2 | System audit trail |
| 25 | `inventory` | PK + SK, GSI1, GSI2 | Provisioned resource lifecycle tracking |
| 26 | `resource_cache` | PK + SK | Cached cloud resources from discovery/sync |
| 27 | `resource_actions` | PK + SK | Resource action execution logs |

### Governance & Policy (No TTL — Permanent Data)

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 28 | `policies` | PK + SK | Policy-as-Code governance rules |
| 29 | `quotas` | PK + SK | Per-user/group/tenant resource quotas |

### Events & Automation

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 30 | `events` | PK + SK, GSI1, GSI2 | Event log (Event-Driven Architecture) |
| 31 | `event_rules` | PK + SK | Automation trigger rules |

### Notifications & Scheduling

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 32 | `in_app_notifications` | PK + SK | Bell notifications (**TTL enabled**) |
| 33 | `scheduled_jobs` | PK + SK | Cron-based scheduled catalog executions |
| 34 | `scheduled_reports` | PK + SK | Report scheduling and delivery config |
| 35 | `insights_dashboards` | PK + SK | Saved analytics dashboards |

### AI

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 36 | `ai_conversations` | PK + SK | Persistent AI chat conversation history |

### Dashboard

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 37 | `AdminMessages` | PK + SK, GSI1 | Admin broadcast messages |
| 38 | `UserFeedback` | PK + SK, GSI1 | User feedback submissions |
| 39 | `UserQuestions` | PK + SK, GSI1 | User support questions |
| 40 | `app_settings` | PK + SK | User favorites, branding config |

### Terraform

| # | Table | Key Schema | Purpose |
|---|-------|-----------|---------|
| 41 | `terraform_templates` | PK + SK, GSI1 | Terraform template definitions |
| 42 | `terraform_modules` | PK + SK | Reusable Terraform modules |
| 43 | `terraform_workspaces` | PK + SK, GSI1, GSI2, GSI3 | Workspace instances |
| 44 | `terraform_state_versions` | PK + SK | State file version history |
| 45 | `terraform_drift_alerts` | PK + SK | Drift detection alerts |
| 46 | `terraform_state_backend_config` | PK + SK | State backend config per tenant |

---

## TTL Recommendations

### Tier 1 — Strong Candidates (Transient / Log Data)

| Table | Suggested TTL | TTL Attribute | Rationale |
|-------|--------------|---------------|-----------|
| `executions` | 30 days | `ttl` | Historical runs; archive before expiry if compliance needed |
| `task_runs` | 30 days | `ttl` | Same lifecycle as executions |
| `approvals` | 30 days | `ttl` | Only approved/rejected records (condition: status=approved,rejected) |
| `orders` | 365 days | `ttl` | Archived after 1 year |
| `user_login_logs` | 365 days | `ttl` | 1 year compliance retention |
| `audit_logs` | 180 days | `ttl` | Compliance-driven retention (SOC2, ISO 27001) |
| `inventory` | 30 days | `ttl` | Destroyed resources only (condition: status=destroyed) |
| `resource_actions` | 30 days | `ttl` | Action execution logs |
| `events` | 30 days | `ttl` | Event log grows fast; keep recent for debugging |
| `ai_conversations` | 30 days | `ttl` | Chat history is transient |
| `AdminMessages` | 30 days | `ttl` | Broadcasts expire naturally |
| `UserFeedback` | 30 days | `ttl` | Archived after 30 days |
| `UserQuestions` | 15 days | `ttl` | TTL starts after resolution (condition: status=resolved) |

### Tier 2 — Future Consideration

| Table | Strategy | Condition |
|-------|----------|-----------|
| `resource_cache` | Could add TTL 1–7 days | All records |
| `carts` | Could add TTL 7–14 days | Abandoned carts |
| `shared_context` | Could add TTL 7–30 days | Post-execution cleanup |
| `workflow_versions` | Keep N latest versions, TTL the rest | Set TTL on versions older than latest 10 |
| `terraform_state_versions` | Keep N latest versions, TTL the rest | Set TTL on versions older than latest 20 |
| `terraform_drift_alerts` | Could add TTL 30–90 days | All resolved alerts |
| `cost_ledger` | Could add TTL 90–365 days | Older snapshots |

### No TTL (Permanent)

These tables contain configuration, definitions, or active-state data that must persist indefinitely:

`users`, `credentials`, `groups`, `tenants`, `settings`, `sso_config`, `catalogs`, `catalog_components`, `tasks`, `workflows`, `flows`, `cost_models`, `resource_action_definitions`, `budgets`, `policies`, `quotas`, `event_rules`, `scheduled_jobs`, `scheduled_reports`, `insights_dashboards`, `app_settings`, `terraform_templates`, `terraform_modules`, `terraform_workspaces`, `terraform_state_backend_config`

---

## Implementation Notes

### Admin UI

A "Data Retention" tab is available under **Administration → Settings** (the last tab). It shows only the tables that support TTL configuration:

- Each table has a toggle (enabled/disabled), retention days input, and condition label
- Admins can adjust retention periods and enable/disable per table
- Saving applies TTL to DynamoDB tables automatically
- Existing items without a `ttl` attribute are filtered at read time using their `created_at` timestamp and the configured retention days (no write/update required)

### Backend Endpoints

- `GET /api/v1/settings/data-retention` — Returns current retention config (admin only)
- `PUT /api/v1/settings/data-retention` — Saves config and applies TTL to DynamoDB (admin only)

### How DynamoDB TTL Works

1. Add a numeric attribute (epoch timestamp in seconds) to items
2. Enable TTL on the table specifying that attribute name
3. DynamoDB automatically deletes items where the attribute value < current time
4. Deletion is eventual (typically within 48 hours of expiry)
5. Expired items still appear in queries until physically deleted — filter with `ttl > current_time` if needed

### TTL Attribute Convention

Use `ttl` as the attribute name across all tables for consistency.

### Setting TTL Value on Write

```python
import time

def set_ttl(days: int) -> int:
    """Return epoch timestamp `days` from now."""
    return int(time.time()) + (days * 86400)

# Example: 90-day TTL
item["ttl"] = set_ttl(90)
```

### Enabling TTL on a Table

```python
dynamodb.meta.client.update_time_to_live(
    TableName="<table_name>",
    TimeToLiveSpecification={
        "Enabled": True,
        "AttributeName": "ttl",
    },
)
```

### Filtering Expired (But Not Yet Deleted) Items

```python
from boto3.dynamodb.conditions import Attr
import time

response = table.query(
    KeyConditionExpression=...,
    FilterExpression=Attr("ttl").not_exists() | Attr("ttl").gt(int(time.time()))
)
```

### Application-Level Filtering for Pre-Existing Records

Records written before TTL stamping was implemented lack a `ttl` attribute. These items won't be auto-deleted by DynamoDB TTL. To ensure consistent retention behavior, the CRUD layer applies application-level filtering at read time:

1. Items **with** a `ttl` attribute: filtered if `ttl <= now_epoch` (standard TTL check for DynamoDB's eventual deletion lag)
2. Items **without** a `ttl` attribute: filtered based on their `created_at` timestamp plus the configured retention days

```python
# Example from crud/execution.py — filtering pre-existing records
from app.services.data_retention import get_retention_days_for_table

retention_days = await get_retention_days_for_table(TABLE_NAME)

for item in items:
    ttl_val = item.get("ttl")
    if ttl_val:
        # Standard: skip if DynamoDB TTL expired but not yet deleted
        if int(ttl_val) <= now_epoch:
            continue
    elif retention_days is not None:
        # Pre-existing record without TTL stamp: apply retention via created_at
        created_at = item.get("created_at")
        if created_at:
            created_dt = datetime.fromisoformat(created_at.replace("Z", "+00:00"))
            expiry_epoch = int(created_dt.timestamp()) + (retention_days * 86400)
            if expiry_epoch <= now_epoch:
                continue
```

This ensures that old records respect the admin-configured retention period even though they were never stamped with a `ttl` attribute. The filtering is applied in the `list_executions()` query function so that API responses and UI displays are consistent with the configured retention policy.

---

## Implementation Priority

1. **Phase 1 (Done):** Admin UI + backend settings API — Data Retention tab under Administration → Settings
2. **Phase 2 (Done):** CRUD layer integration — all 13 tables now stamp `ttl` attribute on writes
3. **Phase 3 (Done):** DynamoDB TTL enablement — `apply_retention_settings()` runs at app startup + on admin save
4. **Phase 4 (Future consideration):** Pre-expiry archival to S3, additional tables (resource_cache, carts, shared_context, etc.)

### Files Modified for TTL Stamping

| CRUD File | Table | Stamp Location |
|-----------|-------|----------------|
| `crud/execution.py` | executions | `create_execution()` |
| `crud/task.py` | task_runs | `create_task_run()` |
| `crud/approval.py` | approvals | `update_approval_status()` (conditional: approved/rejected) |
| `crud/order.py` | orders | `create_order()` |
| `crud/user_login_log.py` | user_login_logs | `create_login_log()` |
| `crud/audit_log.py` | audit_logs | `create_audit_log()` |
| `crud/inventory.py` | inventory | `update_inventory_status()` (conditional: destroyed) |
| `crud/resources.py` | resource_actions | `log_action()` |
| `crud/event.py` | events | `upsert_event()` |
| `crud/ai_conversation.py` | ai_conversations | `create_conversation()` |
| `crud/dashboard.py` | AdminMessages | `create_admin_message()` |
| `crud/dashboard.py` | UserFeedback | `create_feedback()` |
| `crud/dashboard.py` | UserQuestions | `update_question()` (conditional: resolved) |

---

## Inventory Archival Strategy

For `inventory` records: when a resource is destroyed, the record gets a 30-day TTL. However, the full provisioning history (who provisioned what, when, from which catalog) should remain queryable for audits. Recommended approach:

- On destroy, the `audit_logs` table already captures the action (180-day retention)
- The `executions` table keeps the execution record (30-day retention)
- For longer-term archival, implement a scheduled job that exports expiring records to S3 before TTL deletion
- Records can be pulled back from S3 on demand for compliance audits

## Architecture

```
┌─────────────────┐     ┌───────────────────────┐     ┌─────────────┐
│  Admin Settings │────▶│  settings table        │────▶│  DynamoDB   │
│  (Frontend Tab) │     │  id="data_retention"   │     │  TTL config │
└─────────────────┘     └───────────────────────┘     └─────────────┘
                                   │
                                   ▼
                        ┌───────────────────────┐
                        │  In-memory cache       │
                        │  (5-min refresh)       │
                        └───────────────────────┘
                                   │
                   ┌───────────────┴───────────────┐
                   ▼                               ▼
┌─────────────────────────────┐     ┌─────────────────────────────┐
│  CRUD Functions (on write)  │     │  CRUD Functions (on read)   │
│  stamp_ttl()                │     │  filter by ttl OR created_at│
│  stamp_ttl_conditional()    │     │  + retention_days           │
└──────────────┬──────────────┘     └──────────────┬──────────────┘
               ▼                                   ▼
        ┌─────────────┐                    ┌─────────────────┐
        │  DynamoDB   │                    │  Filtered API   │
        │  item.ttl   │                    │  response       │
        └──────┬──────┘                    └─────────────────┘
               ▼
        ┌─────────────┐
        │  DynamoDB   │
        │  auto-delete│
        │  (≤48 hrs)  │
        └─────────────┘
```

---

*Last updated: June 2026*

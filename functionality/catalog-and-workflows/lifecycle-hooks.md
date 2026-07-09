# Lifecycle Hooks

Lifecycle Hooks allow administrators and developers to attach pre/post automation flows to resource lifecycle operations. When a resource action is performed (start, stop, terminate, lease, Day 2 changes), configured hook flows execute automatically — enabling custom notifications, compliance checks, audit logging, or any workflow automation tied to resource events.

## Overview

| Aspect | Detail |
|--------|--------|
| UI Navigation | Sidebar → **Lifecycle Hooks** (under Resources section, requires `resources` feature toggle) |
| API Prefix | `/api/v1/lifecycle-hooks` |
| Required Role | `admin` or `developer` |
| License Gate | None (always available) |
| Hook Behavior | Advisory — hook failures do not prevent the main action; results are tracked as step_log entries |

## Supported Operations

Hooks can be attached to these resource lifecycle events:

| Operation | Description |
|-----------|-------------|
| `start` | VM/resource start |
| `stop` | VM/resource stop |
| `restart` | VM/resource restart |
| `terminate` | VM/resource termination |
| `lease` | Lease assignment to a resource |
| `unlease` | Lease removal from a resource |
| `day2_variable_update` | Terraform variable change (Day 2) |
| `day2_resource_addition` | Adding resources to a Terraform stack |
| `day2_resource_removal` | Removing resources from a Terraform stack |
| `day2_destroy` | Full Terraform stack destruction |

### Custom Operations

In addition to the built-in operations above, you can define **custom operations** when creating a hook. In the UI, select **"+ Custom operation..."** from the operation dropdown and type any operation name. This is useful for:

- Attaching hooks to custom resource actions specific to your organization
- Supporting new resource operations before they are added to the built-in list
- Integrating with external systems that trigger CMP operations via API

Custom operations appear automatically in the operations dropdown for all users in the tenant once a hook references them (via the `/operations` endpoint). The backend accepts any non-empty string as an operation name.

## Phases

Each hook runs in one of two phases:

- **pre** — Executes before the main action. Useful for validation, approval notifications, or audit logging.
- **post** — Executes after the main action completes. Receives the action result (success/failure) in context. Useful for notifications, downstream automation, or cleanup.

## API Endpoints

### List Available Operations

```
GET /api/v1/lifecycle-hooks/operations
```

Returns a sorted list of all available operations for lifecycle hooks. This combines the well-known operations (listed above) with any custom operations found in existing hooks for the tenant. Useful for populating dropdowns in the UI — new resource actions automatically appear once a hook references them.

**Response:** `string[]`

```json
[
  "day2_destroy",
  "day2_resource_addition",
  "day2_resource_removal",
  "day2_variable_update",
  "lease",
  "restart",
  "start",
  "stop",
  "terminate",
  "unlease"
]
```

> Custom operations created via hooks will also appear in this list automatically.

### List Hooks

```
GET /api/v1/lifecycle-hooks
```

Query parameters:
- `operation` (optional) — Filter by operation type (e.g., `stop`)
- `phase` (optional) — Filter by phase (`pre` or `post`)

Returns all lifecycle hooks for the current tenant.

### Create Hook

```
POST /api/v1/lifecycle-hooks
```

Request body:

```json
{
  "operation": "stop",
  "phase": "pre",
  "flow_id": "flow-abc-123",
  "enabled": true,
  "description": "Notify Slack before stopping production VMs",
  "providers": ["aws"],
  "resource_types": ["ec2"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `operation` | string | Yes | A supported operation or any custom operation name |
| `phase` | string | Yes | `pre` or `post` |
| `flow_id` | string | Yes | ID of the automation flow to execute |
| `enabled` | boolean | No | Default `true`. Set `false` to disable without deleting |
| `description` | string | No | Human-readable description |
| `providers` | string[] | No | Filter: only fire for these cloud providers (e.g., `["aws", "azure"]`) |
| `resource_types` | string[] | No | Filter: only fire for these resource types (e.g., `["ec2", "vm"]`) |

### Update Hook

```
PUT /api/v1/lifecycle-hooks/{hook_id}
```

Request body (all fields optional):

```json
{
  "flow_id": "flow-new-456",
  "enabled": false,
  "description": "Updated description",
  "providers": ["aws", "gcp"],
  "resource_types": []
}
```

### Delete Hook

```
DELETE /api/v1/lifecycle-hooks/{hook_id}
```

Returns `204 No Content` on success.

## Hook Context

When a hook flow executes, it receives a context payload as `form_data` with all relevant operation details:

| Field | Available In | Description |
|-------|-------------|-------------|
| `operation` | pre, post | The operation being performed |
| `phase` | pre, post | `pre` or `post` |
| `hook_id` | pre, post | The hook that triggered this execution |
| `resource_id` | pre, post | CMP resource identifier |
| `resource_name` | pre, post | Human-readable resource name |
| `resource_type` | pre, post | Resource type (e.g., `ec2`, `vm`) |
| `instance_id` | pre, post | Cloud instance identifier |
| `cloud_provider` | pre, post | Provider (aws, azure, gcp) |
| `region` | pre, post | Cloud region |
| `user_id` | pre, post | User performing the action |
| `username` | pre, post | Username of the actor |
| `tenant_id` | pre, post | Tenant scope |
| `credential_id` | pre, post | Credential used for the operation |
| `workspace_id` | pre, post | Terraform workspace (Day 2 operations) |
| `template_id` | pre, post | Terraform template (Day 2 operations) |
| `updated_variables` | pre, post | Variable changes (Day 2 variable updates) |
| `destroy_date` | pre, post | Lease expiry date (lease operations) |
| `action_success` | post only | Whether the main action succeeded |
| `action_result` | post only | Result details from the action |
| `action_error` | post only | Error message if action failed |

## Filtering

Hooks support optional filtering so they only fire for matching resources:

- **providers** — If set, the hook only fires when the resource's cloud provider matches one of the listed values.
- **resource_types** — If set, the hook only fires when the resource type matches.

If both filters are empty, the hook fires for all resources on the matching operation/phase.

## Execution Behavior

- Hooks are **advisory** — they do not block or gate the main action. If a pre-hook fails, the resource action still proceeds.
- Hook flows execute **synchronously** — the system awaits each hook's completion before continuing. This enables capturing execution results and tracking them alongside the parent action.
- **Timeout:** Each hook phase (pre or post) has a **30-second execution timeout**. If hooks do not complete within 30 seconds, execution is cancelled and a timeout error step_log entry is recorded. This prevents hanging hooks from blocking resource operations indefinitely.
- Each hook execution creates a tracked execution record (catalog: `_internal:lifecycle-hook:{hook_id}`). These internal executions are **hidden from the top-level Executions list** to reduce noise — they appear as sub-steps within their parent resource action's execution record instead. This provides a cleaner execution history while maintaining full auditability through the parent action's step_logs.
- Hook results are returned as **step_log entries** that can be appended to the parent execution's `step_logs` for unified tracking and visibility.
- If the hook infrastructure itself is unavailable, operations continue without hooks (returns empty step_logs).

### Step Log Format

Each hook execution produces a step_log entry with this structure:

```json
{
  "step_id": "pre_hook_abc12345",
  "step_name": "PRE Hook: My Notification Flow",
  "status": "success",
  "started_at": "2025-01-15T10:30:00+00:00",
  "completed_at": "2025-01-15T10:30:02+00:00",
  "duration_ms": 2000,
  "output": {
    "hook_id": "hook-abc-123",
    "flow_id": "flow-xyz-456",
    "flow_name": "My Notification Flow",
    "execution_id": "exec-789",
    "description": "Notify Slack before stopping",
    "context": {
      "resource_id": "res-001",
      "resource_name": "my-server",
      "provider": "aws",
      "operation": "stop",
      "actor": "user@example.com"
    }
  },
  "sub_steps": [
    {
      "step_id": "task_1",
      "step_name": "Send Slack Notification",
      "status": "success",
      "duration_ms": 850,
      "live_logs": [
        { "timestamp": "2025-01-15T10:30:01Z", "stream": "stdout", "message": "Posting to #ops channel..." },
        { "timestamp": "2025-01-15T10:30:01Z", "stream": "stdout", "message": "Message delivered successfully" }
      ],
      "output": { "channel": "#ops", "ts": "1705312201.000100" },
      "error": null
    }
  ],
  "error": null
}
```

### Nested Flow Tasks (Sub-Steps)

When a lifecycle hook triggers an automation flow, the individual **flow tasks** are captured as `sub_steps` within the parent step_log entry. This provides granular visibility into what each task in the hook flow did.

Each sub-step contains:

| Field | Type | Description |
|-------|------|-------------|
| `step_id` | string | Unique identifier for the flow task |
| `step_name` | string | Human-readable task name |
| `status` | string | Task status: `pending`, `running`, `success`, `failed`, `skipped` |
| `duration_ms` | number | Execution time in milliseconds |
| `live_logs` | array | Real-time log entries with `timestamp`, `stream` (stdout/stderr), and `message` |
| `output` | object | Task output data (if any) |
| `error` | string | Error message if the task failed |

#### Viewing Sub-Steps in the UI

On the **Executions** page, lifecycle hook step_log entries that contain `sub_steps` display a collapsible **"Flow Tasks"** section. Each sub-step shows:

- Status icon and colored badge matching the task status
- Task name and execution duration
- Expandable detail panel with:
  - **Live logs** — terminal-style display with color-coded stdout (green) and stderr (red) entries
  - **Output** — formatted JSON output from the task
  - **Error** — error message highlighted in red (if applicable)

This nested view allows operators to drill down from a high-level hook execution into the individual flow tasks without navigating away from the execution timeline.
```

The `context` field contains the full `form_data` that triggered the hook, including resource details, provider, operation, and actor information. This is useful for debugging hook behavior and for audit trail purposes.

If a hook phase times out (exceeds 30 seconds), a timeout entry is recorded:

```json
{
  "step_id": "pre_hook_timeout",
  "step_name": "PRE Hooks (timed out)",
  "section": "pre_hooks",
  "status": "failed",
  "error": "Hook execution timed out after 30s"
}
```

These entries integrate into the parent action's execution record, providing end-to-end visibility of hook activity within the resource operation timeline.

## Usage Examples

### Notify Slack Before Terminating Any Resource

```json
{
  "operation": "terminate",
  "phase": "pre",
  "flow_id": "flow-slack-notify",
  "description": "Alert #ops channel before any resource termination"
}
```

### Run Compliance Check After Day 2 Variable Changes on AWS

```json
{
  "operation": "day2_variable_update",
  "phase": "post",
  "flow_id": "flow-compliance-scan",
  "providers": ["aws"],
  "description": "Run compliance validation after terraform variable updates"
}
```

### Disable a Hook Temporarily

```
PUT /api/v1/lifecycle-hooks/{hook_id}
```
```json
{
  "enabled": false
}
```

## Integration with Flows

Hook flows are standard CMP automation flows. Any existing flow can be used as a hook target. The flow's tasks have access to the full hook context via `form_data`, enabling conditional logic based on provider, resource type, operation result, etc.

To use a flow as a lifecycle hook:
1. Create or identify an existing automation flow
2. Create a lifecycle hook referencing that flow's ID
3. The flow will automatically execute when the matching operation occurs

## Developer Integration

To add lifecycle hook support to a new resource action in the backend:

```python
from app.services.lifecycle_hooks import run_pre_hooks, run_post_hooks, build_hook_context
import asyncio

# Build context
context = build_hook_context(
    operation="my_operation",
    phase="pre",
    resource_id=resource_id,
    cloud_provider="aws",
    user_id=current_user["user_id"],
    username=current_user["username"],
    tenant_id=tenant_id,
)

# Run pre-hooks with 30s timeout — returns step_log entries
try:
    pre_logs = await asyncio.wait_for(
        run_pre_hooks("my_operation", context, tenant_id),
        timeout=30.0
    )
except asyncio.TimeoutError:
    pre_logs = [{"step_id": "pre_hook_timeout", "step_name": "PRE Hooks (timed out)", "section": "pre_hooks", "status": "failed", "error": "Hook execution timed out after 30s"}]

# ... perform the main action ...

# Update context with result for post-hooks
context.phase = "post"
context.action_success = True
context.action_result = {"instance_id": "i-abc123"}

# Run post-hooks with 30s timeout — returns step_log entries
try:
    post_logs = await asyncio.wait_for(
        run_post_hooks("my_operation", context, tenant_id),
        timeout=30.0
    )
except asyncio.TimeoutError:
    post_logs = [{"step_id": "post_hook_timeout", "step_name": "POST Hooks (timed out)", "section": "post_hooks", "status": "failed", "error": "Hook execution timed out after 30s"}]

# Append hook logs to the parent execution's step_logs for unified tracking
all_step_logs = pre_logs + action_step_logs + post_logs
```

### Return Type

Both `run_pre_hooks` and `run_post_hooks` return `List[Dict[str, Any]]` — a list of step_log entries (one per hook that executed). If no hooks matched or the hook infrastructure was unavailable, an empty list is returned.

This allows the calling code to integrate hook execution results into the parent action's step_logs, giving operators a single timeline view of the full operation including all hook activity.

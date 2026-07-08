# Lifecycle Hooks

Lifecycle Hooks allow administrators and developers to attach pre/post automation flows to resource lifecycle operations. When a resource action is performed (start, stop, terminate, lease, Day 2 changes), configured hook flows execute automatically — enabling custom notifications, compliance checks, audit logging, or any workflow automation tied to resource events.

## Overview

| Aspect | Detail |
|--------|--------|
| API Prefix | `/api/v1/lifecycle-hooks` |
| Required Role | `admin` or `developer` |
| License Gate | None (always available) |
| Hook Behavior | Advisory (non-blocking) — hook failures do not prevent the main action |

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

## Phases

Each hook runs in one of two phases:

- **pre** — Executes before the main action. Useful for validation, approval notifications, or audit logging.
- **post** — Executes after the main action completes. Receives the action result (success/failure) in context. Useful for notifications, downstream automation, or cleanup.

## API Endpoints

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
| `operation` | string | Yes | One of the supported operations |
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
- Hook flows execute **asynchronously** in the background. The main action does not wait for hooks to complete.
- Each hook execution creates a tracked execution record visible in the Executions list (catalog: `lifecycle-hook:{hook_id}`).
- If the hook infrastructure itself is unavailable, operations continue without hooks.

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

# Run pre-hooks (non-blocking)
await run_pre_hooks("my_operation", context, tenant_id)

# ... perform the main action ...

# Update context with result for post-hooks
context.phase = "post"
context.action_success = True
context.action_result = {"instance_id": "i-abc123"}

await run_post_hooks("my_operation", context, tenant_id)
```

# Configuration Management via Chat

> View, modify, and bulk-manage resource configurations through natural language — with preview diffs, rollback support, and audit trails.

---

## Overview

Configuration Management via Chat lets users read and modify cloud resource settings (tags, environment variables, parameters) conversationally through the AI Assistant. Changes are previewed before application, and every modification is recorded for rollback.

This feature is part of the **Conversational Infrastructure Management** suite (Feature 4) and requires the `chat_config_management` feature toggle to be enabled.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| View Config | Retrieve current configuration for any resource |
| Modify Config | Change tags, env vars, or settings via chat |
| Bulk Tagging | Apply tags across multiple resources at once |
| Preview Changes | See a diff before any modification is applied |
| Rollback | Revert a previous configuration change by changeset ID |
| Audit Trail | All changes logged with user, timestamp, and before/after values |

---

## Prerequisites

1. **AI Assistant** enabled (Settings → Feature Toggles → `AI Chatbot`)
2. **Config Management via Chat** enabled (Settings → Feature Toggles → `chat_config_management`)
3. **Role:** Developer or Admin (Users and Readonly roles cannot modify configurations)
4. **Credentials:** Valid cloud credentials with appropriate IAM permissions stored in CMP

---

## How to Use

### View Resource Configuration

```
"Show me the configuration of my payment-service Lambda"
"What tags are on instance i-0abc123?"
"Get the config for my-api-gateway"
```

Returns the resource's current settings, tags, and parameters.

### Modify Configuration

```
"Set the DATABASE_URL env var on payment-service to postgres://new-host:5432/db"
"Add a tag team=platform to instance i-0abc123"
"Change the timeout on my Lambda to 60 seconds"
```

The AI will show a preview of proposed changes and ask for confirmation before applying.

### Bulk Tag Resources

```
"Tag all untagged EC2 instances with team=platform"
"Add environment=production tag to all resources in us-east-1"
```

> **Note:** Bulk tagging requires the **Admin** role.

### Preview Changes (Dry Run)

```
"Preview what would change if I set memory to 1024 on my Lambda"
"Show me the diff before applying"
```

Returns a structured diff without modifying anything.

### Rollback a Change

```
"Rollback the last config change"
"Revert changeset abc-123"
```

Restores the previous configuration state using stored rollback data.

---

## Confirmation Flow

All configuration modifications use a two-step confirmation pattern:

1. **Preview phase** — The AI shows proposed changes, affected resources, and a before/after diff.
2. **Apply phase** — After user confirms, the changes are executed.

Bulk operations always require explicit confirmation regardless of risk level.

---

## Supported Change Types

| Change Type | Description | Example |
|-------------|-------------|---------|
| `set_tag` | Add or update a resource tag | `team=platform` |
| `delete_tag` | Remove a tag from a resource | Remove `temp` tag |
| `set_env_var` | Set an environment variable | `DATABASE_URL=...` |
| `delete_env_var` | Remove an environment variable | Remove `DEBUG` |
| `modify_setting` | Change a resource parameter | Timeout, memory, etc. |

---

## Permissions

| Action | Admin | Developer | User | Readonly |
|--------|-------|-----------|------|----------|
| View configuration | ✓ | ✓ | — | — |
| Modify configuration | ✓ | ✓ | — | — |
| Bulk tag resources | ✓ | — | — | — |
| Preview changes | ✓ | ✓ | — | — |
| Rollback changes | ✓ | ✓ | — | — |

---

## Example Conversation

```
User: "Update the DATABASE_URL env var on my payment-service Lambda"

AI:   "What should the new value be?"

User: "postgres://new-rds.cluster.us-east-1.rds.amazonaws.com:5432/payments"

AI:   "Here's the proposed change:
       Resource: payment-service (Lambda)
       Change: SET_ENV_VAR
         DATABASE_URL: postgres://old-host:5432/payments → postgres://new-rds.cluster...
       
       Confirm to apply?"

User: "Yes"

AI:   "Done. DATABASE_URL updated on payment-service. 
       The change takes effect on next invocation."
```

---

## Audit & Compliance

Every configuration change creates:
- A **changeset record** with full before/after data (stored in DynamoDB)
- An **event** in the event log for automation triggers
- Sensitive values are masked in logs when `sensitive=true` is set

---

## Limitations

- Currently supports AWS resources (EC2, Lambda, RDS, S3)
- Bulk operations limited to tagging; bulk env var changes are not yet supported
- Rollback requires the original changeset to still exist in storage
- Secret rotation is planned but not yet available via chat


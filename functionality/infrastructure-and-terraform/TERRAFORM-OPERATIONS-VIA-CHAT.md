# Terraform Operations via Chat

> Plan, apply, inspect drift, and manage Terraform workspaces through natural language — with human-readable summaries and confirmation-gated applies.

---

## Overview

Terraform Operations via Chat brings Terraform workflow management into the AI Assistant. Instead of switching to a terminal or navigating to the Terraform workspace UI, users can plan, apply, check drift, and manage variables conversationally.

This feature requires the `chat_terraform_ops` feature toggle to be enabled.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Plan | Run `terraform plan` and get a human-readable summary |
| Apply | Execute `terraform apply` with confirmation gating |
| Drift Detection | Check for infrastructure drift against Terraform state |
| Workspace Status | View current state, outputs, and resource counts |
| Variable Management | Set or update Terraform variables (including sensitive) |

---

## Prerequisites

1. **AI Assistant** enabled (Settings → Feature Toggles → `AI Chatbot`)
2. **Terraform Ops via Chat** enabled (Settings → Feature Toggles → `chat_terraform_ops`)
3. **Role:** Developer or Admin
4. **Terraform Workspace:** Must have an existing workspace configured in CMP

---

## How to Use

### Run a Plan

```
"Run terraform plan on my production workspace"
"Plan the staging workspace"
"What changes are pending for workspace ws-123?"
```

The AI runs `terraform plan` and returns a human-readable summary showing:
- Resources to add, change, or destroy
- Key attribute changes highlighted
- Estimated impact assessment

### Apply Changes

```
"Apply the plan for my staging workspace"
"Terraform apply on workspace ws-123"
```

> **Important:** Apply always requires explicit confirmation. The AI will show the plan summary and ask you to confirm before executing.

### Check for Drift

```
"Is there any drift on my production workspace?"
"Check terraform drift for ws-456"
"Show me what's drifted"
```

Returns a summary of resources where actual state differs from Terraform state.

### View Workspace Status

```
"What's the status of my terraform workspace?"
"Show me workspace ws-123 details"
```

Returns current state information, resource counts, last apply time, and outputs.

### Set Variables

```
"Set the instance_count variable to 5 on workspace ws-123"
"Update the db_password variable (sensitive) on production"
```

For sensitive variables, values are stored encrypted and masked in all logs and chat output.

---

## Confirmation Flow

### Plan (no confirmation needed)
Plans are read-only and execute immediately when requested.

### Apply (always requires confirmation)

```
User: "Apply terraform on staging"

AI:   "Here's the plan for workspace staging-infra:
       
       + 1 resource to add (aws_instance.web)
       ~ 2 resources to change (aws_security_group.api, aws_lb.main)
       - 0 resources to destroy
       
       ⚠️ This will modify live infrastructure. Confirm to apply?"

User: "Yes, apply it"

AI:   "Apply started. I'll report the result when complete."
       ...
AI:   "✓ Apply complete. 1 added, 2 changed, 0 destroyed."
```

---

## Variable Management

| Parameter | Description |
|-----------|-------------|
| `workspace_id` | Target workspace identifier |
| `variable_name` | Name of the variable to set |
| `variable_value` | New value for the variable |
| `sensitive` | If true, value is encrypted and masked in logs |

Sensitive variables:
- Never displayed in plain text in chat responses
- Stored encrypted using Fernet encryption
- Masked in audit logs and event history

---

## Permissions

| Action | Admin | Developer | User | Readonly |
|--------|-------|-----------|------|----------|
| Terraform plan | ✓ | ✓ | — | — |
| Terraform apply | ✓ | ✓ | — | — |
| Check drift | ✓ | ✓ | — | — |
| View workspace status | ✓ | ✓ | — | — |
| Set variables | ✓ | ✓ | — | — |
| Set sensitive variables | ✓ | ✓ | — | — |

---

## Example Conversations

### Planning and Applying

```
User: "Plan my staging workspace"

AI:   "Running terraform plan on staging-infra...
       
       Plan complete:
       • + aws_instance.worker (t3.medium, us-west-2)
       • ~ aws_rds_instance.main (instance_class: db.t3.medium → db.r5.large)
       
       2 changes total. No resources will be destroyed.
       Would you like to apply?"

User: "Yes"

AI:   "Applying... ✓ Done. 1 created, 1 modified."
```

### Drift Detection

```
User: "Check drift on production"

AI:   "Checking terraform drift for prod-infra...
       
       ⚠️ 2 resources have drifted:
       
       1. aws_security_group.api
          - ingress rule added manually (port 8080 from 0.0.0.0/0)
       
       2. aws_instance.web-1
          - instance_type changed from t3.medium to t3.large (manual resize)
       
       Run 'terraform apply' to reconcile, or update your .tf files to match."
```

### Setting Variables

```
User: "Set instance_count to 5 on staging workspace"

AI:   "Updated variable 'instance_count' = 5 on workspace staging-infra.
       Run a plan to see the effect of this change."
```

---

## Integration with CMP Terraform Workspaces

This feature builds on CMP's existing Terraform workspace management. Workspaces must be:
- Created and configured in the Terraform Workspaces section of CMP
- Linked to a VCS repository or have inline configuration
- Associated with valid cloud credentials

The chat interface provides a conversational layer over the same operations available in the workspace UI.

---

## Limitations

- Workspace must already exist in CMP (cannot create workspaces via chat yet)
- Apply operations are synchronous — the chat waits for completion
- Large plans (50+ resource changes) are summarized rather than showing full details
- Terraform state locking is respected — concurrent applies will fail gracefully
- Only one apply can run per workspace at a time


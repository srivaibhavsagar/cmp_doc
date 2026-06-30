# Cloud Operations via Chat

> Perform direct cloud infrastructure operations through natural language conversation — resize instances, scale groups, modify security groups, manage DNS, and more — with built-in risk assessment and cost impact estimation.

---

## Overview

Cloud Operations via Chat extends the AI Assistant into an operational control plane. Instead of navigating provider consoles or writing scripts, users issue natural language commands and the AI executes validated cloud operations with appropriate guardrails.

This feature is part of the `ai_assistant` license key (Enterprise tier).

### Implementation Status

Phase 1 tools are registered in the AI chat endpoint and available for use:

| Tool | Status | Since |
|------|--------|-------|
| `cloud_resize_instance` | ✅ Implemented | v8.15.55 |
| `cloud_scale_group` | ✅ Implemented | v8.15.55 |
| `cloud_modify_security_group` | ✅ Implemented | v8.15.55 |
| `cloud_create_snapshot` | ✅ Implemented | v8.15.55 |
| `cloud_modify_database` | ✅ Implemented | v8.15.55 |
| `cloud_update_function` | ✅ Implemented | v8.15.55 |
| `cloud_operation_dry_run` | ✅ Implemented | v8.15.55 |

---

## Supported Operations

| Category | Operations | Providers | Tool Name |
|----------|-----------|-----------|-----------|
| Compute | Resize instance (change type/size) | AWS EC2, Azure VM, GCP Compute | `cloud_resize_instance` |
| Scaling | Modify auto-scaling group capacity (desired, min, max) | AWS ASG, Azure VMSS, GCP MIG | `cloud_scale_group` |
| Security | Add/remove security group or firewall rules | AWS SG, Azure NSG, GCP Firewall | `cloud_modify_security_group` |
| DNS | Create, modify, or delete DNS records | AWS Route53, Azure DNS, GCP Cloud DNS | — (planned) |
| Storage | Modify bucket lifecycle, versioning, access settings | S3, Azure Blob, GCS | — (planned) |
| Serverless | Update function config (memory, timeout, env vars) | Lambda, Azure Functions, Cloud Functions | `cloud_update_function` |
| Snapshots | Create disk or database snapshots | All providers | `cloud_create_snapshot` |
| Database | Resize, modify parameters, create read replicas | RDS, Azure SQL, Cloud SQL | `cloud_modify_database` |
| Dry Run | Preview any operation without applying it | All providers | `cloud_operation_dry_run` |

---

## Risk Assessment

Every cloud operation is automatically assigned a risk level before execution:

| Risk Level | Description | User Experience |
|------------|-------------|-----------------|
| **Low** | Read-only or non-destructive (e.g., create snapshot) | AI confirms and executes |
| **Medium** | Reversible changes (e.g., stop instance, resize, scale) | AI shows impact and asks for confirmation |
| **High** | Destructive or security-impacting (e.g., terminate, modify security groups) | AI requires explicit confirmation with warnings |
| **Critical** | Irreversible data loss (e.g., delete bucket with data) | Requires admin approval before execution |

Each operation receives a numeric risk score (0–100) based on factors such as:
- Whether the operation is reversible
- Whether it affects security boundaries
- Whether it could cause data loss
- The blast radius (single resource vs. group)

Operations exceeding the configurable risk threshold require admin approval via the existing approval workflow.

---

## Cost Impact Estimation

Before applying changes, the AI provides estimated cost impact:

- **Monthly cost before** the change
- **Monthly cost after** the change
- **Monthly delta** (increase or decrease)
- **Plain-language description** of the financial impact

Example:
> "Resizing from t3.medium to t3.xlarge will increase monthly cost from ~$30 to ~$121 (+$91/month)."

---

## Operation Lifecycle

Each cloud operation moves through a defined status flow:

```
planned → confirmed → executing → completed
                                 → failed
         → cancelled
         → approval_required (if risk threshold exceeded)
```

| Status | Meaning |
|--------|---------|
| `planned` | AI has proposed the operation, awaiting user response |
| `confirmed` | User approved the operation |
| `executing` | Operation is in progress |
| `completed` | Operation finished successfully |
| `failed` | Operation encountered an error |
| `cancelled` | User declined the operation |
| `approval_required` | High-risk operation sent for admin approval |

---

## Usage Examples

### Resize an instance

```
User: "Resize my api-server to t3.xlarge"
AI:   "I'll resize api-server (i-0abc123) from t3.medium → t3.xlarge.
       Cost impact: +$91/month. The instance will be stopped during resize.
       Proceed?"
User: "Yes"
AI:   "Done. api-server is now t3.xlarge and running."
```

### Scale an auto-scaling group

```
User: "Scale web-asg to 6 instances"
AI:   "I'll update web-asg desired capacity from 3 → 6.
       Cost impact: +$45/day (~$1,350/month).
       Proceed?"
User: "Go ahead"
AI:   "Done. web-asg desired_capacity=6. Changes propagate in ~2 minutes."
```

### Modify a security group

```
User: "Open port 443 on my api-sg security group"
AI:   "⚠️ This is a HIGH risk operation (modifies network security).
       I'll add an inbound rule: TCP 443 from 0.0.0.0/0 on sg-0def456.
       Confirm?"
User: "Yes, but only from 10.0.0.0/8"
AI:   "Updated plan: TCP 443 from 10.0.0.0/8 only. Proceed?"
User: "Yes"
AI:   "Done. Inbound rule added to api-sg."
```

### Dry-run preview

```
User: "What would happen if I resize my database to db.r5.xlarge?"
AI:   "Here's the dry-run preview for resizing my-prod-db:
       Current:  db.r5.large (2 vCPU, 16 GB RAM, $230/mo)
       Proposed: db.r5.xlarge (4 vCPU, 32 GB RAM, $460/mo)
       Risk: MEDIUM (reversible, but may cause brief downtime if applied immediately)
       Cost impact: +$230/month
       Would you like to proceed with the resize?"
```

### Create a snapshot

```
User: "Take a backup of my payments database"
AI:   "Creating a snapshot of payments-db (db-prod-payments)...
       Done. Snapshot rds:payments-db-2026-06-30 created. It will be available in ~5 minutes."
```

### Update a Lambda function

```
User: "Increase the timeout on my image-processor Lambda to 5 minutes and bump memory to 1024MB"
AI:   "I'll update image-processor:
       • Timeout: 30s → 300s
       • Memory: 512 MB → 1024 MB
       Estimated cost impact: +$4/month (based on current invocation rate).
       Confirm?"
User: "Do it"
AI:   "Done. image-processor updated. Changes take effect on next invocation."
```

---

## Confirmation Flow (Two-Step Execution)

Most operations use a two-step confirmation pattern:

1. **Plan phase** — The AI calls the tool with `confirmed=false` (or omits it). The system returns a preview showing current state, proposed changes, risk level, and cost impact.
2. **Execute phase** — After user confirms, the AI calls the tool again with `confirmed=true` to apply the changes.

This ensures users always see what will happen before any changes are made. The `cloud_operation_dry_run` tool provides an explicit way to preview any operation without risk of execution.

For **low-risk operations** (like creating snapshots), the AI may execute immediately after a brief confirmation since the operation is non-destructive.

For **high-risk operations** (like modifying security groups or production databases), the AI always requires explicit confirmation and displays detailed warnings.

---

## Permissions & Access Control

- Operations are scoped to the user's accessible credentials only — users cannot operate on resources they don't have credentials for.
- Role-based restrictions apply:
 - `readonly` users cannot execute any operations.
 - `user` role cannot execute operations (read-only chat access).
 - `developer` role can execute: resize instance, scale group, create snapshot, modify database, update function.
 - `admin` role can execute all operations, including security group modifications.
- All operations emit audit events (`CLOUD_OPERATION_EXECUTED`) and are logged in the event system.
- Admin approval is required when the operation's risk score exceeds the org-configured threshold.

---

## Admin Configuration

Admins can configure:

| Setting | Description | Default |
|---------|-------------|---------|
| Risk threshold for approval | Operations above this score require admin sign-off | 75 |
| Allowed operation types | Which operation categories are enabled | All |
| Dry-run enforcement | Require dry-run preview before any execution | Off |
| Cost threshold alert | Alert when estimated cost impact exceeds amount | $100/month |

---

## Audit & Compliance

Every cloud operation creates:
- An **event** in the event log (`CLOUD_OPERATION_EXECUTED`, `CLOUD_OPERATION_FAILED`)
- An **audit trail** entry with: who, what, when, risk score, cost impact, and result
- A **conversation record** linking the operation to the chat session that triggered it

These records support compliance reporting and incident investigation.

---

## Related Features

- [AI Assistant](./FEATURE-CATALOG.md#5-ai_assistant--ai-assistant) — The chat interface through which operations are triggered
- [Resource Actions](./FEATURE-CATALOG.md#11-resource_actions--resource-actions) — Day-2 operations via the UI (complementary to chat-based operations)
- [Policy Governance](./FEATURE-CATALOG.md#4-policy_governance--policy-governance) — Policies that may block or require approval for certain operations
- [Event Automation](./FEATURE-CATALOG.md#10-event_automation--event-automation) — Automation rules that react to operation events
- [Scaling & Auto-Scaling](./SCALING-AND-AUTO-SCALING.md) — Dedicated scaling policy management via chat (`chat_scaling`)
- [Configuration Management via Chat](./CONFIG-MANAGEMENT-VIA-CHAT.md) — Tags, env vars, and resource config changes (`chat_config_management`)
- [Proactive AI Recommendations](./PROACTIVE-AI-RECOMMENDATIONS.md) — AI-initiated suggestions and alerts (`chat_proactive_ai`)
- [Guided Conversations (Wizards)](./GUIDED-CONVERSATIONS-WIZARDS.md) — Multi-step guided workflows (`chat_guided_flows`)
- [Infrastructure-as-Conversation](./INFRASTRUCTURE-AS-CONVERSATION.md) — Plan and export infrastructure from natural language (`chat_infra_planning`)
- [Terraform Operations via Chat](./TERRAFORM-OPERATIONS-VIA-CHAT.md) — Plan, apply, drift, and variable management (`chat_terraform_ops`)

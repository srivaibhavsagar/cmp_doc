# CMP Feature Catalog — Licensed Features Reference

> What each licensed feature key includes, what capabilities it unlocks, and which pages/APIs are gated behind it.

---

## Overview

CMP uses feature keys in the license to control what capabilities are available. Each key maps to a set of UI pages, API endpoints, and functionality. This document explains exactly what each feature covers.

---

## Feature Keys Summary

| License Key | Display Name | Tier | What It Covers |
|-------------|-------------|------|----------------|
| `catalog` | Service Catalog | Trial+ | Self-service catalog, templates, provisioning requests |
| `workflows` | Workflow Automation | Trial+ | Tasks, workflows, flows, executions |
| `scheduled_jobs` | Scheduled Jobs | Standard+ | Cron-based recurring automation |
| `terraform` | Terraform Provisioning | Standard+ | Templates, workspaces, drift detection, state backend |
| `cost_management` | Cost Management | Trial+ | Cost models, cloud pricing, cost analytics, live projections |
| `reports` | Reports & Analytics | Standard+ | Platform reports and analytics dashboards |
| `policy_governance` | Policy Governance | Standard+ | Policies, quotas, approvals, compliance rules |
| `budgets` | Budget Management | Standard+ | Budget creation, alerts, threshold notifications |
| `event_automation` | Event Automation | Standard+ | Event-driven rules, triggers, event log |
| `webhooks` | Inbound Webhooks | Standard+ | Webhook endpoint configuration |
| `cloud_accounts` | Cloud Accounts | Trial+ | AWS, Azure, GCP account management (per-provider limits) |
| `multi_tenancy` | Multi-Tenancy | Enterprise | Multiple tenants, tenant switching, isolated environments |
| `sso` | Single Sign-On (SSO) | Enterprise | OIDC/SAML integration, SSO group mappings |
| `ai_assistant` | AI Assistant | Enterprise | AI chatbot, Terraform generation, natural language queries |

---

## Detailed Feature Breakdown

### 1. `catalog` — Service Catalog

**What it is:** A self-service portal where users can browse and request pre-built infrastructure templates.

**UI Pages:**
- `/catalog` — Browse available catalog items
- `/catalog/:id` — View catalog item details
- `/catalog/create` — Create new catalog items (admin/developer)
- `/catalog-components` — Manage reusable components

**API Endpoints:**
- `GET /api/v1/catalog` — List catalog items
- `POST /api/v1/catalog` — Create catalog item
- `GET /api/v1/catalog/:id` — Get catalog item
- `PUT /api/v1/catalog/:id` — Update catalog item
- `DELETE /api/v1/catalog/:id` — Delete catalog item
- `POST /api/v1/catalog/:id/request` — Request provisioning
- `GET /api/v1/catalog-components` — List components

**What users can do:**
- Browse available infrastructure templates (VMs, databases, networks, etc.)
- Request provisioning of catalog items
- View request status and history
- Admins: create/edit/delete catalog items and components

---

### 2. `workflows` — Workflow Automation

**What it is:** The automation engine that powers provisioning and orchestration.

**UI Pages:**
- `/tasks` — Task definitions (individual automation steps)
- `/workflows` — Workflow definitions (multi-step orchestrations)
- `/flows` — Flow designer (visual workflow builder)
- `/executions` — Execution history and monitoring

**API Endpoints:**
- `GET/POST /api/v1/tasks` — Task CRUD
- `GET/POST /api/v1/workflows` — Workflow CRUD
- `GET/POST /api/v1/flows` — Flow CRUD
- `GET /api/v1/executions` — List executions
- `GET /api/v1/executions/:id` — Execution details + logs
- `POST /api/v1/executions/:id/cancel` — Cancel execution

**What users can do:**
- Create automation tasks (shell scripts, Terraform, API calls)
- Build multi-step workflows with conditions and branching
- Design flows visually with drag-and-drop
- Monitor execution progress and view logs

**Sub-features controlled by this key:**
| Sub-feature | What it does |
|-------------|-------------|
| Tasks | Individual automation steps (scripts, API calls) |
| Workflows | Multi-step orchestrations with dependencies |
| Flows | Visual workflow designer |
| Visual DAG Editor | Drag-and-drop DAG canvas for building workflows visually (gated by `visual_dag_editor` toggle) |
| Executions | Run history, logs, status monitoring |

---

### 3. `cost_management` — Cost Management

**What it is:** Tools for tracking, analyzing, and optimizing cloud spending across all providers.

**UI Pages:**
- `/cost-models` — Define cost allocation models
- `/reports` — Cost reports and analytics
- Cloud pricing reference

**API Endpoints:**
- `GET/POST /api/v1/cost-models` — Cost model CRUD
- `GET /api/v1/cost-analytics` — Cost analytics data
- `GET /api/v1/cloud-pricing` — Cloud provider pricing reference
- `GET /api/v1/reports` — Generate cost reports

**What users can do:**
- Define cost models (how costs are allocated to teams/projects)
- View cost breakdowns by provider, service, team, project
- Generate cost reports (daily, weekly, monthly)
- Compare costs across cloud providers
- Identify cost optimization opportunities
- Track spending trends over time

---

### 4. `policy_governance` — Policy Governance

**What it is:** Compliance and governance engine that enforces rules across all cloud operations.

**UI Pages:**
- `/policies` — Policy definitions and management
- `/quotas` — Resource quotas per team/user
- `/approvals` — Approval workflows for sensitive operations

**API Endpoints:**
- `GET/POST /api/v1/policies` — Policy CRUD
- `POST /api/v1/policies/:id/evaluate` — Evaluate a policy
- `GET/POST /api/v1/quotas` — Quota CRUD
- `GET /api/v1/quotas/usage` — Current quota usage
- `GET/POST /api/v1/approvals` — Approval requests
- `POST /api/v1/approvals/:id/approve` — Approve request
- `POST /api/v1/approvals/:id/reject` — Reject request

**What users can do:**
- Define policies (e.g., "no public S3 buckets", "max instance size = m5.xlarge")
- Set resource quotas per team or user
- Require approvals for specific operations (e.g., production deployments)
- View compliance status across all resources
- Get alerts when policies are violated

**Sub-features controlled by this key:**
| Sub-feature | What it does |
|-------------|-------------|
| Policies | Define and enforce compliance rules |
| Quotas | Limit resource usage per team/user |
| Approvals | Multi-level approval workflows |

---

### 5. `ai_assistant` — AI Assistant

**What it is:** AI-powered copilot that helps with cloud operations using natural language, with contextual inline suggestions, smart catalog recommendations, and persistent conversation history.

**UI Components:**
- AI chat panel (⌘K shortcut)
- Inline AI suggestion buttons on execution, resource, catalog, policy, and budget pages
- AI-powered catalog recommendations on the catalog page
- Conversation history panel with resumable sessions

**API Endpoints:**
- `POST /api/v1/chat` — Send message to AI assistant
- `POST /api/v1/ai/inline-suggestions` — Get contextual AI action buttons for current UI state
- `POST /api/v1/ai/catalog-recommendations` — Get personalized catalog suggestions based on usage patterns
- `GET /api/v1/ai/conversations` — List user's saved conversations
- `POST /api/v1/ai/conversations` — Create a new conversation
- `GET /api/v1/ai/conversations/:id` — Get conversation with full message history
- `POST /api/v1/ai/conversations/:id/messages` — Append messages to a conversation
- `PATCH /api/v1/ai/conversations/:id` — Rename a conversation
- `DELETE /api/v1/ai/conversations/:id` — Delete a conversation
- `DELETE /api/v1/ai/conversations` — Clear all conversations

**What users can do:**
- Ask questions in natural language ("How much did we spend on EC2 last month?")
- Generate Terraform code from descriptions ("Create a VPC with 3 subnets")
- Get explanations of errors and resources
- Get optimization recommendations
- Troubleshoot issues with AI guidance
- Click inline AI buttons on failed executions ("Explain this error", "Suggest a fix")
- See personalized catalog recommendations based on their usage history and intent
- Resume past AI conversations from any previous session
- Reference previous AI answers without re-asking questions

**Powered by:** Google Gemini 2.5 Flash Lite (configurable via admin UI or `GEMINI_API_KEY`)

---

### 6. `sso` — Single Sign-On (SSO)

**What it is:** Enterprise identity provider integration for centralized authentication.

**UI Pages:**
- `/admin/sso/config` — SSO configuration (admin)
- `/admin/sso/group-mappings` — Map IdP groups to CMP roles

**API Endpoints:**
- `GET/POST /api/v1/auth/sso/config` — SSO configuration
- `GET/POST /api/v1/auth/sso/group-mappings` — Group mapping CRUD
- `GET /api/v1/auth/sso/providers` — List configured providers
- `POST /api/v1/auth/sso/saml/callback` — SAML assertion consumer

**What users can do:**
- Login with corporate credentials (no separate CMP password)
- Automatic role assignment based on IdP group membership
- Support for SAML 2.0 and OIDC providers
- Multiple IdP support

**Supported providers:**
- Azure Active Directory (Entra ID)
- Okta
- OneLogin
- Google Workspace
- Any SAML 2.0 / OIDC compliant IdP

---

### 7. `multi_tenancy` — Multi-Tenancy

**What it is:** Isolated environments for different teams, departments, or customers within one CMP instance.

**UI Pages:**
- `/tenants` — Tenant management (admin)
- Tenant switcher in the top navigation bar

**API Endpoints:**
- `GET/POST /api/v1/tenants` — Tenant CRUD
- `POST /api/v1/auth/switch-tenant` — Switch active tenant
- All data endpoints are automatically scoped to the active tenant

**What users can do:**
- Create isolated tenant environments
- Switch between tenants (if user has access to multiple)
- Each tenant has its own: resources, credentials, catalog, workflows, policies, budgets
- Custom branding per tenant (logo, colors, company name)
- Separate user management per tenant

---

### 8. `budgets` — Budget Management

**What it is:** Financial controls for cloud spending with alerts and thresholds.

**UI Pages:**
- `/budgets` — Budget definitions and status

**API Endpoints:**
- `GET/POST /api/v1/budgets` — Budget CRUD
- `GET /api/v1/budgets/:id/status` — Budget utilization
- `GET /api/v1/budgets/alerts` — Active budget alerts

**What users can do:**
- Create budgets (monthly, quarterly, annual)
- Set alert thresholds (e.g., notify at 80%, block at 100%)
- Assign budgets to teams, projects, or cloud accounts
- View real-time budget utilization
- Get notifications when thresholds are crossed
- Integrate with approval workflows (require approval when budget exceeded)

---

### 9. `scheduled_jobs` — Scheduled Jobs

**What it is:** Cron-based automation for recurring operational tasks.

**UI Pages:**
- `/scheduled-jobs` — Job definitions and execution history

**API Endpoints:**
- `GET/POST /api/v1/scheduled-jobs` — Job CRUD
- `GET /api/v1/scheduled-jobs/:id/history` — Execution history
- `POST /api/v1/scheduled-jobs/:id/trigger` — Manual trigger
- `POST /api/v1/scheduled-jobs/:id/pause` — Pause job
- `POST /api/v1/scheduled-jobs/:id/resume` — Resume job

**What users can do:**
- Schedule recurring tasks (e.g., "stop dev instances every night at 8pm")
- Define cron expressions for timing
- View execution history and logs
- Pause/resume jobs
- Get notifications on job failures
- Link to workflows for complex scheduled operations

---

### 10. `event_automation` — Event Automation

**What it is:** Event-driven automation that triggers actions when specific events occur in the system (resource changes, budget thresholds, policy violations, etc.).

**UI Pages:**
- `/events` — Event automation rules (admin only)

**API Endpoints:**
- `GET/POST /api/v1/events/rules` — Event rule CRUD
- `GET /api/v1/events/rules/:id` — Get rule details
- `PUT /api/v1/events/rules/:id` — Update rule
- `DELETE /api/v1/events/rules/:id` — Delete rule
- `GET /api/v1/events/log` — Event log (history of triggered events)

**What users can do:**
- Define event rules (e.g., "when a resource is created, notify Slack")
- Set conditions and filters (e.g., "only for production resources")
- Trigger workflows automatically on events
- View event history and triggered actions
- Integrate with notification channels (Slack, Teams, email, webhooks)
- Authenticate outbound HTTP calls using Bearer tokens, Basic auth, API keys, or CMP-managed credentials

**Example event triggers:**
- Resource created/deleted/modified
- Budget threshold exceeded
- Policy violation detected
- User login from new location
- Scheduled job failed
- Approval requested/approved/rejected

**Sold separately from workflows** — a customer can have event automation without the full workflow engine, or vice versa.

---

### 11. `resource_actions` — Resource Actions

**What it is:** Day-2 operations on provisioned resources — start, stop, resize, terminate, install agent, and custom actions defined by admins.

**UI Pages:**
- `/resource-actions` — Define and manage resource actions (admin/developer)
- Resource action buttons appear on `/resources/:id` detail page

**API Endpoints:**
- `GET/POST /api/v1/resource-actions` — Resource action CRUD
- `GET /api/v1/resource-actions/:id` — Get action details
- `PUT /api/v1/resource-actions/:id` — Update action
- `DELETE /api/v1/resource-actions/:id` — Delete action
- `POST /api/v1/resource-actions/:id/execute` — Execute an action on a resource

**What users can do:**
- Define custom actions for resource types (e.g., "Restart", "Scale Up", "Take Snapshot")
- Execute day-2 operations on provisioned resources
- Set approval requirements for dangerous actions (requires `policy_governance`)
- Attach cost estimates to actions (requires `cost_management`)
- View execution history for actions

**Events emitted:**
- `resource.action.started` — When a resource action execution begins
- `resource.action.completed` — When a resource action finishes successfully
- `resource.action.failed` — When a resource action fails

These events can be used with automation rules (requires `event_automation`) to trigger notifications, webhooks, or remediation workflows on action completion or failure.

**Cross-feature dependencies:**
- Approval settings within resource actions require `policy_governance` license
- Cost estimate toggles within resource actions require `cost_management` license

---

## License Tiers — What's Included

### Trial (30 days, free)
| Feature | Included |
|---------|----------|
| `catalog` | ✓ |
| `workflows` | ✓ |
| `cost_management` | ✓ |
| `cloud_accounts` | ✓ (AWS: 1 only) |
| `scheduled_jobs` | ✗ |
| `terraform` | ✗ |
| `reports` | ✗ |
| `policy_governance` | ✗ |
| `budgets` | ✗ |
| `event_automation` | ✗ |
| `webhooks` | ✗ |
| `multi_tenancy` | ✗ |
| `sso` | ✗ |
| `ai_assistant` | ✗ |

### Standard
| Feature | Included |
|---------|----------|
| `catalog` | ✓ |
| `workflows` | ✓ |
| `scheduled_jobs` | ✓ |
| `terraform` | ✓ |
| `cost_management` | ✓ |
| `reports` | ✓ |
| `policy_governance` | ✓ |
| `budgets` | ✓ |
| `event_automation` | ✓ |
| `webhooks` | ✓ |
| `cloud_accounts` | ✓ (AWS: 3, Azure: 2, GCP: 2) |
| `multi_tenancy` | ✗ |
| `sso` | ✗ |
| `ai_assistant` | ✗ |

### Enterprise (all features)
| Feature | Included |
|---------|----------|
| `catalog` | ✓ |
| `workflows` | ✓ |
| `scheduled_jobs` | ✓ |
| `terraform` | ✓ |
| `cost_management` | ✓ |
| `reports` | ✓ |
| `policy_governance` | ✓ |
| `budgets` | ✓ |
| `event_automation` | ✓ |
| `webhooks` | ✓ |
| `cloud_accounts` | ✓ (AWS: 10, Azure: 10, GCP: 10) |
| `multi_tenancy` | ✓ |
| `sso` | ✓ |
| `ai_assistant` | ✓ |

---

## How Feature Gating Works

### Two-Layer System

```
Feature Accessible = License Key Present AND Admin Toggle Enabled
```

1. **License layer:** The license file defines which feature keys are available
2. **Admin toggle layer:** Admins can additionally disable licensed features via Settings → Feature Toggles

Both must be true for a feature to be accessible.

### What Happens When a Feature is NOT Licensed

- **Frontend:** Feature is hidden from navigation (user never sees it)
- **API:** Returns HTTP 403: `"This feature requires a license upgrade. Contact your administrator."`
- **Feature toggle endpoint:** Reports the feature as `disabled`

### What Happens When a Feature IS Licensed but Admin-Disabled

- **Frontend:** Feature is hidden from navigation
- **API:** Returns HTTP 403 (same as unlicensed)
- **Difference:** Admin can re-enable it without a license change

---

## Always Available (Not Gated by License)

These features are always available regardless of license:

| Feature | Why |
|---------|-----|
| Dashboard | Core navigation |
| User management | Required for basic operation |
| Login/logout | Authentication |
| Profile | User self-service |
| Health endpoints | Monitoring |
| Resource inventory | Core functionality (viewing resources) |
| Resource actions | Day-2 operations on existing resources |
| Notifications | System alerts |
| Logging & Monitoring | Platform observability |

---

## For Vendors: How to Add a New Licensed Feature

1. Add the feature key to `backend/app/models/enterprise_config.py` → `SECRET_KEYS` or relevant list
2. Add the feature toggle in `backend/app/models/toggle_settings.py`
3. Gate the API endpoint with `Depends(require_licensed_feature("new_feature_key"))`
4. Gate the frontend nav item with `isFeatureEnabled('new_feature_key')`
5. Add to this document
6. Add to the license generation script's feature list
7. Update tier definitions (which tiers include it)

---

## For Customers: How to Check Your Licensed Features

```bash
# Via API
curl -s -H "Authorization: Bearer <token>" \
  https://your-cmp.com/api/v1/license/status | jq .features_enabled

# Via UI
Login as admin → Administration → System → System Status → Licensed Features section
```


License feature keys vs toggle coverage:

All 17 license keys in license.json are referenced by at least one toggle:

catalog ← service_catalog
workflows ← executions, provisioning, visual_dag_editor
cost_management ← cost_models, cost_analytics, live_cost_projections
policy_governance ← approvals, policies, quotas, audit_logs, compliance_dashboard
multi_tenancy ← tenants
budgets ← budgets
scheduled_jobs ← scheduled_jobs
event_automation ← event_automation, event_log
terraform ← terraform_provisioning
webhooks ← webhooks
reports ← reports
cloud_accounts ← credentials, resources
sso ← sso
ai_assistant ← chatbot, ai_inline_suggestions, ai_catalog_recommendations, ai_conversation_history
notifications ← notifications
infrastructure ← infrastructure
logging_monitoring ← logging_monitoring
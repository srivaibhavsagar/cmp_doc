# CMP License & Feature Gating Reference

## Overview

CMP uses a license-based feature gating system. Each license file contains:
- A list of **license keys** that control access to platform features
- **Cloud provider limits** that control which clouds are enabled and how many accounts can be added
- **Usage limits** for users and managed resources

Features not included in the license are:
- Hidden from the UI (navigation items removed)
- Locked in the admin Feature Toggles panel (cannot be enabled)
- Blocked at the API level (returns HTTP 403)

---

## All License Keys

| License Key | Description | API Endpoints Gated |
|---|---|---|
| `catalog` | Service Catalog | `/catalog`, `/catalog-components` |
| `workflows` | Workflow Automation | `/tasks`, `/workflows`, `/flows`, `/executions` |
| `scheduled_jobs` | Scheduled Jobs | `/scheduled-jobs` |
| `terraform` | Terraform Provisioning | `/terraform/*` (templates, modules, workspaces, drift, state-backend) |
| `cost_management` | Cost Management | `/cost-models`, `/cloud-pricing`, `/cost-analytics` |
| `reports` | Reports & Analytics | `/reports` |
| `policy_governance` | Policy & Governance | `/policies`, `/quotas`, `/approvals` |
| `multi_tenancy` | Multi-Tenancy | `/tenants` |
| `budgets` | Budget Management | `/budgets` |
| `event_automation` | Event Automation + Event Log | `/events`, `/event-log` |
| `webhooks` | Inbound Webhooks | `/webhooks` |
| `cloud_accounts` | Cloud Account Management | `/cloud` (credential CRUD — per-provider limits enforced), `/credentials`, `/resources` |
| `sso` | Single Sign-On | `/auth/sso/*` (login, callback, config, group-mappings) |
| `ai_assistant` | AI Assistant | `/chat` |
| `notifications` | Notifications | `/notifications` |
| `infrastructure` | Infrastructure Management | `/resource-actions`, `/context` (shared variables) |
| `logging_monitoring` | Logging & Monitoring | `/admin/logging` |

---

## Cloud Provider Limits

The license includes a `cloud_providers` section that controls:
- **Which cloud providers are enabled** (0 = disabled)
- **How many accounts can be added per provider**

```json
"cloud_providers": {
  "aws": 2,
  "azure": 2,
  "gcp": 1
}
```

| Provider | Limit = 0 | Limit > 0 |
|---|---|---|
| AWS | Cannot add AWS accounts | Can add up to N AWS accounts |
| Azure | Cannot add Azure accounts | Can add up to N Azure accounts |
| GCP | Cannot add GCP accounts | Can add up to N GCP accounts |

Enforcement happens at credential creation time. Existing accounts are not removed if the license is downgraded.

---

## Feature → License Key Mapping

### Licensed Features (require a license key)

| Feature (Toggle Key) | UI Label | License Key Required |
|---|---|---|
| `service_catalog` | Service Catalog | `catalog` |
| `executions` | Executions | `workflows` |
| `approvals` | Approvals | `policy_governance` |
| `provisioning` | Provisioning | `workflows` |
| `scheduled_jobs` | Scheduled Jobs | `scheduled_jobs` |
| `terraform_provisioning` | Terraform Provisioning | `terraform` |
| `infrastructure` | Infrastructure | `infrastructure` |
| `event_automation` | Event Automation | `event_automation` |
| `event_log` | Event Log | `event_automation` |
| `reports` | Reports & Analytics | `reports` |
| `cost_models` | Cost Models | `cost_management` |
| `cost_analytics` | Cost Analytics | `cost_management` |
| `live_cost_projections` | Live Cost Projections | `cost_management` |
| `budgets` | Budgets | `budgets` |
| `policies` | Policies | `policy_governance` |
| `quotas` | Quotas | `policy_governance` |
| `tenants` | Tenants | `multi_tenancy` |
| `webhooks` | Inbound Webhooks | `webhooks` |
| `credentials` | Accounts & Credentials | `cloud_accounts` |
| `resources` | Resources | `cloud_accounts` |
| `sso` | Single Sign-On (SSO) | `sso` |
| `chatbot` | AI Chatbot | `ai_assistant` |
| `notifications` | Notifications | `notifications` |
| `logging_monitoring` | Logging & Monitoring | `logging_monitoring` |

### Core Features (always available, no license key needed)

| Feature (Toggle Key) | UI Label | Notes |
|---|---|---|
| `self_registration` | Self Registration | User self-registration on login page |
| `forgot_password` | Forgot Password | Password reset flow |

---

## License Plans

### Starter Plan

**Target:** Small teams getting started with cloud management.

| | |
|---|---|
| **License Keys** | `catalog`, `workflows`, `cloud_accounts`, `notifications`, `infrastructure` |
| **Cloud Providers** | AWS: 1, Azure: 0, GCP: 0 |
| **Max Users** | 5 |
| **Max Managed Resources** | 50 |
| **Support Tier** | Basic |

**Included Features:**
- Service Catalog (browse & order)
- Workflow Automation (tasks, workflows, flows, executions)
- Cloud Accounts (AWS only, 1 account)
- Accounts & Credentials, Resources
- Infrastructure (resource actions, shared variables)
- Notifications
- All core features (self-registration, forgot password)

**Not Included:**
- Scheduled Jobs, Terraform, Reports, Cost Management
- Policy & Governance, Multi-Tenancy, Budgets
- Event Automation, Webhooks, SSO, AI Assistant
- Logging & Monitoring
- Azure / GCP accounts

---

### Professional Plan

**Target:** Growing teams needing multi-cloud, cost visibility, and governance.

| | |
|---|---|
| **License Keys** | `catalog`, `workflows`, `scheduled_jobs`, `terraform`, `cost_management`, `reports`, `policy_governance`, `budgets`, `event_automation`, `webhooks`, `cloud_accounts`, `notifications`, `infrastructure`, `logging_monitoring` |
| **Cloud Providers** | AWS: 3, Azure: 2, GCP: 2 |
| **Max Users** | 25 |
| **Max Managed Resources** | 500 |
| **Support Tier** | Standard |

**Included Features:**
- Everything in Starter, plus:
- Scheduled Jobs
- Terraform Provisioning (templates, workspaces, drift, state backend)
- Cost Management (cost models, cloud pricing, cost analytics, live projections)
- Reports & Analytics
- Policy & Governance (policies, quotas, approvals)
- Budget Management
- Event Automation + Event Log
- Inbound Webhooks
- Logging & Monitoring
- Multi-cloud (AWS, Azure, GCP)

**Not Included:**
- Multi-Tenancy, SSO, AI Assistant

---

### Enterprise Plan

**Target:** Large organizations with multi-tenant, SSO, and AI requirements.

| | |
|---|---|
| **License Keys** | All keys |
| **Cloud Providers** | AWS: 10, Azure: 10, GCP: 10 |
| **Max Users** | 999 (or custom) |
| **Max Managed Resources** | 99,999 (or custom) |
| **Support Tier** | Premium |

**Included Features:**
- Everything in Professional, plus:
- Multi-Tenancy (tenant management, tenant switching)
- Single Sign-On (Microsoft, Google, Okta, AWS Identity Center, Ping Identity)
- AI Assistant (Gemini-powered chatbot)
- Unlimited cloud accounts (up to 10 per provider)

---

## How Gating Works (Four Layers)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: API Router (backend enforcement)                          │
│  require_licensed_feature("key") → HTTP 403 if not licensed         │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 2: Feature Toggles (/feature-toggles/me)                     │
│  Returns false for unlicensed features → frontend hides UI          │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 3: Admin Panel (known-features endpoint)                     │
│  Returns licensed:false → toggle switch disabled/locked             │
│  PUT /menu-toggles rejects enabling unlicensed features             │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 4: Cloud Provider Limits (credential creation)               │
│  Checks provider enabled + account count on POST /cloud/            │
└─────────────────────────────────────────────────────────────────────┘
```

**Dev mode behavior:** When no valid license is loaded (local development), all features are allowed and cloud provider limits are not enforced.

---

## License File Structure

```json
{
  "version": "1.0",
  "license_id": "lic_xxx",
  "customer": { "name": "...", "id": "..." },
  "type": "enterprise|standard|trial",
  "environment": "saas|on-premises|hybrid|air-gapped",
  "issued_at": "2026-01-01T00:00:00Z",
  "expires_at": "2027-01-01T00:00:00Z",
  "limits": {
    "max_users": 100,
    "max_managed_resources": 1000
  },
  "cloud_providers": {
    "aws": 5,
    "azure": 3,
    "gcp": 2
  },
  "features": ["catalog", "workflows", "..."],
  "permitted_versions": { "min": "1.0.0", "max": "2.99.99" },
  "support_tier": "premium|standard|basic",
  "deployment_model": "saas|on-premises|hybrid|air-gapped",
  "vendor_verification": { "enabled": true, "endpoint": "https://..." },
  "signature": "<RSA 4096-bit signature>"
}
```

---

## Adding a New Licensed Feature

1. Choose or create a license key (e.g., `my_feature`)
2. Add `require_licensed_feature("my_feature")` to the router in `api/v1/api.py`
3. Add the toggle key → license key mapping in `TOGGLE_TO_LICENSE_KEY` in `models/toggle_settings.py`
4. Add the feature to `KNOWN_FEATURES` if it should appear in the admin toggle panel
5. Include the license key in customer license files as needed
6. Update this document

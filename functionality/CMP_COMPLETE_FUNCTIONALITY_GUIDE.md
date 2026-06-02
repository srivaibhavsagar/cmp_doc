# Cloud Management Platform (CMP) — Complete Functionality Guide

**Version:** 0.2.0 (Alpha)  
**Last Updated:** May 21, 2026  
**Platform:** Multi-tenant, Cloud-Agnostic Resource Management

---

## Table of Contents

1. [Platform Overview](#1-platform-overview)
2. [Architecture & Technical Design](#2-architecture--technical-design)
3. [Authentication & Identity](#3-authentication--identity)
4. [Role-Based Access Control (RBAC)](#4-role-based-access-control-rbac)
5. [Navigation & UI Structure](#5-navigation--ui-structure)
6. [Dashboard](#6-dashboard)
7. [Service Catalog](#7-service-catalog)
8. [Provisioning Engine](#8-provisioning-engine)
9. [Cloud Infrastructure Management](#9-cloud-infrastructure-management)
10. [Terraform Provisioning](#10-terraform-provisioning)
11. [Governance & Compliance](#11-governance--compliance)
12. [Cost Management & Finance](#12-cost-management--finance)
13. [Event-Driven Automation](#13-event-driven-automation)
14. [Administration](#14-administration)
15. [AI Assistant](#15-ai-assistant)
16. [Notifications & Channels](#16-notifications--channels)
17. [Feature Toggle System](#17-feature-toggle-system)
18. [Multi-Tenancy](#18-multi-tenancy)
19. [API Reference Summary](#19-api-reference-summary)
20. [Role-Based Usage Guide](#20-role-based-usage-guide)

---

## 1. Platform Overview

The Cloud Management Platform (CMP) is an enterprise-grade, multi-tenant platform that enables organizations to provision, manage, monitor, and govern cloud resources across AWS, Azure, and GCP from a single unified interface.

### Core Capabilities

| Capability | Description |
|-----------|-------------|
| **Self-Service Catalog** | Users order pre-approved cloud services through a form-driven catalog |
| **Workflow Automation** | Multi-step provisioning workflows with DAG-based orchestration |
| **Multi-Cloud Support** | AWS, Azure, GCP with unified resource management |
| **RBAC & Governance** | Role hierarchy, policies, quotas, and approval workflows |
| **Cost Management** | Cost models, budgets, live pricing, and analytics |
| **Terraform IaC** | Template management, workspaces, Day 2 operations, drift detection |
| **Event-Driven Automation** | 60+ event types with configurable automation rules |
| **Multi-Tenancy** | Complete tenant isolation with per-tenant branding and configuration |
| **SSO Integration** | Microsoft, Google, Okta, AWS Identity Center, Ping Identity |
| **AI Assistant** | Gemini-powered chatbot for platform guidance |

### Tech Stack

| Layer | Technologies |
|-------|-------------|
| **Backend** | Python 3.12, FastAPI, Uvicorn, DynamoDB (single-table design), Redis, Celery |
| **Frontend** | React 18, TypeScript, Vite, Tailwind CSS, React Router v6, Axios, Recharts |
| **Auth** | JWT (python-jose, HS256), bcrypt, Fernet encryption |
| **Cloud SDKs** | boto3 (AWS), azure-sdk (Azure), google-cloud (GCP) |
| **AI** | google-generativeai (Gemini 2.5 Flash Lite) |
| **Infrastructure** | Docker Compose (local), GitHub Actions CI/CD |

---

## 2. Architecture & Technical Design

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Frontend (React 18 + TypeScript)              │
│         Vite Build │ Tailwind CSS │ React Router v6             │
│         Context Providers: Auth, Branding, Feature, AI          │
└──────────────────────────────┬──────────────────────────────────┘
                               │ REST API (JSON)
┌──────────────────────────────▼──────────────────────────────────┐
│                     Backend (FastAPI + Uvicorn)                  │
├─────────────────────────────────────────────────────────────────┤
│  API Layer (v1)  │  Services Layer  │  CRUD Layer  │  Models   │
├─────────────────────────────────────────────────────────────────┤
│  Security: JWT + RBAC │ Event Bus │ Policy Engine │ Cost Engine │
└──────┬──────────────┬──────────────┬────────────────────────────┘
       │              │              │
┌──────▼──────┐ ┌────▼────┐ ┌──────▼──────┐
│  DynamoDB   │ │  Redis  │ │ Cloud APIs  │
│ (Single-    │ │ (Cache, │ │ AWS│Azure│  │
│  Table)     │ │  Token  │ │    GCP      │
│             │ │  Blackl)│ │             │
└─────────────┘ └─────────┘ └─────────────┘
```

### Three-Layer Backend Architecture

| Layer | Location | Responsibility |
|-------|----------|---------------|
| **Models** | `backend/app/models/` | Pydantic schemas: `XCreate`, `XUpdate`, `XInDB`, enums |
| **CRUD** | `backend/app/crud/` | Async DynamoDB operations only — zero business logic |
| **Endpoints** | `backend/app/api/v1/endpoints/` | FastAPI routers — call CRUD/services, emit events |
| **Services** | `backend/app/services/` | Business logic, orchestration, cloud integrations |

### DynamoDB Single-Table Design

- **Key Schema:** `PK = "TENANT#{tenant_id}"`, `SK = "DOMAIN#{item_id}"`
- **Listing Pattern:** `begins_with(SK, "DOMAIN#")` within a tenant partition
- **All writes** pass through `_sanitize_for_dynamodb()` (converts float→Decimal, strips None)
- **All calls** wrapped with `await loop.run_in_executor(None, fn)` for async compatibility
- **Reserved words** (`name`, `status`, `type`, `data`) use `ExpressionAttributeNames`

### Event-Driven Architecture

The platform uses an in-process async EventBus with:
- 60+ event types covering all domain operations
- Max 10,000 event queue with exponential backoff retry
- Dead-letter store for failed events
- Automation rules that trigger actions (notifications, webhooks, workflows)

---

## 3. Authentication & Identity

### Login Methods

| Method | Description |
|--------|-------------|
| **Local Auth** | Username/email + password with JWT tokens |
| **SSO (OIDC)** | Microsoft, Google, Okta, AWS Identity Center, Ping Identity |
| **API Tokens** | Long-lived tokens for programmatic access |

### JWT Token System

- **Algorithm:** HS256
- **Expiry:** 30 minutes
- **Payload:** `sub` (user_id), `tenant_id`, `jti` (unique token ID), `role_override`
- **Refresh:** Token refresh endpoint for session extension
- **Blacklist:** Redis-backed token revocation (graceful degradation if Redis unavailable)

### Password Requirements

- Minimum 8 characters
- At least one uppercase letter
- At least one lowercase letter
- At least one digit

### SSO Configuration

Supported identity providers:
- **Microsoft Entra ID** (Azure AD)
- **Google Workspace**
- **Okta**
- **AWS Identity Center**
- **Ping Identity**

SSO features:
- OIDC protocol with PKCE
- Automatic user provisioning on first login
- Group synchronization from IdP to CMP roles
- Single Logout (SLO) support
- Domain-based access restriction

### Login Flow

1. User navigates to `/login` (or `/t/{tenant}/login` for tenant-specific)
2. Chooses local login or SSO provider
3. On success, receives JWT token stored in `localStorage`
4. Token attached to all API requests via `Authorization: Bearer <token>`

### Self-Registration

When enabled via feature toggle (`self_registration`), new users can register from the login page with username, email, password, first name, and last name.

---

## 4. Role-Based Access Control (RBAC)

### Role Hierarchy

```
admin > developer > user > readonly
```

### Role Definitions

| Role | Access Level | Description |
|------|-------------|-------------|
| **admin** | Full | Complete platform access including all administration, governance, and system settings |
| **developer** | Provisioning + Infrastructure | Can create/manage tasks, workflows, flows, catalogs, credentials, resources, Terraform |
| **user** | Self-Service | Can order from catalog, view own executions, manage own credentials, view resources |
| **readonly** | View Only | Dashboard and Service Catalog (view only), can view approvals |

### Role Assignment Sources

Roles can be assigned from multiple sources (merged, deduplicated):
1. **Direct assignment** — Set on the user record by an admin
2. **Group membership** — Inherited from CMP groups the user belongs to
3. **SSO group mapping** — Derived from external IdP group memberships

### Role Resolution

The system resolves the "active" role using priority:
1. If user has `admin` in any source → active role = `admin`
2. Else if `developer` → active role = `developer`
3. Else if `user` → active role = `user`
4. Else → `readonly`

### Role Override

Users with multiple roles can switch their active role via the token's `role_override` field (useful for testing lower-privilege views).

### Endpoint Protection

```python
# Require authentication
@router.get("/resource", dependencies=[Depends(get_current_user)])

# Require specific roles
@router.post("/admin-action", dependencies=[Depends(require_role(["admin"]))])

# Admin + Developer access
@router.post("/provision", dependencies=[Depends(require_role(["admin", "developer"]))])
```

---

## 5. Navigation & UI Structure

### URL Pattern

All authenticated routes use tenant-scoped paths: `/t/{tenantSlug}/...`

Example: `/t/default/dashboard`, `/t/acme-corp/catalog`

### Top Navigation Bar

The navigation bar is customizable per tenant (color, logo, company name) and adapts based on user role:

#### Always Visible (All Roles)
- **Dashboard** — Platform overview
- **Service Catalog** — Browse and order services
- **Notifications** (bell icon) — In-app notification center
- **Profile** (avatar) — User settings
- **AI Assistant** (⌘K) — AI chatbot toggle

#### User Role (Standard Users)
- **Executions** — View own provisioning executions
- **Approvals** — Submit and track approval requests
- **Resources** — View provisioned cloud resources
- **Credentials** — Manage own cloud accounts

#### Admin + Developer (Dropdown Menus)

**Provisioning Dropdown:**
- Tasks
- Workflows
- Flows
- Executions
- Catalog Components
- Scheduled Jobs
- Terraform Templates
- Event Automation (admin only)

**Infrastructure Dropdown:**
- Accounts & Credentials
- Resources
- Resource Actions
- Shared Variables
- Workspace Management (Terraform)
- State Backend (admin only)

#### Admin Only

**Administration Dropdown (grouped by section):**

*Identity & Access:*
- Users & Groups
- SSO Configuration
- SSO Group Mappings

*Governance:*
- Approvals
- Policies
- Quotas

*Finance:*
- Cost Models
- Budgets
- Reports & Analytics

*System:*
- Tenants
- Logging & Monitoring
- Inbound Webhooks
- Event Log

### Developer Mode Toggle

Admin and Developer users can toggle "Developer Mode" which enables additional technical details and debugging information in the UI.

---

## 6. Dashboard

**Route:** `/t/{tenant}/dashboard`  
**Access:** All authenticated users

### Functionality

The Dashboard provides a real-time overview of the platform state:

- **Resource Summary** — Total active resources by provider (AWS/Azure/GCP)
- **Execution Statistics** — Recent executions with success/failure rates
- **Approval Queue** — Pending approvals requiring action
- **Cost Overview** — Current spend vs. budget
- **Recent Activity** — Latest events and actions across the tenant
- **Quick Actions** — Shortcuts to common operations

### Technical Details

- Data fetched from `/api/v1/dashboard` endpoint
- Aggregates data from inventory, executions, approvals, and cost analytics
- Auto-refreshes on navigation

---

## 7. Service Catalog

**Route:** `/t/{tenant}/catalog`  
**Access:** All authenticated users (items filtered by role and group)

### Overview

The Service Catalog is the primary self-service interface for users to request cloud resources and services. Admins/developers create catalog items; users browse and order them.

### Catalog Item Structure

Each catalog item consists of:

| Field | Description |
|-------|-------------|
| **Name** | Display name (max 200 chars) |
| **Description** | Rich description (max 2000 chars) |
| **Tags** | Categorization labels (max 50) |
| **Catalog Type** | `day1` (provisioning with credential) or `day2` (operational actions) |
| **UI Mode** | `form_builder` (drag-and-drop form) or `byoui` (custom HTML) |
| **Form Schema** | Dynamic form with fields, layout, sections, tabs |
| **Flow ID** | Linked provisioning flow to execute on submission |
| **Status** | `draft`, `published`, or `archived` |
| **Allowed Roles** | Which roles can see/order this item |
| **Allowed Groups** | Which groups can see/order this item |
| **Approval Settings** | Whether approval is required, who can approve |
| **Cost Estimate** | Whether to show CMP charges and live cloud pricing |

### Form Builder

The form builder supports these field types:
- `string`, `number`, `boolean`, `select`, `multiselect`, `radio`
- `date`, `email`, `url`, `password`, `range`, `file`
- `hidden`, `readonly`, `secret`, `cloud_region`, `textarea`, `table`

Advanced form features:
- **Dynamic options** — Populate select fields from external APIs
- **Conditional display** — Show/hide fields based on other field values
- **Reusable components** — Reference shared form components
- **Layout** — Sections (collapsible), tabs, multi-column rows
- **Validation** — Min/max, regex patterns, required fields

### Ordering Flow

1. User browses published catalog items (filtered by role/group)
2. Selects an item and fills out the dynamic form
3. System evaluates **policies** (deny/warn) against form data
4. System checks **quotas** for the user/group/tenant
5. System calculates **cost estimate** (CMP charges + cloud pricing)
6. If approval required → creates approval request
7. If no approval → triggers the linked flow immediately
8. Execution tracked in the Executions page

### Approval Conditions

Approval can be conditional based on form field values:
- Operators: `equals`, `not_equals`, `contains`, `greater_than`, `less_than`, `in`
- Logic: `any` (OR) or `all` (AND)
- Example: Require approval only when `instance_type` is `x2idn.metal`

### BYOUI Mode

For advanced use cases, catalog items can use custom HTML/JS instead of the form builder. The custom UI communicates via a `submit_contract` that defines expected field names.

---

## 8. Provisioning Engine

The provisioning engine is the core automation layer that executes cloud operations. It consists of four building blocks: Tasks, Workflows, Flows, and Executions.

### 8.1 Tasks

**Route:** `/t/{tenant}/tasks`  
**Access:** Admin, Developer

Tasks are the atomic units of execution — individual code scripts or API calls.

**Supported Languages:**
- Python
- TypeScript/JavaScript
- Bash/Shell
- Go
- SQL
- REST API calls

**Task Features:**
- Inline code editor with syntax highlighting
- GitHub repository linking for version-controlled code
- Input parameters with type definitions
- Output capture for downstream consumption
- Timeout configuration
- Retry policies

**How Tasks Work:**
1. Developer writes code (inline or links to GitHub)
2. Defines input parameters the task expects
3. Task is referenced in Workflows as a step
4. At execution time, the Task Runner executes the code with injected parameters
5. Output is captured and available to subsequent steps

### 8.2 Workflows

**Route:** `/t/{tenant}/workflows`  
**Access:** Admin, Developer

Workflows are multi-step DAGs (Directed Acyclic Graphs) that orchestrate multiple tasks and cloud operations.

**Workflow Step Actions:**
| Action | Description |
|--------|-------------|
| `run_task` | Execute a defined Task |
| `http_request` | Make an HTTP API call |
| `aws.ec2` | AWS EC2 operations |
| `aws.s3` | AWS S3 operations |
| `aws.rds` | AWS RDS operations |
| `aws.lambda` | AWS Lambda operations |
| `azure.vm` | Azure VM operations |
| `gcp.compute` | GCP Compute operations |
| `builtin.delay` | Wait for a specified duration |
| `builtin.condition` | Conditional branching |
| `terraform` | Terraform plan/apply/destroy |

**Step Configuration:**
- `depends_on` — Define execution order (DAG dependencies)
- `on_failure` — `stop`, `continue`, or `rollback`
- `timeout_seconds` — Maximum execution time per step
- `retry_count` — Automatic retry on failure
- `inputs` — Parameters passed to the step (supports variable interpolation)
- `output_key` — Custom key for storing step output

### 8.3 Flows

**Route:** `/t/{tenant}/flows`  
**Access:** Admin, Developer

Flows chain multiple Workflows together with dependency ordering and input mapping. They represent the complete provisioning pipeline for a catalog item.

**Flow Features:**
- Chain multiple workflows in sequence or parallel
- Map outputs from one workflow as inputs to the next
- Credential injection at flow level
- Linked to catalog items for self-service ordering

### 8.4 Executions

**Route:** `/t/{tenant}/executions`  
**Access:** Admin, Developer, User (own executions)

Executions track the real-time status of provisioning operations.

**Execution Lifecycle:**
```
pending → running → success
                  → failed
                  → cancelled
```

**Execution Features:**
- Real-time step-by-step progress tracking
- Per-step logs with output and error details
- Duration tracking (total and per-step)
- Cost tracking (estimated and actual)
- Dry-run mode (preview without executing)
- Step-level retry for failed executions
- Cancel running executions

**Execution Detail View:**
- Step timeline with status indicators
- Expandable step logs
- Input parameters used
- Output data produced
- Error details with stack traces
- Cost breakdown

### 8.5 Catalog Components

**Route:** `/t/{tenant}/catalog-components`  
**Access:** Admin, Developer

Reusable form field components that can be shared across multiple catalog items.

**Use Cases:**
- Standard "Region Selector" dropdown used in all AWS catalogs
- "Environment" radio button (dev/staging/prod) shared across items
- "Instance Type" select with pre-approved options

### 8.6 Scheduled Jobs

**Route:** `/t/{tenant}/scheduled-jobs`  
**Access:** Admin, Developer

Cron-based task scheduling for recurring operations.

**Configuration:**
- **Cron Expression** — Standard cron syntax (e.g., `0 8 * * 1-5` for weekdays at 8 AM)
- **Task ID** — Which task to execute
- **Parameters** — Input params for the task
- **Max Runs** — Optional limit on total executions
- **Status** — `active`, `paused`, `completed`, `failed`

**Run History:**
Each job maintains a history of runs with:
- Run ID, status, start/end time, duration
- stdout/stderr output
- Exit code

---

## 9. Cloud Infrastructure Management

### 9.1 Accounts & Credentials

**Route:** `/t/{tenant}/credentials`  
**Access:** Admin, Developer, User (own credentials)

Manage cloud provider accounts used for provisioning and resource management.

**Supported Providers:**
| Provider | Required Fields |
|----------|----------------|
| **AWS** | Access Key, Secret Key, Region(s) |
| **Azure** | Client ID, Client Secret (in secret_key), Tenant ID, Subscription ID |
| **GCP** | Service Account JSON (in secret_key), Project ID |
| **GitHub** | Personal Access Token |
| **Bearer Token** | Token value |
| **Basic Auth** | Username + Password |

**Credential Features:**
- Fernet encryption for all secrets at rest
- Multi-region support (list of regions per credential)
- Visibility controls: restrict by role, user IDs, or group IDs
- Active/inactive toggle
- Validation endpoint to test connectivity
- Resource groups and tags (Azure)
- Last-used tracking

**Visibility Model:**
- Admin can see all credentials in the tenant
- Users/Developers see credentials where:
  - They are the creator, OR
  - Their role is in `visible_roles`, OR
  - Their user_id is in `visible_user_ids`, OR
  - Their group is in `visible_group_ids`

### 9.2 Resources

**Route:** `/t/{tenant}/resources`  
**Access:** Admin, Developer, User

View and manage cloud resources discovered across all connected accounts.

**Resource Discovery:**
- Fetches resources from all active credentials
- Supports EC2 instances, S3 buckets, RDS databases, Lambda functions, VMs, etc.
- Pagination with filtering by type, status, region, provider
- Sorting by name, status, created date

**Resource Detail View (`/resources/{credentialId}/{resourceId}`):**
- Full resource metadata and tags
- Current status with available actions
- Execution history (all actions performed on this resource)
- Live cost projections (when enabled)
- Lease/expiry information

**Native Resource Actions:**
- Start, Stop, Restart, Terminate/Delete
- Provider-specific actions based on resource type

### 9.3 Resource Actions

**Route:** `/t/{tenant}/resource-actions`  
**Access:** Admin, Developer

Define custom Day 2 operations that users can perform on provisioned resources.

**Action Backend Types:**
| Type | Description |
|------|-------------|
| `native` | Built-in cloud operations (start/stop/terminate) |
| `flow` | Execute a CMP Flow with resource context |
| `terraform` | Run Terraform operations (plan/apply/destroy/targeted-apply) |

**Action Configuration:**
- **Conditions** — When the action is available (status, region, tags, name patterns)
- **Input Schema** — Custom form fields for user input
- **Approval** — Optional approval workflow with conditional triggers
- **Visibility** — Role-based visibility control
- **Cost Estimate** — Show estimated cost before execution
- **Status Message** — Warning/info banner shown to users

**Terraform-Backed Actions:**
- Link to a Terraform template
- Workspace resolution: `inventory_link` (auto-detect) or `explicit` (specify workspace)
- Variable mappings from resource context to Terraform variables
- Operations: `plan`, `apply`, `destroy`, `targeted-apply`

### 9.4 Shared Variables (Context)

**Route:** `/t/{tenant}/context`  
**Access:** Admin, Developer

Global key-value store for variables shared across tasks, workflows, and flows.

**Variable Types:**
- **String** — Plain text values (visible in UI)
- **Secret** — Encrypted values (masked in UI, decrypted at runtime)

**Use Cases:**
- Environment-specific configuration (API URLs, endpoints)
- Shared secrets (API keys, tokens) injected into tasks
- Default values referenced in workflow step inputs

### 9.5 Inventory & Lease Management

The inventory system tracks all provisioned resources with lifecycle management.

**Inventory Record:**
- Resource name, type, cloud provider, region
- Owner (user who provisioned it)
- Group assignment
- Lease date and expiry date
- Status: `active`, `stopped`, `expired`, `destroyed`
- Tags and catalog reference
- Terraform workspace link

**Lease Features:**
- Set destroy/expiry dates on resources
- Automatic warnings at 3 days and 1 day before expiry
- Auto-destruction on lease expiry (configurable)
- Lease extension by admins

---

## 10. Terraform Provisioning

**Feature Toggle:** `terraform_provisioning`  
**Access:** Admin, Developer

### 10.1 Terraform Templates

**Route:** `/t/{tenant}/terraform-templates`

Templates define reusable Infrastructure-as-Code configurations.

**Source Types:**
| Source | Description |
|--------|-------------|
| `inline` | HCL content stored directly (max 1 MB) |
| `git` | Repository URL + branch + folder path (with auth token) |
| `upload` | Uploaded archive (S3/storage key) |
| `registry` | Terraform Registry module reference |

**Template Configuration:**
- Input variables with types (string, number, bool, list, map, object)
- Variable attributes: required, sensitive, default value, Day 2 configurable
- Output definitions
- Required providers with version constraints
- Terraform version constraint
- Supported cloud providers
- Tags for categorization
- Versioning (auto-incremented on update)

### 10.2 Terraform Workspaces

**Route:** `/t/{tenant}/terraform-workspaces`

Workspaces represent deployed instances of templates with their own state.

**Workspace Lifecycle:**
```
initializing → active → updating → active
                      → destroying → destroyed
                      → errored (admin can retry/force-destroy)
```

**Valid Status Transitions:**
- `initializing` → `active`, `errored`
- `active` → `updating`, `destroying`
- `updating` → `active`, `errored`
- `destroying` → `destroyed`, `errored`
- `errored` → `updating` (retry), `destroying` (force), `destroyed` (force)
- `destroyed` → (terminal state)

**Workspace Features:**
- Variable values stored per workspace
- Credential linking for cloud access
- Execution history
- Locking mechanism (prevents concurrent operations)
- Cursor-based pagination with filters (owner, group, template, status)

### 10.3 Day 2 Operations

Post-deployment operations on active workspaces:

| Operation | Description |
|-----------|-------------|
| `variable_update` | Modify Day 2-configurable variables and re-apply |
| `resource_addition` | Add new resources to the stack |
| `resource_removal` | Remove specific resources |
| `stack_destruction` | Destroy the entire workspace |

### 10.4 Drift Detection

Automated detection of configuration drift between desired state and actual infrastructure.

**Features:**
- Scheduled drift checks
- Drift status tracking per workspace (`unknown`, `in_sync`, `drifted`)
- Accept drift (update desired state) or remediate (re-apply)
- Last drift check timestamp

### 10.5 State Backend Management

**Route:** `/t/{tenant}/terraform-state-backend`  
**Access:** Admin only

Configure where Terraform state is stored:

| Backend | Description |
|---------|-------------|
| **CMP Backend** | Built-in state storage in DynamoDB |
| **S3 Backend** | AWS S3 bucket with DynamoDB locking |
| **TFC Backend** | Terraform Cloud remote state |

### 10.6 Module Registry

Manage reusable Terraform modules that can be referenced in templates.

---

## 11. Governance & Compliance

### 11.1 Policies

**Route:** `/t/{tenant}/policies`  
**Access:** Admin

Governance policies that evaluate catalog requests and enforce organizational rules.

**Policy Structure:**
- **Name & Description** — Human-readable policy identification
- **Scope** — Apply to specific catalog IDs, catalog tags, or all catalogs
- **Rules** — List of conditions with effects
- **Enabled** — Toggle policy on/off
- **Version** — Track policy changes

**Policy Rules:**
Each rule contains:
- **Conditions** — Field-based checks on form data
  - Operators: `equals`, `not_equals`, `in`, `not_in`, `greater_than`, `less_than`, `contains`
  - Condition logic: `ALL` (AND) or `ANY` (OR)
- **Effect** — `deny` (block the request) or `warn` (show warning but allow)
- **Message** — Displayed to the user when triggered

**Example Policy:**
```
Policy: "Restrict Large Instances"
Scope: All catalogs tagged "compute"
Rule: IF instance_type IN ["x2idn.metal", "p4d.24xlarge"] THEN DENY
Message: "Metal and GPU instances require a separate approval process"
```

**Evaluation Flow:**
1. User submits catalog order
2. Policy engine evaluates all active policies matching the catalog
3. If any rule has `deny` effect → request blocked with message
4. If rules have `warn` effect → warning shown, user can proceed

### 11.2 Quotas

**Route:** `/t/{tenant}/quotas`  
**Access:** Admin

Resource limits that prevent over-provisioning.

**Quota Scopes:**
| Scope | Description |
|-------|-------------|
| `user` | Limit per individual user |
| `group` | Limit per group |
| `tenant` | Limit for the entire tenant |

**Quota Configuration:**
- **Resource Type** — What's being limited (e.g., `ec2_instance`, `s3_bucket`, `catalog_request`)
- **Max Count** — Maximum allowed quantity
- **Current Count** — Tracked automatically

**Quota Check Result:**
- `allowed: true/false`
- Current count vs. max count
- Human-readable message

### 11.3 Approvals

**Route:** `/t/{tenant}/approvals`  
**Access:** All users (submit); Admin + designated approvers (approve/reject)

Multi-level approval workflow for catalog requests and resource actions.

**Approval Lifecycle:**
```
pending → approved → (triggers execution)
        → rejected
        → cancelled (by requester)
        → timed_out
```

**Approval Configuration (per catalog/action):**
- **Approver User IDs** — Specific users who can approve
- **Approver Group IDs** — Members of these groups can approve
- **Approver Roles** — Users with these roles can approve
- **Conditional Approval** — Only require approval when form data matches conditions

**Approval Features:**
- Comments thread on each request
- Cost estimate attached to approval
- Justification field for requesters
- Automatic execution on approval
- Flow triggers on approve/reject/cancel events
- Full audit trail

---

## 12. Cost Management & Finance

### 12.1 Cost Models

**Route:** `/t/{tenant}/cost-models`  
**Access:** Admin

Define CMP platform charges applied on top of cloud costs.

**Cost Model Scope:**
| Scope | Description |
|-------|-------------|
| `global` | Applies to all catalog items and resource actions |
| `catalog` | Applies only to specified catalog IDs |
| `resource_action` | Applies only to specified resource action IDs |

**Cost Rules:**
Each model contains rules with:
- **Conditions** — When the rule applies (based on form fields or cloud cost)
  - Operators: `equals`, `not_equals`, `contains`, `greater_than`, `less_than`, `in`, `not_in`
  - Logic: `any` (OR) or `all` (AND)
- **Charge Type** — `percentage` (% of cloud cost) or `fixed` (flat amount)
- **Charge Value** — The percentage or fixed amount
- **Charge Label** — Displayed in cost breakdown
- **Priority** — Evaluation order (lower = higher priority)

**Cost Estimation Flow:**
1. User fills catalog form
2. System fetches live cloud pricing for selected resources
3. Cost engine evaluates all matching cost models
4. Returns breakdown: cloud cost + each CMP charge + total

**Example:**
```
Model: "Platform Management Fee"
Scope: Global
Rule: Always apply
Charge: 15% of cloud cost
Label: "CMP Management Fee"

Model: "Premium Support"
Scope: Catalog (production catalogs only)
Rule: IF environment == "production"
Charge: Fixed $50/month
Label: "Premium Support Surcharge"
```

### 12.2 Budgets

**Route:** `/t/{tenant}/budgets`  
**Access:** Admin

Set spending limits with alert thresholds.

**Budget Configuration:**
- **Amount** — Budget limit in specified currency
- **Currency** — Default USD
- **Period** — `monthly`, `quarterly`, or `annual`
- **Scope** — All providers, specific provider, specific credential, or specific group
- **Thresholds** — Alert at percentage levels (e.g., 80%, 90%, 100%)
- **Notification Channels** — Where to send alerts (email, Slack, Teams)

**Budget Tracking:**
- `current_spend` tracked automatically via cost analytics
- Alerts triggered when thresholds are crossed
- Events emitted: `cost.threshold.exceeded`

### 12.3 Reports & Analytics

**Route:** `/t/{tenant}/reports`  
**Access:** Admin

Platform-wide analytics dashboards:
- Resource utilization by provider/region/type
- Cost trends over time
- Execution success/failure rates
- User activity metrics
- Approval turnaround times

### 12.4 Cost Analytics

**API:** `/api/v1/cost-analytics`  
**Access:** Admin

Aggregated cost summaries broken down by:
- User
- Group
- Catalog item
- Cloud provider
- Time period

### 12.5 Live Cloud Pricing

**API:** `/api/v1/cloud-pricing`

Real-time pricing data from cloud providers used during catalog ordering to show estimated costs before submission.

---

## 13. Event-Driven Automation

### 13.1 Event System

The platform emits events for every significant operation. Events follow a standard schema:

**Event Schema:**
```
{
  event_id, event_type, timestamp, tenant_id,
  actor: { user_id, username, role, ip },
  resource: { type, id, name },
  metadata: { ... },
  source, correlation_id, status, retry_count
}
```

**Event Categories (60+ types):**

| Category | Events |
|----------|--------|
| **Catalog** | requested, approval_requested, approved, rejected, cancelled |
| **Resource** | provision started/completed/failed, created, updated, deleted, started, stopped, terminated |
| **Credential** | added, updated, deleted, validated, failed |
| **User** | created, updated, deleted, login, logout, password_changed |
| **SSO** | login_success, login_failure, user_provisioned |
| **Group** | created, updated, deleted |
| **Approval** | pending, approved, rejected, timed_out, cancelled |
| **Workflow** | created, updated, deleted, started, completed, failed |
| **Flow** | created, updated, deleted |
| **Execution** | started, completed, failed, retried, cancelled |
| **Task** | created, updated, deleted, started, completed, failed |
| **Cost** | threshold_exceeded, estimate_generated, anomaly_detected |
| **Notification** | sent, failed |
| **System** | health_degraded, health_restored |
| **Automation** | rule_triggered, rule_failed |
| **Terraform** | action_started, action_completed, action_failed |
| **Webhook** | delivered, failed |

### 13.2 Event Automation Rules

**Route:** `/t/{tenant}/events`  
**Access:** Admin

Configure rules that automatically trigger actions when specific events occur.

**Rule Structure:**
- **Event Type** — Which event triggers this rule
- **Conditions** — Additional filters on event data
  - Field path (dot notation into event, e.g., `metadata.cost`)
  - Operators: equals, not_equals, contains, greater_than, less_than, exists, in
  - Logic: AND or OR
- **Actions** — What to do when triggered
- **Priority** — Execution order (lower = higher priority)
- **Enabled** — Toggle on/off

**Available Actions:**

| Action Type | Description |
|-------------|-------------|
| `send_notification` | Send email/in-app notification to specified recipients |
| `send_slack` | Post to Slack via configured webhook |
| `send_teams` | Post to Microsoft Teams via configured webhook |
| `trigger_workflow` | Start a CMP workflow with specified parameters |
| `call_webhook` | Make HTTP request to external URL |
| `call_api` | Call external API with authentication |
| `execute_task` | Run a specific CMP task |
| `publish_event` | Emit another event (event chaining) |

**Example Rules:**
```
Rule: "Notify on Failed Provisioning"
Event: resource.provision.failed
Action: send_slack { body: "⚠️ Provisioning failed for {{resource.name}}" }

Rule: "Auto-Remediate Drift"
Event: terraform.action.completed
Conditions: metadata.drift_detected == true
Action: trigger_workflow { workflow_id: "remediate-drift" }

Rule: "Budget Alert"
Event: cost.threshold.exceeded
Conditions: metadata.percentage >= 90
Action: send_notification { recipients: ["finance-team"], subject: "Budget 90% reached" }
```

### 13.3 Event Log

**Route:** `/t/{tenant}/event-log`  
**Access:** Admin

System-wide audit trail showing all events with:
- Timestamp, event type, status
- Actor information
- Resource details
- Full metadata
- Filtering by type, date range, actor, resource

---

## 14. Administration

### 14.1 Users & Groups

**Route:** `/t/{tenant}/admin`  
**Access:** Admin

#### User Management

- Create users with username, email, password, roles
- Edit user details and role assignments
- Activate/deactivate user accounts
- View user activity and last login
- Assign users to multiple tenants

#### Group Management

- Create groups with name, description, and linked role
- Add/remove members (user IDs)
- Groups grant their linked role to all members
- Groups can be used for:
  - Credential visibility
  - Catalog item access
  - Approval routing
  - Quota scoping
  - Budget assignment

### 14.2 SSO Configuration

**Route:** `/t/{tenant}/admin/sso/config`  
**Access:** Admin

Configure external identity providers for Single Sign-On.

**Provider Setup:**
- Provider type (Microsoft, Google, Okta, AWS Identity Center, Ping Identity)
- Client ID and Client Secret
- Issuer URL (must be HTTPS)
- Redirect URL
- Allowed email domains
- Auto user creation on first SSO login
- Group sync enabled/disabled
- Single Logout enabled/disabled
- Display name for login button

### 14.3 SSO Group Mappings

**Route:** `/t/{tenant}/admin/sso/group-mappings`  
**Access:** Admin

Map external IdP groups to CMP roles.

**Mapping Configuration:**
- External Group ID (from IdP)
- Group Name (display name)
- CMP Role (admin, developer, user, readonly)
- Priority (1-100, for conflict resolution)

**How It Works:**
1. User logs in via SSO
2. IdP returns user's group memberships
3. CMP matches groups against configured mappings
4. Matched roles are merged with any direct role assignments
5. Highest-priority role becomes the active role

### 14.4 Branding

**API:** `/api/v1/branding`  
**Access:** Admin

Customize the platform appearance per tenant:
- **Company Name** — Displayed in header and footer
- **Logo URL** — Company logo in navigation bar
- **Top Bar Color** — Navigation bar background color (hex)
- Text color auto-calculated for contrast

### 14.5 Logging & Monitoring

**Route:** `/t/{tenant}/admin/logging`  
**Access:** Admin

Platform health and operational monitoring:
- Application logs
- API request metrics
- Error rates and patterns
- System health indicators

### 14.6 Inbound Webhooks

**Route:** `/t/{tenant}/webhooks`  
**Access:** Admin

Configure webhook endpoints that external systems can call to trigger CMP actions.

---

## 15. AI Assistant

**Feature Toggle:** `chatbot`  
**Access:** All authenticated users  
**Shortcut:** ⌘K (Mac) / Ctrl+K (Windows)

### Overview

The AI Assistant is an **intelligent, context-aware enterprise cloud copilot** powered by Google Gemini 2.5 Flash Lite. It has full awareness of the CMP platform data (inventory, executions, catalogs, audit logs, approvals, budgets) while enforcing strict role-based access control on all responses and actions.

### Intelligence Model

The assistant dynamically adapts based on:
- **User Role** — Only suggests/executes actions the user is permitted to perform
- **Current Page** — Tailors suggestions to the user's current workflow context
- **Selected Resource** — Understands "this", "it" when viewing a specific resource
- **Ownership Filtering** — Non-admin users only see their own data; admins see platform-wide
- **Catalog Intelligence** — Recommends catalogs based on intent, role eligibility, and tags
- **Historical Activity** — Can query user's past executions, audit logs, and provisioning history

### Capabilities

- **Resource Actions** — Start, stop, restart, terminate resources directly via natural language
- **Catalog Provisioning** — Open provisioning forms, recommend catalogs based on intent
- **Troubleshooting** — Analyze failed executions, explain errors, suggest fixes
- **Analytics** — Platform summaries, budget status, execution metrics, resource counts
- **API Execution** — Developers/admins can run any CMP API endpoint through the assistant
- **Cloud Pricing** — Live pricing lookups across AWS, Azure, GCP
- **Lease Management** — Apply/remove resource leases via conversation
- **Approval Management** — View, approve, or reject requests (admin)
- **Navigation** — Guide users to the correct page for any task

### Ownership-Aware Data Access

| User Role | Data Scope |
|-----------|-----------|
| USER | Own resources, own executions, own audit logs |
| DEVELOPER | Own resources + create catalogs/workflows/tasks |
| ADMIN | Full platform-wide access to all data |

### UI

- Slide-out panel on the right side of the screen
- Dual mode: Command palette (quick actions) + Chat conversation
- Context pill showing current page, role, and selected resource
- Clickable suggestion chips for follow-up actions
- Persistent conversation history per user
- Toggle via ⌘K shortcut or nav bar button

---

## 16. Notifications & Channels

### 16.1 In-App Notifications

**Route:** `/t/{tenant}/notifications`  
**Access:** All authenticated users

- Real-time notification center
- Unread count badge in navigation bar
- Mark as read/unread
- Notification types: approvals, executions, alerts, system messages

### 16.2 Notification Channels

**API:** `/api/v1/settings/notification-channels`  
**Access:** Admin

Configure external notification delivery:

| Channel | Configuration |
|---------|--------------|
| **Email (SMTP)** | Host, port, username, password, from address |
| **Slack** | Webhook URL |
| **Microsoft Teams** | Webhook URL |
| **PagerDuty** | Integration key |

### 16.3 Notification Triggers

Notifications are sent via:
- Event automation rules (configurable per event type)
- Budget threshold alerts
- Approval requests (to approvers)
- Execution completions/failures (to submitter)
- Lease expiry warnings (to resource owner)
- Drift detection alerts (to workspace owner)

### 16.4 Mail Body Templates

**API:** `/api/v1/settings/mail-templates`  
**Access:** Admin

Admins can customize the email subject and body for each notification event type. Templates support `{placeholder}` variables that are automatically replaced with event-specific metadata at send time.

#### Configurable Events

| Event Key | When It Fires | Available Placeholders |
|-----------|---------------|----------------------|
| `approval.pending` | New approval request submitted | `catalog_name`, `requested_by`, `approval_id`, `justification` |
| `approval.approved` | Request approved | `catalog_name`, `actioned_by`, `comment` |
| `approval.rejected` | Request rejected | `catalog_name`, `actioned_by`, `comment` |
| `resource.provision.completed` | Resource provisioned successfully | `resource_name`, `resource_type`, `execution_id`, `duration_ms` |
| `resource.provision.failed` | Resource provisioning failed | `resource_name`, `resource_type`, `execution_id`, `duration_ms` |
| `cost.threshold.exceeded` | Budget threshold crossed | `budget_name`, `threshold_pct`, `current_spend`, `budget_amount`, `current_pct`, `resource_name` |
| `catalog.requested` | Catalog item requested | `catalog_name`, `requested_by`, `catalog_type`, `status` |
| `catalog.completion.success` | Catalog execution succeeded | `catalog_name`, `execution_id`, `duration_ms`, `step_count`, `total_steps` |
| `catalog.completion.failure` | Catalog execution failed | `catalog_name`, `execution_id`, `failed_step`, `error` |

#### Per-Event Enable/Disable

Each mail template has an **enabled** toggle (default: `true`). When set to `false`, no email or notification is sent for that event — the template is effectively silenced without deleting it. This allows admins to selectively mute specific notification types (e.g., disable provisioning success emails while keeping failure alerts active).

To disable a notification, update the template via the API or Administration → Settings page and set `enabled` to `false`.

#### How It Works

1. Each event type has a **default template** with a pre-written subject and body.
2. Admins can override any template via the Administration → Settings page.
3. Placeholders are written as `{placeholder_name}` in both subject and body fields.
4. Only placeholders listed for that event type are substituted — unknown placeholders are left as-is.
5. If an admin hasn't customized a template, the system default is used.
6. If a template's `enabled` flag is `false`, the notification is skipped entirely — no email is sent for that event.

#### Example Template

**Event:** `approval.pending`

```
Subject: [CMP] Approval Required: {catalog_name}

Body:
A new approval request has been submitted.

Catalog: {catalog_name}
Requested by: {requested_by}
Approval ID: {approval_id}
Justification: {justification}

Please log in to the Cloud Management Platform to review.
```

#### API Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/settings/mail-templates` | Retrieve all mail templates (defaults + overrides) |
| PUT | `/api/v1/settings/mail-templates` | Update one or more mail templates |

---

## 17. Feature Toggle System

**API:** `/api/v1/settings/feature-toggles`  
**Access:** Admin (manage), All users (read own toggles)

### Overview

24 features can be independently enabled/disabled with granular access control.

### Toggle Configuration

Each toggle has:
- **key** — Feature identifier (e.g., `service_catalog`)
- **enabled** — Global on/off switch
- **roles** — If set, only these roles see the feature (empty = all roles)
- **groups** — If set, only members of these groups see the feature
- **users** — If set, only these specific user IDs see the feature

### Available Features

| Key | Label | Group | Description |
|-----|-------|-------|-------------|
| `service_catalog` | Service Catalog | General | Browse and order from the service catalog |
| `executions` | Executions | General | View and manage task/workflow executions |
| `approvals` | Approvals | General | Submit and manage approval requests |
| `resources` | Resources | General | View cloud resources across accounts |
| `credentials` | Credentials | General | Access cloud account credentials |
| `notifications` | Notifications | General | User notification center |
| `chatbot` | AI Chatbot | General | AI assistant in the navigation bar |
| `live_cost_projections` | Live Cost Projections | General | Real-time cost projections |
| `self_registration` | Self Registration | Authentication | Allow new user registration |
| `forgot_password` | Forgot Password | Authentication | Password reset on login page |
| `provisioning` | Provisioning | Admin/Developer | Tasks, Workflows, Flows, Components, Jobs |
| `infrastructure` | Infrastructure | Admin/Developer | Credentials, Resources, Actions, Variables |
| `event_automation` | Event Automation | Admin/Developer | Event-driven workflow configuration |
| `terraform_provisioning` | Terraform | Admin/Developer | Templates, workspaces, drift, state |
| `reports` | Reports & Analytics | Admin | Platform analytics dashboards |
| `cost_models` | Cost Models | Admin | Cost allocation model management |
| `budgets` | Budgets | Admin | Spending budget management |
| `policies` | Policies | Admin | Cloud governance policies |
| `quotas` | Quotas | Admin | Resource quota limits |
| `tenants` | Tenants | Admin | Tenant organization management |
| `webhooks` | Inbound Webhooks | Admin | Webhook endpoint configuration |
| `event_log` | Event Log | Admin | System-wide event audit log |
| `logging_monitoring` | Logging & Monitoring | Admin | Platform logs and monitoring |
| `cost_analytics` | Cost Analytics | Admin | Aggregated cost summaries |

### Frontend Integration

```typescript
const { isFeatureEnabled } = useFeature()

// Conditionally render UI elements
{isFeatureEnabled('service_catalog') && <CatalogLink />}
```

### Backend Integration

Feature toggles are checked at the route level — disabled features return redirects to the dashboard.

---

## 18. Multi-Tenancy

### Overview

CMP supports complete multi-tenant isolation where each tenant is an independent organization with its own users, resources, configurations, and branding.

### Tenant Structure

| Field | Description |
|-------|-------------|
| `tenant_id` | Unique identifier (same as slug) |
| `name` | Display name |
| `slug` | URL-safe identifier used in paths |
| `admin_email` | Primary admin contact |
| `plan` | Subscription plan (e.g., "standard") |
| `status` | `active`, `suspended`, or `trial` |
| `max_users` | User limit for the tenant |
| `max_credentials` | Credential limit |
| `branding` | Custom logo, colors, company name |

### Tenant Management

**Route:** `/t/{tenant}/tenants`  
**Access:** Admin

- Create new tenants (provisions admin user automatically)
- Update tenant settings and limits
- Suspend/activate tenants
- Configure per-tenant branding

### Tenant Switching

Users who belong to multiple tenants can switch between them:
1. Dropdown in the navigation bar shows available tenants
2. Switching issues a new JWT token scoped to the selected tenant
3. All subsequent API calls use the new tenant context

### Data Isolation

- Every DynamoDB record includes `tenant_id` in the partition key
- All queries are scoped to the current tenant — no cross-tenant access
- Credentials, resources, policies, budgets, etc. are all tenant-isolated

### URL Structure

All authenticated routes include the tenant slug:
```
/t/{tenantSlug}/dashboard
/t/{tenantSlug}/catalog
/t/{tenantSlug}/admin
```

---

## 19. API Reference Summary

**Base URL:** `http://localhost:8001/api/v1` (development)  
**Documentation:** `/docs` (Swagger UI), `/redoc` (ReDoc)

### Authentication Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/login` | Login with JSON body (username + password) |
| POST | `/auth/register` | Register new user |
| POST | `/auth/logout` | Logout (blacklists token) |
| POST | `/auth/refresh` | Refresh JWT token |
| POST | `/auth/switch-tenant` | Switch active tenant |
| GET | `/auth/me` | Get current user info |
| POST | `/auth/sso/{provider}/authorize` | Initiate SSO flow |
| POST | `/auth/sso/{provider}/callback` | SSO callback handler |
| POST | `/auth/api-tokens` | Create API token |
| GET | `/auth/api-tokens` | List API tokens |
| DELETE | `/auth/api-tokens/{token_id}` | Revoke API token |

### Cloud & Resources

| Method | Path | Description |
|--------|------|-------------|
| POST | `/cloud/credentials` | Add credential |
| GET | `/cloud/credentials` | List credentials |
| GET | `/cloud/credentials/{id}` | Get credential |
| PATCH | `/cloud/credentials/{id}` | Update credential |
| DELETE | `/cloud/credentials/{id}` | Delete credential |
| POST | `/cloud/credentials/{id}/validate` | Validate credential |
| GET | `/cloud/resources` | List all resources (paginated) |
| GET | `/cloud/resources/{credential_id}` | List resources for credential |
| POST | `/cloud/resources/{credential_id}/{resource_id}/action` | Execute action |

### Service Catalog

| Method | Path | Description |
|--------|------|-------------|
| GET | `/catalog` | List catalog items |
| POST | `/catalog` | Create catalog item |
| GET | `/catalog/{id}` | Get catalog item |
| PUT | `/catalog/{id}` | Update catalog item |
| DELETE | `/catalog/{id}` | Delete catalog item |
| POST | `/catalog/{id}/order` | Order from catalog |

### Provisioning

| Method | Path | Description |
|--------|------|-------------|
| CRUD | `/tasks` | Task management |
| POST | `/tasks/{id}/run` | Execute task |
| CRUD | `/workflows` | Workflow management |
| CRUD | `/flows` | Flow management |
| GET | `/executions` | List executions |
| GET | `/executions/{id}` | Execution detail |
| POST | `/executions/{id}/cancel` | Cancel execution |
| POST | `/executions/{id}/retry` | Retry failed execution |
| CRUD | `/scheduled-jobs` | Scheduled job management |

### Governance

| Method | Path | Description |
|--------|------|-------------|
| CRUD | `/policies` | Policy management |
| POST | `/policies/evaluate` | Evaluate policies |
| CRUD | `/quotas` | Quota management |
| GET | `/quotas/check` | Check quota availability |
| CRUD | `/approvals` | Approval management |
| POST | `/approvals/{id}/approve` | Approve request |
| POST | `/approvals/{id}/reject` | Reject request |

### Cost & Finance

| Method | Path | Description |
|--------|------|-------------|
| CRUD | `/cost-models` | Cost model management |
| POST | `/cost-models/estimate` | Calculate cost estimate |
| CRUD | `/budgets` | Budget management |
| GET | `/cost-analytics/summary` | Cost analytics |
| GET | `/cloud-pricing/{provider}` | Live cloud pricing |

### Terraform

| Method | Path | Description |
|--------|------|-------------|
| CRUD | `/terraform/templates` | Template management |
| CRUD | `/terraform/workspaces` | Workspace management |
| POST | `/terraform/workspaces/{id}/plan` | Run plan |
| POST | `/terraform/workspaces/{id}/apply` | Run apply |
| POST | `/terraform/workspaces/{id}/destroy` | Destroy workspace |
| GET | `/terraform/workspaces/{id}/drift` | Check drift |
| CRUD | `/terraform/modules` | Module management |
| CRUD | `/terraform/state-backend` | State backend config |

### Events & Automation

| Method | Path | Description |
|--------|------|-------------|
| GET | `/events` | List events |
| POST | `/events/publish` | Publish event |
| CRUD | `/events/rules` | Automation rule management |

### Administration

| Method | Path | Description |
|--------|------|-------------|
| CRUD | `/tenants` | Tenant management |
| GET/PUT | `/settings/feature-toggles` | Feature toggle management |
| GET | `/settings/feature-toggles/me` | Current user's enabled features |
| CRUD | `/settings/notification-channels` | Notification channel config |
| GET/PUT | `/settings/mail-templates` | Mail body template customization |
| GET/PUT | `/branding` | Branding configuration |
| CRUD | `/groups` | Group management |
| CRUD | `/webhooks` | Webhook management |
| GET | `/monitoring/health` | System health |

---

## 20. Role-Based Usage Guide

### 20.1 Admin Perspective

**Primary Responsibilities:** Platform governance, user management, cost control, security

#### Day-to-Day Tasks

1. **User & Access Management**
   - Create/deactivate user accounts
   - Assign roles and group memberships
   - Configure SSO providers and group mappings
   - Monitor login activity

2. **Governance Setup**
   - Define policies to restrict dangerous configurations
   - Set quotas to prevent resource sprawl
   - Configure approval workflows for high-cost items
   - Review and act on pending approvals

3. **Cost Control**
   - Create cost models with CMP charges
   - Set budgets with alert thresholds
   - Monitor cost analytics dashboards
   - Review reports for optimization opportunities

4. **Platform Configuration**
   - Enable/disable features via toggles
   - Configure notification channels (Slack, Teams, email)
   - Customize mail body templates for each event type
   - Set up event automation rules
   - Manage tenant settings and branding
   - Configure Terraform state backends

5. **Security & Compliance**
   - Review event logs for suspicious activity
   - Monitor system health
   - Manage inbound webhooks
   - Review credential usage

#### Key Workflows

- **Onboarding a new team:** Create tenant → Create admin user → Configure SSO → Set up groups → Define policies/quotas → Enable features
- **Cost governance:** Create budget → Set thresholds → Configure alert rules → Review monthly reports
- **Compliance audit:** Review event log → Check policy violations → Verify approval trails

---

### 20.2 Developer Perspective

**Primary Responsibilities:** Build provisioning automation, create catalog items, manage infrastructure

#### Day-to-Day Tasks

1. **Build Automation**
   - Write Tasks (Python/TS/Bash/Go/SQL/REST)
   - Compose Workflows (multi-step DAGs)
   - Create Flows (chain workflows together)
   - Test with dry-run executions
   - Set up scheduled jobs for recurring tasks

2. **Catalog Management**
   - Design catalog item forms (form builder or BYOUI)
   - Link flows to catalog items
   - Configure approval requirements
   - Set role/group visibility
   - Create reusable form components

3. **Infrastructure Setup**
   - Add cloud credentials (AWS/Azure/GCP)
   - Define resource actions for Day 2 operations
   - Manage shared variables (context)
   - Create Terraform templates
   - Manage Terraform workspaces

4. **Monitoring**
   - Track execution success/failure rates
   - Debug failed executions via step logs
   - Monitor drift detection alerts
   - Review resource inventory

#### Key Workflows

- **New service offering:** Create Task(s) → Build Workflow → Create Flow → Design Catalog Form → Publish → Test Order
- **Terraform deployment:** Create Template → Configure Variables → Deploy Workspace → Set up Drift Detection
- **Custom Day 2 action:** Define Resource Action → Configure conditions → Link to Flow → Set approval rules

---

### 20.3 User Perspective

**Primary Responsibilities:** Order cloud services, manage own resources, track requests

#### Day-to-Day Tasks

1. **Self-Service Ordering**
   - Browse the Service Catalog
   - Fill out request forms
   - Review cost estimates before submission
   - Provide justification for approval-required items
   - Track order status in Executions

2. **Resource Management**
   - View provisioned resources
   - Perform allowed actions (start/stop/restart)
   - Check resource lease/expiry dates
   - Request lease extensions

3. **Approvals**
   - Submit approval requests
   - Track approval status
   - Add comments to pending requests
   - Cancel own pending requests

4. **Credentials**
   - Add personal cloud credentials
   - Select credentials during catalog ordering
   - Validate credential connectivity

#### Key Workflows

- **Order a service:** Browse Catalog → Select Item → Fill Form → Review Cost → Submit → (Approval if needed) → Track Execution → View Resource
- **Manage resource:** View Resources → Select Resource → Check Status → Perform Action (start/stop) → View History

---

### 20.4 Readonly Perspective

**Primary Responsibilities:** View platform state, monitor without making changes

#### Available Actions

- View Dashboard (read-only)
- Browse Service Catalog (cannot order)
- View Approvals (cannot approve/reject)
- View Notifications
- Access AI Assistant for questions

---

### 20.5 Product Owner Perspective

**Primary Responsibilities:** Define service offerings, monitor adoption, control costs

#### Recommended Workflow

1. **Define Service Strategy**
   - Work with developers to identify automatable services
   - Define approval policies for high-risk operations
   - Set cost models and budgets

2. **Monitor Adoption**
   - Review Reports & Analytics for usage patterns
   - Track catalog item popularity
   - Monitor execution success rates
   - Review cost trends

3. **Governance**
   - Define policies aligned with organizational standards
   - Set quotas based on team allocations
   - Configure budget alerts for early warning
   - Review approval queues for bottlenecks

4. **Iterate**
   - Use cost analytics to optimize pricing
   - Adjust policies based on violation patterns
   - Refine catalog items based on user feedback
   - Enable/disable features based on maturity

---

## Appendix A: Complete Page/Tab Reference

| # | Page | Route | Roles | Feature Toggle |
|---|------|-------|-------|----------------|
| 1 | Login | `/login` | Public | — |
| 2 | Dashboard | `/dashboard` | All | — |
| 3 | Service Catalog | `/catalog` | All | `service_catalog` |
| 4 | Catalog Create/Edit | `/catalog/create`, `/catalog/:id/edit` | Admin, Dev | `service_catalog` |
| 5 | Catalog View | `/catalog/:id` | All | `service_catalog` |
| 6 | Tasks | `/tasks` | Admin, Dev | `provisioning` |
| 7 | Workflows | `/workflows` | Admin, Dev | `provisioning` |
| 8 | Flows | `/flows` | Admin, Dev | `provisioning` |
| 9 | Executions | `/executions` | Admin, Dev, User | `executions` |
| 10 | Execution Detail | `/executions/:id` | Admin, Dev, User | `executions` |
| 11 | Catalog Components | `/catalog-components` | Admin, Dev | `provisioning` |
| 12 | Scheduled Jobs | `/scheduled-jobs` | Admin, Dev | `provisioning` |
| 13 | Credentials | `/credentials` | Admin, Dev, User | `credentials` |
| 14 | Resources | `/resources` | Admin, Dev, User | `resources` |
| 15 | Resource Detail | `/resources/:credId/:resId` | Admin, Dev, User | `resources` |
| 16 | Resource Actions | `/resource-actions` | Admin, Dev | `infrastructure` |
| 17 | Shared Variables | `/context` | Admin, Dev | `infrastructure` |
| 18 | Terraform Templates | `/terraform-templates` | Admin, Dev | `terraform_provisioning` |
| 19 | Terraform Workspaces | `/terraform-workspaces` | Admin, Dev | `terraform_provisioning` |
| 20 | Terraform State Backend | `/terraform-state-backend` | Admin | `terraform_provisioning` |
| 21 | Users & Groups | `/admin` | Admin | — |
| 22 | SSO Configuration | `/admin/sso/config` | Admin | — |
| 23 | SSO Group Mappings | `/admin/sso/group-mappings` | Admin | — |
| 24 | Approvals | `/approvals` | All (non-admin top-nav) | `approvals` |
| 25 | Policies | `/policies` | Admin | `policies` |
| 26 | Quotas | `/quotas` | Admin | `quotas` |
| 27 | Cost Models | `/cost-models` | Admin | `cost_models` |
| 28 | Budgets | `/budgets` | Admin | `budgets` |
| 29 | Reports & Analytics | `/reports` | Admin | `reports` |
| 30 | Tenants | `/tenants` | Admin | `tenants` |
| 31 | Logging & Monitoring | `/admin/logging` | Admin | `logging_monitoring` |
| 32 | Inbound Webhooks | `/webhooks` | Admin | `webhooks` |
| 33 | Event Log | `/event-log` | Admin | `event_log` |
| 34 | Event Automation | `/events` | Admin | `event_automation` |
| 35 | Notifications | `/notifications` | All | `notifications` |
| 36 | Profile | `/profile` | All | — |
| 37 | SSO Callback | `/sso/callback` | Public | — |

---

## Appendix B: Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `SECRET_KEY` | JWT signing key | (required) |
| `ALGORITHM` | JWT algorithm | `HS256` |
| `ENCRYPTION_KEY` | Fernet key for credential encryption | (required) |
| `DYNAMODB_ENDPOINT` | DynamoDB endpoint URL | `http://dynamodb-local:8000` |
| `REDIS_URL` | Redis connection URL | `redis://redis:6379` |
| `GEMINI_API_KEY` | Google Gemini API key for AI assistant | (optional) |
| `SMTP_HOST` | SMTP server for email notifications | (optional) |
| `SMTP_PORT` | SMTP port | `587` |
| `SMTP_USER` | SMTP username | (optional) |
| `SMTP_PASSWORD` | SMTP password | (optional) |

---

## Appendix C: Deployment

### Local Development

```bash
# Full stack via Docker Compose
docker compose up -d

# Frontend: http://localhost:3000
# Backend API: http://localhost:8001/docs
```

### Production

- **Backend:** Docker container on AWS ECS/Fargate
- **Frontend:** Static build served via CloudFront CDN
- **Database:** AWS DynamoDB (managed)
- **Cache:** AWS ElastiCache (Redis)
- **CI/CD:** GitHub Actions (deploy-dev.yml, deploy.yml)

---

*This document covers the complete functionality of the Cloud Management Platform as of version 0.2.0. For the latest updates, refer to the application's built-in API documentation at `/docs`.*

# CMP Environment Promoter — Functionality Guide

> Promote CMP configuration artifacts between environments safely, with dependency awareness, conflict detection, and full audit trail.

---

## Overview

The CMP Environment Promoter is a standalone product that enables platform administrators to promote (push) Cloud Management Platform artifacts from one environment to another — for example, dev → test → prod. It provides a unified interface for exporting, comparing, and importing all CMP configuration artifacts while preserving referential integrity between dependent artifacts.

**Key capabilities:**
- Dependency-aware promotion (automatically detects and includes referenced artifacts)
- Field-level conflict detection with resolution options (overwrite, skip, rename)
- ID remapping (source IDs are translated to target IDs, internal references rewritten)
- Selective rollback within a 30-day window
- Full environment bulk promotion with runtime data exclusion
- Complete audit trail with per-artifact outcome tracking
- Fernet-encrypted credential storage for environment connections
- Standalone deployment — communicates with CMP exclusively via its public REST API

**What it does NOT promote:**
- Runtime data (executions, audit logs, events, user login logs, notifications)
- Sensitive credential values (secrets are replaced with placeholders requiring manual entry)

---

## Key Concepts

### Environments

An **Environment** is a running CMP instance (e.g., `dev.company.com`, `test.company.com`, `prod.company.com`). Each environment has its own database, users, and configuration. The Promoter connects to environments via their public REST API using configured **Environment Connections**.

### Environment Connections

An **Environment Connection** stores the details needed to communicate with a CMP instance:
- **Name** — unique per user (1–100 characters)
- **Base URL** — the CMP instance URL (HTTP or HTTPS)
- **API Token** — authentication credential (stored Fernet-encrypted)
- **Tenant ID** — which tenant to access in that environment

Users can configure up to 10+ connections and test connectivity before use.

### Artifacts

An **Artifact** is any CMP configuration entity that can be promoted between environments. The Promoter supports 12 artifact types (see [Supported Artifact Types](#supported-artifact-types)).

### Artifact Bundles

An **Artifact Bundle** is a packaged collection of artifacts exported from a source environment. It contains:
- An **Artifact Manifest** (metadata: types, IDs, names, timestamps, dependency relationships)
- The full artifact payloads (with sensitive fields replaced by placeholders)
- Maximum 500 artifacts per bundle, 50 MB size limit

Bundles can be downloaded as a single JSON file for offline transfer or archival.

### Promotion Plans

A **Promotion Plan** is a preview of changes that will be applied to the target environment. Before any modifications are made, the plan categorizes each artifact as:
- **Create** — artifact does not exist in the target
- **Update** — artifact exists but has field-level differences
- **Unchanged** — artifact exists with identical values (no action needed)
- **Conflict** — artifact matches by ID but not name, or by name but not ID

### Promotion Sessions

A **Promotion Session** tracks a single promotion operation from start to finish. It records:
- Who initiated it, when, and between which environments
- Per-artifact outcomes (success, skipped, failed)
- The ID mapping generated (source ID → target ID)
- Final status (completed, partially_completed, failed)

Sessions are retained for a minimum of 90 days.

### ID Mappings

An **ID Mapping** translates artifact identifiers from the source environment to corresponding identifiers in the target environment. When an artifact is created in the target, it gets a new unique ID. The mapping ensures all internal references (e.g., a catalog item pointing to a flow) are rewritten to use the correct target IDs.

---

## Supported Artifact Types

The Promoter supports 12 artifact types organized by their role in the CMP platform:

| # | Artifact Type | Description | Has Dependencies |
|---|---------------|-------------|------------------|
| 1 | **Task** | Individual automation steps (scripts, commands) | No (leaf node) |
| 2 | **Workflow** | Orchestrated sequences of tasks and sub-workflows | Yes → Tasks, Workflows |
| 3 | **Flow** | Higher-level orchestration containing workflows | Yes → Workflows |
| 4 | **Catalog Item** | Self-service catalog entries with forms and flows | Yes → Flows, Catalog Components |
| 5 | **Catalog Component** | Reusable form components for catalog items | No (leaf node) |
| 6 | **Policy** | Governance rules applied to catalog items | Yes → Catalog Items |
| 7 | **Credential** | Cloud provider credentials (secrets excluded) | No (leaf node) |
| 8 | **Budget** | Cost budgets and thresholds | No |
| 9 | **Dashboard** | Custom monitoring dashboards | No |
| 10 | **Scheduled Job** | Cron-based automation schedules | No |
| 11 | **Cost Model** | Cost calculation and chargeback models | No |
| 12 | **Resource Action** | Day 2 operations on provisioned resources | No |

### Dependency Relationships

```
Policy ──────────► Catalog Item ──────────► Flow ──────────► Workflow ──────────► Task
  │                     │                                        │
  │                     ├──► Approval Flows (approve/reject/cancel)
  │                     │                                        ├──► Sub-Workflow
  │                     └──► Catalog Component                   └──► Task
  │
  └──► Catalog Item (by tag — optional)
```

**Dependency extraction rules:**

| Parent Type | Reference Field | Target Type | Classification |
|-------------|----------------|-------------|----------------|
| Workflow | `steps[].task_id` (action="run_task") | Task | Required |
| Workflow | `steps[].inputs.workflow_id` (action="builtin.subworkflow") | Workflow | Required |
| Flow | `workflows[].workflow_id` | Workflow | Required |
| Catalog Item | `flow_id` | Flow | Required |
| Catalog Item | `on_approve_flow_id` | Flow | Required |
| Catalog Item | `on_reject_flow_id` | Flow | Required |
| Catalog Item | `on_cancel_flow_id` | Flow | Required |
| Catalog Item | `form_schema.fields[].reusable_component_id` | Catalog Component | Required |
| Policy | `catalog_ids[]` | Catalog Item | Required |
| Policy | `catalog_tags[]` | Catalog Item (by tag) | Optional |

---

## Promotion Workflow

The complete promotion workflow follows these steps:

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│  1. Connect     │────►│  2. Select Artifacts │────►│  3. Resolve     │
│  (Setup envs)   │     │  (Browse & pick)     │     │  Dependencies   │
└─────────────────┘     └──────────────────────┘     └────────┬────────┘
                                                              │
┌─────────────────┐     ┌──────────────────────┐     ┌───────▼────────┐
│  6. Audit       │◄────│  5. Execute Plan     │◄────│  4. Generate   │
│  (Track results)│     │  (Import to target)  │     │  Plan & Review │
└─────────────────┘     └──────────────────────┘     └────────────────┘
```

### Step 1: Environment Connection Setup

1. Create connections to your source and target CMP environments
2. Provide: name, base URL, API token, tenant ID
3. The Promoter validates connectivity (10-second timeout health check)
4. Credentials are stored encrypted (Fernet)
5. Test connections at any time via the "Test Connection" button

### Step 2: Artifact Selection

1. Select a source environment from your configured connections
2. Browse artifacts grouped by type (12 categories)
3. View artifact metadata: name, description, tags, last modified date
4. Select individual artifacts or "select all" for a given type
5. Search by name or tag (case-insensitive, minimum 2 characters)
6. Paginated display (20 items per page by default)

### Step 3: Dependency Resolution

1. The Promoter automatically computes the full dependency graph
2. Dependencies are resolved recursively up to 10 levels deep
3. Each dependency is labeled:
   - **Required** — directly referenced by ID (e.g., `flow_id` in a catalog item)
   - **Optional** — referenced by tag or pattern (e.g., `catalog_tags` in a policy)
4. You can deselect optional dependencies
5. Deselecting a required dependency shows a warning listing broken references
6. Missing dependencies (not found in source) are marked as "unresolved"

### Step 4: Generate Promotion Plan

1. The Promoter compares selected artifacts against the target environment
2. Each artifact is categorized: create, update, unchanged, or conflict
3. For updates: a field-level diff shows exactly what will change
4. For conflicts: you choose a resolution strategy per artifact
5. Plan generation completes within 30 seconds
6. The plan is validated — if the target changes after plan generation, you must regenerate

### Step 5: Execute Promotion

1. Confirm the plan to begin execution
2. Artifacts are imported in dependency order (tasks first, then workflows, etc.)
3. New IDs are generated in the target; all internal references are remapped
4. Credential artifacts are created with empty sensitive fields (status: "requires_configuration")
5. If an artifact fails, its dependents are skipped; independent artifacts continue
6. A promotion session is created tracking every outcome

### Step 6: Audit and Review

1. View the promotion session with full details
2. See per-artifact outcomes: success, skipped, or failed (with error messages)
3. Review the ID mapping (source ID → target ID)
4. Session history is paginated, filterable by date range and environment
5. Sessions are retained for 90 days minimum

---

## Dependency Resolution

### How It Works

When you select artifacts for promotion, the Promoter automatically identifies all artifacts that your selection depends on. This prevents promoting incomplete configurations that would break in the target environment.

**Example:** Selecting a catalog item automatically pulls in:
- Its linked flow
- All workflows in that flow
- All tasks referenced by those workflows
- Any approval flows (approve, reject, cancel)
- Any reusable catalog components in the form

### Resolution Rules

- **Recursive traversal** — dependencies of dependencies are included (up to 10 levels)
- **Required vs Optional** — ID-based references are required; tag-based references are optional
- **Deselection warnings** — removing a required dependency shows which artifacts will have broken references
- **Unresolved markers** — if a referenced artifact doesn't exist in the source, it's flagged but doesn't block promotion
- **Depth limit** — traversal stops at 10 levels to prevent infinite loops in circular reference scenarios

### Dependency Classification

| Classification | Meaning | User Action |
|----------------|---------|-------------|
| **Required** | Parent artifact directly references this by ID | Cannot deselect without warning |
| **Optional** | Parent references this via tag/pattern matching | Can freely deselect |
| **Unresolved** | Referenced artifact not found in source | Warning displayed; promotion continues |
| **Missing** | Dependency cannot be retrieved from source | Warning displayed |

---

## Conflict Detection and Resolution

### What Is a Conflict?

A conflict occurs when an artifact in your bundle collides with an existing artifact in the target environment in an ambiguous way:

- **ID match, name mismatch** — same internal ID exists in target but with a different name
- **Name match, ID mismatch** — same name exists in target but with a different internal ID

This is distinct from a simple "update" (same ID and name, different field values).

### Resolution Strategies

| Strategy | What Happens | When to Use |
|----------|-------------|-------------|
| **Overwrite Target** | Replace the target artifact entirely with the source version | You want the source version to win |
| **Skip Artifact** | Leave the target artifact unchanged; do not import this artifact | The target version is correct |
| **Rename in Target** | Import the source artifact with a new unique name (max 128 chars) | You want both versions to coexist |

### Conflict Resolution Workflow

1. Generate the promotion plan
2. Conflicts are highlighted in the plan review screen
3. For each conflict, choose one of the three strategies
4. Provide a new name if choosing "Rename in Target"
5. The new name must be unique within the target environment and ≤ 128 characters
6. Confirm the plan to proceed with your chosen resolutions

---

## Rollback

### Overview

The Promoter supports selective rollback of completed promotions within a 30-day window. Before any artifact is modified during promotion, a snapshot of its pre-promotion state is stored. This enables reverting changes if something goes wrong.

### 30-Day Rollback Window

- Rollback is available for 30 days after a promotion session completes
- After 30 days, snapshots expire and rollback is no longer possible
- The Promoter displays whether a session is within or outside the rollback window

### What Rollback Does

| Original Action | Rollback Action |
|-----------------|-----------------|
| Artifact was **updated** (overwritten) | **Restore** to pre-promotion state from snapshot |
| Artifact was **created** (new) | **Delete** from target environment |

### Rollback Workflow

1. Navigate to Promotion History
2. Select a completed session (within 30 days)
3. Click "Rollback" to generate a rollback plan
4. Review the plan: which artifacts will be restored, which will be deleted
5. Confirm to execute the rollback
6. The rollback is recorded as a new promotion session linked to the original

### Rollback Statuses

| Status | Meaning |
|--------|---------|
| **rolled_back** | All artifacts successfully reverted |
| **partially_rolled_back** | Some artifacts failed to revert (failures are logged) |

### Failure Handling

- If a specific artifact fails to rollback, the error is recorded
- Remaining artifacts continue processing (rollback doesn't stop on failure)
- The final session shows which artifacts succeeded and which failed

---

## Bulk Promotion Mode

### Full Environment Promotion

Bulk promotion mode selects **all promotable artifact types** from the source environment in a single operation. This is useful for:
- Replicating a complete environment configuration (e.g., setting up a new test environment)
- Standardizing configuration across environments
- Disaster recovery (restoring a full environment from a known-good source)

### Included Artifact Types

All 12 promotable types are selected by default:
- Tasks, Workflows, Catalog Items, Catalog Components, Flows, Policies
- Credentials, Budgets, Dashboards, Scheduled Jobs, Cost Models, Resource Actions

### Excluded Runtime Data

The following are **never** included in bulk promotion (they are environment-specific runtime data):
- Executions (workflow/task run history)
- Audit logs
- Events
- User login logs
- Notifications

### Customization

- You can **deselect** one or more artifact types before executing bulk promotion
- All other promotion features apply: dependency resolution, conflict detection, ID remapping

### Processing Order

Bulk promotion processes artifacts in dependency order to ensure references resolve correctly:

```
1. Tasks
2. Workflows
3. Flows
4. Catalog Items
5. Policies
6. Credentials
7. Budgets
8. Dashboards
9. Scheduled Jobs
10. Cost Models
11. Resource Actions
12. Catalog Components
```

### Progress Tracking

During bulk promotion:
- A progress indicator shows the current artifact type being processed
- Overall completion percentage is displayed
- Progress updates each time processing advances to a new artifact type
- Final summary shows succeeded, skipped, and failed counts by type

### Partial Failure Handling

- If artifacts fail during bulk promotion, they are skipped
- Processing continues with remaining artifacts and types
- The session status is set to "partially_completed"
- A summary report lists all failures with artifact type and identifier

---

## Security

### Authentication

- Users authenticate via JWT tokens obtained from CMP's `POST /api/v1/auth/login` endpoint
- Authentication is required for both source and target environments
- The **admin** role is required in both environments to perform promotions
- Token refresh is attempted up to 3 times (5-second intervals) if a token expires during promotion
- If refresh fails, the operation halts and the user must re-authenticate

### Authorization Validation Order

The Promoter validates access in this specific sequence:
1. Authenticate with Source Environment
2. Authenticate with Target Environment
3. Verify "admin" role in Source Environment
4. Verify "admin" role in Target Environment
5. Verify tenant_id access in both environments

### Credential Storage

- Environment connection API tokens are encrypted at rest using **Fernet symmetric encryption**
- The encryption key is configured via the `ENCRYPTION_KEY` environment variable
- Tokens are decrypted only when making API calls to CMP environments
- Sensitive fields in promoted credential artifacts are **never** stored or transferred in plaintext

### Tenant Isolation

- All Promoter data is scoped to `tenant_id` in DynamoDB partition keys
- Users can only access connections and sessions within their tenant
- The Promoter verifies the user has access to the specified tenant_id in both CMP environments

---

## Audit Trail

### Session Tracking

Every promotion operation creates a **Promotion Session** that captures:
- **Who** — the initiating user's identifier
- **When** — UTC timestamp of initiation and completion
- **Where** — source and target environment names
- **What** — every artifact promoted with its individual outcome
- **Result** — final session status and any error messages

### Per-Artifact Outcomes

Each artifact in a session has one of three outcomes:

| Outcome | Meaning |
|---------|---------|
| **Success** | Artifact was created or updated in the target |
| **Skipped** | Artifact was intentionally skipped (conflict resolution) or skipped due to a failed dependency |
| **Failed** | Artifact import failed (error message recorded) |

### Session Statuses

| Status | Condition |
|--------|-----------|
| **completed** | All artifacts succeeded |
| **partially_completed** | At least one succeeded and at least one failed |
| **failed** | All artifacts failed |
| **rolled_back** | Rollback completed successfully |
| **partially_rolled_back** | Rollback completed with some failures |

### History and Filtering

- Sessions are displayed in a paginated list (max 50 per page)
- Sorted by timestamp descending (most recent first)
- Filterable by:
  - Date range
  - Source environment
  - Target environment

### Retention

- Promotion session records are retained for a minimum of **90 days**
- Rollback snapshots are retained for **30 days** (the rollback window)
- Write failures for audit records are retried up to 3 times before logging the failure

---

## UI Screens

### Connections Page

Manage environment connections:
- List all configured connections with status indicators
- Create new connections with form validation
- Edit existing connections
- Test connectivity with real-time feedback
- Delete connections (blocked if referenced by in-progress promotions)

### Artifact Selection Page

Browse and select artifacts for promotion:
- Source environment dropdown selector
- Artifact type category browser (12 types grouped)
- Paginated artifact list with metadata (name, description, tags, last modified)
- Individual and bulk selection controls
- Search bar with instant filtering
- Dependency graph visualization after selection

### Promotion Plan Page

Review and confirm the promotion plan:
- Categorized artifact list (create/update/unchanged/conflict)
- Field-level diff viewer for updates
- Conflict resolution interface (overwrite/skip/rename)
- Stale plan warning with regeneration prompt
- Execute button with confirmation

### Promotion History Page

View past promotion sessions:
- Paginated session list with status badges
- Date range and environment filters
- Session detail view with per-artifact outcomes
- Rollback button (for eligible sessions)

### Rollback View

Execute and monitor rollbacks:
- Rollback plan showing restore and delete actions
- Window status indicator (available/expired)
- Confirmation dialog
- Results display with per-artifact outcomes

---

## Standalone Architecture

The CMP Environment Promoter is a fully independent application:

| Aspect | Details |
|--------|---------|
| **Codebase** | Separate `cmp_promoter/` workspace folder |
| **Backend** | Python 3.12, FastAPI, DynamoDB (single-table design) |
| **Frontend** | React 18, TypeScript, Vite, Tailwind CSS |
| **Communication** | CMP public REST API only — no direct DB access |
| **Storage** | Own DynamoDB table for connections, sessions, mappings, snapshots |
| **Deployment** | Own Docker Compose, CI/CD pipeline, versioning |
| **Versioning** | Independent semantic versioning (`cmp_promoter/VERSION`) |

---

## Limits and Constraints

| Constraint | Value |
|------------|-------|
| Max artifacts per bundle | 500 |
| Max bundle file size | 50 MB |
| Dependency resolution depth | 10 levels |
| Connection health check timeout | 10 seconds |
| Promotion plan generation timeout | 30 seconds |
| Token refresh retries | 3 attempts (5-second intervals) |
| Connections per user | 10+ |
| Session history page size | 50 per page |
| Artifact list page size | 20 per page (default) |
| Session retention | 90 days minimum |
| Rollback window | 30 days |
| Connection name max length | 100 characters |
| Rename max length | 128 characters |
| Search query minimum length | 2 characters |
| Audit write retries | 3 attempts |

# Spacelift vs CMP: Feature Comparison & Implementation Plan

**Created:** June 22, 2026  
**Purpose:** Reference document for adopting Spacelift-inspired features into the Cloud Management Platform  
**Source:** Analysis of Spacelift platform (spacelift.io), documentation (docs.spacelift.io), and CMP codebase

---

## Table of Contents

1. [Spacelift Platform Overview](#1-spacelift-platform-overview)
2. [Spacelift Core Concepts](#2-spacelift-core-concepts)
3. [Feature-by-Feature Comparison](#3-feature-by-feature-comparison)
4. [Features CMP Should Adopt](#4-features-cmp-should-adopt)
5. [Detailed Implementation Designs](#5-detailed-implementation-designs)
6. [CMP's Existing Advantages Over Spacelift](#6-cmps-existing-advantages-over-spacelift)
7. [Implementation Roadmap](#7-implementation-roadmap)
8. [Data Models](#8-data-models)
9. [API Contracts](#9-api-contracts)
10. [Frontend Requirements](#10-frontend-requirements)

---

## 1. Spacelift Platform Overview

### What Is Spacelift?

Spacelift is an infrastructure orchestration platform focused on IaC management. It supports
Terraform, OpenTofu, Terragrunt, CloudFormation, Pulumi, Kubernetes, and Ansible. It integrates
with VCS providers and provides governance, policy enforcement, and self-service capabilities.

### Spacelift's Three Pillars

| Pillar | Description |
|--------|-------------|
| **Provision** | Stacks, Runs, Stack Dependencies, Drift Detection, State Management, Templates/Blueprints, Intent (AI) |
| **Configure** | Contexts, Env Variables, Worker Pools, VCS Integrations, Cloud Integrations, Webhooks, Module Registry, Plugins |
| **Govern** | OPA Policies, RBAC, SSO/SAML, Approval Workflows, Audit Logs, Spaces, Resources View, Dashboard |

### Spacelift's Target Users

- Platform Engineering teams managing IaC at scale
- DevOps teams needing governed self-service
- Security teams requiring policy enforcement on infrastructure changes
- Developers who need to deploy infrastructure without deep IaC knowledge

---

## 2. Spacelift Core Concepts

### 2.1 Stacks

A **Stack** is the fundamental unit in Spacelift. It combines:
- Source code (a directory in a git repo containing IaC)
- State (Terraform state file, managed by Spacelift)
- Configuration (environment variables, mounted files, attached contexts)

Stacks are isolated and independent. Each stack tracks one Terraform root module.

**Stack States:** None → Initializing → Active (Finished) → Failed → Errored

**Stack Settings:**
- Repository + branch + project root (subfolder)
- IaC vendor (Terraform, Pulumi, CloudFormation, etc.)
- Runner image (Docker-based execution environment)
- Administrative flag (can manage other Spacelift resources)
- Auto-deploy toggle (auto-apply on tracked branch push)
- Auto-retry toggle (retry failed runs automatically)
- Labels (for filtering, context auto-attachment, policy targeting)

### 2.2 Runs

A **Run** is a single execution cycle on a stack. Two primary types:

| Run Type | Trigger | What It Does | Can Apply? |
|----------|---------|--------------|------------|
| **Proposed** | Push to non-tracked branch (PR) | Runs `terraform plan` only | No (preview only) |
| **Tracked** | Push to tracked branch (merge) or manual trigger | Runs `terraform plan` + optional `terraform apply` | Yes |

**Run Phases:**
1. Preparing
2. Initializing (terraform init)
3. Planning (terraform plan)
4. [Proposed stops here]
5. Awaiting approval (if policies require it)
6. Applying (terraform apply)
7. Finished / Failed

**Run Serialization:** All state-changing operations on a stack are serialized. No parallel applies.

### 2.3 Run Promotion

Convert a proposed run (plan preview from a PR) into a tracked run (apply). The exact plan
that was reviewed gets applied — no re-planning. Available from the Spacelift UI or from
GitHub PR "Deploy" button.

**Prerequisites:**
- Run Promotion enabled in stack settings
- The commit to be promoted is newer than the stack's current commit

### 2.4 Stack Dependencies

Stacks can form a **Directed Acyclic Graph (DAG)** of dependencies:

```
Infrastructure → Database → PaymentService
              → Network  → CartService
              → Storage  → StorageService
```

**Key behaviors:**
- When a parent stack finishes a tracked run, child stacks are automatically triggered
- Parent stack outputs can be passed as environment variables to child stacks
- If trigger condition is "on_output_change" (default), child only runs if referenced output changed
- If trigger condition is "always", child runs regardless
- If a parent run fails, the entire downstream chain is cancelled
- Parallel execution: independent siblings run in parallel
- No cycles allowed (DAG enforcement)
- Stack cannot be deleted if it has downstream dependencies

**Output References:**
- Parent stack defines Terraform outputs
- Child stack maps parent outputs to environment variables (e.g., `vpc_id` → `TF_VAR_vpc_id`)
- Sensitive outputs supported (requires explicit opt-in)

### 2.5 Contexts

A **Context** is a reusable bundle of configuration that can be attached to multiple stacks:
- Environment variables (plain text)
- Secret variables (write-only, encrypted)
- Mounted files (injected into the run environment)
- Hooks (shell commands run at specific phases)

**Context features:**
- Priority system: higher priority contexts override lower ones
- Auto-attach via labels: "Attach this context to any stack labeled `env:production`"
- Manual attachment to specific stacks
- Changes to a context affect all attached stacks on their next run

### 2.6 Policies (OPA/Rego-Based)

Spacelift uses Open Policy Agent with Rego language for policy-as-code. Policy types:

| Policy Type | When Evaluated | What It Controls |
|-------------|---------------|-----------------|
| **Login Policy** | User authentication | Who can access Spacelift, what role they get |
| **Access Policy** | Stack access check | Who can read/write/admin a stack (deprecated for Spaces) |
| **Push Policy** | Git push event | Should this push trigger a run? Which type? |
| **Plan Policy** | After terraform plan | Warn/deny/require approval based on plan contents |
| **Approval Policy** | Before apply phase | How many approvals needed, who can approve |
| **Trigger Policy** | After a run completes | Should other stacks be triggered? |
| **Notification Policy** | On run events | Where to send notifications |
| **Intent Policy** | Before Intent execution | Validate AI-generated infrastructure requests |

**Plan Policy example (Rego):**
```rego
package spacelift

# Deny any plan that creates a public S3 bucket
deny["S3 bucket must not be public"] {
    some resource
    input.terraform.resource_changes[resource].type == "aws_s3_bucket"
    input.terraform.resource_changes[resource].change.after.acl == "public-read"
}

# Require approval for production changes
warn["Production change requires review"] {
    input.spacelift.stack.labels[_] == "env:production"
    input.terraform.resource_changes[_].change.actions[_] == "delete"
}
```

### 2.7 Blueprints

A **Blueprint** is a template for creating a stack with pre-configured settings:
- Which repository and branch to use
- Which policies to attach
- Which contexts to include
- Default variable values
- Labels to apply
- Worker pool selection

Blueprints enable admins to define "golden paths" — standardized, governed ways to
deploy infrastructure. End users fill in a simple form and get a fully configured stack.

### 2.8 Templates (Self-Service)

**Templates** go beyond Blueprints — they provide a form-based interface for developers:
- Platform teams define what can be deployed and how
- Developers deploy through a simplified form (no IaC knowledge needed)
- Templates can create multiple stacks in a single deployment
- Templates Workbench for platform team authoring and version management
- Deployment tracking: see all instances created from a template

### 2.9 Intent (AI-Powered Provisioning)

**Spacelift Intent** enables natural language infrastructure provisioning:
- User describes infrastructure needs in plain English
- AI generates the appropriate IaC code
- Standard governance policies are enforced before execution
- No HCL or Terraform knowledge required
- Agentic model: the AI acts as an infrastructure operator

**Intent flow:**
1. User types natural language request
2. Intent agent generates Terraform/IaC
3. Intent Policy evaluates the request (OPA/Rego)
4. Plan is generated and shown to user
5. Standard approval workflow applies
6. Apply executes with full audit trail

### 2.10 Drift Detection

Spacelift's drift detection:
- Periodically runs `terraform plan -refresh-only` on stable (FINISHED) stacks
- Configurable schedule per stack (cron expression)
- Optional auto-reconciliation: if drift detected, automatically apply to restore desired state
- Drift runs don't trigger dependent stacks
- Notifications sent when drift is detected
- Integration with run history for audit trail

### 2.11 Worker Pools

Worker Pools allow running Terraform executions in customer infrastructure:
- Public worker pool (Spacelift-managed, shared)
- Private worker pools (customer-hosted, Docker or Kubernetes)
- Network isolation: runs can access private VPCs/networks
- Custom runner images with pre-installed tools
- Pool scheduling and capacity management
- Auto-scaling based on queue depth

### 2.12 Module Registry

Private Terraform module registry:
- Publish modules from git repositories
- Version management via git tags
- Module test cases (automated testing)
- Usage tracking across stacks
- Documentation auto-generation
- Integration with stack configuration (source modules from registry)

### 2.13 Spaces (Organizational Hierarchy)

Spaces provide hierarchical organizational boundaries:
- Tree structure: Root → Child → Grandchild
- Policy inheritance: child spaces inherit parent policies
- Access control scoped to spaces
- Stacks belong to exactly one space
- Contexts and policies can be scoped to a space
- Logical separation: by team, environment, project

### 2.14 Plugins

First-class integration framework:
- **Infracost**: Cost estimation on every plan
- **Checkov**: Security scanning of IaC
- **Wiz**: Cloud security posture checks
- Custom plugins via run hooks
- Plugin results shown in run UI

### 2.15 Notifications & Integrations

- Slack integration (run status, approvals)
- Microsoft Teams integration
- Webhook notifications (outbound)
- Datadog/Prometheus observability
- ServiceNow integration
- Backstage integration
- GraphQL API for automation

---

## 3. Feature-by-Feature Comparison

### Current State: What CMP Has vs What Spacelift Has

| Feature Area | CMP (Current) | Spacelift | Gap |
|-------------|---------------|-----------|-----|
| **IaC Vendors** | Terraform only | Terraform, OpenTofu, Terragrunt, Pulumi, CloudFormation, K8s, Ansible | Multi-vendor support |
| **Template Management** | Templates with inline/git/upload/registry source, versioning | Similar, plus module registry | Comparable |
| **Workspace/Stack** | Workspaces with state machine, variable_values, credential linking | Stacks with richer configuration model | Contexts, hooks |
| **Runs/Executions** | Direct apply (no plan preview separation) | Proposed (plan) + Tracked (apply) with promotion | Plan preview |
| **Dependencies** | None between workspaces | Full DAG with output passing | Major gap |
| **Drift Detection** | On-demand + alert-based, accept/remediate | Scheduled + on-demand, optional auto-reconcile | Scheduling |
| **Policies** | Custom policy engine (rules-based) | OPA/Rego policies at every lifecycle point | Plan policies |
| **Self-Service** | Full catalog with cart, ordering, forms | Blueprints + Templates | CMP is richer |
| **RBAC** | 4-level hierarchy + groups + SSO mapping | Spaces + role bindings + IdP groups | Comparable |
| **Approval** | Multi-level with SLA tracking, conditional | Policy-driven, simpler | CMP is richer |
| **Audit** | Full audit logs + event system | Audit trail integration | Comparable |
| **VCS Integration** | Git source parsing for templates | Deep VCS integration (auto-trigger runs on push) | GitOps gap |
| **AI** | Gemini assistant with TF generation, inline suggestions | Intent (natural language → deploy) | Different focus |
| **Cost** | Full cost management, budgets, analytics | Infracost plugin only | CMP is far richer |
| **Day-2 Operations** | Native + Flow + Terraform backends, approval, conditions | Re-run IaC only (no native cloud API actions) | CMP is far richer |
| **Resource Inventory** | Multi-cloud discovery, detail views, actions | Terraform state viewer only | CMP is richer |
| **Event Automation** | 60+ events, configurable rules, webhooks | Webhooks + notification policies | CMP is richer |
| **Multi-Tenancy** | Full tenant isolation, switching, branding | Single account with Spaces | CMP is richer |
| **Worker/Execution Env** | Local Docker-based execution | Private worker pools (Docker/K8s) | Worker pools |
| **Module Registry** | Basic module management | Full private registry with versioning | Enhancement needed |
| **Notifications** | Multi-channel (email, webhook, in-app) | Slack, Teams, webhook, custom policies | Comparable |
| **Scheduling** | Scheduled jobs system (separate from TF) | Per-stack scheduling (drift, runs, destroy) | TF-specific scheduling |

---

## 4. Features CMP Should Adopt

### Tier 1: High-Impact, Core Infrastructure Orchestration

#### 4.1 Workspace Dependencies (Stack Dependencies)

**What Spacelift does:** DAG-based relationships between stacks. Parent outputs flow into child
inputs as environment variables. Automatic cascade triggering on parent completion.

**Why CMP needs this:** Currently workspaces are isolated. Complex deployments (VPC → Subnet →
EC2 → ALB) require manual coordination. Users must deploy in order and manually copy outputs.

**CMP Implementation Design:**

Introduce a `WorkspaceDependency` model and a dependency resolution engine:

- Workspaces can declare dependencies on other workspaces
- Output references map parent outputs to child input variables
- When a workspace completes a Day-2 or initial apply, check for dependents
- Dependent workspaces auto-trigger with injected variables
- Failure in parent cancels downstream chain
- Visual dependency graph in UI
- Deletion protection: cannot delete workspace with dependents

**Key behaviors to implement:**
1. DAG validation (no cycles)
2. Parallel execution of independent siblings
3. Output value capture and injection
4. Chain cancellation on failure
5. Force-trigger (ignore output change detection)
6. Cascade destroy (reverse order)

---

#### 4.2 Proposed Runs / Plan Preview + Promotion

**What Spacelift does:** Every change starts as a plan-only "proposed run". Users review the diff.
If approved, it's "promoted" to a tracked run that applies. The exact reviewed plan is applied.

**Why CMP needs this:** Currently Day-2 operations go directly to apply. Users cannot preview
what will change. Approvers approve based on variable values, not actual infrastructure impact.

**CMP Implementation Design:**

Extend the workspace execution model with two-phase runs:

```
Phase 1: Plan (proposed)
  - Runs terraform plan
  - Captures plan output (JSON + human-readable)
  - Stores plan file for later apply
  - Shows resource changes (create/modify/destroy counts)
  - Calculates cost impact
  - Evaluates plan policies

Phase 2: Apply (tracked) — only after approval/confirmation
  - Uses stored plan file (terraform apply planfile)
  - No re-planning (what you saw is what gets applied)
  - Full execution tracking with step logs
```

**User experience:**
1. User triggers Day-2 variable update
2. System shows: "Planning... 2 resources to modify, 1 to create"
3. User sees detailed diff (like terraform plan output)
4. User clicks "Confirm & Apply" or "Discard"
5. If approval required: approver sees the plan, approves/rejects
6. Apply uses the exact plan file

---

#### 4.3 Contexts (Shared Configuration Bundles)

**What Spacelift does:** Reusable bundles of env vars, secrets, and files that attach to multiple
stacks. Auto-attachment via labels. Priority-based override resolution.

**Why CMP needs this:** Currently each workspace has its own variable_values. Common settings
(region defaults, tagging standards, compliance vars) must be duplicated across workspaces.

**CMP Implementation Design:**

New domain: `TerraformContext`

- Bundle of key-value pairs (plain + encrypted/secret)
- Attachable to workspaces (many-to-many)
- Auto-attach rules based on workspace labels/tags
- Priority system for conflict resolution
- Changes propagate: updating a context value affects next run of all attached workspaces
- Separate from existing CMP "context" (shared variables) — this is Terraform-specific

---

#### 4.4 Plan Policies (Evaluate Terraform Plan Output)

**What Spacelift does:** After `terraform plan`, the plan JSON is evaluated against OPA/Rego
policies that can WARN, DENY, or REQUIRE_APPROVAL.

**Why CMP needs this:** Current policies evaluate resource metadata (tags, types, regions) but
cannot inspect what Terraform will actually do. A policy like "deny if plan destroys more than
5 resources" is impossible today.

**CMP Implementation Design:**

Extend the policy engine with a `plan_evaluation` policy type:

- After plan phase, extract plan JSON
- Evaluate against policy rules
- Rules can inspect: resource_changes, output_changes, variable values
- Actions: allow, warn (proceed with warning), deny (block apply), require_approval
- Pre-built rule templates for common scenarios
- Custom rules via JSON-based condition language (simpler than Rego)

**Example rules:**
- Deny: Plan destroys any resource tagged `protected=true`
- Deny: Plan creates publicly accessible RDS instance
- Require Approval: Plan modifies more than 10 resources
- Warn: Plan creates resources without `cost-center` tag
- Deny: Plan removes encryption from any storage resource

---

#### 4.5 Blueprints / Infrastructure Templates (Self-Service)

**What Spacelift does:** Blueprints are stack-creation templates. Templates go further with
form-based self-service for non-IaC-savvy users.

**Why CMP needs this:** CMP already has catalog items, but they're generic. A dedicated
"Infrastructure Blueprint" type that pre-configures workspace settings, policies, contexts,
and post-deploy actions would provide a richer governed self-service experience.

**CMP Implementation Design:**

New catalog type: `infrastructure_blueprint`

- Pre-selects: template, version, contexts, policies, credential
- Locks certain variables (user can't change them)
- Exposes only relevant variables as a simple form
- Attaches post-deploy resource actions automatically
- Sets drift detection schedule
- Configures dependency chains (deploy A then B then C)
- Governance: max instances per user, required approval tier, budget check

---

### Tier 2: Medium-Impact Differentiators

#### 4.6 Intent-Based Provisioning (Natural Language → Infrastructure)

**What Spacelift does:** Users describe infrastructure in natural language. AI generates IaC,
creates a stack, and runs it — all under standard governance.

**Why CMP should enhance this:** CMP already has AI-generated Terraform. The gap is the
end-to-end flow: natural language → governed deployment → tracked resources.

**CMP Enhancement:**
- AI assistant generates Terraform code from natural language
- System creates a temporary template from the generated code
- Creates a proposed run (plan preview) → user reviews
- If user confirms, triggers full deployment flow (approval, apply, inventory registration)
- Generated code can be saved as a reusable template for future use
- Intent policies: define what AI is allowed to create (no public databases, enforce encryption)

---

#### 4.7 Workspace Scheduling

**What Spacelift does:** Cron-based scheduling for: drift detection, periodic applies (GitOps
reconciliation), timed destroy (ephemeral environments).

**Why CMP needs this:** Currently drift detection is on-demand only. Scheduled jobs exist but
aren't tightly integrated with workspace lifecycle.

**CMP Implementation Design:**

Per-workspace schedule configuration:

| Schedule Type | Description | Example |
|--------------|-------------|---------|
| `drift_detection` | Run periodic drift scan | Every 4 hours on production |
| `auto_apply` | Periodic reconciliation apply | Daily at 2am (ensure state matches code) |
| `auto_destroy` | Destroy after TTL expires | Destroy dev workspace after 72 hours |
| `variable_update` | Timed variable changes | Scale down to 1 instance at 8pm daily |
| `snapshot` | Create state snapshot/backup | Weekly state backup |

Integration with existing CMP scheduled jobs system — workspace schedules become a specialized
scheduled job type with Terraform-specific semantics.

---

#### 4.8 Trigger Policies (Cross-Workspace Orchestration Rules)

**What Spacelift does:** After a run completes, Rego-based trigger policies evaluate whether
other stacks should be triggered. Enables complex multi-stack workflows beyond simple DAG deps.

**Why CMP needs this:** Workspace dependencies (4.1) cover simple DAG cases. Trigger policies
handle advanced scenarios: "if workspace A deploys to production AND output `api_url` changed,
trigger workspace B only if B's last run was more than 1 hour ago."

**CMP Implementation (simplified, rule-based):**

```
WorkspaceTriggerRule:
  source_workspace_id: str
  target_workspace_id: str
  trigger_on: "success" | "failure" | "output_change" | "always"
  conditions:
    - output_name: "api_endpoint"  # only trigger if this output changed
    - min_interval_minutes: 60     # don't re-trigger within 1 hour
    - source_labels_match: ["env:production"]  # only if source has this label
  inject_outputs: bool
  override_variables: Dict[str, Any]  # override specific vars on target
```

---

#### 4.9 Resources View (Cross-Workspace Resource Inventory)

**What Spacelift does:** Unified view of all Terraform-managed resources across all stacks,
showing resource address, type, provider, and owning stack.

**Why CMP should enhance this:** CMP already has resource inventory. Enhance with Terraform
workspace linking metadata.

**Enhancement:**
- Each inventory item shows which workspace manages it
- Show Terraform resource address (e.g., `module.vpc.aws_subnet.private[0]`)
- Filter resources by workspace, template, drift status
- "Managed by Terraform" badge on resources discovered via state
- Click resource → shows workspace detail + relevant state attributes
- Orphan detection: resources in state but not in inventory (and vice versa)

---

#### 4.10 Run Hooks (Before/After Execution Steps)

**What Spacelift does:** Shell commands that run at specific phases of a run:
before_init, after_init, before_plan, after_plan, before_apply, after_apply, after_run.

**Why CMP needs this:** Currently no way to run custom scripts around Terraform operations.
Common needs: security scanning before apply, CMDB updates after apply, secret injection before init.

**CMP Implementation:**

```
WorkspaceHook:
  hook_id: str
  workspace_id: str  (or null for context-attached hooks)
  phase: "before_init" | "after_init" | "before_plan" | "after_plan" |
         "before_apply" | "after_apply" | "on_failure" | "after_destroy"
  command: str       # shell command or script
  environment: Dict[str, str]  # additional env vars for the hook
  timeout_seconds: int = 300
  run_on_failure: bool = False
  enabled: bool = True
```

**Use cases:**
- `before_plan`: Inject secrets from HashiCorp Vault
- `after_plan`: Run Checkov security scan, fail if critical issues found
- `before_apply`: Notify Slack channel "Deploying to production..."
- `after_apply`: Update ServiceNow CMDB, trigger health checks
- `on_failure`: Create PagerDuty incident, rollback notification
- `after_destroy`: Clean up DNS records, remove monitoring dashboards

---

### Tier 3: Strategic Long-Term Additions

#### 4.11 Private Module Registry

**What Spacelift does:** Full module registry for publishing, versioning, and consuming reusable
Terraform modules. Includes testing, documentation, and usage tracking.

**CMP Implementation:**
- Extend existing terraform_modules domain
- Add version management (git tag → module version)
- Auto-generate documentation from variable/output descriptions
- Usage tracking: "Used by 14 workspaces across 3 tenants"
- Deprecation workflow: mark old versions, notify consumers
- Publish from: git repo, upload, inline
- Integration with template source_type: "registry" already in the model

---

#### 4.12 Workspace Groups (Spaces)

**What Spacelift does:** Hierarchical containers that scope access, policies, and contexts.
Child spaces inherit from parent spaces.

**CMP Implementation:**
- Add workspace grouping within tenants (by team, environment, project)
- Groups can have attached policies and contexts (inherited by all workspaces in the group)
- RBAC scoped to groups: "Team A can manage workspaces in group 'payments'"
- Default credential per group
- Group-level settings: auto-drift schedule, default labels, notification channels

---

#### 4.13 GitOps / VCS Integration

**What Spacelift does:** Deep VCS integration — pushes to tracked branch auto-trigger applies,
PRs auto-trigger plan previews, PR comments show plan output.

**CMP Implementation:**
- Connect workspace to a git repository + branch
- On push to tracked branch: auto-trigger proposed run (plan)
- On PR: auto-comment with plan output (creates/modifies/destroys)
- Auto-apply on merge (configurable: auto vs require manual confirmation)
- Push policies: filter which file changes should trigger which workspaces
- Requires: GitHub/GitLab webhook integration

---

#### 4.14 Worker Pools (Self-Hosted Execution)

**What Spacelift does:** Customers run Terraform in their own infrastructure (Docker or K8s)
for network access to private resources and compliance requirements.

**CMP Implementation (future):**
- Currently CMP runs Terraform locally (Docker-based)
- Add support for remote execution agents deployed in customer VPCs
- Agent registration and health monitoring
- Job routing: "This workspace runs on the 'production-vpc' worker pool"
- Useful for: accessing private networks, compliance (data doesn't leave customer infra)
- Lower priority: CMP's current Docker-based execution works for most cases

---

#### 4.15 Multi-IaC Support

**What Spacelift does:** Supports Terraform, OpenTofu, Terragrunt, Pulumi, CloudFormation,
Kubernetes, and Ansible in a unified workflow.

**CMP Implementation (future):**
- Current: Terraform only (via TerraformEngine)
- Phase 1: Add OpenTofu support (nearly identical to Terraform, swap binary)
- Phase 2: Add CloudFormation support (for AWS-only customers)
- Phase 3: Consider Pulumi (TypeScript/Python-based IaC)
- Each IaC vendor would implement a common `IaCEngine` interface
- Same workspace model, different execution engine

---

## 5. Detailed Implementation Designs

### 5.1 Workspace Dependencies — Full Design

#### Data Model

```python
class WorkspaceDependency(BaseModel):
    """A dependency relationship between two workspaces."""
    dependency_id: str
    workspace_id: str              # child (depends on parent)
    depends_on_workspace_id: str   # parent
    tenant_id: str
    trigger_condition: str         # "on_output_change" | "always" | "never" (manual only)
    output_references: List[OutputReference] = []
    created_at: str
    created_by: str

class OutputReference(BaseModel):
    """Maps a parent workspace output to a child workspace input variable."""
    output_name: str          # name of the output in parent workspace
    input_variable_name: str  # variable name in child workspace to inject into
    sensitive: bool = False   # whether to treat as secret

class DependencyGraph(BaseModel):
    """The full dependency DAG for visualization."""
    nodes: List[DependencyNode]
    edges: List[DependencyEdge]

class DependencyNode(BaseModel):
    workspace_id: str
    workspace_name: str
    template_name: str
    status: WorkspaceStatus
    last_run_status: Optional[str]

class DependencyEdge(BaseModel):
    source_workspace_id: str   # parent
    target_workspace_id: str   # child
    output_references: List[OutputReference]
    trigger_condition: str
```

#### DynamoDB Schema

```
Table: terraform_workspace_dependencies
PK: TENANT#{tenant_id}
SK: DEP#{dependency_id}

GSI: workspace_dependents
PK: PARENT#{depends_on_workspace_id}
SK: CHILD#{workspace_id}

GSI: workspace_parents
PK: CHILD#{workspace_id}
SK: PARENT#{depends_on_workspace_id}
```

#### API Endpoints

```
POST   /api/v1/terraform/workspaces/{id}/dependencies       — Add dependency
GET    /api/v1/terraform/workspaces/{id}/dependencies       — List dependencies (parents + children)
DELETE /api/v1/terraform/workspaces/{id}/dependencies/{dep_id} — Remove dependency
GET    /api/v1/terraform/workspaces/{id}/dependency-graph   — Full DAG visualization
POST   /api/v1/terraform/workspaces/{id}/cascade-trigger    — Manually trigger cascade
```

#### Cascade Trigger Logic (Service Layer)

```python
async def on_workspace_run_completed(workspace_id: str, tenant_id: str, outputs: Dict):
    """Called after a successful workspace apply. Triggers dependents."""
    dependents = await get_workspace_dependents(workspace_id, tenant_id)
    
    for dep in dependents:
        should_trigger = False
        
        if dep.trigger_condition == "always":
            should_trigger = True
        elif dep.trigger_condition == "on_output_change":
            # Compare current outputs with previously stored outputs
            previous_outputs = await get_stored_outputs(workspace_id, tenant_id)
            referenced_outputs = [ref.output_name for ref in dep.output_references]
            should_trigger = any(
                outputs.get(name) != previous_outputs.get(name)
                for name in referenced_outputs
            )
        
        if should_trigger:
            # Inject parent outputs as variables into child workspace
            injected_vars = {}
            for ref in dep.output_references:
                if ref.output_name in outputs:
                    injected_vars[ref.input_variable_name] = outputs[ref.output_name]
            
            # Trigger child workspace with injected variables
            await trigger_workspace_run(
                dep.workspace_id, tenant_id,
                override_variables=injected_vars,
                triggered_by=f"dependency:{workspace_id}"
            )
    
    # Store current outputs for future comparison
    await store_workspace_outputs(workspace_id, tenant_id, outputs)
```

### 5.2 Plan Preview + Promotion — Full Design

#### Data Model

```python
class RunType(str, Enum):
    PROPOSED = "proposed"     # plan-only, for review
    TRACKED = "tracked"       # full plan + apply

class RunPhase(str, Enum):
    QUEUED = "queued"
    INITIALIZING = "initializing"
    PLANNING = "planning"
    PLAN_COMPLETE = "plan_complete"    # proposed run stops here
    AWAITING_APPROVAL = "awaiting_approval"
    APPLYING = "applying"
    APPLIED = "applied"
    FAILED = "failed"
    DISCARDED = "discarded"
    CANCELLED = "cancelled"

class TerraformRun(BaseModel):
    run_id: str
    workspace_id: str
    tenant_id: str
    run_type: RunType
    current_phase: RunPhase
    
    # Plan output
    plan_json: Optional[str]           # terraform show -json planfile
    plan_human_readable: Optional[str]  # terraform plan stdout
    plan_file_key: Optional[str]        # S3/storage key for binary plan file
    
    # Plan summary
    resources_to_create: int = 0
    resources_to_modify: int = 0
    resources_to_destroy: int = 0
    resources_unchanged: int = 0
    
    # Cost impact (from plan evaluation)
    estimated_monthly_cost_change: Optional[float]
    
    # Policy evaluation results
    policy_warnings: List[str] = []
    policy_denials: List[str] = []
    requires_approval: bool = False
    approval_reason: Optional[str]
    
    # Promotion
    promoted_from_run_id: Optional[str]  # if this tracked run was promoted from a proposed run
    promoted_by: Optional[str]
    promoted_at: Optional[str]
    
    # Metadata
    triggered_by: str          # "user:{user_id}" | "dependency:{workspace_id}" | "schedule:{schedule_id}"
    variables_used: Dict[str, Any]
    created_at: str
    started_at: Optional[str]
    completed_at: Optional[str]
    duration_ms: Optional[int]

class PromoteRunRequest(BaseModel):
    """Promote a proposed run to a tracked run (apply the reviewed plan)."""
    run_id: str
    confirmation: str = "confirm"  # user must type "confirm"
```

#### API Endpoints

```
POST   /api/v1/terraform/workspaces/{id}/runs         — Create a new run (proposed or tracked)
GET    /api/v1/terraform/workspaces/{id}/runs         — List runs for workspace
GET    /api/v1/terraform/workspaces/{id}/runs/{run_id} — Get run details + plan output
POST   /api/v1/terraform/workspaces/{id}/runs/{run_id}/promote  — Promote proposed → tracked
POST   /api/v1/terraform/workspaces/{id}/runs/{run_id}/discard  — Discard a proposed run
POST   /api/v1/terraform/workspaces/{id}/runs/{run_id}/cancel   — Cancel a running run
GET    /api/v1/terraform/workspaces/{id}/runs/{run_id}/plan     — Get plan output (JSON + human)
```

#### Execution Flow

```
User Action: "Update variables" or "Deploy"
         │
         ▼
┌─────────────────────┐
│  Create Proposed Run │ (run_type = proposed)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   terraform init     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   terraform plan     │ → Save plan file + JSON output
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Evaluate Plan       │ → Run plan policies, calculate cost
│  Policies            │ → Check: deny / warn / require_approval
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  PLAN_COMPLETE       │ ← User reviews plan here
│  (awaiting action)   │
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
 [Discard]   [Promote/Confirm]
                  │
                  ▼
        ┌────────────────┐
        │ If requires     │
        │ approval →      │ → Approval workflow
        │ wait for        │
        │ approve/reject  │
        └────────┬───────┘
                 │
                 ▼
        ┌────────────────┐
        │ terraform apply │ (using saved plan file)
        │ -auto-approve   │
        └────────┬───────┘
                 │
                 ▼
        ┌────────────────┐
        │ APPLIED         │ → Update inventory, trigger dependents
        └────────────────┘
```

### 5.3 Contexts — Full Design

#### Data Model

```python
class TerraformContext(BaseModel):
    context_id: str
    name: str
    description: Optional[str]
    tenant_id: str
    
    # Configuration
    environment_variables: Dict[str, str] = {}      # plain-text vars
    secret_variables: List[str] = []                # keys only (values encrypted in DB)
    mounted_files: Dict[str, str] = {}              # filename → content
    
    # Hooks (shell commands at specific phases)
    hooks: List[ContextHook] = []
    
    # Auto-attachment rules
    auto_attach_labels: List[str] = []  # attach to workspaces with these labels
    
    # Priority (higher overrides lower when multiple contexts conflict)
    priority: int = 0
    
    # Metadata
    created_by: str
    created_at: str
    updated_at: str

class ContextHook(BaseModel):
    phase: str  # "before_init" | "after_plan" | "before_apply" | "after_apply"
    command: str
    
class ContextAttachment(BaseModel):
    """Links a context to a workspace."""
    attachment_id: str
    context_id: str
    workspace_id: str
    priority_override: Optional[int]  # override context default priority
    tenant_id: str
```

#### Variable Resolution Order

When a workspace runs, variables are resolved in this order (later overrides earlier):
1. Template defaults (from template `input_variables[].default`)
2. Context variables (sorted by priority, ascending — highest priority wins)
3. Workspace `variable_values` (user-provided)
4. Dependency-injected variables (from parent workspace outputs)
5. Run-specific overrides (from Day-2 operation `updated_variables`)

#### API Endpoints

```
POST   /api/v1/terraform/contexts              — Create context
GET    /api/v1/terraform/contexts              — List contexts
GET    /api/v1/terraform/contexts/{id}         — Get context (secrets masked)
PUT    /api/v1/terraform/contexts/{id}         — Update context
DELETE /api/v1/terraform/contexts/{id}         — Delete context
POST   /api/v1/terraform/contexts/{id}/attach  — Attach to workspace
DELETE /api/v1/terraform/contexts/{id}/detach/{workspace_id} — Detach from workspace
GET    /api/v1/terraform/workspaces/{id}/resolved-variables — Show final resolved vars
```

### 5.4 Plan Policies — Full Design

#### Data Model

```python
class PlanPolicyType(str, Enum):
    DENY = "deny"                      # Block the apply entirely
    WARN = "warn"                      # Allow but show warning
    REQUIRE_APPROVAL = "require_approval"  # Force approval workflow

class PlanPolicyConditionOperator(str, Enum):
    EQUALS = "equals"
    NOT_EQUALS = "not_equals"
    CONTAINS = "contains"
    NOT_CONTAINS = "not_contains"
    GREATER_THAN = "greater_than"
    LESS_THAN = "less_than"
    REGEX_MATCH = "regex_match"
    EXISTS = "exists"
    NOT_EXISTS = "not_exists"
    IN_LIST = "in_list"

class PlanPolicyRule(BaseModel):
    """A single rule evaluated against the terraform plan JSON."""
    rule_id: str
    description: str
    
    # What to inspect in the plan
    target: str  # "resource_change" | "output_change" | "plan_summary" | "variable"
    
    # Filters (narrow down which resources/changes to inspect)
    resource_type_filter: Optional[str]      # e.g., "aws_security_group", "aws_s3_bucket"
    change_action_filter: Optional[List[str]] # ["create", "update", "delete"]
    
    # Condition to evaluate
    attribute_path: Optional[str]            # JSON path within the resource change
    operator: PlanPolicyConditionOperator
    expected_value: Optional[Any]
    
    # Invert: trigger when condition is NOT met (useful for "require encryption")
    negate: bool = False

class PlanPolicy(BaseModel):
    policy_id: str
    name: str
    description: Optional[str]
    tenant_id: str
    enabled: bool = True
    
    # What happens when rules match
    action: PlanPolicyType
    
    # Rules (evaluated with AND logic — all must match for policy to trigger)
    rules: List[PlanPolicyRule]
    
    # Scope
    applies_to_labels: List[str] = []       # only workspaces with these labels
    applies_to_workspace_ids: List[str] = [] # specific workspaces
    applies_to_all: bool = False            # all workspaces in tenant
    
    # Metadata
    created_by: str
    created_at: str
    updated_at: str

class PlanPolicyEvaluationResult(BaseModel):
    """Result of evaluating all plan policies against a run's plan."""
    run_id: str
    workspace_id: str
    evaluated_at: str
    
    denied: List[PolicyViolation] = []
    warnings: List[PolicyViolation] = []
    approvals_required: List[PolicyViolation] = []
    passed: List[str] = []  # policy IDs that passed
    
    overall_verdict: str  # "allowed" | "denied" | "approval_required"

class PolicyViolation(BaseModel):
    policy_id: str
    policy_name: str
    rule_id: str
    rule_description: str
    action: PlanPolicyType
    affected_resources: List[str]  # resource addresses that triggered the rule
    details: Optional[str]
```

#### Example Policies (Pre-Built Templates)

```json
[
  {
    "name": "No Public S3 Buckets",
    "description": "Deny any plan that creates or modifies S3 buckets with public ACLs",
    "action": "deny",
    "rules": [{
      "target": "resource_change",
      "resource_type_filter": "aws_s3_bucket",
      "change_action_filter": ["create", "update"],
      "attribute_path": "values.acl",
      "operator": "in_list",
      "expected_value": ["public-read", "public-read-write"]
    }]
  },
  {
    "name": "Limit Destructive Changes",
    "description": "Require approval if plan destroys more than 3 resources",
    "action": "require_approval",
    "rules": [{
      "target": "plan_summary",
      "attribute_path": "resources_to_destroy",
      "operator": "greater_than",
      "expected_value": 3
    }]
  },
  {
    "name": "Enforce Encryption",
    "description": "Deny RDS instances without encryption enabled",
    "action": "deny",
    "rules": [{
      "target": "resource_change",
      "resource_type_filter": "aws_db_instance",
      "change_action_filter": ["create"],
      "attribute_path": "values.storage_encrypted",
      "operator": "equals",
      "expected_value": false
    }]
  },
  {
    "name": "Require Cost-Center Tag",
    "description": "Warn if any new resource is missing a cost-center tag",
    "action": "warn",
    "rules": [{
      "target": "resource_change",
      "change_action_filter": ["create"],
      "attribute_path": "values.tags.cost-center",
      "operator": "not_exists",
      "expected_value": null
    }]
  }
]
```

#### API Endpoints

```
POST   /api/v1/terraform/plan-policies           — Create plan policy
GET    /api/v1/terraform/plan-policies           — List plan policies
GET    /api/v1/terraform/plan-policies/{id}      — Get plan policy
PUT    /api/v1/terraform/plan-policies/{id}      — Update plan policy
DELETE /api/v1/terraform/plan-policies/{id}      — Delete plan policy
GET    /api/v1/terraform/plan-policies/templates — Get pre-built policy templates
POST   /api/v1/terraform/runs/{run_id}/policy-evaluation — Get evaluation results for a run
```

### 5.5 Infrastructure Blueprints — Full Design

#### Data Model

```python
class BlueprintInputField(BaseModel):
    """A simplified form field exposed to the end user."""
    variable_name: str           # maps to template variable
    label: str                   # human-readable label
    description: Optional[str]
    field_type: str              # "text" | "number" | "select" | "boolean" | "password"
    required: bool = True
    default_value: Optional[Any]
    options: Optional[List[Dict[str, str]]]  # for select fields
    validation_regex: Optional[str]
    validation_message: Optional[str]
    placeholder: Optional[str]

class BlueprintPostDeployAction(BaseModel):
    """Actions to automatically attach after deployment."""
    resource_action_id: str
    auto_execute: bool = False   # execute immediately after deploy?
    delay_minutes: int = 0

class InfrastructureBlueprint(BaseModel):
    blueprint_id: str
    tenant_id: str
    name: str
    description: Optional[str]
    icon: Optional[str]          # Lucide icon name
    category: str                # "networking" | "compute" | "database" | "application" | "security"
    tags: List[str] = []
    
    # Template configuration (pre-selected)
    template_id: str
    template_version: Optional[int]   # null = always latest
    
    # Pre-configured settings (user cannot change these)
    locked_variables: Dict[str, Any] = {}
    required_contexts: List[str] = []         # context IDs to auto-attach
    required_plan_policies: List[str] = []    # plan policy IDs to enforce
    default_credential_id: Optional[str]
    workspace_labels: List[str] = []          # labels to apply to created workspace
    
    # Self-service form (only these variables exposed to user)
    exposed_inputs: List[BlueprintInputField] = []
    
    # Governance
    requires_approval: bool = False
    approver_roles: List[str] = []
    max_instances_per_user: Optional[int]     # limit deployments per user
    allowed_roles: List[str] = ["user", "developer", "admin"]
    budget_check: bool = False                # validate against budget before deploy
    
    # Post-deploy configuration
    post_deploy_actions: List[BlueprintPostDeployAction] = []
    drift_detection_schedule: Optional[str]   # cron expression
    auto_destroy_after_hours: Optional[int]   # TTL for ephemeral environments
    
    # Dependencies (deploy these in order)
    dependency_chain: List[BlueprintDependencyStep] = []
    
    # Metadata
    created_by: str
    created_at: str
    updated_at: str
    usage_count: int = 0         # how many times deployed
    last_deployed_at: Optional[str]

class BlueprintDependencyStep(BaseModel):
    """A step in a multi-workspace deployment chain."""
    step_order: int
    template_id: str
    workspace_name_suffix: str   # e.g., "-network", "-compute"
    locked_variables: Dict[str, Any] = {}
    exposed_inputs: List[BlueprintInputField] = []
    output_mappings: List[OutputReference] = []  # from previous step

class DeployBlueprintRequest(BaseModel):
    """User request to deploy a blueprint."""
    blueprint_id: str
    workspace_name: str
    input_values: Dict[str, Any]
    credential_id: Optional[str]  # override default if allowed
    
class DeployBlueprintResponse(BaseModel):
    """Response after blueprint deployment starts."""
    order_id: str
    workspace_ids: List[str]     # one per dependency step
    execution_id: str
    status: str
```

#### User Experience Flow

```
1. User browses "Infrastructure Blueprints" page
2. Filters by category (Networking, Compute, Database, Application)
3. Selects "Production-Ready PostgreSQL on AWS"
4. Sees blueprint card with:
   - Description, estimated cost, deployment time
   - "Deploy" button (if authorized)
5. Clicks "Deploy" → opens simple form:
   - Database Name: [_____________]
   - Instance Size: [dropdown: db.t3.medium, db.r5.large, ...]
   - Storage (GB): [_____________]
   - Multi-AZ: [toggle]
   - Backup Retention (days): [_____________]
6. Clicks "Review & Deploy"
7. System shows:
   - Plan preview (what will be created)
   - Estimated monthly cost
   - Policy check results
8. If approval needed → routes to approver with plan attached
9. On confirmation → creates workspace, runs apply
10. Resources appear in inventory with Day-2 actions pre-attached
```

#### API Endpoints

```
POST   /api/v1/terraform/blueprints              — Create blueprint (admin/developer)
GET    /api/v1/terraform/blueprints              — List blueprints (filtered by role)
GET    /api/v1/terraform/blueprints/{id}         — Get blueprint details
PUT    /api/v1/terraform/blueprints/{id}         — Update blueprint
DELETE /api/v1/terraform/blueprints/{id}         — Delete blueprint
POST   /api/v1/terraform/blueprints/{id}/deploy  — Deploy a blueprint (creates workspace + runs)
GET    /api/v1/terraform/blueprints/{id}/deployments — List all deployments of this blueprint
```

### 5.6 Workspace Scheduling — Full Design

#### Data Model

```python
class WorkspaceScheduleType(str, Enum):
    DRIFT_DETECTION = "drift_detection"
    AUTO_APPLY = "auto_apply"            # periodic reconciliation
    AUTO_DESTROY = "auto_destroy"        # TTL-based destruction
    VARIABLE_UPDATE = "variable_update"  # scheduled variable changes
    SNAPSHOT = "snapshot"                # state backup

class WorkspaceSchedule(BaseModel):
    schedule_id: str
    workspace_id: str
    tenant_id: str
    schedule_type: WorkspaceScheduleType
    enabled: bool = True
    
    # Timing
    cron_expression: str          # e.g., "0 */4 * * *" (every 4 hours)
    timezone: str = "UTC"
    
    # Type-specific configuration
    # For VARIABLE_UPDATE:
    scheduled_variables: Optional[Dict[str, Any]]   # variables to set when schedule fires
    
    # For AUTO_DESTROY:
    destroy_after_hours: Optional[int]              # hours after creation
    destroy_at: Optional[str]                       # specific datetime
    
    # For DRIFT_DETECTION:
    auto_reconcile: bool = False                    # auto-apply if drift detected
    notify_on_drift: bool = True
    notification_channels: List[str] = []           # channel IDs
    
    # Metadata
    last_triggered_at: Optional[str]
    next_trigger_at: Optional[str]
    created_by: str
    created_at: str

class ScheduleExecutionLog(BaseModel):
    log_id: str
    schedule_id: str
    workspace_id: str
    triggered_at: str
    result: str          # "success" | "failed" | "skipped" | "drift_found" | "no_drift"
    run_id: Optional[str]
    error_message: Optional[str]
```

#### Integration with Existing Scheduled Jobs

- Workspace schedules register as a specialized type in the existing `scheduled_jobs` system
- The scheduler service checks workspace schedules alongside regular jobs
- Each schedule execution creates a proper TerraformRun with audit trail

---

### 5.7 Trigger Policies — Full Design

#### Data Model

```python
class TriggerConditionType(str, Enum):
    ON_SUCCESS = "on_success"
    ON_FAILURE = "on_failure"
    ON_OUTPUT_CHANGE = "on_output_change"
    ON_ANY_CHANGE = "on_any_change"
    ALWAYS = "always"

class WorkspaceTriggerRule(BaseModel):
    trigger_id: str
    tenant_id: str
    name: str
    description: Optional[str]
    enabled: bool = True
    
    # Source: which workspace completion triggers this rule
    source_workspace_id: str
    
    # Target: which workspace to trigger
    target_workspace_id: str
    
    # Conditions
    trigger_on: TriggerConditionType
    output_filter: List[str] = []       # only trigger if these specific outputs changed
    
    # Throttling
    min_interval_minutes: int = 0       # don't re-trigger within this window
    
    # Variable injection
    inject_outputs: bool = True         # pass source outputs to target
    override_variables: Dict[str, Any] = {}  # static overrides for target
    
    # Additional conditions
    source_labels_require: List[str] = []    # source must have these labels
    time_window: Optional[Dict[str, str]]    # only trigger during specific hours
    
    # Metadata
    last_triggered_at: Optional[str]
    trigger_count: int = 0
    created_by: str
    created_at: str
```

---

## 6. CMP's Existing Advantages Over Spacelift

These are areas where CMP is already stronger and should be preserved/promoted:

| Area | CMP Advantage | Spacelift Limitation |
|------|---------------|---------------------|
| **Self-Service Catalog** | Full shopping cart, multi-item ordering, form builder, catalog components | Blueprints/Templates are simpler, no cart concept |
| **Cost Management** | Native cost models, budgets, analytics, live projections, cost anomaly detection | Relies on Infracost plugin, no budget management |
| **Day-2 Operations** | Three backend types (native API + flow + terraform), conditional actions, custom input schemas | Only re-running IaC, no native cloud API actions |
| **Resource Actions** | Rich conditions (status, region, tags, name), approval per action, role-based visibility | No equivalent — Spacelift doesn't manage individual resources |
| **Event Automation** | 60+ event types, configurable rules with conditions, webhook/notification actions | Basic webhook notifications only |
| **Multi-Tenancy** | Full tenant isolation, branding, per-tenant config, tenant switching | Single account with Spaces (not true multi-tenancy) |
| **Approval Workflows** | Multi-level, SLA-tracked, conditional, role-based routing | Simple approval counts, no SLA |
| **Resource Inventory** | Multi-cloud discovery, native API queries, resource metadata, lease management | Only Terraform state viewer, no actual cloud API queries |
| **AI Assistant** | Conversational AI with history, inline suggestions, catalog recommendations | Intent is deploy-focused only, no conversational context |
| **Compliance Dashboard** | Policies + quotas + compliance scoring | OPA policies only, no quotas concept |
| **Order Management** | Full order lifecycle (draft → approved → provisioning → completed) | No order/request concept |
| **Notifications** | Multi-channel (email, in-app, webhook), user preferences | Slack/Teams/webhook only |
| **Reports & Analytics** | Built-in reporting, insights, scheduled reports | No native reporting |
| **Leases** | Resource TTL management with auto-destroy scheduling | Stack scheduling covers some of this |
| **Audit Logging** | Comprehensive audit logs with search, export | Basic audit trail |

### Key Positioning Message

CMP = **Spacelift's IaC orchestration** + **ServiceNow's ITSM** + **CloudHealth's FinOps** in one platform.

Spacelift is excellent at IaC orchestration but is purely an engineering tool. CMP bridges
the gap between engineering (IaC) and business (cost, compliance, self-service, governance).

---

## 7. Implementation Roadmap

### Phase 1: Foundation (4-6 weeks)

| Feature | Priority | Effort | Dependencies |
|---------|----------|--------|-------------|
| Plan Preview + Promotion (5.2) | P0 | 3 weeks | TerraformEngine changes |
| Workspace Scheduling — Drift (5.6, drift only) | P0 | 1 week | Existing scheduler |
| Contexts (5.3) | P1 | 2 weeks | New domain |

**Deliverables:**
- Users can preview Terraform plan before applying
- Scheduled drift detection on production workspaces
- Shared configuration bundles attachable to workspaces

### Phase 2: Orchestration (4-6 weeks)

| Feature | Priority | Effort | Dependencies |
|---------|----------|--------|-------------|
| Workspace Dependencies (5.1) | P0 | 3 weeks | Plan Preview (Phase 1) |
| Plan Policies (5.4) | P1 | 2 weeks | Plan Preview (Phase 1) |
| Run Hooks (4.10) | P2 | 1 week | None |

**Deliverables:**
- Multi-workspace deployment chains with output passing
- Automated policy evaluation on plan output
- Custom scripts at execution lifecycle points

### Phase 3: Self-Service (4-6 weeks)

| Feature | Priority | Effort | Dependencies |
|---------|----------|--------|-------------|
| Infrastructure Blueprints (5.5) | P0 | 3 weeks | Dependencies + Contexts |
| Workspace Scheduling — Full (5.6) | P1 | 2 weeks | Phase 1 drift scheduling |
| Trigger Policies (5.7) | P2 | 2 weeks | Dependencies (Phase 2) |

**Deliverables:**
- Governed self-service infrastructure deployment for all roles
- Full workspace lifecycle scheduling (destroy, scale, reconcile)
- Advanced cross-workspace orchestration rules

### Phase 4: Advanced (6-8 weeks)

| Feature | Priority | Effort | Dependencies |
|---------|----------|--------|-------------|
| Intent-Based Provisioning (4.6) | P1 | 3 weeks | Blueprints + Plan Preview |
| Private Module Registry (4.11) | P2 | 2 weeks | None |
| Resources View Enhancement (4.9) | P2 | 1 week | None |
| Workspace Groups (4.12) | P2 | 2 weeks | Contexts + Policies |
| GitOps / VCS Integration (4.13) | P3 | 4 weeks | Plan Preview |

**Deliverables:**
- Natural language → governed infrastructure deployment
- Internal module registry with versioning
- Organizational workspace hierarchy
- Git-triggered infrastructure deployments

---

## 8. Data Models — DynamoDB Table Designs

### New Tables Required

| Table | Purpose | PK | SK |
|-------|---------|----|----|
| `terraform_runs` | Run history per workspace | `WORKSPACE#{workspace_id}` | `RUN#{run_id}` |
| `terraform_contexts` | Shared config bundles | `TENANT#{tenant_id}` | `CONTEXT#{context_id}` |
| `terraform_context_attachments` | Context-workspace links | `WORKSPACE#{workspace_id}` | `CONTEXT#{context_id}` |
| `terraform_workspace_dependencies` | DAG edges | `TENANT#{tenant_id}` | `DEP#{dependency_id}` |
| `terraform_workspace_outputs` | Stored outputs for change detection | `WORKSPACE#{workspace_id}` | `OUTPUTS#latest` |
| `terraform_plan_policies` | Plan evaluation rules | `TENANT#{tenant_id}` | `PLANPOLICY#{policy_id}` |
| `terraform_blueprints` | Blueprint definitions | `TENANT#{tenant_id}` | `BLUEPRINT#{blueprint_id}` |
| `terraform_workspace_schedules` | Per-workspace cron jobs | `WORKSPACE#{workspace_id}` | `SCHEDULE#{schedule_id}` |
| `terraform_trigger_rules` | Cross-workspace triggers | `TENANT#{tenant_id}` | `TRIGGER#{trigger_id}` |
| `terraform_hooks` | Execution hooks | `WORKSPACE#{workspace_id}` | `HOOK#{hook_id}` |

### GSI Designs

```
terraform_runs:
  GSI: tenant-runs — PK: TENANT#{tenant_id}, SK: RUN#{created_at}#{run_id}
  GSI: run-by-status — PK: STATUS#{status}, SK: WORKSPACE#{workspace_id}

terraform_workspace_dependencies:
  GSI: parent-lookup — PK: PARENT#{depends_on_workspace_id}, SK: CHILD#{workspace_id}
  GSI: child-lookup — PK: CHILD#{workspace_id}, SK: PARENT#{depends_on_workspace_id}

terraform_context_attachments:
  GSI: context-workspaces — PK: CONTEXT#{context_id}, SK: WORKSPACE#{workspace_id}

terraform_workspace_schedules:
  GSI: next-trigger — PK: TENANT#{tenant_id}, SK: NEXT#{next_trigger_at}
```

---

## 9. API Contracts — Complete Endpoint Registry

### New Router: Terraform Runs

```
Router prefix: /api/v1/terraform/workspaces/{workspace_id}/runs
Tags: ["Terraform Runs"]

POST   /                    — Create run (proposed or tracked)
GET    /                    — List runs (paginated, filtered by type/status)
GET    /{run_id}            — Get run details
GET    /{run_id}/plan       — Get plan output (JSON + human-readable)
GET    /{run_id}/logs       — Get execution logs (streaming via SSE)
POST   /{run_id}/promote    — Promote proposed → tracked (apply)
POST   /{run_id}/confirm    — Confirm tracked run (start apply phase)
POST   /{run_id}/discard    — Discard a proposed run
POST   /{run_id}/cancel     — Cancel a running run
GET    /{run_id}/policy-results — Get plan policy evaluation results
POST   /{run_id}/approve    — Approve a run awaiting approval
POST   /{run_id}/reject     — Reject a run awaiting approval
```

### New Router: Terraform Contexts

```
Router prefix: /api/v1/terraform/contexts
Tags: ["Terraform Contexts"]

POST   /                    — Create context
GET    /                    — List contexts for tenant
GET    /{context_id}        — Get context (secrets masked)
PUT    /{context_id}        — Update context
DELETE /{context_id}        — Delete context (fails if attached)
POST   /{context_id}/attach — Attach to workspace(s)
POST   /{context_id}/detach — Detach from workspace(s)
GET    /{context_id}/workspaces — List attached workspaces
```

### New Router: Workspace Dependencies

```
Router prefix: /api/v1/terraform/workspaces/{workspace_id}/dependencies
Tags: ["Terraform Dependencies"]

POST   /                    — Add dependency (this workspace depends on another)
GET    /                    — List all dependencies (parents + children)
DELETE /{dependency_id}     — Remove dependency
GET    /graph               — Full dependency DAG for visualization
POST   /cascade-trigger     — Manually trigger cascade from this workspace
GET    /resolved-order      — Get execution order for a full cascade
```

### New Router: Plan Policies

```
Router prefix: /api/v1/terraform/plan-policies
Tags: ["Terraform Plan Policies"]

POST   /                    — Create plan policy
GET    /                    — List plan policies for tenant
GET    /templates           — Get pre-built policy templates
GET    /{policy_id}         — Get policy details
PUT    /{policy_id}         — Update policy
DELETE /{policy_id}         — Delete policy
POST   /{policy_id}/test    — Test policy against a sample plan JSON
```

### New Router: Infrastructure Blueprints

```
Router prefix: /api/v1/terraform/blueprints
Tags: ["Terraform Blueprints"]

POST   /                    — Create blueprint (admin/developer)
GET    /                    — List blueprints (role-filtered, categorized)
GET    /categories          — List available categories
GET    /{blueprint_id}      — Get blueprint details
PUT    /{blueprint_id}      — Update blueprint
DELETE /{blueprint_id}      — Delete blueprint
POST   /{blueprint_id}/deploy — Deploy blueprint (creates workspace + runs)
GET    /{blueprint_id}/deployments — List all deployments from this blueprint
GET    /{blueprint_id}/cost-estimate — Estimate cost for given inputs
```

### New Router: Workspace Schedules

```
Router prefix: /api/v1/terraform/workspaces/{workspace_id}/schedules
Tags: ["Terraform Scheduling"]

POST   /                    — Create schedule
GET    /                    — List schedules for workspace
GET    /{schedule_id}       — Get schedule details
PUT    /{schedule_id}       — Update schedule
DELETE /{schedule_id}       — Delete schedule
POST   /{schedule_id}/trigger — Manually trigger schedule now
GET    /{schedule_id}/history — Get schedule execution history
```

### New Router: Trigger Rules

```
Router prefix: /api/v1/terraform/trigger-rules
Tags: ["Terraform Triggers"]

POST   /                    — Create trigger rule
GET    /                    — List trigger rules for tenant
GET    /{trigger_id}        — Get trigger rule details
PUT    /{trigger_id}        — Update trigger rule
DELETE /{trigger_id}        — Delete trigger rule
GET    /{trigger_id}/history — Get trigger execution history
```

### Modified Existing Endpoints

```
# Existing workspace endpoints — add new capabilities:
GET  /api/v1/terraform/workspaces/{id}/resolved-variables  — Show final resolved vars (context + workspace + deps)
GET  /api/v1/terraform/workspaces/{id}/hooks              — List hooks for workspace
POST /api/v1/terraform/workspaces/{id}/hooks              — Add hook to workspace
PUT  /api/v1/terraform/workspaces/{id}/hooks/{hook_id}    — Update hook
DELETE /api/v1/terraform/workspaces/{id}/hooks/{hook_id}  — Remove hook
```

---

## 10. Frontend Requirements

### New Pages

| Page | Route | Description | Access |
|------|-------|-------------|--------|
| Infrastructure Blueprints | `/t/:tenant/blueprints` | Browse and deploy blueprints | All roles |
| Blueprint Detail | `/t/:tenant/blueprints/:id` | Blueprint info + deploy form | All roles |
| Blueprint Admin | `/t/:tenant/blueprints/manage` | Create/edit blueprints | admin, developer |
| Terraform Contexts | `/t/:tenant/terraform/contexts` | Manage shared contexts | admin, developer |
| Plan Policies | `/t/:tenant/terraform/plan-policies` | Manage plan policies | admin |
| Trigger Rules | `/t/:tenant/terraform/triggers` | Manage cross-workspace triggers | admin, developer |

### Enhanced Existing Pages

| Page | Enhancement |
|------|-------------|
| **TerraformWorkspaces** | Add dependency graph visualization, schedule indicators, context badges |
| **Workspace Detail** | Add: Dependencies tab, Runs tab (with plan preview), Contexts tab, Schedules tab, Hooks tab |
| **Workspace Day-2** | Two-phase: show plan preview first, then "Confirm Apply" button |
| **Executions** | Show run type (proposed/tracked), plan output viewer, policy results |

### New Components

| Component | Purpose |
|-----------|---------|
| `DependencyGraph` | Visual DAG renderer (React Flow or similar) |
| `PlanDiffViewer` | Terraform plan output renderer (color-coded create/modify/destroy) |
| `PolicyResultBadge` | Shows policy evaluation result (passed/warned/denied) |
| `BlueprintCard` | Card component for blueprint catalog browsing |
| `BlueprintDeployForm` | Dynamic form generated from blueprint `exposed_inputs` |
| `ContextAttachmentManager` | UI for attaching/detaching contexts to workspaces |
| `ScheduleConfigForm` | Cron expression builder with human-readable preview |
| `RunTimeline` | Visual timeline of run phases (queued → planning → applying → done) |

### UX Flow: Plan Preview

```
Current (before):
  User clicks "Apply Day-2 Operation" → Loading... → Success/Fail

Enhanced (after):
  User clicks "Plan Changes"
    → Loading spinner: "Planning..."
    → Plan result panel appears:
      ┌─────────────────────────────────────────────┐
      │  Plan Summary                                │
      │  ✅ 2 to create  ⚡ 1 to modify  ❌ 0 destroy│
      │                                              │
      │  Estimated Cost Impact: +$45/month           │
      │                                              │
      │  Policy Check: ✅ Passed (2 policies)        │
      │                                              │
      │  [View Full Plan]  [View Policy Details]     │
      │                                              │
      │  [Discard]            [Confirm & Apply →]    │
      └─────────────────────────────────────────────┘
    → User clicks "Confirm & Apply"
    → Progress: "Applying... (step 1/3)"
    → Success panel with created resources
```

### UX Flow: Dependency Graph

```
┌──────────────────────────────────────────────────┐
│  Workspace Dependencies                           │
│                                                   │
│       ┌─────────┐                                │
│       │  VPC    │ ← active ✅                     │
│       └────┬────┘                                │
│            │                                      │
│     ┌──────┴──────┐                              │
│     │             │                              │
│  ┌──▼──┐    ┌────▼────┐                         │
│  │ RDS │    │ EC2 ASG │ ← updating ⟳            │
│  └──┬──┘    └────┬────┘                         │
│     │            │                              │
│     └──────┬─────┘                              │
│            │                                      │
│       ┌────▼────┐                                │
│       │   ALB   │ ← queued (waiting)             │
│       └─────────┘                                │
│                                                   │
│  Legend: ✅ Active  ⟳ Running  ⏳ Queued  ❌ Failed│
└──────────────────────────────────────────────────┘
```

---

## 11. Feature Toggle & License Keys

### New Feature Toggles

| Toggle Key | Display Name | License Key | Default |
|-----------|-------------|-------------|---------|
| `terraform_plan_preview` | Terraform Plan Preview | `terraform` | enabled |
| `terraform_contexts` | Terraform Contexts | `terraform` | enabled |
| `terraform_dependencies` | Workspace Dependencies | `terraform` | enabled |
| `terraform_plan_policies` | Plan Policies | `terraform` + `policy_governance` | enabled |
| `terraform_blueprints` | Infrastructure Blueprints | `terraform` + `catalog` | enabled |
| `terraform_scheduling` | Workspace Scheduling | `terraform` + `scheduled_jobs` | enabled |
| `terraform_triggers` | Trigger Rules | `terraform` | disabled (admin enables) |
| `terraform_hooks` | Run Hooks | `terraform` | disabled (admin enables) |
| `terraform_intent` | Intent Provisioning | `terraform` + `ai_assistant` | disabled |
| `terraform_module_registry` | Module Registry | `terraform` | enabled |

### License Tier Mapping

| Feature | Starter | Professional | Enterprise |
|---------|---------|-------------|-----------|
| Plan Preview | ✗ | ✓ | ✓ |
| Contexts | ✗ | ✓ | ✓ |
| Dependencies | ✗ | ✓ | ✓ |
| Plan Policies | ✗ | ✓ | ✓ |
| Blueprints | ✗ | ✓ | ✓ |
| Scheduling | ✗ | ✓ | ✓ |
| Trigger Rules | ✗ | ✗ | ✓ |
| Run Hooks | ✗ | ✓ | ✓ |
| Intent Provisioning | ✗ | ✗ | ✓ |
| Module Registry | ✗ | ✓ | ✓ |

---

## 12. Day-2 / Resource Actions — Spacelift-Inspired Enhancements

### What Spacelift Lacks (CMP's Opportunity)

Spacelift has **no concept of Day-2 operations** beyond re-running IaC. It cannot:
- Call native cloud APIs (start/stop/restart instances)
- Execute custom workflows on individual resources
- Show resource-specific actions based on state/type/tags
- Provide approval workflows per action
- Estimate cost impact of individual operations

CMP already leads here. Spacelift-inspired enhancements to Day-2:

### 12.1 Plan Preview for Terraform-Backed Resource Actions

Currently, Terraform-backed resource actions go directly to apply. Apply plan preview:

```
User clicks "Resize Instance" (terraform backend action)
  → System runs terraform plan with new variables
  → Shows: "1 resource to modify (aws_instance.web: instance_type m5.large → m5.2xlarge)"
  → Shows: "Estimated cost change: +$85/month"
  → User confirms → apply
```

### 12.2 Action Chaining with Dependencies

Inspired by stack dependencies, allow resource actions to form chains:

```python
class ActionChain(BaseModel):
    chain_id: str
    name: str  # "Full Maintenance Cycle"
    steps: List[ActionChainStep]

class ActionChainStep(BaseModel):
    order: int
    action_id: str
    wait_for_success: bool = True
    timeout_minutes: int = 30
    rollback_action_id: Optional[str]  # if this step fails, run this action
```

**Example chain: "Safe Resize"**
1. Create Snapshot (backup)
2. Stop Instance
3. Resize (terraform apply with new instance_type)
4. Start Instance
5. Health Check (custom flow)
6. If health check fails → Rollback (restore from snapshot)

### 12.3 Action Scheduling (Per-Resource)

```python
class ResourceActionSchedule(BaseModel):
    schedule_id: str
    resource_id: str
    action_id: str
    cron_expression: str
    params: Dict[str, Any]  # pre-filled params for the action
    enabled: bool
    # Example: "Stop this instance every day at 8pm"
    # Example: "Take snapshot every Sunday at 2am"
```

### 12.4 Bulk Actions with Dependency Awareness

```
User selects 5 EC2 instances → clicks "Stop All"
  → System checks workspace dependencies:
    "Instance web-3 is in workspace 'frontend' which depends on workspace 'api'.
     Stopping it may affect dependent services."
  → Shows impact analysis
  → User confirms → executes in parallel (or serial if dependencies exist)
```

### 12.5 Action Templates Library

Pre-built action definitions that admins can import:

```json
{
  "library": [
    {
      "id": "aws-ec2-resize",
      "name": "Resize EC2 Instance",
      "description": "Change instance type with optional stop/start",
      "providers": ["aws"],
      "resource_types": ["ec2"],
      "backend_type": "terraform",
      "terraform_operation": "apply",
      "input_schema": [
        {"key": "new_instance_type", "label": "New Instance Type", "type": "select",
         "options": [{"label": "t3.small", "value": "t3.small"}, ...]}
      ],
      "terraform_variable_mappings": [
        {"source": "input.new_instance_type", "target": "instance_type"}
      ]
    },
    {
      "id": "aws-rds-snapshot",
      "name": "Create RDS Snapshot",
      "description": "Create a manual snapshot of an RDS instance",
      "providers": ["aws"],
      "resource_types": ["rds"],
      "backend_type": "native",
      "native_action": "snapshot"
    }
  ]
}
```

---

## 13. Event Integration Points

### New Events to Add

| Event Type | When Fired | Source |
|-----------|-----------|--------|
| `terraform.run.proposed` | Plan-only run created | terraform_runs |
| `terraform.run.plan_complete` | Plan finished, awaiting action | terraform_runs |
| `terraform.run.promoted` | Proposed run promoted to tracked | terraform_runs |
| `terraform.run.applying` | Apply phase started | terraform_runs |
| `terraform.run.applied` | Apply completed successfully | terraform_runs |
| `terraform.run.failed` | Run failed at any phase | terraform_runs |
| `terraform.run.discarded` | Proposed run discarded | terraform_runs |
| `terraform.dependency.triggered` | Dependent workspace auto-triggered | dependencies |
| `terraform.dependency.cascade_failed` | Cascade cancelled due to parent failure | dependencies |
| `terraform.policy.denied` | Plan policy denied a run | plan_policies |
| `terraform.policy.approval_required` | Plan policy requires approval | plan_policies |
| `terraform.context.updated` | Context variables changed | contexts |
| `terraform.schedule.triggered` | Scheduled action fired | schedules |
| `terraform.schedule.drift_found` | Scheduled drift check found drift | schedules |
| `terraform.blueprint.deployed` | Blueprint deployment completed | blueprints |

### Event Automation Rule Examples

```json
{
  "name": "Notify on Production Plan Denial",
  "trigger": "terraform.policy.denied",
  "conditions": {"workspace_labels": ["env:production"]},
  "actions": [
    {"type": "notify_slack", "channel": "#infra-alerts"},
    {"type": "create_incident", "severity": "warning"}
  ]
}
```

---

## 14. Security Considerations

### Plan File Storage
- Plan files contain sensitive data (resource attributes, secrets)
- Store encrypted at rest (S3 with SSE-KMS or DynamoDB encryption)
- Auto-delete plan files after apply or after 24-hour TTL
- Access control: only workspace owner, admin, and approvers can view plans

### Context Secrets
- Secret variables: write-only (cannot be read back via API)
- Encrypted with Fernet (consistent with existing credential encryption)
- Masked in logs and UI (show `***` for secret values)
- Audit log entry when secrets are accessed during a run

### Policy Evaluation
- Plan policies run server-side (users cannot bypass)
- Policy results are immutable (cannot be altered after evaluation)
- Admin override: admin can force-approve a denied run (with audit trail)

### Dependency Security
- Output injection is scoped to tenant (cannot reference cross-tenant workspaces)
- Sensitive outputs require explicit opt-in per dependency
- Dependency creation requires admin role on parent workspace

---

## 15. References & Sources

- Spacelift Platform Capabilities: https://spacelift.io/platform/capabilities
- Spacelift Stack Documentation: https://docs.spacelift.io/concepts/stack
- Spacelift Stack Dependencies: https://docs.spacelift.io/concepts/stack/stack-dependencies
- Spacelift Drift Detection: https://docs.spacelift.io/concepts/stack/drift-detection
- Spacelift Blueprints: https://docs.spacelift.io/concepts/blueprint
- Spacelift Templates: https://docs.spacelift.io/concepts/template
- Spacelift Contexts: https://docs.spacelift.io/concepts/configuration/context
- Spacelift Policies: https://docs.spacelift.io/concepts/policy
- Spacelift Trigger Policy: https://docs.spacelift.io/concepts/policy/trigger-policy
- Spacelift Plan Policy: https://docs.spacelift.io/concepts/policy/terraform-plan-policy
- Spacelift Runs: https://docs.spacelift.io/concepts/run
- Spacelift Run Promotion: https://docs.spacelift.io/concepts/run/run-promotion
- Spacelift Intent: https://docs.spacelift.io/concepts/intent
- Spacelift Spaces: https://docs.spacelift.io/concepts/spaces
- Spacelift Worker Pools: https://docs.spacelift.io/concepts/worker-pools
- Spacelift Module Registry: https://docs.spacelift.io/vendors/terraform/module-registry

---

*Content was rephrased for compliance with licensing restrictions. All information is sourced from publicly available documentation and marketing materials.*

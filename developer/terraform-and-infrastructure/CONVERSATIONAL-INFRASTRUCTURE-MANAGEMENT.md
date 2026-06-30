# Conversational Infrastructure Management — Requirements & Design

## Overview

This document defines the requirements and technical design for extending CMP's AI chat copilot into a full **Conversational Infrastructure Management** layer. Users will deploy, scale, configure, and monitor cloud infrastructure entirely through natural language — with guardrails, approvals, and audit trails.

### Current State

CMP already has:
- AI chat endpoint with 50+ function-calling tools (Gemini + OpenAI-compatible)
- Role-aware, page-aware, resource-aware context injection
- Conversation persistence (DynamoDB)
- Event bus with automation rules engine
- Workflow orchestration (DAG-based)
- Terraform workspace management
- Resource actions (start/stop/restart/terminate)
- Shopping cart & bulk ordering

### Target State

Ten new capability areas that transform the chat from a read-heavy assistant into a full infrastructure control plane.

---

## Feature 1: Direct Cloud Operations via Chat

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| DCO-1 | Users can perform cloud operations (scale, resize, modify) directly via chat without creating a catalog/workflow | Must |
| DCO-2 | Operations are scoped to the user's accessible credentials only | Must |
| DCO-3 | Destructive operations (terminate, delete, modify security groups) require confirmation | Must |
| DCO-4 | Operations exceeding a configurable risk threshold require admin approval | Must |
| DCO-5 | All operations emit events and create audit log entries | Must |
| DCO-6 | Dry-run mode shows what WOULD change without applying | Should |
| DCO-7 | Cost impact estimation before applying changes | Should |
| DCO-8 | Support for AWS, Azure, and GCP operations | Must |

### Supported Operations (Phase 1)

| Category | Operations | Providers |
|----------|-----------|-----------|
| Compute | Start, stop, resize, reboot, change instance type | AWS EC2, Azure VM, GCP Compute |
| Scaling | Modify ASG/VMSS desired count, min/max | AWS ASG, Azure VMSS, GCP MIG |
| Storage | Create/delete bucket, modify lifecycle rules | S3, Azure Blob, GCS |
| Networking | Modify security group rules, create/delete DNS records | AWS SG/Route53, Azure NSG/DNS, GCP Firewall/DNS |
| Database | Resize, create read replica, modify parameters | RDS, Azure SQL, Cloud SQL |
| Serverless | Update function config, env vars, memory/timeout | Lambda, Azure Functions, Cloud Functions |

### Design

#### New Backend Components

```
backend/app/
├── services/
│   └── cloud_operations/
│       ├── __init__.py
│       ├── base.py              # Abstract CloudOperator interface
│       ├── aws_operator.py      # AWS-specific implementations
│       ├── azure_operator.py    # Azure-specific implementations
│       ├── gcp_operator.py      # GCP-specific implementations
│       ├── risk_assessor.py     # Calculates risk score for operations
│       └── operation_planner.py # Dry-run & plan generation
├── models/
│   └── cloud_operation.py       # CloudOperation, OperationPlan, RiskAssessment
```

#### CloudOperator Interface

```python
class CloudOperator(ABC):
    @abstractmethod
    async def execute(self, operation: CloudOperation, credential: dict) -> OperationResult:
        """Execute a cloud operation."""
        
    @abstractmethod
    async def dry_run(self, operation: CloudOperation, credential: dict) -> OperationPlan:
        """Show what would change without applying."""
        
    @abstractmethod
    async def estimate_cost_impact(self, operation: CloudOperation) -> CostImpact:
        """Estimate cost change from this operation."""
```

#### Risk Assessment Model

```python
class RiskLevel(str, Enum):
    LOW = "low"         # Read-only, non-destructive (e.g., tag, describe)
    MEDIUM = "medium"   # Reversible changes (e.g., stop, resize)
    HIGH = "high"       # Destructive or security-impacting (e.g., terminate, modify SG)
    CRITICAL = "critical"  # Irreversible data loss (e.g., delete bucket with data)

class RiskAssessment(BaseModel):
    level: RiskLevel
    score: int  # 0-100
    factors: List[str]
    requires_approval: bool
    requires_confirmation: bool
```

#### New Chat Tools

| Tool Name | Description | Risk | Status |
|-----------|-------------|------|--------|
| `cloud_resize_instance` | Change instance type/size | Medium | ✅ Registered (v8.15.55) |
| `cloud_scale_group` | Modify auto-scaling group capacity | Medium | ✅ Registered (v8.15.55) |
| `cloud_modify_security_group` | Add/remove SG rules | High | ✅ Registered (v8.15.55) |
| `cloud_manage_dns` | Create/modify/delete DNS records | Medium | Planned |
| `cloud_modify_storage` | Bucket lifecycle, versioning | Medium | Planned |
| `cloud_update_function` | Lambda/Function config changes | Medium | ✅ Registered (v8.15.55) |
| `cloud_create_snapshot` | Create disk/DB snapshot | Low | ✅ Registered (v8.15.55) |
| `cloud_modify_database` | Resize DB, modify params | High | ✅ Registered (v8.15.55) |
| `cloud_operation_dry_run` | Preview any operation without applying | None | ✅ Registered (v8.15.55) |

> **Implementation Note (v8.15.55):** Tool definitions for the above registered tools are declared in
> `backend/app/api/v1/endpoints/chat.py` within the function-calling tools list. Each tool uses a two-step
> confirmation pattern (`confirmed` parameter) — first call with `confirmed=false` to generate a plan,
> then `confirmed=true` to execute after user approval. The `cloud_operation_dry_run` tool provides a
> generic preview endpoint that accepts any `operation_type` and returns current state, proposed state,
> risk assessment, and cost impact without executing.

#### Flow

```
User: "Scale my web-asg to 6 instances"
  → AI resolves "web-asg" to resource in user's inventory
  → AI calls cloud_scale_group(resource_id, desired=6, credential_id)
  → risk_assessor scores it MEDIUM (reversible, no data loss)
  → AI confirms: "I'll scale web-asg from 3 → 6 instances. Est. cost: +$45/day. Proceed?"
  → User: "Yes"
  → aws_operator.execute() modifies ASG
  → Event emitted: CLOUD_OPERATION_EXECUTED
  → Audit log created
  → AI: "Done. web-asg now has desired_capacity=6. Changes take ~2 min to propagate."
```

---

## Feature 2: Conversational Deployment Pipelines

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| CDP-1 | Users can trigger deployments via natural language (e.g., "deploy v2.3 to staging") | Must |
| CDP-2 | AI infers deployment parameters from context (service name, version, environment) | Must |
| CDP-3 | Support rollback to previous version via chat | Must |
| CDP-4 | Deployment progress streamed back to chat (see Feature 9) | Should |
| CDP-5 | Blue/green and canary deployment orchestration via conversation | Should |
| CDP-6 | Pre-deployment checks (health, dependencies) run automatically | Must |
| CDP-7 | Production deployments require approval (integrates with existing approval system) | Must |
| CDP-8 | Deployment history queryable via chat | Must |

### Design

#### New Backend Components

```
backend/app/
├── services/
│   └── deployment/
│       ├── __init__.py
│       ├── deployment_service.py    # Core deployment orchestration
│       ├── deploy_strategies.py     # Rolling, blue/green, canary
│       ├── version_resolver.py      # Resolve "latest", "previous", semver
│       └── health_checker.py        # Pre/post deployment health checks
├── models/
│   └── deployment.py                # Deployment, DeploymentStrategy, DeployTarget
├── crud/
│   └── deployment.py                # DynamoDB CRUD for deployment records
```

#### Data Model

```python
class DeploymentStrategy(str, Enum):
    ROLLING = "rolling"
    BLUE_GREEN = "blue_green"
    CANARY = "canary"
    IN_PLACE = "in_place"

class DeployTarget(BaseModel):
    service_name: str
    environment: str           # dev, staging, production
    version: str               # semver, git sha, "latest", "previous"
    provider: str              # aws, azure, gcp
    target_resource_id: str    # ECS service, AKS deployment, Cloud Run, etc.
    credential_id: str

class DeploymentCreate(BaseModel):
    target: DeployTarget
    strategy: DeploymentStrategy = DeploymentStrategy.ROLLING
    pre_checks: List[str] = ["health", "dependencies"]
    rollback_on_failure: bool = True
    canary_percentage: Optional[int] = None  # For canary strategy
    approval_required: bool = False

class DeploymentInDB(BaseModel):
    deployment_id: str
    tenant_id: str
    target: DeployTarget
    strategy: DeploymentStrategy
    status: str  # pending, running, succeeded, failed, rolled_back
    triggered_by: str
    triggered_via: str = "chat"  # chat, api, automation
    previous_version: Optional[str]
    started_at: str
    completed_at: Optional[str]
    steps: List[Dict[str, Any]]
    error: Optional[str]
```

#### New Chat Tools

| Tool Name | Description |
|-----------|-------------|
| `deploy_service` | Trigger a deployment to a target environment |
| `rollback_deployment` | Rollback to previous version |
| `list_deployments` | Show deployment history |
| `get_deployment_status` | Check current deployment progress |
| `list_deployable_services` | Show services available for deployment |
| `compare_versions` | Show diff between two versions |

#### Flow

```
User: "Deploy order-service v2.3.1 to staging"
  → AI resolves service, finds ECS/AKS target via inventory
  → AI calls deploy_service(service="order-service", version="v2.3.1", env="staging")
  → Pre-checks: health OK, dependencies OK
  → Strategy: rolling (default for staging)
  → AI: "Deploying order-service v2.3.1 to staging (rolling). Currently running v2.3.0."
  → [Progress updates via streaming — Feature 9]
  → AI: "✓ Deployment complete. 3/3 replicas running v2.3.1. Health checks passing."

User: "Something's wrong, roll it back"
  → AI: "Rolling back order-service on staging from v2.3.1 → v2.3.0. Confirm?"
  → User: "Yes"
  → AI: "Rollback complete. All replicas running v2.3.0."
```

---

## Feature 3: Scaling & Auto-Scaling Configuration

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| ASC-1 | Users can view current scaling configuration via chat | Must |
| ASC-2 | Users can modify auto-scaling policies (min, max, target metrics) | Must |
| ASC-3 | Scheduled scaling (e.g., "scale up during business hours") | Should |
| ASC-4 | AI recommends scaling changes based on metrics/patterns | Should |
| ASC-5 | Cost projection for scaling changes shown before applying | Must |
| ASC-6 | Support horizontal (add instances) and vertical (resize) scaling | Must |
| ASC-7 | Cooldown period enforcement to prevent flapping | Must |
| ASC-8 | Scaling history and metrics queryable via chat | Should |

### Design

#### New Backend Components

```
backend/app/
├── services/
│   └── scaling/
│       ├── __init__.py
│       ├── scaling_service.py       # Core scaling operations
│       ├── scaling_scheduler.py     # Scheduled scaling (cron-based)
│       ├── scaling_advisor.py       # AI-powered scaling recommendations
│       └── metrics_collector.py     # CloudWatch/Monitor/Stackdriver metrics
├── models/
│   └── scaling.py                   # ScalingPolicy, ScalingSchedule, ScalingEvent
```

#### Data Model

Located in `backend/app/models/scaling.py`:

```python
class ScalingMetric(str, Enum):
    CPU_UTILIZATION = "cpu_utilization"
    MEMORY_UTILIZATION = "memory_utilization"
    REQUEST_COUNT = "request_count"
    NETWORK_IN = "network_in"
    NETWORK_OUT = "network_out"
    CUSTOM = "custom"

class ScalingPolicyConfig(BaseModel):
    """Current scaling policy configuration for a resource."""
    resource_id: str
    provider: str                             # aws, azure, gcp
    resource_type: str = "asg"               # asg, vmss, instance_group
    min_capacity: int = 1
    max_capacity: int = 10
    desired_capacity: int = 2
    target_metric: ScalingMetric = ScalingMetric.CPU_UTILIZATION
    target_value: float = 70.0               # e.g., 70% CPU
    scale_in_cooldown: int = 300             # seconds
    scale_out_cooldown: int = 60             # seconds
    region: Optional[str] = None
    credential_id: Optional[str] = None

class ScalingSchedule(BaseModel):
    """Scheduled scaling action."""
    schedule_id: str
    resource_id: str
    cron_expression: str                     # e.g., "0 8 * * MON-FRI"
    desired_capacity: int
    min_capacity: Optional[int] = None
    max_capacity: Optional[int] = None
    timezone: str = "UTC"
    enabled: bool = True
    description: Optional[str] = None

class ScalingRecommendation(BaseModel):
    """AI-generated scaling recommendation."""
    resource_id: str
    resource_name: Optional[str] = None
    current_config: Dict[str, Any] = Field(default_factory=dict)
    recommended_config: Dict[str, Any] = Field(default_factory=dict)
    reason: str = ""
    estimated_savings: Optional[float] = None
    confidence: float = 0.0                  # 0.0 - 1.0

class ScalingScheduleCreate(BaseModel):
    """Request to create a scaling schedule."""
    resource_id: str
    cron_expression: str
    desired_capacity: int
    min_capacity: Optional[int] = None
    max_capacity: Optional[int] = None
    timezone: str = "UTC"
    description: Optional[str] = None
    credential_id: str
    region: str = "us-east-1"

class ScalingPolicyUpdate(BaseModel):
    """Request to update scaling policy."""
    min_capacity: Optional[int] = None
    max_capacity: Optional[int] = None
    desired_capacity: Optional[int] = None
    target_metric: Optional[ScalingMetric] = None
    target_value: Optional[float] = None
    scale_in_cooldown: Optional[int] = None
    scale_out_cooldown: Optional[int] = None
```

#### New Chat Tools

| Tool Name | Description |
|-----------|-------------|
| `get_scaling_config` | Show current scaling settings for a resource |
| `update_scaling_policy` | Modify auto-scaling policy (min/max/target) |
| `scale_now` | Immediately scale to a specific count |
| `create_scaling_schedule` | Set up scheduled scaling |
| `get_scaling_recommendations` | AI-powered scaling advice based on metrics |
| `get_scaling_history` | Show past scaling events |
| `estimate_scaling_cost` | Project cost for a scaling change |

#### Flow

```
User: "My app is slow, can you help?"
  → AI checks current metrics via metrics_collector
  → AI: "Your web-cluster CPU is at 89% (4 instances). I recommend scaling to 6 instances. 
         Cost impact: +$30/day. Or I can set auto-scaling to target 70% CPU. What do you prefer?"
User: "Set auto-scaling to target 70% CPU, max 8 instances"
  → AI calls update_scaling_policy(target_metric="cpu", target_value=70, max=8)
  → AI: "Done. Auto-scaling updated: target CPU 70%, min 4, max 8. 
         The group should scale to ~6 instances within 5 minutes."
```

---

## Feature 4: Configuration Management via Chat

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| CFG-1 | Users can view resource configurations via chat | Must |
| CFG-2 | Users can modify resource configurations (env vars, settings, tags) | Must |
| CFG-3 | Bulk operations: apply changes to multiple resources at once | Should |
| CFG-4 | Secret rotation with automatic propagation to dependent services | Should |
| CFG-5 | Configuration drift detection (actual vs. desired) | Should |
| CFG-6 | Configuration templates/presets (e.g., "apply production hardening") | Should |
| CFG-7 | Change preview (diff) before applying | Must |
| CFG-8 | Configuration versioning and rollback | Should |

### Design

#### New Backend Components

```
backend/app/
├── services/
│   └── config_management/
│       ├── __init__.py
│       ├── config_service.py        # Core config read/write operations
│       ├── config_diff.py           # Generate diffs between current and desired
│       ├── config_templates.py      # Predefined config presets
│       ├── secret_rotation.py       # Credential/secret rotation orchestration
│       └── bulk_operations.py       # Apply changes to resource groups
├── models/
│   └── config_management.py         # ConfigChange, ConfigTemplate, BulkOperation
```

#### Data Model

```python
class ConfigChangeType(str, Enum):
    SET_ENV_VAR = "set_env_var"
    DELETE_ENV_VAR = "delete_env_var"
    SET_TAG = "set_tag"
    DELETE_TAG = "delete_tag"
    MODIFY_SETTING = "modify_setting"
    APPLY_TEMPLATE = "apply_template"
    ROTATE_SECRET = "rotate_secret"

class ConfigChange(BaseModel):
    change_type: ConfigChangeType
    resource_id: str
    key: str
    new_value: Optional[str]        # None for deletions
    old_value: Optional[str]        # For diff display
    sensitive: bool = False         # Mask value in logs

class ConfigChangeSet(BaseModel):
    changeset_id: str
    tenant_id: str
    changes: List[ConfigChange]
    status: str  # planned, previewed, applied, rolled_back
    applied_by: str
    applied_at: Optional[str]
    rollback_data: Optional[Dict]   # Stores old values for rollback

class ConfigTemplate(BaseModel):
    template_id: str
    name: str                       # "production-hardening", "cost-optimization"
    description: str
    changes: List[ConfigChange]     # Template of changes to apply
    applicable_resource_types: List[str]
    tags: List[str]
```

#### New Chat Tools

| Tool Name | Description |
|-----------|-------------|
| `get_resource_config` | Show current configuration of a resource |
| `modify_resource_config` | Change env vars, settings, or parameters |
| `apply_config_template` | Apply a predefined configuration template |
| `preview_config_changes` | Show diff without applying |
| `bulk_tag_resources` | Add/modify tags across multiple resources |
| `rotate_secret` | Rotate a secret and update all references |
| `list_config_templates` | Show available configuration presets |
| `rollback_config_change` | Revert a previous configuration change |

#### Flow

```
User: "Update the DATABASE_URL env var on my payment-service Lambda"
  → AI resolves Lambda function from inventory
  → AI calls preview_config_changes:
    "Current: DATABASE_URL = postgres://old-host:5432/payments
     New:     DATABASE_URL = [you haven't specified the new value]
     What should the new value be?"
User: "postgres://new-rds.cluster.us-east-1.rds.amazonaws.com:5432/payments"
  → AI shows preview diff, asks for confirmation
  → AI applies change via modify_resource_config
  → Event emitted: CONFIG_CHANGE_APPLIED
  → AI: "Updated. The Lambda will use the new DATABASE_URL on next invocation."

User: "Tag all untagged EC2 instances in us-east-1 with team=platform"
  → AI calls bulk_tag_resources(filter={region: "us-east-1", untagged: true}, tag={team: "platform"})
  → AI: "Found 7 untagged instances. I'll add team=platform to all of them. Confirm?"
  → User: "Go"
  → AI: "Done. 7 instances tagged. Here's the list: [...]"
```

---

## Feature 5: Proactive AI-Initiated Conversations

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| PAI-1 | AI generates proactive recommendations based on events, metrics, and patterns | Must |
| PAI-2 | Notifications delivered as AI messages (in-app, with suggested actions) | Must |
| PAI-3 | Categories: cost anomalies, drift, health degradation, lease expiry, security | Must |
| PAI-4 | Users can configure notification preferences (frequency, categories, severity) | Must |
| PAI-5 | Recommendations include one-click actionable suggestions | Should |
| PAI-6 | AI learns from user actions to reduce noise (snooze, dismiss patterns) | Should |
| PAI-7 | Digest mode: daily/weekly summary instead of real-time alerts | Should |
| PAI-8 | Admin can configure org-wide proactive rules | Should |

### Design

#### New Backend Components

```
backend/app/
├── services/
│   └── proactive_ai/
│       ├── __init__.py
│       ├── recommendation_engine.py  # Core recommendation generation
│       ├── pattern_detector.py       # Anomaly/pattern detection
│       ├── digest_service.py         # Daily/weekly digest compilation
│       └── user_preferences.py       # Per-user noise/preference management
├── models/
│   └── proactive_ai.py              # Recommendation, ProactiveAlert, DigestConfig
```

#### Data Model

```python
class RecommendationCategory(str, Enum):
    COST = "cost"                    # Budget overruns, idle resources, savings
    SECURITY = "security"            # Open ports, public buckets, expired certs
    HEALTH = "health"                # Degraded services, high error rates
    DRIFT = "drift"                  # Terraform drift, config drift
    LIFECYCLE = "lifecycle"          # Lease expiry, unused resources
    PERFORMANCE = "performance"      # High CPU/memory, slow queries
    COMPLIANCE = "compliance"        # Policy violations, audit findings

class RecommendationSeverity(str, Enum):
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"

class ProactiveRecommendation(BaseModel):
    recommendation_id: str
    tenant_id: str
    user_id: str                     # Target user
    category: RecommendationCategory
    severity: RecommendationSeverity
    title: str
    message: str                     # Natural language explanation
    suggested_actions: List[SuggestedAction]  # Clickable actions
    resource_ids: List[str]          # Related resources
    data: Dict[str, Any]             # Supporting metrics/evidence
    status: str                      # pending, seen, acted, dismissed, snoozed
    created_at: str
    expires_at: Optional[str]

class SuggestedAction(BaseModel):
    label: str                       # "Stop idle instances"
    action_type: str                 # "chat_command", "navigate", "api_call"
    payload: Dict[str, Any]          # Command/path/API details
    risk_level: str

class ProactiveConfig(BaseModel):
    """Per-user configuration for proactive alerts."""
    user_id: str
    enabled_categories: List[RecommendationCategory]
    severity_threshold: RecommendationSeverity  # Only show >= this level
    delivery_mode: str               # "realtime", "digest_daily", "digest_weekly"
    quiet_hours: Optional[Dict]      # {"start": "22:00", "end": "08:00"}
    snoozed_resources: List[str]     # Resources to ignore
```

#### Event-Driven Triggers

Hooks into the existing EventBus:

| Event Type | Proactive Check |
|------------|----------------|
| `COST_THRESHOLD_EXCEEDED` | Generate savings recommendations |
| `COST_ANOMALY_DETECTED` | Alert with root cause analysis |
| `RESOURCE_PROVISION_COMPLETED` | Check for missing tags, no lease, no budget |
| `TERRAFORM_DRIFT_DETECTED` | Alert with remediation options |
| Scheduled (hourly) | Idle resource detection, health checks |
| Scheduled (daily) | Compile digest, lease expiry warnings |

#### Frontend Integration

New component: `ProactiveInsightsPanel` — shows as a badge on the AI chat icon with unread recommendation count. Recommendations render as AI messages with action buttons.

```typescript
interface ProactiveInsight {
  id: string
  category: string
  severity: string
  title: string
  message: string
  actions: Array<{
    label: string
    onClick: () => void  // Sends command to chat or navigates
  }>
  timestamp: string
}
```

---

## Feature 6: Multi-Step Guided Conversations (Wizards)

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| MGC-1 | AI conducts multi-step conversations for complex operations | Must |
| MGC-2 | Steps are context-aware — AI skips questions it can infer from context | Must |
| MGC-3 | Users can go back to previous steps | Should |
| MGC-4 | Progress indicator shows current step in the flow | Should |
| MGC-5 | Wizard templates configurable by admins (define step sequences) | Should |
| MGC-6 | AI summarizes the full plan before execution | Must |
| MGC-7 | Partial completion: save progress and resume later | Should |
| MGC-8 | Contextual defaults based on user's history and preferences | Should |

### Design

#### New Backend Components

```
backend/app/
├── services/
│   └── guided_flows/
│       ├── __init__.py
│       ├── wizard_engine.py         # Core wizard state machine
│       ├── wizard_templates.py      # Template definitions and CRUD
│       ├── step_resolver.py         # Decides which steps to skip/show
│       └── plan_generator.py        # Generates execution plan summary
├── models/
│   └── guided_flow.py              # WizardTemplate, WizardSession, WizardStep
├── crud/
│   └── wizard_session.py           # Persist wizard sessions (resume later)
```

#### Data Model

```python
class WizardStepType(str, Enum):
    CHOICE = "choice"           # Select from options
    INPUT = "input"             # Free text/number input
    CONFIRM = "confirm"         # Yes/no confirmation
    MULTI_SELECT = "multi"      # Select multiple options
    RESOURCE_PICKER = "resource" # Pick from user's resources
    REVIEW = "review"           # Show plan summary

class WizardStep(BaseModel):
    step_id: str
    question: str               # What to ask the user
    step_type: WizardStepType
    options: Optional[List[str]]
    default_value: Optional[str]
    skip_condition: Optional[str]  # Expression: "context.provider == 'aws'"
    validation: Optional[str]      # Regex or expression for input validation
    infer_from: Optional[str]      # Context key to auto-fill from (e.g., "selected_resource.provider")

class WizardTemplate(BaseModel):
    template_id: str
    name: str                   # "Setup Microservice Environment"
    description: str
    category: str               # "provisioning", "security", "networking"
    steps: List[WizardStep]
    output_action: str          # What to do with collected data (e.g., "provision", "configure")
    required_role: str          # Minimum role to use this wizard
    tenant_id: str

class WizardSession(BaseModel):
    session_id: str
    template_id: str
    user_id: str
    tenant_id: str
    current_step: int
    collected_data: Dict[str, Any]  # Answers so far
    status: str                     # active, completed, abandoned
    created_at: str
    updated_at: str
```

#### Wizard Execution Flow

```
1. User intent triggers wizard detection (or user explicitly starts one)
2. AI loads template, resolves context-skippable steps
3. For each step:
   a. Check skip_condition — if met, auto-fill from context
   b. Check infer_from — if context has the value, use it
   c. Otherwise, ask the user
4. After all steps: generate plan summary (REVIEW step)
5. On confirmation: execute the output_action with collected_data
6. Session saved at each step for resume capability
```

#### Built-in Wizard Templates

| Template | Steps | Output |
|----------|-------|--------|
| Setup Microservice Environment | Provider → Region → VPC → Compute → DB → DNS | Multi-resource provisioning |
| Secure a Public Resource | Identify resource → Current exposure → Apply fixes | Security config changes |
| Migrate Database | Source → Target → Strategy → Schedule → Confirm | Migration execution |
| Set Up CI/CD Pipeline | Repo → Build → Test → Deploy Target → Strategy | Pipeline configuration |
| Onboard New Team Member | Name → Role → Groups → Credentials → Welcome | User + access creation |

---

## Feature 7: Infrastructure-as-Conversation (Desired State)

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| IAC-1 | Users describe desired infrastructure state in natural language | Must |
| IAC-2 | AI generates an infrastructure plan (Terraform/CloudFormation or direct API calls) | Must |
| IAC-3 | Plan shown as visual diagram + resource list before execution | Must |
| IAC-4 | Cost estimation for the complete plan | Must |
| IAC-5 | Support incremental changes to existing infrastructure | Should |
| IAC-6 | Compliance checking against org policies before execution | Must |
| IAC-7 | AI explains trade-offs between options (e.g., "HA costs 2x more") | Should |
| IAC-8 | Generated IaC code exportable (user can take the Terraform and run elsewhere) | Should |

### Design

#### New Backend Components

```
backend/app/
├── services/
│   └── infra_planner/
│       ├── __init__.py
│       ├── intent_parser.py         # Parse NL intent into structured requirements
│       ├── plan_generator.py        # Generate infra plan from requirements
│       ├── terraform_generator.py   # Generate Terraform HCL from plan
│       ├── cost_estimator.py        # Full-plan cost estimation
│       ├── compliance_checker.py    # Check plan against policies/quotas
│       └── plan_executor.py         # Execute plan (direct or via Terraform)
├── models/
│   └── infra_plan.py               # InfraPlan, InfraRequirement, PlanResource
```

#### Data Model

```python
class InfraRequirement(BaseModel):
    """Parsed user intent into structured requirements."""
    description: str                 # Original user request
    resources_needed: List[ResourceSpec]
    constraints: InfraConstraints
    preferences: InfraPreferences

class ResourceSpec(BaseModel):
    resource_type: str              # "compute", "database", "storage", "network", "dns"
    provider: Optional[str]         # "aws", "azure", "gcp" or None (AI picks)
    specific_type: Optional[str]    # "t3.medium", "db.r5.large"
    count: int = 1
    properties: Dict[str, Any]      # Additional specs (e.g., engine, version, size)

class InfraConstraints(BaseModel):
    budget_monthly: Optional[float]
    region: Optional[str]
    high_availability: bool = False
    multi_az: bool = False
    encryption_required: bool = True
    public_access: bool = False
    compliance_frameworks: List[str] = []  # "hipaa", "soc2", "pci"

class InfraPreferences(BaseModel):
    prefer_serverless: bool = False
    prefer_managed: bool = True
    minimize_cost: bool = False
    maximize_performance: bool = False

class InfraPlan(BaseModel):
    plan_id: str
    tenant_id: str
    user_id: str
    requirements: InfraRequirement
    resources: List[PlannedResource]
    dependencies: List[Dict[str, str]]   # [{source, target}]
    estimated_monthly_cost: float
    estimated_setup_cost: float
    terraform_code: Optional[str]        # Generated HCL
    policy_violations: List[str]         # Any compliance issues
    alternatives: List[InfraPlan]        # Cheaper/simpler alternatives
    status: str                          # draft, approved, executing, completed, failed
    created_at: str

class PlannedResource(BaseModel):
    resource_id: str                     # Temp ID within plan
    name: str
    provider: str
    resource_type: str
    specific_type: str                   # e.g., "aws_instance", "aws_db_instance"
    properties: Dict[str, Any]
    estimated_monthly_cost: float
    tags: Dict[str, str]
```

#### New Chat Tools

| Tool Name | Description |
|-----------|-------------|
| `plan_infrastructure` | Generate an infra plan from natural language description |
| `review_infra_plan` | Show plan details, resources, cost, compliance |
| `modify_infra_plan` | Adjust a generated plan (change provider, size, options) |
| `execute_infra_plan` | Apply the approved plan |
| `export_terraform` | Export the plan as Terraform HCL |
| `estimate_plan_cost` | Detailed cost breakdown for a plan |
| `check_plan_compliance` | Validate plan against org policies |

#### Flow

```
User: "I need a highly available web application with a database and CDN"
  → AI calls plan_infrastructure with parsed requirements:
    - 2 compute instances (HA, multi-AZ)
    - 1 managed database (multi-AZ, encrypted)
    - 1 load balancer
    - 1 CDN distribution
    - VPC, subnets, security groups
  → AI: "Here's my plan:
    ┌─────────────────────────────────────────┐
    │ VPC (10.0.0.0/16) — us-east-1           │
    │  ├── Public Subnet (AZ-a) ── ALB        │
    │  ├── Public Subnet (AZ-b) ── ALB        │
    │  ├── Private Subnet (AZ-a) ── EC2       │
    │  ├── Private Subnet (AZ-b) ── EC2       │
    │  └── Private Subnet (AZ-a) ── RDS (Multi-AZ) │
    │                                         │
    │ CloudFront CDN → ALB                    │
    └─────────────────────────────────────────┘
    
    Estimated: $285/mo | 8 resources | Multi-AZ HA
    
    Policy check: ✓ Passes all governance rules
    
    Want me to execute this, show Terraform code, or adjust anything?"

User: "Can you make it cheaper? I don't need CDN."
  → AI regenerates without CDN: "Updated plan: $195/mo (saves $90). Proceed?"
```

---

## Feature 8: Conversational Terraform Operations

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| TFO-1 | Users can run `terraform plan` on workspaces via chat | Must |
| TFO-2 | Plan output summarized in natural language (not raw HCL diff) | Must |
| TFO-3 | Users can approve and apply plans via chat | Must |
| TFO-4 | Drift detection results explained conversationally | Must |
| TFO-5 | Import existing resources into Terraform state via guided chat | Should |
| TFO-6 | Show workspace status, recent runs, state resources | Must |
| TFO-7 | Lock/unlock workspaces via chat | Should |
| TFO-8 | Variable management (set/update variables) via chat | Should |

### Design

#### Integration with Existing Terraform Module

CMP already has Terraform workspace management. This feature adds a conversational layer on top.

```
backend/app/
├── services/
│   └── terraform_chat/
│       ├── __init__.py
│       ├── plan_summarizer.py       # Summarize TF plan in natural language
│       ├── drift_explainer.py       # Explain drift in human terms
│       ├── import_wizard.py         # Guided resource import
│       └── variable_manager.py      # Manage workspace variables
```

#### New Chat Tools

| Tool Name | Description |
|-----------|-------------|
| `terraform_plan` | Run plan on a workspace, return NL summary |
| `terraform_apply` | Apply a pending plan (requires confirmation) |
| `terraform_show_drift` | Detect and explain drift |
| `terraform_import_resource` | Guided import of existing resource |
| `terraform_workspace_status` | Show workspace state summary |
| `terraform_set_variable` | Set/update a workspace variable |
| `terraform_lock_workspace` | Lock workspace to prevent changes |
| `terraform_list_resources` | List resources in state |
| `terraform_show_run_history` | Show recent plan/apply runs |

#### Plan Summarization

Raw Terraform plan output is translated to natural language:

```
Raw: + aws_instance.web (create)
       ami = "ami-12345"
       instance_type = "t3.medium"
     ~ aws_security_group_rule.web (modify)
       from_port: 80 -> 443
     - aws_s3_bucket.old_logs (destroy)

AI Summary:
"This plan will:
 1. Create a new t3.medium EC2 instance (web server)
 2. Change the security group rule from port 80 → 443 (HTTPS)
 3. ⚠️ Delete the 'old_logs' S3 bucket (DESTRUCTIVE — data will be lost)

 Total: +1 create, ~1 modify, -1 destroy
 Estimated cost change: +$30/mo (new instance) 
 
 The bucket deletion is irreversible. Confirm you want to proceed?"
```

#### Flow

```
User: "Run a plan on my networking workspace"
  → AI resolves workspace by name
  → AI calls terraform_plan(workspace_id="ws-abc123")
  → Plan runs asynchronously, result streamed back (Feature 9)
  → AI presents NL summary with risk flags
  → User: "Apply it"
  → AI: "Applying changes to 'networking' workspace..."
  → AI streams progress: "Creating VPC... Creating subnets... Done. 5 resources created."

User: "I think my production workspace has drifted"
  → AI calls terraform_show_drift(workspace_id="ws-prod")
  → AI: "Drift detected on 2 resources:
    1. aws_instance.app-server: instance_type changed from t3.medium to t3.large 
       (someone resized it manually via console)
    2. aws_security_group.app-sg: rule for port 8080 was added outside Terraform
    
    Options:
    - **Refresh state** to accept current reality
    - **Apply** to revert to Terraform-defined state
    - **Update code** to match current state"
```

---

## Feature 9: Real-Time Operation Streaming

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| RTS-1 | Long-running operations stream progress to the chat in real-time | Must |
| RTS-2 | Support Server-Sent Events (SSE) for streaming | Must |
| RTS-3 | Each operation step shows status (pending, running, success, failed) | Must |
| RTS-4 | Users can cancel a running operation mid-stream | Must |
| RTS-5 | Reconnection support (resume stream after disconnect) | Should |
| RTS-6 | Progress shown as structured steps (not raw log dump) | Must |
| RTS-7 | Final summary after operation completes | Must |
| RTS-8 | Multiple concurrent operations tracked independently | Should |

### Design

#### Architecture: Server-Sent Events (SSE)

SSE chosen over WebSocket because:
- Simpler to implement (HTTP/1.1 compatible)
- Works through most proxies/load balancers
- Automatic reconnection built into browser EventSource API
- One-directional (server → client) fits the progress streaming use case
- Cancel operations still use REST endpoint (POST /cancel)

#### New Backend Components

```
backend/app/
├── api/v1/endpoints/
│   └── chat_stream.py               # SSE streaming endpoint
├── services/
│   └── streaming/
│       ├── __init__.py
│       ├── stream_manager.py         # Manages active streams per user
│       ├── operation_tracker.py      # Tracks operation progress
│       └── sse_formatter.py          # Formats events for SSE protocol
```

#### SSE Endpoint

```python
@router.get("/chat/stream/{operation_id}")
async def stream_operation(
    operation_id: str,
    request: Request,
    current_user: dict = Depends(get_current_user),
):
    """Stream real-time progress for a long-running operation."""
    
    async def event_generator():
        async for event in operation_tracker.subscribe(operation_id, current_user["user_id"]):
            yield {
                "event": event.event_type,  # "step_started", "step_completed", "step_failed", "operation_complete"
                "data": json.dumps({
                    "step": event.step_index,
                    "total_steps": event.total_steps,
                    "step_name": event.step_name,
                    "status": event.status,
                    "message": event.message,
                    "timestamp": event.timestamp,
                    "duration_ms": event.duration_ms,
                })
            }
    
    return EventSourceResponse(event_generator())
```

#### Stream Event Data Model

```python
class StreamEventType(str, Enum):
    OPERATION_STARTED = "operation_started"
    STEP_STARTED = "step_started"
    STEP_PROGRESS = "step_progress"      # Intermediate progress within a step
    STEP_COMPLETED = "step_completed"
    STEP_FAILED = "step_failed"
    OPERATION_COMPLETED = "operation_completed"
    OPERATION_FAILED = "operation_failed"
    OPERATION_CANCELLED = "operation_cancelled"

class StreamEvent(BaseModel):
    event_type: StreamEventType
    operation_id: str
    step_index: int
    total_steps: int
    step_name: str
    status: str
    message: str
    timestamp: str
    duration_ms: Optional[int]
    metadata: Optional[Dict[str, Any]]  # Step-specific data

class OperationStream(BaseModel):
    operation_id: str
    operation_type: str          # "deployment", "terraform_apply", "scaling", "provisioning"
    user_id: str
    tenant_id: str
    steps: List[str]             # Ordered step names
    current_step: int
    status: str                  # running, completed, failed, cancelled
    started_at: str
    completed_at: Optional[str]
```

#### Frontend Integration

```typescript
// Hook for consuming operation streams
function useOperationStream(operationId: string) {
  const [steps, setSteps] = useState<StreamStep[]>([])
  const [status, setStatus] = useState<'running' | 'completed' | 'failed'>('running')
  
  useEffect(() => {
    const source = new EventSource(
      `${getApiUrl()}/api/v1/chat/stream/${operationId}`,
      { headers: { Authorization: `Bearer ${token}` } }
    )
    
    source.addEventListener('step_completed', (e) => {
      const data = JSON.parse(e.data)
      setSteps(prev => [...prev, { ...data, icon: '✓' }])
    })
    
    source.addEventListener('operation_completed', (e) => {
      setStatus('completed')
      source.close()
    })
    
    return () => source.close()
  }, [operationId])
  
  return { steps, status }
}
```

#### Chat UI Rendering

Operations render as a special "streaming card" in the chat:

```
┌──────────────────────────────────────────────┐
│ 🚀 Deploying order-service v2.3.1 → staging │
├──────────────────────────────────────────────┤
│ ✓ Building container image         (12s)     │
│ ✓ Pushing to ECR                   (5s)      │
│ ✓ Updating task definition         (2s)      │
│ ⏳ Rolling update (3/5 replicas)...          │
│ ○ Health check                               │
│ ○ Cleanup old tasks                          │
├──────────────────────────────────────────────┤
│ [Cancel Operation]                           │
└──────────────────────────────────────────────┘
```

---

## Feature 10: ChatOps Integration (Slack/Teams)

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| COP-1 | CMP chat accessible from Slack via slash commands and mentions | Must |
| COP-2 | CMP chat accessible from Microsoft Teams via bot | Must |
| COP-3 | Authentication links Slack/Teams identity to CMP user | Must |
| COP-4 | Same RBAC enforcement as web UI (role-based tool access) | Must |
| COP-5 | Interactive messages (buttons, dropdowns) for confirmations | Should |
| COP-6 | Thread-based conversations (each thread = one CMP conversation) | Should |
| COP-7 | Proactive alerts delivered to configured Slack/Teams channels | Should |
| COP-8 | Approval workflows actionable directly from Slack/Teams | Should |
| COP-9 | Audit trail captures Slack/Teams as the source channel | Must |

### Design

#### Architecture

```
┌─────────────┐     ┌───────────────┐     ┌─────────────────┐
│   Slack     │────▶│  CMP ChatOps  │────▶│  Existing Chat  │
│   Teams     │◀────│  Gateway      │◀────│  Engine         │
└─────────────┘     └───────────────┘     └─────────────────┘
                           │
                    ┌──────┴──────┐
                    │ Identity    │
                    │ Mapping     │
                    └─────────────┘
```

The ChatOps gateway is a thin adapter layer that:
1. Receives messages from Slack/Teams APIs
2. Maps external identity to CMP user
3. Forwards to the same chat engine (tool calling, context)
4. Formats responses back to platform-specific message format

#### New Backend Components

```
backend/app/
├── api/v1/endpoints/
│   └── chatops/
│       ├── __init__.py
│       ├── slack_handler.py         # Slack event/command handler
│       ├── teams_handler.py         # Teams bot framework handler
│       └── chatops_gateway.py       # Shared gateway logic
├── services/
│   └── chatops/
│       ├── __init__.py
│       ├── identity_mapper.py       # Map Slack/Teams ID → CMP user
│       ├── message_formatter.py     # Format responses for each platform
│       ├── interactive_handler.py   # Handle button clicks, dropdown selections
│       └── channel_config.py        # Channel → notification routing config
├── models/
│   └── chatops.py                   # ChatOpsConfig, IdentityMapping, ChannelBinding
├── crud/
│   └── chatops.py                   # DynamoDB CRUD for mappings and config
```

#### Data Model

```python
class ChatOpsPlatform(str, Enum):
    SLACK = "slack"
    TEAMS = "teams"

class IdentityMapping(BaseModel):
    mapping_id: str
    platform: ChatOpsPlatform
    external_user_id: str        # Slack user ID or Teams user principal
    external_username: str       # Display name on external platform
    cmp_user_id: str             # Linked CMP user
    tenant_id: str
    linked_at: str
    verified: bool = False

class ChatOpsConfig(BaseModel):
    """Per-tenant ChatOps configuration."""
    tenant_id: str
    slack_enabled: bool = False
    slack_bot_token: Optional[str]       # Encrypted
    slack_signing_secret: Optional[str]  # Encrypted
    slack_app_id: Optional[str]
    teams_enabled: bool = False
    teams_bot_id: Optional[str]
    teams_app_password: Optional[str]    # Encrypted
    teams_tenant_id: Optional[str]       # Azure AD tenant
    alert_channels: List[ChannelBinding] = []

class ChannelBinding(BaseModel):
    """Maps a Slack/Teams channel to receive specific notification types."""
    channel_id: str
    channel_name: str
    platform: ChatOpsPlatform
    notification_categories: List[str]  # ["cost", "security", "deployments"]
    severity_threshold: str             # "info", "warning", "critical"

class ChatOpsMessage(BaseModel):
    """Incoming message from external platform."""
    platform: ChatOpsPlatform
    external_user_id: str
    channel_id: str
    thread_id: Optional[str]
    message_text: str
    timestamp: str
    is_slash_command: bool = False
    command_name: Optional[str]
```

#### Slack Integration

**Slash Commands:**
- `/cmp deploy <service> <environment>` — trigger deployment
- `/cmp status [resource]` — check resource/execution status
- `/cmp approve <request-id>` — approve a pending request
- `/cmp scale <resource> <count>` — scale a resource
- `/cmp ask <question>` — general AI query

**Event Subscriptions:**
- `app_mention` — @CMP bot in any channel triggers AI response
- `message.im` — DM to bot for private queries

**Interactive Components:**
- Confirmation dialogs for destructive operations
- Approval buttons (Approve/Reject) in alert messages
- Dropdown selectors for multi-choice questions

#### Teams Integration

**Bot Commands:**
- Same command set as Slack but via Teams Messaging Extensions
- Adaptive Cards for rich interactive responses
- Task Modules for multi-step flows

#### Authentication Flow

```
1. User types `/cmp link` in Slack
2. CMP returns a one-time auth URL: https://cmp.example.com/chatops/link?code=ABC123
3. User clicks URL, logs into CMP
4. CMP creates IdentityMapping (Slack user → CMP user)
5. Future Slack commands use the mapped CMP identity (full RBAC)
```

#### Response Formatting

```python
class MessageFormatter:
    """Formats AI responses for different platforms."""
    
    def format_slack(self, ai_response: str, actions: List[SuggestedAction]) -> dict:
        """Convert to Slack Block Kit format."""
        blocks = [{"type": "section", "text": {"type": "mrkdwn", "text": ai_response}}]
        if actions:
            blocks.append({
                "type": "actions",
                "elements": [
                    {"type": "button", "text": {"type": "plain_text", "text": a.label}, 
                     "action_id": a.action_type, "value": json.dumps(a.payload)}
                    for a in actions
                ]
            })
        return {"blocks": blocks}
    
    def format_teams(self, ai_response: str, actions: List[SuggestedAction]) -> dict:
        """Convert to Teams Adaptive Card format."""
        card = {
            "type": "AdaptiveCard",
            "body": [{"type": "TextBlock", "text": ai_response, "wrap": True}],
            "actions": [
                {"type": "Action.Submit", "title": a.label, "data": a.payload}
                for a in actions
            ]
        }
        return card
```

---

## Cross-Cutting Concerns

### Security & Authorization

| Concern | Design |
|---------|--------|
| All operations enforce existing RBAC | Every chat tool checks `current_user.roles` before execution |
| Destructive ops require explicit confirmation | AI asks "Confirm?" before applying HIGH/CRITICAL risk ops |
| Multi-tenancy | All new services use `tenant_id` in every query |
| Secrets never exposed in chat | Credential values never returned; only IDs and names |
| ChatOps identity verified | External users must link account before accessing CMP |
| Audit trail for all operations | Every chat-initiated operation logs source="chat" or source="chatops_slack" |
| Rate limiting | All new endpoints use existing `@limiter.limit()` decorators |

### Event System Integration

All new features emit events via the existing EventBus:

```python
# New event types to add to EventType enum
class EventType(str, Enum):
    # ... existing ...
    
    # Feature 1: Direct Cloud Operations
    CLOUD_OPERATION_PLANNED = "cloud_operation.planned"
    CLOUD_OPERATION_EXECUTED = "cloud_operation.executed"
    CLOUD_OPERATION_FAILED = "cloud_operation.failed"
    
    # Feature 2: Deployments
    DEPLOYMENT_STARTED = "deployment.started"
    DEPLOYMENT_COMPLETED = "deployment.completed"
    DEPLOYMENT_FAILED = "deployment.failed"
    DEPLOYMENT_ROLLED_BACK = "deployment.rolled_back"
    
    # Feature 3: Scaling
    SCALING_POLICY_UPDATED = "scaling.policy_updated"
    SCALING_EXECUTED = "scaling.executed"
    SCALING_RECOMMENDATION = "scaling.recommendation"
    
    # Feature 4: Configuration
    CONFIG_CHANGE_APPLIED = "config.change_applied"
    CONFIG_CHANGE_ROLLED_BACK = "config.change_rolled_back"
    SECRET_ROTATED = "secret.rotated"
    
    # Feature 5: Proactive AI
    PROACTIVE_RECOMMENDATION = "proactive.recommendation"
    RECOMMENDATION_ACTED = "proactive.acted"
    
    # Feature 7: Infra Planning
    INFRA_PLAN_CREATED = "infra.plan_created"
    INFRA_PLAN_EXECUTED = "infra.plan_executed"
    
    # Feature 8: Terraform Chat
    TERRAFORM_PLAN_VIA_CHAT = "terraform.chat_plan"
    TERRAFORM_APPLY_VIA_CHAT = "terraform.chat_apply"
```

### Feature Toggle Integration

Each feature gated behind a toggle:

| Toggle Key | Feature | Default |
|------------|---------|---------|
| `chat_cloud_operations` | Direct Cloud Operations | off |
| `chat_deployments` | Conversational Deployments | off |
| `chat_scaling` | Scaling via Chat | off |
| `chat_config_management` | Configuration Management | off |
| `proactive_ai` | Proactive AI Recommendations | off |
| `guided_wizards` | Multi-step Guided Flows | on |
| `infra_as_conversation` | Infrastructure Planning | off |
| `chat_terraform` | Conversational Terraform | off |
| `chat_streaming` | Real-time Streaming | on |
| `chatops_slack` | Slack Integration | off |
| `chatops_teams` | Teams Integration | off |

### Database Schema (DynamoDB)

New tables/indexes needed:

| Table/SK Pattern | Purpose |
|-----------------|---------|
| `CLOUD_OP#{op_id}` | Cloud operation records |
| `DEPLOYMENT#{deploy_id}` | Deployment records |
| `SCALING_SCHEDULE#{sched_id}` | Scaling schedules |
| `CONFIG_CHANGE#{change_id}` | Configuration change history |
| `RECOMMENDATION#{rec_id}` | Proactive recommendations |
| `WIZARD_SESSION#{session_id}` | Wizard session state |
| `INFRA_PLAN#{plan_id}` | Infrastructure plans |
| `CHATOPS_MAPPING#{mapping_id}` | Identity mappings |
| `CHATOPS_CONFIG#{tenant_id}` | ChatOps configuration |
| `STREAM#{operation_id}` | Active operation streams |

All follow existing pattern: `PK = TENANT#{tenant_id}`, `SK = <pattern above>`

---

## Implementation Phases

### Phase 1 — Foundation (4 weeks)

| Week | Deliverable |
|------|-------------|
| 1-2 | Feature 9: Real-Time Streaming (SSE endpoint, operation tracker, frontend hook) |
| 3-4 | Feature 1: Direct Cloud Operations — AWS only (resize, scale ASG, modify SG) |

**Why start here:** Streaming is a dependency for Features 2, 3, 8. Cloud operations on AWS validates the pattern before expanding.

### Phase 2 — Core Operations (6 weeks)

| Week | Deliverable |
|------|-------------|
| 5-6 | Feature 8: Conversational Terraform (plan/apply/drift via existing TF module) |
| 7-8 | Feature 3: Scaling & Auto-Scaling (AWS ASG + scheduling) |
| 9-10 | Feature 4: Configuration Management (env vars, tags, bulk ops) |

### Phase 3 — Intelligence Layer (4 weeks)

| Week | Deliverable |
|------|-------------|
| 11-12 | Feature 5: Proactive AI (cost/health/drift recommendations) |
| 13-14 | Feature 6: Multi-Step Guided Wizards (engine + 3 built-in templates) |

### Phase 4 — Advanced Capabilities (6 weeks)

| Week | Deliverable |
|------|-------------|
| 15-17 | Feature 7: Infrastructure-as-Conversation (planning + Terraform generation) |
| 18-20 | Feature 2: Conversational Deployments (ECS/AKS rolling + blue/green) |

### Phase 5 — External Integrations (4 weeks)

| Week | Deliverable |
|------|-------------|
| 21-22 | Feature 10: Slack ChatOps (slash commands, interactive messages, alerts) |
| 23-24 | Feature 10: Teams ChatOps (bot framework, adaptive cards) |

### Phase 6 — Multi-Cloud Expansion (ongoing)

- Extend all operations to Azure and GCP
- Cloud-specific optimizations
- Platform-specific features (Azure Advisor, AWS Trusted Advisor, GCP Recommender)

---

## API Endpoint Summary

All new endpoints added under `/api/v1/`:

| Method | Path | Feature | Description |
|--------|------|---------|-------------|
| POST | `/chat` | Existing | Enhanced with new tools |
| GET | `/chat/stream/{operation_id}` | 9 | SSE streaming endpoint |
| POST | `/chat/stream/{operation_id}/cancel` | 9 | Cancel running operation |
| POST | `/cloud-operations` | 1 | Execute cloud operation |
| POST | `/cloud-operations/dry-run` | 1 | Dry-run cloud operation |
| GET | `/cloud-operations` | 1 | List operations history |
| POST | `/deployments` | 2 | Trigger deployment |
| POST | `/deployments/{id}/rollback` | 2 | Rollback deployment |
| GET | `/deployments` | 2 | List deployments |
| GET | `/deployments/{id}` | 2 | Get deployment details |
| GET | `/scaling/{resource_id}` | 3 | Get scaling config |
| PUT | `/scaling/{resource_id}` | 3 | Update scaling policy |
| POST | `/scaling/schedules` | 3 | Create scaling schedule |
| GET | `/scaling/recommendations` | 3 | Get scaling recommendations |
| POST | `/config-changes` | 4 | Apply config changes |
| POST | `/config-changes/preview` | 4 | Preview changes (diff) |
| POST | `/config-changes/{id}/rollback` | 4 | Rollback a change |
| GET | `/config-templates` | 4 | List config templates |
| GET | `/recommendations` | 5 | List proactive recommendations |
| PUT | `/recommendations/{id}` | 5 | Act/dismiss/snooze a recommendation |
| GET | `/recommendations/config` | 5 | Get user's proactive config |
| PUT | `/recommendations/config` | 5 | Update proactive config |
| POST | `/wizards/start` | 6 | Start a wizard session |
| POST | `/wizards/{session_id}/step` | 6 | Answer a wizard step |
| GET | `/wizards/{session_id}` | 6 | Get wizard session state |
| GET | `/wizard-templates` | 6 | List available wizards |
| POST | `/infra-plans` | 7 | Generate infrastructure plan |
| PUT | `/infra-plans/{id}` | 7 | Modify plan |
| POST | `/infra-plans/{id}/execute` | 7 | Execute approved plan |
| GET | `/infra-plans/{id}/terraform` | 7 | Export as Terraform |
| POST | `/chatops/slack/events` | 10 | Slack event webhook |
| POST | `/chatops/slack/commands` | 10 | Slack slash commands |
| POST | `/chatops/slack/interactive` | 10 | Slack interactive actions |
| POST | `/chatops/teams/messages` | 10 | Teams bot messages |
| POST | `/chatops/link` | 10 | Link external identity |
| GET | `/chatops/config` | 10 | Get ChatOps config |
| PUT | `/chatops/config` | 10 | Update ChatOps config |

---

## Frontend Components

| Component | Feature | Description |
|-----------|---------|-------------|
| `OperationStreamCard` | 9 | Renders streaming operation progress in chat |
| `ProactiveInsightsBadge` | 5 | Badge on AI chat icon with unread count |
| `ProactiveInsightsPanel` | 5 | Panel showing actionable recommendations |
| `WizardProgress` | 6 | Step indicator for guided flows |
| `InfraPlanViewer` | 7 | Visual diagram of planned infrastructure |
| `TerraformPlanSummary` | 8 | Formatted plan diff view |
| `ChatOpsSetup` | 10 | Admin page for Slack/Teams configuration |
| `IdentityLinkFlow` | 10 | User flow to link Slack/Teams account |

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Chat-initiated operations per week | >50% of total ops | Audit log source="chat" |
| Mean time to resolve (scaling issues) | <2 min via chat vs ~10 min via UI | Operation timestamps |
| Proactive recommendation action rate | >30% acted upon | Recommendation status tracking |
| ChatOps adoption | >40% of active users linked | Identity mapping count |
| Wizard completion rate | >70% started → completed | Wizard session status |
| Streaming operation cancel rate | <5% (ops complete without cancellation) | Stream event tracking |
| Deployment success rate via chat | >95% | Deployment records |
| User satisfaction (NPS) | >60 for AI chat features | In-app survey |

---

## Dependencies & Prerequisites

| Dependency | Required By | Status |
|-----------|-------------|--------|
| Existing chat engine (50+ tools) | All features | ✅ Done |
| EventBus + automation rules | Features 1-5 | ✅ Done |
| Terraform workspace management | Feature 8 | ✅ Done |
| Resource inventory + sync | Features 1, 3, 4 | ✅ Done |
| Feature toggles system | All features | ✅ Done |
| Credential management (encrypted) | Features 1-4, 7, 8 | ✅ Done |
| Approval workflows | Features 1, 2, 7 | ✅ Done |
| `sse-starlette` package | Feature 9 | ❌ To install |
| Slack Bolt SDK (`slack-bolt`) | Feature 10 | ❌ To install |
| Teams Bot Framework SDK | Feature 10 | ❌ To install |
| Cloud SDK metric APIs (CloudWatch, Monitor) | Features 3, 5 | Partial (boto3 exists) |


---

## ⚠️ CRITICAL: Backward Compatibility & Non-Breaking Guarantee

**All features in this document MUST be implemented without breaking any existing functionality.**

### Hard Rules

1. **Existing chat endpoint (`POST /api/v1/chat`) must remain fully functional.** New tools are additive — never remove, rename, or change the signature of existing tools.

2. **Existing API contracts are immutable.** No changes to request/response schemas on existing endpoints. New fields may be added as optional, never required.

3. **Existing event types and handlers must not be modified.** New event types are appended to the `EventType` enum. Existing handler subscriptions remain unchanged.

4. **Feature toggles gate all new behavior.** Every new capability is behind a toggle (default: off). Existing users experience zero change until an admin explicitly enables a feature.

5. **Database schema is additive only.** New SK patterns are added alongside existing ones. No migrations that alter existing items. No table deletions or key schema changes.

6. **Existing orchestrator/execution engine untouched.** New deployment and scaling services operate independently. The existing `orchestrator.py` action registry (`run_task`, `http_request`, `builtin.*`) remains as-is.

7. **Frontend is additive.** New components and hooks are added. Existing pages, contexts (`AuthContext`, `BrandingContext`, `FeatureContext`, `AIContext`), and routing remain unchanged. New UI surfaces behind feature flags.

8. **Existing RBAC and auth flow unchanged.** JWT structure, role hierarchy, `get_current_user`, `require_role` — all preserved. New features layer on top of existing permission checks.

9. **No new required environment variables.** New services use optional env vars with sensible defaults or admin-configured DB settings (like existing `ai_config`).

10. **Existing tests must continue to pass.** New features include their own test coverage. No modification to existing test assertions.

### Implementation Checklist (per feature)

Before merging any feature branch:

- [ ] All existing API endpoint tests pass unchanged
- [ ] All existing chat tools function identically (regression test suite)
- [ ] `docker compose up` starts successfully with no new required config
- [ ] Feature disabled (toggle off) → platform behaves exactly as before
- [ ] No breaking changes to `models/` Pydantic schemas used by existing endpoints
- [ ] No import cycle introduced by new service modules
- [ ] Existing EventBus handlers fire correctly for all pre-existing event types
- [ ] Frontend builds without errors (`tsc && npm run build`)
- [ ] No new required props on existing React components
- [ ] Existing Terraform workspace operations (plan/apply/destroy) unaffected


---

## UX Decision: Side Panel vs. Dedicated Page

### Decision: Keep the existing AI side panel as the primary interface

The current slide-out panel (`⌘J` / `Ctrl+J`, 440px width) remains the single entry point for all conversational infrastructure management. **No separate AI chat page will be created.**

### Rationale

| Factor | Side Panel (chosen) | Dedicated Page (rejected) |
|--------|--------------------|-----------------------------|
| Page context | ✅ Preserves `current_page`, `selected_resource`, `page_category` — AI knows where the user is | ❌ User navigates away, AI loses context about what they're looking at |
| "Operate alongside" | ✅ User says "stop this resource" while viewing the resource | ❌ Forces context switch — user must describe the resource instead of pointing at it |
| Quick interactions | ✅ Open/close instantly without route change | ❌ Full navigation required |
| Complex outputs | Can be solved with expand mode (see below) | Only advantage: more space |
| Discoverability | ✅ Always accessible from any page | ❌ One more nav item competing for attention |
| Mobile | ✅ Full-screen overlay on mobile already works | Equivalent |

### Panel Enhancements (to support infra management UX)

Instead of a new page, enhance the existing panel with:

#### 1. Expandable/Full-Width Mode
- Toggle button in panel header: `⬜ Expand` / `⬜ Collapse`
- Expanded: panel takes ~70% viewport width (for infra plans, Terraform diffs, streaming cards)
- Collapsed: default 440px side panel
- Keyboard shortcut: `⌘⇧J` to toggle expand

#### 2. Operation Stream Cards (inline)
- Long-running operations render as structured cards inside the chat
- Shows step-by-step progress with status icons (✓ ⏳ ✗)
- Cancel button embedded in the card
- No need to navigate elsewhere to monitor

#### 3. Pin/Float Mode
- Users can "pin" the panel so it persists across page navigations
- Currently, navigating away closes context — pinned mode maintains the conversation
- Useful for multi-step wizards where the user needs to check other pages mid-flow

#### 4. Deep-Link Support
- URL format: `/t/{tenant}/any-page?ai=open&conversation={id}`
- Opens the panel with a specific conversation pre-loaded
- Useful for: sharing conversations, linking from notifications, ChatOps "view in CMP" links
- Does NOT create a new route/page — just opens the panel overlay

#### 5. Detached Window (future)
- Pop the AI panel into a separate browser window
- Useful for multi-monitor setups where users want persistent AI alongside the main app
- Low priority — implement only if user demand exists

### When a Dedicated Page IS Appropriate

A dedicated `/ai-ops` page (AI Operations Center) may be added in Phase 3+ as a **read-only dashboard**, not a chat replacement:

| Content | Purpose |
|---------|---------|
| Active streaming operations | Monitor all running deployments/scaling/plans |
| Proactive recommendations feed | Aggregated view of all pending recommendations |
| Recent AI-initiated actions | Audit trail of what AI did on behalf of users |
| ChatOps activity | Messages from Slack/Teams mapped to CMP operations |

This page would complement the panel (link: "View in AI Ops" from a streaming card) but would NOT include a chat input. Chat always happens in the side panel.

### Implementation Notes

```typescript
// AIAssistantPanel.tsx — add expand state
const [isExpanded, setIsExpanded] = useState(false)

// Panel width classes
const panelWidth = isExpanded 
  ? 'lg:w-[70vw] sm:w-full' 
  : 'lg:w-[440px] sm:w-[400px]'

// Main content margin adjustment
const contentMargin = aiPanelOpen 
  ? (isExpanded ? 'lg:mr-[70vw]' : 'lg:mr-[440px] sm:mr-[400px]') 
  : ''
```

```typescript
// Deep-link support in App.tsx
useEffect(() => {
  const params = new URLSearchParams(location.search)
  if (params.get('ai') === 'open') {
    setAiPanelOpen(true)
    const convoId = params.get('conversation')
    if (convoId) {
      // Signal AIAssistantPanel to load this conversation
      setInitialConversationId(convoId)
    }
  }
}, [location.search])
```

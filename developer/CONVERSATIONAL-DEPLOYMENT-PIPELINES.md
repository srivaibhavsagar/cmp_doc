# Conversational Deployment Pipelines — Developer Guide

## Overview

Conversational Deployment Pipelines allow users to trigger, monitor, and rollback service deployments entirely through natural language via the AI chat copilot. This feature sits on top of CMP's execution engine and provides a deployment-specific abstraction focused on service version management and deployment strategies.

**Status:** Core service implemented (v8.15.56) — models, CRUD, and deployment service with rollback, status tracking, and AWS provider support are live. Chat tool registration and additional provider strategies forthcoming.

**Related docs:**
- [Conversational Infrastructure Management (full design)](./CONVERSATIONAL-INFRASTRUCTURE-MANAGEMENT.md) — Feature 2
- [Deployment Architecture Comparison](./DEPLOYMENT-ARCHITECTURE-COMPARISON.md) — CMP's own deployment topology

---

## Data Models

Location: `backend/app/models/deployment.py`

### Enums

| Enum | Values | Purpose |
|------|--------|---------|
| `DeploymentStrategy` | `rolling`, `blue_green`, `canary`, `in_place` | How the new version is rolled out |
| `DeploymentStatus` | `pending`, `pre_checks`, `running`, `completed`, `failed`, `rolled_back`, `cancelled` | Lifecycle state of a deployment |
| `DeploymentProvider` | `aws_ecs`, `aws_lambda`, `aws_ec2`, `azure_app_service`, `azure_aks`, `gcp_cloud_run`, `gcp_gke`, `generic` | Cloud-specific deployment target |

### Core Schemas

| Model | Role |
|-------|------|
| `DeployTarget` | Where to deploy: service name, environment, version, provider, credential, region |
| `DeploymentStep` | A single step in the pipeline (e.g., "pre-check health", "drain old tasks", "register new version") |
| `DeploymentCreate` | Request payload to start a deployment — includes strategy, rollback preference, canary %, pre-checks, and confirmation flag |
| `DeploymentInDB` | Full persisted record in DynamoDB — tracks versions, steps, timing, errors, and rollback lineage |
| `DeploymentSummary` | Lightweight projection for listing/history views |

### Key Design Decisions

1. **Two-step confirmation pattern** — `DeploymentCreate.confirmed` defaults to `False`. First call generates a plan; second call with `confirmed=True` executes. This mirrors the pattern used by cloud operations tools.

2. **Provider-specific targets** — `DeploymentProvider` enumerates concrete cloud services (ECS, Lambda, AKS, Cloud Run, etc.) rather than generic cloud names. This allows provider-specific deployment logic in the service layer.

3. **Rollback lineage** — `DeploymentInDB.rollback_of` links a rollback deployment to the original, enabling audit trail traversal.

4. **Step-level tracking** — Each deployment records individual `DeploymentStep` entries with timing, enabling progress streaming and post-mortem analysis.

5. **Trigger source tracking** — `triggered_via` distinguishes chat-initiated deployments from API calls or automation-triggered ones.

---

## DynamoDB Key Schema

Following CMP's single-table design:

```
PK: TENANT#{tenant_id}
SK: DEPLOYMENT#{deployment_id}
```

For listing by environment or service:

```
GSI1-PK: TENANT#{tenant_id}#ENV#{environment}
GSI1-SK: DEPLOYMENT#{deployment_id}
```

---

## Implemented Components

### Service Layer (`services/deployment/deployment_service.py`)

| Function | Status | Responsibility |
|----------|--------|---------------|
| `execute_deployment()` | ✅ Implemented | Create deployment record, resolve previous version, emit start event |
| `run_deployment()` | ✅ Implemented | Background pipeline execution — iterates steps, updates status, handles failures, emits events |
| `rollback_deployment()` | ✅ Implemented | Creates a new in-place deployment targeting the previous version, links via `rollback_of`, marks original as rolled back |
| `get_deployment_status()` | ✅ Implemented | Returns human-readable progress with step breakdown and rollback availability |
| `_execute_deployment_step()` | ✅ Implemented | Step dispatcher — routes to provider-specific handlers or generic simulation |
| `_aws_ecs_health_check()` | ✅ Implemented | Describes ECS service, checks running vs desired task count |
| `_aws_ecs_rolling_update()` | ✅ Implemented | Forces new ECS deployment via `update_service(forceNewDeployment=True)` |
| `_aws_lambda_update()` | ✅ Implemented | Updates Lambda function configuration with new version description |

### Planned (Not Yet Implemented)

| File | Responsibility |
|------|---------------|
| `deploy_strategies.py` | Dedicated strategy implementations: blue/green canary traffic shifting |
| `version_resolver.py` | Resolve `"latest"`, `"previous"`, semver tags to concrete versions |
| `health_checker.py` | Pre/post deployment health validation (beyond current ECS check) |

### CRUD (`crud/deployment.py`)

Standard async DynamoDB CRUD following CMP patterns:
- `create_deployment(tenant_id, deployment_data)` ✅
- `get_deployment(tenant_id, deployment_id)` ✅
- `get_latest_deployment_for_service(service_name, environment, tenant_id)` ✅
- `list_deployments(tenant_id, filters)` ✅
- `update_deployment_status(tenant_id, deployment_id, status, step_updates)` ✅

### Chat Tools (to be registered in `api/v1/endpoints/chat.py`)

| Tool | Description | Status |
|------|-------------|--------|
| `deploy_service` | Trigger a deployment to a target environment | Pending registration |
| `rollback_deployment` | Rollback to previous version | Pending registration |
| `list_deployments` | Show deployment history | Pending registration |
| `get_deployment_status` | Check current deployment progress | Pending registration |
| `list_deployable_services` | Show services available for deployment | Pending registration |

---

## Example Flow

```
User: "Deploy order-service v2.3.1 to staging"

1. AI resolves "order-service" from tenant's resource inventory
2. AI calls deploy_service(service="order-service", version="v2.3.1", env="staging", confirmed=false)
3. Service returns plan: current version, target version, strategy, pre-checks
4. AI presents: "I'll deploy order-service v2.3.1 to staging using rolling strategy. Currently running v2.3.0. Proceed?"
5. User: "Yes"
6. AI calls deploy_service(..., confirmed=true)
7. Deployment runs: pre-checks → drain → register new version → health check → complete
8. AI: "✓ Deployment complete. 3/3 replicas running v2.3.1."

User: "Roll it back"
→ AI calls rollback_deployment(deployment_id, confirmed=true)
→ Creates new deployment with rollback_of=original_id, version=previous_version
```

---

## Rollback Behavior

Rollbacks create a **new deployment** rather than reverting state in-place. This preserves full audit trail and step-level tracking for the rollback itself.

### How it works

1. User requests rollback (e.g., "Roll back that deployment")
2. System looks up the original deployment's `previous_version`
3. If no previous version exists, rollback is rejected
4. A new `DeploymentCreate` is built targeting the previous version with `strategy=in_place` for speed
5. The new deployment is linked to the original via `rollback_of`
6. The original deployment status is set to `rolled_back`
7. A `DEPLOYMENT_ROLLED_BACK` event is emitted

### Constraints

- Rollback requires `previous_version` to be set on the original deployment
- Rollback is only available when status is `completed` or `failed`
- Rollbacks always use `in_place` strategy (fastest path back to known-good state)
- Rollback deployments themselves have `rollback_on_failure=False` to prevent infinite loops

---

## Deployment Status Response

`get_deployment_status()` returns a structured dict suitable for chat display:

```json
{
  "deployment_id": "dep_abc123",
  "service_name": "order-service",
  "environment": "staging",
  "version": "v2.3.1",
  "previous_version": "v2.3.0",
  "strategy": "rolling",
  "status": "completed",
  "progress": "3/3 steps",
  "steps": [
    {"name": "Pull artifact", "status": "success", "message": "Artifact ready for version v2.3.1", "duration_ms": 1020},
    {"name": "Rolling update", "status": "success", "message": "ECS rolling update initiated for order-service", "duration_ms": 2150},
    {"name": "Health check", "status": "success", "message": "ECS service healthy: 3/3 tasks running", "duration_ms": 1080}
  ],
  "error": null,
  "started_at": "2025-01-15T10:30:00Z",
  "completed_at": "2025-01-15T10:30:05Z",
  "duration_ms": 4250,
  "triggered_by": "vaibhav",
  "rollback_available": true
}
```

The `rollback_available` field is `true` when:
- A `previous_version` is recorded, AND
- The deployment status is `completed` or `failed`

---

## Step Execution (Provider Dispatch)

The `_execute_deployment_step()` function routes steps to provider-specific handlers based on step name keywords and the target provider:

| Step Name Contains | Provider | Handler |
|-------------------|----------|---------|
| "health check" | `aws_ecs` | `_aws_ecs_health_check` — describes service, checks running/desired |
| "health check" | other | Generic pass (simulated) |
| "rolling update" | `aws_ecs` | `_aws_ecs_rolling_update` — force new deployment |
| "rolling update" | `aws_lambda` | `_aws_lambda_update` — update function config |
| "pull" / "build" | any | Simulated artifact preparation |
| "stop service" | any | Simulated service stop |
| "start service" | any | Simulated service start |
| "deploy new version" | any | Simulated version deploy |
| "switch traffic" | any | Simulated traffic shift (blue/green) |
| "canary" | any | Simulated canary rollout |
| "cleanup" | any | Simulated cleanup |

> **Note:** Non-AWS providers currently use simulated steps. As provider support is added (Azure, GCP), real API calls will replace the simulation layer.

---

## Integration Points

| System | Integration |
|--------|-------------|
| **Event Bus** | Emits `DEPLOYMENT_STARTED`, `DEPLOYMENT_COMPLETED`, `DEPLOYMENT_FAILED`, `DEPLOYMENT_ROLLED_BACK` |
| **Credentials** | Uses stored cloud credentials via `credential_id` — Fernet-encrypted |
| **Approvals** | Production deployments can require approval (via existing approval system) |
| **Audit Log** | All deployments create audit entries with actor, resource, and metadata |
| **Feature Toggle** | Gated behind `conversational_deployments` feature toggle |

---

## Provider-Specific Notes

| Provider | Deployment Mechanism | Status |
|----------|---------------------|--------|
| AWS ECS | Force new deployment via `update_service`, health check via `describe_services` | ✅ Implemented |
| AWS Lambda | Update function configuration with version description | ✅ Implemented |
| AWS EC2 | Instance refresh in ASG or in-place code deploy | Planned |
| Azure App Service | Deployment slots for blue/green, direct push for rolling | Planned |
| Azure AKS | kubectl rollout or Helm upgrade | Planned |
| GCP Cloud Run | Deploy new revision, shift traffic | Planned |
| GCP GKE | kubectl rollout or config connector | Planned |
| Generic | Simulated steps with appropriate delays | ✅ Implemented (fallback) |

---

## Security Considerations

- Deployments respect RBAC: only users with `developer` or `admin` role can trigger deployments
- Production environments require additional confirmation or approval
- Credential access validated before deployment starts — user must have access to the referenced `credential_id`
- All deployment actions are tenant-scoped (no cross-tenant access)
- Secrets (image registry tokens, deploy keys) are never logged in step messages

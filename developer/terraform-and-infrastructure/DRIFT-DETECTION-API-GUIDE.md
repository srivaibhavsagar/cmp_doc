# Drift Detection API — Developer Guide

> API reference and data structures for CMP's Terraform drift detection system.

---

## Overview

The drift detection service monitors Terraform workspaces for out-of-band infrastructure changes. It runs `terraform plan -refresh-only` to compare actual cloud state against the Terraform state file, and creates alerts when discrepancies are found.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/terraform/workspaces/{workspace_id}/drift-detect` | Trigger on-demand drift detection |
| GET | `/api/v1/terraform/workspaces/{workspace_id}/drift-alerts` | List drift alerts (optional `?status=` filter) |
| POST | `/api/v1/terraform/drift-alerts/{alert_id}/accept` | Accept drift (sync state to match cloud) |
| POST | `/api/v1/terraform/drift-alerts/{alert_id}/remediate` | Remediate drift (restore desired config state) |

### Authentication

All endpoints require a valid JWT token. Trigger, accept, and remediate require `admin` or `developer` role. Listing alerts is available to any authenticated user.

---

## Response Models

### DriftResult

Returned by `POST .../drift-detect`:

```json
{
  "has_drift": true,
  "alert": { ... },
  "affected_resource_count": 3,
  "resource_additions": 0,
  "resource_changes": 2,
  "resource_destructions": 1
}
```

### DriftAlert

Each alert contains the full details of detected drift:

```json
{
  "alert_id": "uuid",
  "workspace_id": "ws-123",
  "tenant_id": "default",
  "affected_resources": [ ... ],
  "status": "unresolved",
  "detected_at": "2025-01-15T10:30:00+00:00",
  "resolved_at": null,
  "resolution_action": null,
  "resource_additions": 0,
  "resource_changes": 2,
  "resource_destructions": 1
}
```

Alert statuses: `unresolved`, `accepted`, `remediated`, `superseded`

---

## Affected Resources — attribute_diffs Format

Each entry in `affected_resources` describes a single drifted resource:

```json
{
  "resource_address": "aws_security_group.api",
  "change_type": "update",
  "attribute_diffs": {
    "tags.Environment": {
      "cloud_value": "production-manual",
      "config_value": "production",
      "display": "production-manual → production"
    },
    "ingress.0.from_port": {
      "cloud_value": "8080",
      "config_value": "443",
      "display": "8080 → 443"
    }
  }
}
```

### attribute_diffs Structure

Each key in `attribute_diffs` is the Terraform attribute name. The value is an object with:

| Field | Type | Description |
|-------|------|-------------|
| `cloud_value` | string | The actual current value in the cloud (what was detected) |
| `config_value` | string | The desired value from Terraform configuration |
| `display` | string | Human-readable summary: `"{cloud_value} → {config_value}"` |

**Semantics:** In Terraform's `plan -refresh-only` output, the "old" value represents what is actually in the cloud now, and the "new" value represents what Terraform's configuration declares as desired state. The `cloud_value` / `config_value` naming makes this explicit.

### change_type Values

| Value | Meaning |
|-------|---------|
| `update` | Resource exists but attributes differ from config |
| `create` | Resource exists in cloud but not in Terraform state |
| `delete` | Resource missing from cloud but present in state |

---

## Resolution Actions

### Accept Drift

Accepts the current cloud state as the new desired state:

- **CMP-managed backend (S3):** Updates workspace `variable_values` to reflect cloud reality, then runs `terraform apply` (effectively a no-op that syncs state).
- **Terraform Cloud backend:** Runs `terraform apply -refresh-only` on TFC to update remote state.

### Remediate Drift

Restores infrastructure to the Terraform-configured desired state:

- Runs a targeted `terraform apply` scoped only to drifted resources using `-target` flags.
- Prevents unintended side effects on non-drifted resources.
- On failure, the alert remains unresolved.

---

## Scheduling

Configure periodic drift detection per workspace:

```
POST /api/v1/terraform/workspaces/{workspace_id}/drift-schedule
Body: { "interval_hours": 4 }
```

Constraints:
- Minimum interval: 1 hour
- Maximum interval: 168 hours (7 days)
- Default interval: 24 hours

---

## Frontend Integration Notes

The frontend (`TerraformWorkspacePanel.tsx`) handles `attribute_diffs` with backward compatibility:

```typescript
// Supports both structured object and legacy string format
const diffObj = typeof diff === 'object' && diff !== null && 'cloud_value' in diff
  ? diff
  : null;
const cloudVal = diffObj ? diffObj.cloud_value : /* fallback to string parsing */;
```

The TypeScript interface for affected resources:

```typescript
interface DriftAffectedResource {
  resource_address: string
  change_type: string
  attribute_diffs: Record<string, {
    cloud_value: string
    config_value: string
    display: string
  } | string>  // string format for backward compat with older alerts
}
```

---

## DynamoDB Storage

Table: `terraform_drift_alerts`

| Key | Format |
|-----|--------|
| PK | `WORKSPACE#{workspace_id}` |
| SK | `ALERT#{alert_id}` |

Alerts are stored with the full `affected_resources` array serialized. The `attribute_diffs` objects within are stored as DynamoDB maps.

# Scaling Service — Developer Guide

## Overview

The scaling service (`backend/app/services/scaling/scaling_service.py`) provides async operations for managing AWS Auto Scaling Groups. It follows CMP's service layer pattern — no direct endpoint exposure yet, consumed by the chat/operations layer.

---

## Module Location

```
backend/app/
├── models/scaling.py                        # Pydantic models
├── services/scaling/
│   └── scaling_service.py                   # Service implementation
```

---

## Public Functions

### `get_scaling_config(resource_id, credential_data, region)`

Fetches ASG configuration including capacity settings, target tracking policies, and instance health.

```python
result = await get_scaling_config(
    resource_id="my-asg",
    credential_data={"access_key": "...", "secret_key": "..."},
    region="us-east-1",
)
# result["status"] == "success" | "not_found" | "error"
# result["config"] — ScalingPolicyConfig dict
# result["target_tracking_policies"] — list of policy dicts
# result["instances"] — current instance count
# result["health_status"] — list of instance health dicts
```

### `update_scaling_policy(resource_id, updates, credential_data, region)`

Partial update of ASG capacity. Only non-None fields in `ScalingPolicyUpdate` are applied.

```python
from app.models.scaling import ScalingPolicyUpdate

result = await update_scaling_policy(
    resource_id="my-asg",
    updates=ScalingPolicyUpdate(desired_capacity=5, max_capacity=10),
    credential_data=cred_data,
    region="us-east-1",
)
# result["applied"] — dict of actually applied changes
```

### `create_scaling_schedule(data, credential_data)`

Creates a named scheduled scaling action. Schedule name is auto-generated (`cmp-schedule-{uuid8}`).

```python
from app.models.scaling import ScalingScheduleCreate

result = await create_scaling_schedule(
    data=ScalingScheduleCreate(
        resource_id="my-asg",
        cron_expression="0 8 * * MON-FRI",
        desired_capacity=5,
        credential_id="cred-123",
        region="us-east-1",
    ),
    credential_data=cred_data,
)
# result["schedule_name"] — the generated schedule action name
```

### `get_scaling_recommendations(resources, credential_data, region)`

Analyzes CloudWatch CPU metrics over 7 days and returns right-sizing suggestions.

```python
resources = [
    {"resource_id": "web-asg", "name": "Web Tier"},
    {"resource_id": "api-asg", "name": "API Tier"},
]
result = await get_scaling_recommendations(resources, cred_data, "us-east-1")
# result["recommendations"] — list of ScalingRecommendation dicts
# result["analyzed_count"] — total resources submitted
```

---

## Error Handling

All functions return a dict with `"status"` field:

| Status | Meaning |
|--------|---------|
| `success` | Operation completed |
| `not_found` | Resource doesn't exist in the specified region |
| `error` | AWS API or unexpected failure — `"message"` contains details |

Functions catch `botocore.exceptions.ClientError` for AWS-specific errors and general `Exception` for unexpected failures. Errors are logged but never raised — callers should check `result["status"]`.

---

## Adding Azure VMSS Support

To extend the service for Azure:

1. Add a provider detection function based on `resource_type` or explicit `provider` field
2. Implement `_get_azure_client()` using `azure.mgmt.compute.ComputeManagementClient`
3. Map VMSS operations:
   - `get_scaling_config` → `virtual_machine_scale_sets.get()`
   - `update_scaling_policy` → `virtual_machine_scale_sets.update()` with capacity object
   - `create_scaling_schedule` → Azure Autoscale profiles with recurrence
4. Update models if needed (VMSS uses `sku.capacity` instead of min/max/desired)

---

## Adding a Dedicated REST Endpoint

If a standalone scaling API is needed:

1. Create `api/v1/endpoints/scaling.py` with routes:
   - `GET /api/v1/scaling/{resource_id}/config`
   - `PATCH /api/v1/scaling/{resource_id}/policy`
   - `POST /api/v1/scaling/schedules`
   - `GET /api/v1/scaling/recommendations`

2. Register in `api/v1/api.py`:
   ```python
   from app.api.v1.endpoints import scaling
   api_router.include_router(scaling.router, prefix="/scaling", tags=["scaling"])
   ```

3. Add credential resolution in the endpoint layer (resolve `credential_id` → decrypted data)

4. Protect with `Depends(require_role(["admin", "developer"]))`

---

## Testing

When writing tests for the scaling service:

- Mock `boto3.client` to avoid real AWS calls
- Test each status path: `success`, `not_found`, `error`
- Verify partial updates only send non-None fields
- Test recommendation thresholds (< 20% CPU → scale down, > 75% → scale up)
- Verify `run_in_executor` wrapping (ensure async behavior)

```python
# Example test structure
@pytest.mark.asyncio
async def test_get_scaling_config_not_found(mock_boto_client):
    mock_boto_client.describe_auto_scaling_groups.return_value = {
        "AutoScalingGroups": []
    }
    result = await get_scaling_config("missing-asg", cred_data, "us-east-1")
    assert result["status"] == "not_found"
```

---

## Key Design Decisions

1. **No DynamoDB storage** — scaling configs live in AWS; CMP reads them on demand rather than caching (avoids stale state)
2. **Credential passthrough** — callers resolve and decrypt credentials before invoking the service; the service never accesses the credential store directly
3. **Limit of 5 resources** for recommendations — prevents long-running CloudWatch queries from timing out
4. **Instance health capped at 10** — avoids oversized responses for large ASGs

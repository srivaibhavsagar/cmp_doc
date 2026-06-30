# Infrastructure Plan Generator — Developer Guide

Technical reference for the `infra_planner` service module (Feature 7: Infrastructure-as-Conversation).

---

## Architecture

```
backend/app/
├── models/
│   └── infra_plan.py                  # Pydantic models
├── services/
│   └── infra_planner/
│       └── plan_generator.py          # Core plan engine
```

The plan generator is a service-layer module. It does not have its own REST endpoint — it is invoked via AI chat tools (function calling) or can be called directly from other services.

---

## Module: `plan_generator.py`

### Public Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `generate_plan` | `(description, constraints, preferences, tenant_id, user_id) → Dict` | Create a plan from natural language |
| `modify_plan` | `(plan_id, modifications, tenant_id) → Dict` | Modify an existing plan |
| `estimate_plan_cost` | `(plan_id, tenant_id) → Dict` | Get detailed cost breakdown |
| `export_terraform` | `(plan_id, tenant_id) → Dict` | Generate Terraform HCL |

### `generate_plan`

```python
async def generate_plan(
    description: str,
    constraints: Optional[InfraConstraints],
    preferences: Dict[str, Any],
    tenant_id: str,
    user_id: str,
) -> Dict[str, Any]:
```

**Behavior:**
1. Parses description for resource intent (keywords: server, database, storage, lb, etc.)
2. Selects appropriate instance types based on provider + description keywords
3. Extracts resource count from patterns like "3 servers"
4. Automatically adds VPC/networking when compute resources are present
5. Falls back to a basic compute setup if no resources matched
6. Checks budget constraints and adds warnings
7. Builds dependency graph (compute depends on network, LB depends on compute)
8. Persists plan to DynamoDB
9. Returns structured result with resources, costs, and metadata

**Returns:**
```python
{
    "status": "success",
    "plan_id": "uuid",
    "description": "...",
    "resources": [...],
    "dependencies": {...},
    "estimated_monthly_cost": 186.34,
    "provider": "aws",
    "region": "us-east-1",
    "message": "Infrastructure plan generated with 5 resources...",
    "budget_warning": "⚠️ ..."  # Only if over budget
}
```

### `modify_plan`

```python
async def modify_plan(
    plan_id: str,
    modifications: str,
    tenant_id: str,
) -> Dict[str, Any]:
```

**Supported modifications:**
- `"remove <resource_type>"` — removes all resources of that type
- `"add database"` / `"add server"` — appends resources
- `"bigger"` / `"larger"` / `"scale up"` — upgrades compute to m5.large

### `export_terraform`

Generates HCL for these AWS resource types:
- `aws_instance` — compute resources
- `aws_db_instance` — databases (PostgreSQL, with multi-AZ support)
- `aws_s3_bucket` — storage (with random suffix for uniqueness)
- `aws_vpc` — networking
- `aws_lb` — load balancers
- `random_id` — for unique resource naming

---

## Data Models (`models/infra_plan.py`)

### `InfraPlanCreate` (Request)

```python
class InfraPlanCreate(BaseModel):
    description: str                         # Natural language description
    constraints: Optional[InfraConstraints]  # Budget, region, HA, etc.
    preferences: Dict[str, Any] = {}         # Additional hints
```

### `InfraConstraints`

```python
class InfraConstraints(BaseModel):
    budget_monthly: Optional[float] = None
    region: Optional[str] = None
    high_availability: bool = False
    encryption_required: bool = True
    provider_preference: Optional[str] = None  # aws, azure, gcp
    compliance: List[str] = []
```

### `PlannedResource`

```python
class PlannedResource(BaseModel):
    resource_id: str
    name: str
    provider: str
    resource_type: str        # compute, storage, database, network
    specific_type: str        # t3.medium, db.r5.large, etc.
    region: str = "us-east-1"
    properties: Dict[str, Any] = {}
    estimated_monthly_cost: float = 0.0
    depends_on: List[str] = []
```

### `InfraPlanStatus` (Enum)

```python
DRAFT → REVIEWED → APPROVED → EXPORTED → APPLIED
                                    ↘ CANCELLED
```

---

## DynamoDB Storage

| Key | Format |
|-----|--------|
| PK | `TENANT#{tenant_id}` |
| SK | `PLAN#{plan_id}` |
| Table | `infra_plans` |

The table must be created at startup (add to `main.py` lifecycle if not present).

Writes use `_sanitize_for_dynamodb()` and `run_in_executor` per project conventions. Failures are logged but do not raise — plan generation still returns results even if persistence fails.

---

## Resource Selection Logic

### Instance Type Picker (`_pick_instance_type`)

| Provider | Keywords | Selected Type |
|----------|----------|---------------|
| AWS | "large", "production" | m5.xlarge |
| AWS | "medium" | t3.large |
| AWS | "small", "dev", "test" | t3.small |
| AWS | (default) | t3.medium |
| Azure | "large", "production" | Standard_D4s_v3 |
| Azure | "medium" | Standard_D2s_v3 |
| Azure | (default) | Standard_B2s |
| GCP | "large", "production" | n2-standard-4 |
| GCP | "medium" | n2-standard-2 |
| GCP | (default) | e2-medium |

### Count Extraction (`_extract_count`)

- Regex pattern: `(\d+)\s*(server|instance|vm|node|machine)`
- "high availability" / "ha" → defaults to 3
- Maximum capped at 10
- Default: 1

---

## Extending the Plan Generator

### Adding a new resource type

1. Add keyword detection in `generate_plan()`:
   ```python
   if any(kw in desc_lower for kw in ["cache", "redis", "elasticache"]):
       resources.append(PlannedResource(...))
   ```

2. Add cost entries to `_COST_ESTIMATES`:
   ```python
   "cache.t3.medium": 46.08,
   ```

3. Add Terraform generation in `export_terraform()`:
   ```python
   elif r_type == "cache" and provider == "aws":
       hcl_lines.extend([...])
   ```

### Adding a new provider

1. Add instance types to `_pick_instance_type()`
2. Add cost estimates to `_COST_ESTIMATES`
3. Add provider-specific Terraform blocks in `export_terraform()`

### Registering as AI Chat Tool

To expose via the AI chat function-calling interface, register tools in the AI service:

```python
# In the AI tools registration
{
    "name": "plan_infrastructure",
    "description": "Generate an infrastructure plan from natural language",
    "parameters": {
        "description": {"type": "string"},
        "provider": {"type": "string", "enum": ["aws", "azure", "gcp"]},
        "region": {"type": "string"},
        "budget_monthly": {"type": "number"},
        "high_availability": {"type": "boolean"}
    }
}
```

---

## Testing

### Unit Test Approach

```python
import pytest
from app.services.infra_planner.plan_generator import generate_plan, _pick_instance_type, _extract_count
from app.models.infra_plan import InfraConstraints

@pytest.mark.asyncio
async def test_generate_plan_basic():
    result = await generate_plan(
        description="I need a web server with a database",
        constraints=None,
        preferences={},
        tenant_id="test-tenant",
        user_id="test-user",
    )
    assert result["status"] == "success"
    assert len(result["resources"]) >= 2

def test_pick_instance_type_aws_production():
    assert _pick_instance_type("aws", "large production server", None) == "m5.xlarge"

def test_extract_count_explicit():
    assert _extract_count("i need 3 servers") == 3

def test_extract_count_ha():
    assert _extract_count("high availability setup") == 3
```

### Integration Points

- DynamoDB table `infra_plans` must exist (or mock `get_table`)
- No external API calls — all cost data is local reference
- No AI model dependency — uses deterministic pattern matching

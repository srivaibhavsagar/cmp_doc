# CMP Environment Promoter — Developer Architecture Guide

## Overview

The CMP Environment Promoter is a standalone application that enables platform administrators to promote (export and import) CMP configuration artifacts between environments (e.g., dev → test → prod). It communicates exclusively through the CMP public REST API, maintaining complete independence from the CMP platform codebase.

This guide covers the internal architecture, service interfaces, data design, and extension points for developers working on the Promoter.

---

## 1. Project Structure

```
cmp_promoter/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                        # FastAPI app init, CORS, DynamoDB table creation
│   │   ├── api/
│   │   │   ├── deps.py                    # Auth dependencies (get_current_user, require_role)
│   │   │   └── v1/
│   │   │       ├── api.py                 # Central router registration
│   │   │       └── endpoints/
│   │   │           ├── connections.py     # Environment connection CRUD
│   │   │           ├── artifacts.py       # Artifact discovery & search
│   │   │           ├── promotions.py      # Export, plan, execute
│   │   │           ├── sessions.py        # Session history & audit
│   │   │           └── rollbacks.py       # Rollback operations
│   │   ├── core/
│   │   │   ├── config.py                  # Settings (env vars, encryption key, AWS region)
│   │   │   ├── security.py               # Fernet encrypt/decrypt, JWT validation
│   │   │   └── dynamodb.py               # _sanitize_for_dynamodb(), async wrappers
│   │   ├── crud/
│   │   │   ├── connection.py             # Environment_Connection CRUD
│   │   │   ├── session.py               # Promotion_Session CRUD
│   │   │   ├── id_mapping.py            # ID_Mapping CRUD
│   │   │   └── snapshot.py              # Rollback snapshot CRUD
│   │   ├── models/
│   │   │   ├── connection.py            # Connection Pydantic models
│   │   │   ├── artifact.py             # Artifact, bundle, manifest models
│   │   │   ├── promotion.py            # Plan, session, outcome models
│   │   │   ├── dependency.py           # Dependency graph models
│   │   │   ├── rollback.py             # Rollback models
│   │   │   └── error.py                # ErrorResponse model
│   │   └── services/
│   │       ├── cmp_client.py            # HTTP client for CMP REST API
│   │       ├── dependency_resolver.py   # Dependency graph computation (BFS)
│   │       ├── bundle_builder.py        # Artifact export & manifest creation
│   │       ├── plan_generator.py        # Promotion plan & diff engine
│   │       ├── promotion_executor.py    # Import with ID remapping
│   │       ├── rollback_executor.py     # Rollback operations
│   │       └── auth_validator.py        # Role & tenant access validation
│   ├── tests/                           # Property-based & integration tests
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── pytest.ini
├── frontend/
│   ├── src/
│   │   ├── App.tsx                      # Root routing, navigation layout
│   │   ├── config.ts                    # getApiUrl() runtime URL resolution
│   │   ├── main.tsx                     # React entry point
│   │   ├── context/
│   │   │   ├── AuthContext.tsx          # JWT token management
│   │   │   └── PromotionContext.tsx     # Promotion workflow state
│   │   ├── pages/
│   │   │   ├── Login.tsx                # Authentication page
│   │   │   ├── Connections.tsx          # Connection management
│   │   │   ├── ArtifactSelection.tsx    # Browse & select artifacts
│   │   │   ├── PromotionPlan.tsx        # Plan review & execution
│   │   │   ├── PromotionHistory.tsx     # Session audit trail
│   │   │   └── RollbackView.tsx         # Rollback operations
│   │   └── components/
│   │       ├── AdvancedDataTable.tsx     # Reusable data table
│   │       ├── ConflictResolver.tsx     # Conflict resolution UI
│   │       ├── DependencyGraph.tsx      # Visual dependency display
│   │       ├── DiffViewer.tsx           # Field-level diff display
│   │       ├── Pagination.tsx           # Pagination controls
│   │       ├── ProgressIndicator.tsx    # Bulk promotion progress
│   │       └── SkeletonLoader.tsx       # Loading states
│   ├── package.json
│   ├── vite.config.ts
│   ├── tailwind.config.js
│   └── tsconfig.json
├── VERSION                              # Independent semantic versioning
└── README.md
```

---

## 2. Three-Layer Architecture

The backend follows the same three-layer separation as the CMP platform:

```
┌─────────────────────────────────────────────────────┐
│  Endpoints (api/v1/endpoints/)                      │
│  FastAPI routers — call CRUD/services, return JSON  │
├─────────────────────────────────────────────────────┤
│  Services (services/)                               │
│  Business logic, orchestration, external HTTP calls │
├─────────────────────────────────────────────────────┤
│  CRUD (crud/)                                       │
│  Async DynamoDB operations only — zero logic        │
├─────────────────────────────────────────────────────┤
│  Models (models/)                                   │
│  Pydantic schemas for validation & serialization    │
└─────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Location | Responsibility |
|-------|----------|----------------|
| **Models** | `models/` | Pydantic schemas: `XCreate`, `XUpdate`, `XInDB`, enums. Request/response validation. |
| **CRUD** | `crud/` | Async DynamoDB operations only. No business logic. All calls wrapped with `run_in_executor`. |
| **Services** | `services/` | Business logic, dependency resolution, HTTP communication with CMP environments. |
| **Endpoints** | `api/v1/endpoints/` | FastAPI routers. Thin layer that calls services/CRUD and returns responses. |

### Rules

- Business logic belongs in `services/`. Never in endpoints or CRUD.
- All CRUD functions are `async def` and wrap boto3 calls with `await loop.run_in_executor(None, fn)`.
- Always call `_sanitize_for_dynamodb()` before writes (converts `float` → `Decimal`, strips `None`).
- Always use `ExpressionAttributeNames` for DynamoDB reserved words (`name`, `status`, `type`, `data`).

---

## 3. Service Interfaces

### CMPClient (`services/cmp_client.py`)

HTTP client for communicating with CMP environments via their public REST API.

```python
class CMPClient:
    def __init__(self, base_url: str, api_token: str, tenant_id: str): ...
    async def health_check(self, timeout: int = 10) -> bool: ...
    async def authenticate(self, username: str, password: str) -> dict: ...
    async def get_artifacts(self, artifact_type: str, page: int = 1, page_size: int = 20) -> dict: ...
    async def get_artifact_by_id(self, artifact_type: str, artifact_id: str) -> dict: ...
    async def create_artifact(self, artifact_type: str, payload: dict) -> dict: ...
    async def update_artifact(self, artifact_type: str, artifact_id: str, payload: dict) -> dict: ...
    async def delete_artifact(self, artifact_type: str, artifact_id: str) -> bool: ...
    async def refresh_token(self) -> bool: ...
```

- Uses `httpx.AsyncClient` for all HTTP communication
- Token refresh: 3 attempts with 5-second intervals
- Health check timeout: 10 seconds
- All requests include `Authorization: Bearer <token>` header

### DependencyResolver (`services/dependency_resolver.py`)

Computes the full dependency graph using BFS traversal.

```python
class DependencyResolver:
    MAX_DEPTH = 10

    async def resolve(self, selected_artifacts: list[Artifact], cmp_client: CMPClient) -> DependencyGraph: ...
    def extract_workflow_deps(self, workflow: dict) -> list[DependencyRef]: ...
    def extract_flow_deps(self, flow: dict) -> list[DependencyRef]: ...
    def extract_catalog_deps(self, catalog_item: dict) -> list[DependencyRef]: ...
    def extract_policy_deps(self, policy: dict) -> list[DependencyRef]: ...
    def classify_dependency(self, ref: DependencyRef) -> str: ...
```

### BundleBuilder (`services/bundle_builder.py`)

Creates portable artifact bundles for export.

```python
class BundleBuilder:
    MAX_ARTIFACTS = 500
    MAX_BUNDLE_SIZE_MB = 50
    SENSITIVE_FIELDS = ["secret_key", "access_key", "token", "password", "client_secret"]

    async def build(self, artifacts: list[dict], dependency_graph: DependencyGraph) -> ArtifactBundle: ...
    def generate_manifest(self, artifacts: list[dict], graph: DependencyGraph) -> ArtifactManifest: ...
    def sanitize_credentials(self, artifact: dict) -> dict: ...
    def compute_summary(self, bundle: ArtifactBundle) -> ExportSummary: ...
```

### PlanGenerator (`services/plan_generator.py`)

Generates promotion plans by comparing source bundle against target state.

```python
class PlanGenerator:
    async def generate(self, bundle: ArtifactBundle, target_client: CMPClient) -> PromotionPlan: ...
    def categorize_artifact(self, source: dict, target: dict | None, target_by_name: dict | None) -> ArtifactAction: ...
    def compute_diff(self, source: dict, target: dict) -> list[FieldDiff]: ...
    def detect_conflict(self, source: dict, target_by_id: dict | None, target_by_name: dict | None) -> Conflict | None: ...
```

### PromotionExecutor (`services/promotion_executor.py`)

Executes confirmed plans with ID remapping and dependency ordering.

```python
class PromotionExecutor:
    IMPORT_ORDER = ["task", "workflow", "flow", "catalog_item", "catalog_component",
                    "credential", "policy", "budget", "dashboard", "scheduled_job",
                    "cost_model", "resource_action"]

    async def execute(self, plan: PromotionPlan, target_client: CMPClient, session_id: str) -> PromotionResult: ...
    def topological_sort(self, artifacts: list[dict], graph: DependencyGraph) -> list[dict]: ...
    def remap_references(self, artifact_data: dict, id_mapping: dict[str, str]) -> dict: ...
    def sanitize_credential_for_import(self, credential: dict) -> dict: ...
```

### RollbackExecutor (`services/rollback_executor.py`)

Handles rollback operations for completed promotion sessions.

```python
class RollbackExecutor:
    ROLLBACK_WINDOW_DAYS = 30

    async def generate_plan(self, session_id: str) -> RollbackPlan: ...
    async def execute(self, plan: RollbackPlan, target_client: CMPClient) -> RollbackResult: ...
    def is_within_window(self, session_created_at: str) -> bool: ...
```

### AuthValidator (`services/auth_validator.py`)

Validates role-based access and tenant authorization.

```python
class AuthValidator:
    async def validate_promotion_access(self, source_client: CMPClient, target_client: CMPClient, user: dict) -> bool: ...
```

- Validates "admin" role in both source and target environments
- Confirms tenant_id exists in user's authorized tenant list
- Validation order: authenticate source → authenticate target → verify roles → verify tenant access

---

## 4. DynamoDB Single-Table Design

The Promoter uses a single DynamoDB table (`cmp_promoter`) with tenant-scoped partition keys.

### Key Schema

| Entity | PK | SK | Description |
|--------|----|----|-------------|
| Connection | `TENANT#{tenant_id}` | `CONN#{connection_id}` | Environment connection config |
| Session | `TENANT#{tenant_id}` | `SESSION#{session_id}` | Promotion session record |
| SessionArtifact | `TENANT#{tenant_id}` | `SESSION#{session_id}#ART#{artifact_id}` | Per-artifact outcome |
| IDMapping | `TENANT#{tenant_id}` | `IDMAP#{session_id}#SRC#{source_id}` | Source→target ID mapping |
| Snapshot | `TENANT#{tenant_id}` | `SNAP#{session_id}#ART#{artifact_id}` | Pre-promotion artifact state |

### Access Patterns

| Pattern | Key Condition | Use Case |
|---------|--------------|----------|
| List connections | `PK = TENANT#{tid}` AND `begins_with(SK, "CONN#")` | Show user's connections |
| Get connection | `PK = TENANT#{tid}` AND `SK = CONN#{cid}` | Connection detail |
| List sessions | `PK = TENANT#{tid}` AND `begins_with(SK, "SESSION#")` | Session history |
| Get session artifacts | `PK = TENANT#{tid}` AND `begins_with(SK, "SESSION#{sid}#ART#")` | Session detail |
| Get ID mappings | `PK = TENANT#{tid}` AND `begins_with(SK, "IDMAP#{sid}#SRC#")` | Remapping lookup |
| Get snapshots | `PK = TENANT#{tid}` AND `begins_with(SK, "SNAP#{sid}#ART#")` | Rollback data |

### Design Principles

- Every query is scoped to `tenant_id` in the partition key — no cross-tenant access
- All writes go through `_sanitize_for_dynamodb()` (converts floats to Decimal, strips None values)
- All boto3 calls are wrapped with `await loop.run_in_executor(None, fn)` for async compatibility
- Session records retained for minimum 90 days; snapshots expire after 30 days (TTL)
- Write failures retry up to 3 times with exponential backoff

---

## 5. CMP API Communication

The Promoter communicates with CMP environments **exclusively through the CMP public REST API**. There is no direct database access, no shared code imports, and no build-time dependencies on CMP source files.

### HTTP Client Stack

- **Library**: `httpx` (async HTTP client)
- **Auth**: JWT Bearer tokens obtained from `POST /api/v1/auth/login`
- **Content-Type**: Always `application/json` (never form-urlencoded)

### CMP API Endpoints Used

| Operation | Method | CMP Endpoint | Purpose |
|-----------|--------|--------------|---------|
| Health check | GET | `/health` | Validate connectivity (10s timeout) |
| Authenticate | POST | `/api/v1/auth/login` | Obtain JWT token |
| List artifacts | GET | `/api/v1/{artifact_type}` | Paginated artifact retrieval |
| Get artifact | GET | `/api/v1/{artifact_type}/{id}` | Single artifact detail |
| Create artifact | POST | `/api/v1/{artifact_type}` | Import artifact to target |
| Update artifact | PUT | `/api/v1/{artifact_type}/{id}` | Update existing artifact |
| Delete artifact | DELETE | `/api/v1/{artifact_type}/{id}` | Rollback: remove created artifact |

### Artifact Type → API Endpoint Mapping

| Artifact Type | CMP API Path Segment |
|---------------|---------------------|
| `task` | `tasks` |
| `workflow` | `workflows` |
| `catalog_item` | `catalog-items` |
| `catalog_component` | `catalog-components` |
| `flow` | `flows` |
| `policy` | `policies` |
| `credential` | `credentials` |
| `budget` | `budgets` |
| `dashboard` | `dashboards` |
| `scheduled_job` | `scheduled-jobs` |
| `cost_model` | `cost-models` |
| `resource_action` | `resource-actions` |

### Token Refresh Strategy

When a token expires during a promotion operation:
1. Attempt refresh up to 3 times
2. Wait 5 seconds between attempts
3. If all retries fail: halt operation, preserve partial state, report which environment failed

---

## 6. Dependency Resolution Algorithm

The dependency resolver uses **Breadth-First Search (BFS)** to traverse artifact references.

### Algorithm

```
1. Initialize BFS queue with user-selected artifacts at depth 0
2. While queue is not empty:
   a. Dequeue (artifact_type, artifact_id, artifact_data, depth)
   b. If depth >= MAX_DEPTH (10): skip further resolution for this node
   c. Extract dependency references based on artifact type
   d. For each dependency reference:
      - Add edge to dependency graph
      - If already visited: skip
      - Fetch artifact from source CMP environment
      - If not found: mark as UNRESOLVED, add warning
      - If found: add node at depth+1, enqueue for further resolution
3. Return complete DependencyGraph
```

### Extraction Rules

| Artifact Type | Reference Field | Target Type | Classification |
|---------------|----------------|-------------|----------------|
| Workflow | `steps[].task_id` (where action="run_task") | Task | Required |
| Workflow | `steps[].inputs.workflow_id` (where action="builtin.subworkflow") | Workflow | Required |
| Flow | `workflows[].workflow_id` | Workflow | Required |
| Catalog Item | `flow_id` | Flow | Required |
| Catalog Item | `on_approve_flow_id` | Flow | Required |
| Catalog Item | `on_reject_flow_id` | Flow | Required |
| Catalog Item | `on_cancel_flow_id` | Flow | Required |
| Catalog Item | `form_schema.fields[].reusable_component_id` | Catalog Component | Required |
| Policy | `catalog_ids[]` | Catalog Item | Required |
| Policy | `catalog_tags[]` | Catalog Item (by tag) | Optional |

### Classification

- **Required**: ID-based direct reference. Deselecting breaks the parent artifact.
- **Optional**: Tag-based or pattern-based matching. Can be safely deselected.

### Constraints

- `MAX_DEPTH = 10` — traversal stops at 10 levels regardless of remaining references
- Cycle detection via visited set — each artifact processed at most once
- Unresolved dependencies are tracked and surfaced as warnings

---

## 7. ID Remapping Logic

When artifacts are imported into a target environment, they receive new IDs. All internal references must be rewritten to point to the new target IDs.

### Remapping Fields by Artifact Type

| Artifact Type | Fields Requiring Remapping |
|---------------|---------------------------|
| Workflow | `steps[].task_id`, `steps[].inputs.workflow_id` |
| Flow | `workflows[].workflow_id` |
| Catalog Item | `flow_id`, `on_approve_flow_id`, `on_reject_flow_id`, `on_cancel_flow_id`, `form_schema.fields[].reusable_component_id` |
| Policy | `catalog_ids[]` |

### Remapping Process

1. **Topological sort**: Artifacts are sorted by dependency order so dependencies are created first
2. **Import in order**: Tasks → Workflows → Flows → Catalog Items → Catalog Components → Credentials → Policies → Budgets → Dashboards → Scheduled Jobs → Cost Models → Resource Actions
3. **Build ID mapping**: As each artifact is created in the target, store `source_id → target_id` in the mapping
4. **Remap before write**: Before creating/updating an artifact, replace all source IDs in reference fields with corresponding target IDs from the mapping
5. **Non-reference fields untouched**: Only the specific reference fields listed above are modified

### Implementation Detail

The `remap_references()` method performs a deep copy of the artifact data, then walks through each known reference field pattern:

```python
# Workflow steps
for step in data["steps"]:
    if step["task_id"] in id_mapping:
        step["task_id"] = id_mapping[step["task_id"]]
    if step.get("inputs", {}).get("workflow_id") in id_mapping:
        step["inputs"]["workflow_id"] = id_mapping[step["inputs"]["workflow_id"]]

# Flow workflows
for wf_entry in data["workflows"]:
    if wf_entry["workflow_id"] in id_mapping:
        wf_entry["workflow_id"] = id_mapping[wf_entry["workflow_id"]]

# Catalog item flow references
for field in ["flow_id", "on_approve_flow_id", "on_reject_flow_id", "on_cancel_flow_id"]:
    if data.get(field) in id_mapping:
        data[field] = id_mapping[data[field]]

# Policy catalog_ids
data["catalog_ids"] = [id_mapping.get(cid, cid) for cid in data["catalog_ids"]]
```

### Failure Isolation

- If an artifact fails to import, all artifacts that transitively depend on it are skipped
- Independent artifacts continue regardless of failures in unrelated branches
- The `_should_skip()` method checks if any ancestor in the dependency chain has failed

---

## 8. Adding Support for New Artifact Types

To add a new artifact type to the Promoter:

### Step 1: Define the Model

Add the new type to `ArtifactType` enum in `models/artifact.py`:

```python
class ArtifactType(str, Enum):
    # ... existing types ...
    MY_NEW_TYPE = "my_new_type"
```

### Step 2: Add Extraction Rules (if it has dependencies)

In `services/dependency_resolver.py`, add an extraction method:

```python
def extract_my_new_type_deps(self, artifact: dict) -> list[DependencyRef]:
    refs = []
    # Extract references from the artifact's data structure
    if "some_reference_id" in artifact:
        refs.append(DependencyRef(
            artifact_type="referenced_type",
            artifact_id=artifact["some_reference_id"],
            reference_field="some_reference_id",
            classification=DependencyClassification.REQUIRED,
        ))
    return refs
```

Register it in `_extract_deps_for_artifact()`:

```python
def _extract_deps_for_artifact(self, artifact_type: str, data: dict) -> list[DependencyRef]:
    if artifact_type == "my_new_type":
        return self.extract_my_new_type_deps(data)
    # ... existing types ...
```

### Step 3: Add Remapping Fields (if it references other artifacts)

In `services/promotion_executor.py`, update `remap_references()`:

```python
# Remap my_new_type references
if "some_reference_id" in data and data["some_reference_id"] in id_mapping:
    data["some_reference_id"] = id_mapping[data["some_reference_id"]]
```

### Step 4: Add API Endpoint Mapping

In `PromotionExecutor.ARTIFACT_TYPE_ENDPOINTS`:

```python
ARTIFACT_TYPE_ENDPOINTS = {
    # ... existing mappings ...
    "my_new_type": "my-new-types",  # CMP API path segment
}
```

### Step 5: Add to Import Order

In `PromotionExecutor.IMPORT_ORDER`, insert the new type at the correct position based on its dependencies:

```python
IMPORT_ORDER = [
    "task",
    "workflow",
    "flow",
    "catalog_item",
    # Insert here if it depends on catalog_items but is depended on by policies
    "my_new_type",
    "policy",
    # ...
]
```

### Step 6: Update Tests

- Add the new type to property test strategies (Hypothesis generators)
- Add extraction test cases in `test_property_dependency_extraction.py`
- Add remapping test cases in `test_property_id_remapping.py`

---

## 9. Testing Strategy

### Test Framework

- **pytest** — test runner
- **Hypothesis** — property-based testing library
- **httpx** / **unittest.mock** — HTTP mocking for CMP API calls
- **moto** (optional) — DynamoDB mocking for CRUD tests

### Test File Organization

All tests live in `cmp_promoter/backend/tests/`:

```
tests/
├── conftest.py                          # Shared fixtures (mock DynamoDB, mock HTTP)
├── test_property_encryption.py          # Property 2: Fernet round-trip
├── test_property_connection_validation.py  # Property 1: Input validation
├── test_property_connection_crud.py     # Property 3: Name uniqueness, delete protection
├── test_property_artifact_search.py     # Property 4: Search filtering
├── test_property_dependency_extraction.py  # Property 5: Extraction completeness
├── test_property_dependency_depth.py    # Property 6: Recursive resolution + depth limit
├── test_property_dependency_deselection.py # Property 7: Deselection warnings
├── test_property_bundle_limit.py        # Property 8: 500 artifact limit
├── test_property_manifest.py            # Property 9: Manifest completeness
├── test_property_sensitive_fields.py    # Property 10: Sensitive field exclusion
├── test_property_plan_categorization.py # Property 11: Create/update/unchanged
├── test_property_diff_accuracy.py       # Property 12: Field-level diff
├── test_property_conflict_detection.py  # Property 13: ID/name conflicts
├── test_property_id_remapping.py        # Property 14: ID remapping correctness
├── test_property_import_order.py        # Property 15: Dependency-order import
├── test_property_failure_isolation.py   # Property 16: Failure isolation
├── test_property_session_status.py      # Property 17: Status derivation
├── test_property_rollback_window.py     # Property 18: 30-day window
├── test_property_rollback_plan.py       # Property 19: Rollback plan correctness
├── test_property_role_access.py         # Property 20: Role-based access
├── test_property_tenant_access.py       # Property 21: Tenant access validation
├── test_auth_validator.py               # Integration: auth flow
├── test_dependency_resolver.py          # Integration: dependency resolution
├── test_export_import_endpoints.py      # Integration: export/import
├── test_plan_endpoints.py               # Integration: plan generation
├── test_execute_endpoint.py             # Integration: promotion execution
├── test_bulk_promotion_endpoint.py      # Integration: bulk promotion
└── test_sessions_endpoints.py           # Integration: session history
```

### Property-Based Testing Approach

Each property test validates a correctness property from the design document using Hypothesis:

```python
from hypothesis import given, strategies as st

@given(token=st.text(min_size=1, max_size=500, alphabet=st.characters(codec="utf-8")))
def test_encryption_round_trip(token):
    """Property 2: Encrypting then decrypting produces the original value."""
    encrypted = encrypt_token(token)
    decrypted = decrypt_token(encrypted)
    assert decrypted == token
```

Key principles:
- **Smart generators**: Constrain inputs to the valid domain (e.g., valid artifact structures with references)
- **No mocking of core logic**: Property tests validate pure functions directly
- **HTTP/DynamoDB mocked**: External dependencies are mocked at the boundary
- **Requirement traceability**: Each test file documents which requirement it validates

### Running Tests

```bash
cd cmp_promoter/backend

# Run all tests
pytest

# Run only property tests
pytest tests/test_property_*.py

# Run a specific property test
pytest tests/test_property_id_remapping.py -v

# Run with Hypothesis verbose output
pytest tests/test_property_dependency_extraction.py --hypothesis-show-statistics
```

---

## 10. Frontend Architecture

### Tech Stack

| Technology | Purpose |
|-----------|---------|
| React 18 | UI framework (functional components, hooks only) |
| TypeScript | Type safety (strict mode) |
| Vite | Build tool and dev server |
| Tailwind CSS | Utility-first styling |
| React Router v6 | Client-side routing |
| Axios | HTTP client |
| Lucide React | Icon library |

### State Management

The frontend uses React Context exclusively (no Redux/Zustand):

- **AuthContext** (`context/AuthContext.tsx`): JWT token storage, login/logout, user info
- **PromotionContext** (`context/PromotionContext.tsx`): Cross-page promotion workflow state (selected artifacts, current plan, active session)

### API Communication

```typescript
// config.ts
export function getApiUrl(): string {
  // Runtime resolution — never hardcode backend URLs
}
```

All API calls:
- Use `getApiUrl()` for base URL
- Attach `Authorization: Bearer <token>` from AuthContext
- Use JSON content type exclusively
- Handle errors with user-facing messages

### Page Components

| Page | Route | Purpose |
|------|-------|---------|
| Login | `/login` | Authentication |
| Connections | `/connections` | Manage environment connections |
| ArtifactSelection | `/artifacts` | Browse and select artifacts |
| PromotionPlan | `/plan` | Review plan, resolve conflicts, execute |
| PromotionHistory | `/history` | View past sessions and outcomes |
| RollbackView | `/rollback/:sessionId` | Rollback a completed promotion |

### Shared Components

| Component | Purpose |
|-----------|---------|
| `AdvancedDataTable` | Paginated, sortable data tables |
| `DependencyGraph` | Visual dependency tree display |
| `DiffViewer` | Side-by-side field-level diff |
| `ConflictResolver` | Conflict resolution options (overwrite/skip/rename) |
| `ProgressIndicator` | Bulk promotion progress with artifact type tracking |
| `Pagination` | Page navigation controls |
| `SkeletonLoader` | Loading state placeholders |

### Conventions

- PascalCase filenames for components
- Functional components with hooks only
- Props interfaces defined above the component
- `bg-brand-*` / `text-brand-*` Tailwind tokens for theming
- Auth-gated routing (redirect to login if unauthenticated)
- Feature-gated navigation items

---

## Quick Reference: Promoter API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/connections` | Create environment connection |
| GET | `/api/v1/connections` | List user's connections |
| GET | `/api/v1/connections/{id}` | Get connection details |
| PUT | `/api/v1/connections/{id}` | Update connection |
| DELETE | `/api/v1/connections/{id}` | Delete connection |
| POST | `/api/v1/connections/{id}/test` | Test connection health |
| GET | `/api/v1/artifacts/{connection_id}/types` | List artifact types |
| GET | `/api/v1/artifacts/{connection_id}/{type}` | List artifacts of type |
| GET | `/api/v1/artifacts/{connection_id}/{type}/{id}` | Get artifact detail |
| POST | `/api/v1/artifacts/search` | Search artifacts |
| POST | `/api/v1/promotions/resolve-dependencies` | Compute dependency graph |
| POST | `/api/v1/promotions/export` | Export bundle |
| POST | `/api/v1/promotions/import` | Import bundle (upload) |
| POST | `/api/v1/promotions/plan` | Generate promotion plan |
| POST | `/api/v1/promotions/execute` | Execute promotion plan |
| GET | `/api/v1/sessions` | List promotion sessions |
| GET | `/api/v1/sessions/{id}` | Get session details |
| POST | `/api/v1/rollbacks/{session_id}/plan` | Generate rollback plan |
| POST | `/api/v1/rollbacks/{session_id}/execute` | Execute rollback |

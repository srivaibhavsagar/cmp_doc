# Smart Credential Selection — Developer Guide

This document covers the architecture, implementation details, and extension points for the Smart Credential Selection feature in CMP.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Data Models](#data-models)
4. [Service Layer](#service-layer)
5. [API Endpoints](#api-endpoints)
6. [Frontend Integration](#frontend-integration)
7. [Template Syntax](#template-syntax)
8. [Adding New Match Strategies](#adding-new-match-strategies)
9. [DynamoDB Schema](#dynamodb-schema)
10. [Testing](#testing)
11. [Configuration Examples](#configuration-examples)

---

## Overview

Smart Credential Selection adds intelligent credential filtering and suggestion to the catalog provisioning workflow. Instead of showing all accessible credentials to users, the system:

1. **Filters by cloud provider** — Only shows credentials matching the catalog's target cloud (AWS/Azure/GCP)
2. **Evaluates selection rules** — Matches credentials against admin-configured rules based on user form inputs
3. **Auto-selects the best match** — When exactly one credential matches all rules, it's pre-selected in the dropdown

This works for both Day 1 (provisioning) and Day 2 (operational) catalog types.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend                                 │
│  ┌──────────────┐    ┌───────────────────┐    ┌──────────────┐ │
│  │ CatalogCreate│    │   CatalogView     │    │CredentialDrop│ │
│  │ (Admin UI)   │    │ (Provisioning)    │    │   down       │ │
│  └──────┬───────┘    └────────┬──────────┘    └──────┬───────┘ │
└─────────┼─────────────────────┼──────────────────────┼─────────┘
          │                     │                      │
          ▼                     ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Backend API Layer                           │
│  ┌──────────────┐    ┌───────────────────┐    ┌──────────────┐ │
│  │ PUT /catalog │    │POST /catalog/{id}/ │    │GET /cloud/   │ │
│  │ (update)     │    │credential-suggest  │    │accessible    │ │
│  └──────┬───────┘    └────────┬──────────┘    └──────┬───────┘ │
└─────────┼─────────────────────┼──────────────────────┼─────────┘
          │                     │                      │
          ▼                     ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Service Layer                               │
│  ┌────────────────────────────────────────────┐                 │
│  │   credential_selection_service.py          │                 │
│  │   - filter_credentials_by_provider()       │                 │
│  │   - evaluate_selection_rules()             │                 │
│  │   - resolve_template()                     │                 │
│  │   - score_credentials()                    │                 │
│  │   - suggest_credentials()                  │                 │
│  └────────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Data Layer (DynamoDB)                       │
│  PK=TENANT#{tenant_id}, SK=CATALOG#{id}                         │
│  + cloud_provider: "aws"|"azure"|"gcp"|null                     │
│  + credential_selection_config: {...}                            │
└─────────────────────────────────────────────────────────────────┘
```

### File Locations

| Component | Path |
|-----------|------|
| Models | `backend/app/models/credential_selection.py` |
| Catalog model additions | `backend/app/models/catalog.py` |
| Service logic | `backend/app/services/credential_selection_service.py` |
| API endpoint | `backend/app/api/v1/endpoints/catalog.py` |
| Credential endpoint (provider filter) | `backend/app/api/v1/endpoints/credentials.py` |
| Frontend provisioning | `frontend/src/pages/CatalogView.tsx` |
| Frontend admin config | `frontend/src/pages/CatalogCreate.tsx` |
| Tests | `backend/tests/test_credential_selection_service.py` |

---

## Data Models

### CatalogCloudProvider Enum

```python
# backend/app/models/catalog.py
class CatalogCloudProvider(str, Enum):
    AWS = "aws"
    AZURE = "azure"
    GCP = "gcp"
```

### SelectionRule

```python
class SelectionRule(BaseModel):
    form_field_name: str          # References a field in the catalog's form_schema
    match_strategy: MatchStrategy  # "tag" or "name_pattern"
    # Tag matching fields (required when strategy is "tag")
    tag_key: Optional[str]
    tag_value_template: Optional[str]
    # Name pattern fields (required when strategy is "name_pattern")
    name_pattern_template: Optional[str]
```

**Validation rules:**
- `form_field_name` must be non-empty
- When `match_strategy == "tag"`: both `tag_key` and `tag_value_template` are required
- When `match_strategy == "name_pattern"`: `name_pattern_template` is required

### CredentialSelectionConfig

```python
class CredentialSelectionConfig(BaseModel):
    rules: List[SelectionRule] = []
    show_credential_dropdown: bool = True   # Show dropdown to users (False = silent resolution)
    allow_override: bool = True             # Show "Change" button when auto-resolved
```

**Admin options:**
| Field | Default | Effect |
|-------|---------|--------|
| `show_credential_dropdown` | `true` | When `false`, the credential dropdown is completely hidden from users. Credential is resolved silently based on rules. |
| `allow_override` | `true` | When `false`, the "Change" button is hidden after auto-resolution. Users cannot override the selected credential. |

### Request/Response Models

```python
class CredentialSuggestRequest(BaseModel):
    form_values: Dict[str, Any]  # Form field names → current values

class CredentialSuggestionItem(BaseModel):
    credential_id: str
    name: str
    provider: str
    region: Optional[str]
    regions: Optional[List[str]]
    tags: Optional[Dict[str, str]]
    matched_rules: int        # Number of rules this credential matched
    is_recommended: bool      # True if this is the single best match

class CredentialSuggestResponse(BaseModel):
    suggestions: List[CredentialSuggestionItem]
    total_available: int      # Total credentials after provider filtering
    rules_evaluated: int      # Number of rules that were evaluated
```

### Catalog Model Additions

| Model | New Fields |
|-------|-----------|
| `CatalogCreate` | `cloud_provider: Optional[CatalogCloudProvider]`, `credential_selection_config: Optional[CredentialSelectionConfig]` |
| `CatalogUpdate` | Same as above |
| `CatalogInDB` | `cloud_provider: Optional[str]`, `credential_selection_config: Optional[Dict[str, Any]]` |

---

## Service Layer

All business logic lives in `backend/app/services/credential_selection_service.py`.

### Function Reference

| Function | Purpose |
|----------|---------|
| `resolve_template(template, form_values)` | Replace `{{field_name}}` placeholders with form values |
| `filter_credentials_by_provider(credentials, cloud_provider)` | Filter by provider (case-insensitive); returns all if provider is None |
| `evaluate_tag_rule(credential, rule, form_values)` | Check if credential has a tag matching the resolved template value |
| `evaluate_name_pattern_rule(credential, rule, form_values)` | Check if credential name contains the resolved pattern (case-insensitive substring) |
| `evaluate_rule(credential, rule, form_values)` | Dispatch to the correct evaluator based on `match_strategy` |
| `get_evaluable_rules(config, form_values)` | Return only rules whose form field has a non-empty value |
| `score_credentials(credentials, rules, form_values)` | Score each credential by matched rule count; sorted descending |
| `suggest_credentials(catalog, form_values, ...)` | Main orchestration: filter → evaluate → score → recommend |

### Suggestion Algorithm

```
1. Get all accessible credentials for the user (respects RBAC)
2. Filter by catalog's cloud_provider (if set)
3. Determine evaluable rules (skip rules with empty form values)
4. Score each credential: count how many rules it matches
5. Determine recommendation:
   - If exactly ONE credential matches ALL rules → mark as recommended
   - If multiple match all rules → no recommendation (ambiguous)
   - If none match all rules → no recommendation
6. Return scored list sorted by matched_rules descending
```

### Key Behaviors

- **AND logic**: Multiple rules use AND logic — a credential must match ALL evaluable rules to be the recommended pick
- **Empty field skipping**: Rules whose referenced form field is empty/null are skipped entirely (not counted as failures)
- **Case-insensitive**: Both tag value matching and name pattern matching are case-insensitive
- **Non-blocking**: If the suggestion API fails, the frontend silently falls back to manual selection

---

## API Endpoints

### POST `/api/v1/catalog/{catalog_id}/credential-suggestions`

**Auth:** Bearer token required. User must have access to the catalog (role or group-based).

**Request:**
```json
{
  "form_values": {
    "environment": "production",
    "region": "us-east-1"
  }
}
```

**Response (200):**
```json
{
  "suggestions": [
    {
      "credential_id": "uuid-1",
      "name": "prod-aws-account",
      "provider": "aws",
      "region": "us-east-1",
      "regions": ["us-east-1", "us-west-2"],
      "tags": {"environment": "production"},
      "matched_rules": 2,
      "is_recommended": true
    }
  ],
  "total_available": 5,
  "rules_evaluated": 2
}
```

**Error Responses:**
| Status | Condition |
|--------|-----------|
| 404 | Catalog not found |
| 403 | User lacks access to catalog |
| 401 | Unauthenticated |

**Edge cases:**
- No `credential_selection_config` on catalog → returns empty suggestions, `rules_evaluated: 0`
- All form values empty → returns empty suggestions (no evaluable rules)
- No credentials match → returns empty suggestions

### GET `/api/v1/cloud/accessible?provider=aws`

**New optional parameter:** `provider` (string, case-insensitive)

When set, filters the returned credentials server-side to only those matching the specified provider. When omitted, returns all accessible credentials (backward compatible).

---

## Frontend Integration

### CatalogView (Provisioning Page)

**Auto-selection flow:**

1. User fills form fields referenced by selection rules
2. After 500ms debounce, frontend calls the suggestion endpoint
3. If a single recommendation is returned, the credential dropdown auto-selects it
4. A green indicator shows "Auto-selected: Matched N rule(s)"
5. User can override by manually selecting a different credential
6. Changing form fields re-triggers the suggestion (debounced)

**Key state variables:**
```typescript
const [credentialSuggestion, setCredentialSuggestion] = useState<any>(null)
const [suggestionReason, setSuggestionReason] = useState<string>('')
const [isAutoSelected, setIsAutoSelected] = useState(false)
```

**Provider filtering:**
- The credential list fetch passes `catalog.cloud_provider` as a `provider` query param
- Client-side fallback filtering is also applied for defense-in-depth

### CatalogCreate (Admin Configuration)

**Cloud Provider dropdown:**
- Options: None, AWS, Azure, GCP
- Stored as `cloud_provider` on the catalog

**Selection Rules UI:**
- Add/remove rules dynamically
- Each rule: form field selector → match strategy → conditional inputs
- Client-side validation before save (form_field required, strategy-specific fields required)
- Saved as `credential_selection_config: { rules: [...] }`

---

## Template Syntax

Templates use `{{placeholder}}` syntax to inject dynamic values. Three sources are available:

### Form Field Values
Reference the user's form input directly by field name.

| Template | Form Values | Resolves to |
|----------|-------------|-------------|
| `{{environment}}` | `{"environment": "production"}` | `production` |
| `{{env}}-account` | `{"env": "prod"}` | `prod-account` |
| `{{env}}-{{region}}` | `{"env": "prod", "region": "us-east-1"}` | `prod-us-east-1` |
| `{{missing}}` | `{"env": "prod"}` | `{{missing}}` (unresolved) |

### Shared Variables
Reference admin-defined shared variables (Infrastructure → Shared Variables) using `vars.` prefix.

| Template | Shared Variable | Resolves to |
|----------|----------------|-------------|
| `{{vars.default_env}}` | `default_env = "production"` | `production` |
| `{{vars.team_prefix}}-aws` | `team_prefix = "platform"` | `platform-aws` |
| `{{vars.account_id}}` | `account_id = "123456"` | `123456` |

### User Context
Reference the logged-in user's information using `user.` prefix.

| Template | User Field | Example Value |
|----------|-----------|---------------|
| `{{user.username}}` | Login username | `john.doe` |
| `{{user.email}}` | Email address | `john@company.com` |
| `{{user.role}}` | Current role | `developer` |
| `{{user.first_name}}` | First name | `John` |
| `{{user.last_name}}` | Last name | `Doe` |
| `{{user.tenant_id}}` | Tenant ID | `default` |
| `{{user.user_id}}` | User UUID | `550e8400-...` |

### Combining Sources

You can mix all three sources in a single template:

```
{{vars.team_prefix}}-{{environment}}-{{user.role}}
→ "platform-production-developer"
```

**Rules:**
- Placeholders use double curly braces: `{{...}}`
- Dotted paths support one level: `{{vars.key}}` or `{{user.field}}`
- If the referenced value is missing or None, the placeholder remains as literal text
- Multiple placeholders in one template are resolved independently
- Values are converted to strings via `str(value)`
- Resolution is case-sensitive for keys

---

## Adding New Match Strategies

To add a new match strategy (e.g., "regex" or "prefix"):

1. **Add to enum** in `models/credential_selection.py`:
   ```python
   class MatchStrategy(str, Enum):
       TAG = "tag"
       NAME_PATTERN = "name_pattern"
       REGEX = "regex"  # New
   ```

2. **Add validation** in `SelectionRule.validate_strategy_fields()`:
   ```python
   elif self.match_strategy == MatchStrategy.REGEX:
       if not self.regex_pattern:
           raise ValueError("regex_pattern is required when match_strategy is 'regex'")
   ```

3. **Add evaluator** in `credential_selection_service.py`:
   ```python
   def evaluate_regex_rule(credential, rule, form_values):
       # Your logic here
       pass
   ```

4. **Update dispatcher** in `evaluate_rule()`:
   ```python
   elif rule.match_strategy == MatchStrategy.REGEX:
       return evaluate_regex_rule(credential, rule, form_values)
   ```

5. **Update frontend** — Add the new strategy option in `CatalogCreate.tsx` rule cards

---

## DynamoDB Schema

Two new optional attributes on the catalog item:

| Attribute | Type | Example |
|-----------|------|---------|
| `cloud_provider` | String (nullable) | `"aws"`, `"azure"`, `"gcp"`, or absent |
| `credential_selection_config` | Map (nullable) | `{"rules": [...]}` |

**Key schema** (unchanged): `PK = TENANT#{tenant_id}`, `SK = CATALOG#{catalog_id}`

**Write handling:**
- Setting to `null` uses DynamoDB `REMOVE` expression
- Config is sanitized via `_sanitize_for_dynamodb()` before write
- No new tables or GSIs required

---

## Testing

### Running Tests

```bash
cd backend
python -m pytest tests/test_credential_selection_service.py -v
```

### Test Coverage

| Test Class | Functions Covered |
|-----------|-------------------|
| `TestResolveTemplate` | Template placeholder resolution |
| `TestFilterCredentialsByProvider` | Provider-based filtering |
| `TestEvaluateTagRule` | Tag matching with templates |
| `TestEvaluateNamePatternRule` | Name substring matching |
| `TestEvaluateRule` | Strategy dispatch |
| `TestGetEvaluableRules` | Empty value skipping |
| `TestScoreCredentials` | Scoring and sorting |
| `TestSuggestCredentials` | End-to-end orchestration |

### Writing New Tests

When adding tests for this feature, mock `get_accessible_credentials`:

```python
@pytest.mark.asyncio
async def test_my_scenario(monkeypatch):
    from app.services import credential_selection_service

    async def mock_get_accessible(*args, **kwargs):
        return [
            {"credential_id": "c1", "name": "prod-aws", "provider": "aws", "tags": {"env": "prod"}},
        ]

    monkeypatch.setattr(
        credential_selection_service,
        "get_accessible_credentials",
        mock_get_accessible,
    )

    result = await credential_selection_service.suggest_credentials(...)
```

---

## Configuration Examples

### Example 1: Environment-Based Tag Matching

A catalog for deploying EC2 instances where the "environment" form field maps to a credential tag:

```json
{
  "cloud_provider": "aws",
  "credential_selection_config": {
    "rules": [
      {
        "form_field_name": "environment",
        "match_strategy": "tag",
        "tag_key": "env",
        "tag_value_template": "{{environment}}"
      }
    ]
  }
}
```

**Behavior:** When user selects "production" in the environment dropdown, the system finds credentials tagged with `env=production`.

### Example 2: Multi-Rule Configuration

A catalog that matches by both environment tag AND account name pattern:

```json
{
  "cloud_provider": "aws",
  "credential_selection_config": {
    "rules": [
      {
        "form_field_name": "environment",
        "match_strategy": "tag",
        "tag_key": "environment",
        "tag_value_template": "{{environment}}"
      },
      {
        "form_field_name": "team",
        "match_strategy": "name_pattern",
        "name_pattern_template": "{{team}}"
      }
    ]
  }
}
```

**Behavior:** Credential must have the matching environment tag AND contain the team name in its credential name. Both rules must match for a recommendation.

### Example 3: Name Pattern with Static Text

```json
{
  "credential_selection_config": {
    "rules": [
      {
        "form_field_name": "environment",
        "match_strategy": "name_pattern",
        "name_pattern_template": "{{environment}}-aws-account"
      }
    ]
  }
}
```

**Behavior:** If user enters "prod", matches credentials whose name contains "prod-aws-account" (case-insensitive).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| No suggestions returned | Form fields empty or not matching rule's `form_field_name` | Verify field names in rules match the form_schema field_name values |
| Multiple matches, no recommendation | More than one credential matches all rules | Add more specific rules to narrow down to one credential |
| Suggestion not updating | Debounce delay (500ms) | Wait for debounce; check browser network tab |
| 403 on suggestion endpoint | User lacks catalog access | Verify user's role is in `allowed_roles` or user is in `allowed_group_ids` |
| Provider filter not working | Credential `provider` field doesn't match | Check credential provider values are lowercase (aws/azure/gcp) |

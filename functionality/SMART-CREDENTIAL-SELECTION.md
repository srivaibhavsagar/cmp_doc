# Smart Credential Selection — Functionality Guide

> Intelligent credential filtering and auto-selection for catalog provisioning and Day 2 operations.

---

## Overview

Smart Credential Selection reduces friction in the provisioning workflow by automatically suggesting the right cloud credential based on what the user is filling in the form. Instead of scrolling through a list of all accessible credentials, users see only relevant ones — and in many cases, the correct credential is pre-selected for them.

**Key capabilities:**
- Cloud provider filtering (show only AWS/Azure/GCP credentials based on catalog target)
- Rule-based matching (map form field values to credential tags or name patterns)
- Auto-selection with user override (pre-select when exactly one credential matches)
- Works for both Day 1 provisioning and Day 2 operations

---

## How It Works

### For Users (Provisioning)

1. You open a catalog item and start filling the request form
2. As you fill fields (like "environment" or "region"), the system evaluates configured rules
3. If a single credential matches all rules, it's automatically selected in the dropdown
4. A green indicator shows: "Auto-selected: Matched 2 rule(s)"
5. You can always override by manually selecting a different credential
6. If no match is found, the dropdown works normally (manual selection)

### For Admins (Configuration)

1. When creating/editing a catalog, set the **Cloud Provider** (AWS, Azure, or GCP)
2. Add **Credential Selection Rules** that map form fields to credential matching criteria
3. Each rule specifies:
   - Which form field to watch
   - How to match (by credential tag or by credential name pattern)
   - The matching template (can reference the form field value)

---

## Features

### 1. Cloud Provider Filtering

**What it does:** When a catalog has a cloud provider set, only credentials of that provider type are shown to users.

**Example:** A catalog targeting AWS will only show AWS credentials in the dropdown — Azure and GCP credentials are hidden.

**Configuration:**
- Set via the "Cloud Provider" dropdown in the catalog editor
- Options: None (show all), AWS, Azure, GCP
- Applies to both the credential dropdown and the suggestion engine

**Behavior when not set:** All accessible credentials are shown (existing behavior preserved).

---

### 2. Tag-Based Matching

**What it does:** Matches credentials by checking their tags against a value derived from the user's form input.

**Prerequisites:** Credentials must have tags assigned. Tags can be added when creating or editing a credential in the Infrastructure → Cloud Accounts page.

**Example scenario:**
- Credentials are tagged with `environment=production`, `environment=staging`, etc.
- The catalog form has an "Environment" dropdown
- Rule: When user selects "production", find credentials tagged `environment=production`

**Configuration:**
```
Form Field: environment
Strategy: Tag Match
Tag Key: environment
Tag Value Template: {{environment}}
```

**How templates work:**
- `{{environment}}` — replaced with the user's form field value
- `{{vars.key}}` — replaced with a shared variable (Infrastructure → Shared Variables)
- `{{user.username}}` — replaced with the logged-in user's username (also: `user.email`, `user.role`, `user.first_name`, `user.last_name`, `user.tenant_id`)
- You can add static text: `env-{{environment}}` resolves to `env-production`
- Multiple placeholders: `{{vars.team}}-{{environment}}` resolves to `platform-production`

---

### 3. Name Pattern Matching

**What it does:** Matches credentials whose name contains a pattern derived from the user's form input, shared variables, or user context (case-insensitive substring match).

**Example scenario:**
- Credentials are named like `prod-aws-platform`, `staging-aws-platform`, `prod-aws-data`
- Rule: When user selects "prod" as environment, find credentials with "prod" in the name

**Configuration:**
```
Form Field: environment
Strategy: Name Pattern
Name Pattern Template: {{environment}}
```

**Advanced patterns:**
- `{{environment}}-aws` matches credentials containing "prod-aws" in their name
- `{{vars.team_prefix}}-{{environment}}` matches credentials containing "platform-prod"
- `{{user.username}}` matches credentials containing the logged-in user's username

---

### 3b. Using Shared Variables and User Context

Beyond form field values, templates can reference:

**Shared Variables** (set by admins under Infrastructure → Shared Variables):
```
Tag Value Template: {{vars.default_environment}}
Name Pattern Template: {{vars.account_prefix}}-{{environment}}
```

**User Context** (automatically available for the logged-in user):
```
Tag Value Template: {{user.role}}
Name Pattern Template: {{user.username}}
Tag Value Template: {{user.tenant_id}}
```

**Example use cases:**
| Goal | Template | Resolves to |
|------|----------|-------------|
| Match credential by user's team (shared var) | `{{vars.team}}` | `platform` |
| Match credential by username in name | `{{user.username}}` | `john.doe` |
| Match credential by role tag | `{{user.role}}` | `developer` |
| Combine shared var + form field | `{{vars.org}}-{{environment}}` | `acme-production` |

---

### 4. Multiple Rules (AND Logic)

When multiple rules are configured, a credential must match **ALL** rules to be recommended.

**Example:**
- Rule 1: Tag `environment` matches form field "environment"
- Rule 2: Name contains form field "team"

A credential is recommended only if it has the right environment tag AND contains the team name.

**Scoring:** Credentials are ranked by how many rules they match. Even partial matches are shown in the suggestion list, but only a full match gets the auto-selection.

---

### 5. Empty Field Skipping

Rules are only evaluated when their referenced form field has a non-empty value.

**Example:** If you have rules for "environment" and "region", but the user has only filled "environment" so far:
- Only the environment rule is evaluated
- The region rule is skipped (not counted as a failure)
- As the user fills more fields, more rules activate

This means suggestions improve progressively as the user fills the form.

---

### 6. Auto-Selection with Override

**Auto-selection triggers when:**
- Exactly ONE credential matches ALL evaluated rules
- The system is confident in the recommendation

**Auto-selection does NOT trigger when:**
- Multiple credentials match all rules (ambiguous)
- No credentials match all rules
- The catalog has no selection rules configured

**Admin controls:**

| Setting | Effect |
|---------|--------|
| Show credential dropdown (checked) | Users see the full credential dropdown and can select manually |
| Show credential dropdown (unchecked) | Credential is resolved silently — users don't see any credential UI |
| Allow override (checked) | A "Change" button appears after auto-resolution, letting users pick a different credential |
| Allow override (unchecked) | No "Change" button — the auto-resolved credential is final |

**Behavior matrix:**

| Show Dropdown | Allow Override | User Experience |
|:---:|:---:|---|
| ✓ | ✓ | Full dropdown shown; auto-selects when match found; user can change anytime |
| ✓ | ✗ | Full dropdown shown initially; after auto-resolve, locked to resolved credential |
| ✗ | ✓ | No dropdown; shows resolved credential with "Change" button |
| ✗ | ✗ | Completely hidden; credential resolved silently in background |

---

### 7. Day 1 and Day 2 Support

Smart credential selection works for both catalog types:

| Catalog Type | Behavior |
|-------------|----------|
| Day 1 (Provisioning) | Credential dropdown always shown; auto-selection active |
| Day 2 (Operations) | Credential dropdown shown when cloud_provider or selection rules are configured |

---

## Admin Configuration Guide

### Step 1: Set Cloud Provider

In the catalog editor (Step 3: Provisioning):
1. Find the "Cloud Provider" dropdown
2. Select the target cloud: AWS, Azure, or GCP
3. Leave as "None" to show all credentials

### Step 2: Add Selection Rules

Below the cloud provider dropdown:
1. Click "+ Add Rule"
2. Select the **Form Field** — this is the field whose value drives the matching
3. Choose a **Match Strategy**:
   - **Tag Match** — Match by credential tag key/value
   - **Name Pattern** — Match by credential name substring
4. Fill in the strategy-specific fields:
   - For Tag Match: enter the tag key and value template
   - For Name Pattern: enter the name pattern template
5. Add more rules as needed (they combine with AND logic)

### Step 3: Validate and Save

- The system validates rules before saving
- Required fields are enforced (form field, tag key, templates)
- Validation errors appear inline in red

### Tips for Effective Configuration

| Tip | Why |
|-----|-----|
| Tag your credentials first | Go to Infrastructure → Cloud Accounts, edit each credential, and add key-value tags (e.g., `environment=production`) |
| Use descriptive tag keys on credentials | Makes tag matching intuitive (e.g., `environment`, `team`, `project`) |
| Use consistent naming conventions | Makes name pattern matching reliable |
| Start with one rule, add more if needed | Simpler configs are easier to debug |
| Test with the actual form | Fill the form as a user would to verify suggestions work |
| Use `{{field_name}}` syntax in templates | References the exact form field value dynamically |

---

## API Reference

### Credential Suggestion Endpoint

```
POST /api/v1/catalog/{catalog_id}/credential-suggestions
```

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "form_values": {
    "environment": "production",
    "region": "us-east-1",
    "team": "platform"
  }
}
```

**Response:**
```json
{
  "suggestions": [
    {
      "credential_id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "prod-aws-platform",
      "provider": "aws",
      "region": "us-east-1",
      "regions": ["us-east-1", "us-west-2"],
      "tags": {
        "environment": "production",
        "team": "platform"
      },
      "matched_rules": 2,
      "is_recommended": true
    },
    {
      "credential_id": "550e8400-e29b-41d4-a716-446655440001",
      "name": "prod-aws-data",
      "provider": "aws",
      "region": "us-east-1",
      "regions": ["us-east-1"],
      "tags": {
        "environment": "production",
        "team": "data"
      },
      "matched_rules": 1,
      "is_recommended": false
    }
  ],
  "total_available": 8,
  "rules_evaluated": 2
}
```

**Response fields:**
| Field | Description |
|-------|-------------|
| `suggestions` | Credentials that matched at least one rule, sorted by relevance |
| `suggestions[].matched_rules` | How many rules this credential satisfied |
| `suggestions[].is_recommended` | `true` only when exactly one credential matches ALL rules |
| `total_available` | Total credentials available after provider filtering |
| `rules_evaluated` | Number of rules that had non-empty form values |

### Accessible Credentials with Provider Filter

```
GET /api/v1/cloud/accessible?provider=aws
```

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `provider` | string (optional) | Filter by cloud provider (case-insensitive) |

---

## UI Components

### Credential Dropdown (CatalogView)

The credential dropdown in the request form shows:
- Only credentials matching the catalog's cloud provider (when set)
- Auto-selection indicator (green text with info icon) when a recommendation is active
- Standard dropdown behavior when no suggestion is available

### Selection Rules Editor (CatalogCreate)

The admin configuration UI provides:
- Dynamic rule cards with add/remove
- Form field selector populated from the catalog's form schema
- Strategy-specific conditional inputs
- Inline validation errors
- Template syntax helper text

---

## Limitations and Edge Cases

| Scenario | Behavior |
|----------|----------|
| No credentials match any rule | Empty suggestion list; dropdown shows all filtered credentials |
| Multiple credentials match all rules | All shown in suggestions; none auto-selected (ambiguous) |
| User has no accessible credentials | Empty dropdown regardless of rules |
| Catalog has no form_schema | Rules can't reference fields; suggestion returns empty |
| Template placeholder references missing field | Placeholder remains unresolved (literal `{{field_name}}`) |
| Credential has no tags | Tag-based rules won't match that credential |
| Very fast typing | 500ms debounce prevents excessive API calls |

---

## Security

- The suggestion endpoint respects existing RBAC — users only see credentials they have access to
- Cloud provider filtering is applied server-side (not just client-side)
- The endpoint verifies catalog access (role-based and group-based)
- No credential secrets are exposed in the suggestion response (only metadata: name, provider, tags)

---

## Credential Tagging

For tag-based matching to work, credentials must have tags assigned. Tags are key-value pairs that describe the credential's purpose or environment.

### Adding Tags to a Credential

1. Go to **Infrastructure → Cloud Accounts**
2. Click on a credential to expand it, then click **Edit**
3. Scroll to the **Tags** section at the bottom of the form
4. Enter a key (e.g., `environment`) and value (e.g., `production`)
5. Click **+ Add** to add the tag
6. Add as many tags as needed
7. Click **Update Credential** to save

### Common Tag Patterns

| Tag Key | Example Values | Use Case |
|---------|---------------|----------|
| `environment` | `production`, `staging`, `development` | Match by deployment environment |
| `team` | `platform`, `data`, `frontend` | Match by team ownership |
| `project` | `ecommerce`, `analytics`, `infra` | Match by project |
| `region` | `us-east-1`, `eu-west-1` | Match by target region |
| `account_type` | `shared`, `dedicated` | Match by account classification |

### Tags in the API

Tags are included in credential responses:
```json
{
  "credential_id": "...",
  "name": "prod-aws-platform",
  "provider": "aws",
  "tags": {
    "environment": "production",
    "team": "platform"
  }
}
```

Tags can be set on create (`POST /api/v1/cloud`) and updated (`PATCH /api/v1/cloud/{id}`).

---

## Related Documentation

- [Reserved Keys & Catalog Guide](../developer/RESERVED-KEYS-AND-CATALOG-GUIDE.md) — Form field conventions
- [Feature Catalog](./FEATURE-CATALOG.md) — Licensed feature reference
- [CMP Complete Functionality Guide](./CMP_COMPLETE_FUNCTIONALITY_GUIDE.md) — Full platform overview

---

## Event Automation Credential Filtering

The accessible credentials endpoint (`/api/v1/cloud/accessible`) is also used by the **Event Automation** UI when configuring `call_webhook` or `call_api` actions. In this context, the credential picker filters to only show webhook-compatible provider types:

| Provider Type | Shown in Event Automation |
|---------------|:---:|
| `bearer_token` | ✓ |
| `basic_auth` | ✓ |
| `github` | ✓ |
| `aws` | ✗ |
| `azure` | ✗ |
| `gcp` | ✗ |

This ensures users only see credentials that are meaningful for HTTP authentication when configuring webhook actions — cloud provider credentials (which use signature-based auth) are excluded from the dropdown.

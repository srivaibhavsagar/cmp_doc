# Visual DAG Editor — Admin Configuration Guide

> Feature toggle management, role-based access control, AI configuration, and deployment verification for the Visual DAG Editor.

---

## Overview

The Visual DAG Editor is a canvas-based workflow builder that allows developers to compose workflows as Directed Acyclic Graphs (DAGs). As an administrator, you control:

- Whether the feature is available to users (feature toggle)
- Which roles can access the editor and in what mode (RBAC)
- AI-powered suggestions and natural language generation (AI configuration)
- Deployment health and troubleshooting

The editor is gated behind two independent controls:
1. **Feature toggle** (`visual_dag_editor`) — controls access to the DAG editor UI
2. **License feature** (`ai_assistant`) — controls whether AI features within the editor are active

---

## Feature Toggle Configuration

### Enabling the Visual DAG Editor

The `visual_dag_editor` toggle is **disabled by default** after deployment. An admin must explicitly enable it.

**Via the Admin UI:**

1. Log in as an admin user
2. Navigate to **Administration → Settings → Feature Toggles**
3. Find **"Visual DAG Editor"** in the **Admin/Developer** group
4. Toggle it to **Enabled**
5. Click **Save**

**Via the API:**

```bash
curl -X PUT \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  https://your-cmp.com/api/v1/settings/menu-toggles \
  -d '{
    "toggles": [
      {
        "key": "visual_dag_editor",
        "enabled": true,
        "roles": [],
        "groups": [],
        "users": []
      }
    ]
  }'
```

### Disabling the Visual DAG Editor

To disable the editor (e.g., during maintenance or rollback):

1. Navigate to **Administration → Settings → Feature Toggles**
2. Toggle **"Visual DAG Editor"** to **Disabled**
3. Click **Save**

When disabled:
- The Workflows page shows only the form-based WorkflowBuilder
- No entry point to the DAG editor is rendered
- Existing workflows created with the DAG editor remain intact and can still be edited via the form builder
- API endpoints for AI DAG suggestions return 403

---

## Role-Based Access Control

### Access Levels

The Visual DAG Editor enforces role-based access when the toggle is enabled:

| Role | Access Level | Capabilities |
|------|-------------|--------------|
| `admin` | Full | Create, edit, delete nodes/edges, use AI features, save workflows |
| `developer` | Full | Create, edit, delete nodes/edges, use AI features, save workflows |
| `user` | Read-only | View the DAG graph, inspect node properties (no editing) |
| `readonly` | Read-only | View the DAG graph, inspect node properties (no editing) |

### Restricting Access by Role

You can further restrict which roles see the DAG editor by configuring the toggle's `roles` field:

```json
{
  "key": "visual_dag_editor",
  "enabled": true,
  "roles": ["admin", "developer"],
  "groups": [],
  "users": []
}
```

When `roles` is non-empty, only users with those roles can access the feature. When `roles` is empty (default), all roles can access it (with read-only mode for `user` and `readonly` roles).

### Restricting Access by Group or User

For more granular control:

- **Groups:** Set `"groups": ["engineering", "devops"]` to limit access to specific user groups
- **Users:** Set `"users": ["user-id-1", "user-id-2"]` to limit access to specific users

These restrictions combine with AND logic — a user must satisfy all non-empty restriction lists.

---

## AI Configuration

The DAG editor's AI features (next-step suggestions, optimization recommendations, and natural language workflow generation) use the platform's shared AI infrastructure powered by Google Gemini.

### Prerequisites

AI features in the DAG editor require:
1. The `ai_assistant` license feature to be present in your license
2. A valid Gemini API key configured
3. The `visual_dag_editor` toggle enabled

### API Key Setup

The AI system resolves the API key in priority order:

1. **Database-stored key** (configured via Platform Settings UI) — recommended
2. **Environment variable** (`GEMINI_API_KEY`) — fallback for deployments without UI access

**Option 1: Configure via Platform Settings (recommended)**

1. Navigate to **Administration → Platform Settings → AI Configuration**
2. Select provider: **Google Gemini**
3. Select model: **gemini-2.5-flash-lite** (default, recommended for DAG suggestions)
4. Enter your Gemini API key
5. Click **Save**

The key is encrypted at rest using Fernet encryption before storage in DynamoDB.

**Option 2: Configure via environment variable**

Set the `GEMINI_API_KEY` environment variable in your deployment:

```bash
# In .env or deployment configuration
GEMINI_API_KEY=your-gemini-api-key-here
```

**Verify key configuration:**

```bash
curl -s -H "Authorization: Bearer <admin_token>" \
  https://your-cmp.com/api/v1/ai/config | jq .

# Expected response:
# {
#   "provider": "gemini",
#   "model": "gemini-2.5-flash-lite",
#   "api_key_configured": true,
#   "api_key_source": "database"
# }
```

### Rate Limiting

The AI DAG endpoints enforce per-user rate limits to prevent abuse and control costs:

| Endpoint | Rate Limit | Purpose |
|----------|-----------|---------|
| `POST /api/v1/ai/dag-suggestions` | 30 requests/minute | Next-step suggestions and optimization recommendations |
| `POST /api/v1/ai/dag-generate` | 10 requests/minute | Full DAG generation from natural language |

When a user exceeds the rate limit, the API returns HTTP 429 with a `retry_after` header. The frontend displays a user-friendly message.

Rate limits are enforced per client IP (respecting `X-Forwarded-For` from trusted proxies).

### Disabling AI Independently

AI features in the DAG editor can be disabled independently of the editor itself. The AI is gated behind the `ai_assistant` license feature key.

**Scenario: DAG editor enabled, AI disabled**

If your license does not include `ai_assistant`, or if the AI-related toggles are disabled:
- The DAG editor canvas, toolbar, and property panel work normally
- The AI Suggestion Panel is hidden
- The natural language prompt input is hidden
- No AI API calls are made

**To disable AI while keeping the editor:**

1. Ensure the `ai_assistant` license key is not present in your license, OR
2. Disable the `chatbot` toggle (which gates all AI features platform-wide), OR
3. Remove the Gemini API key from both the database and environment variable

**To verify AI status:**

```bash
curl -s -H "Authorization: Bearer <admin_token>" \
  https://your-cmp.com/api/v1/ai/config | jq .api_key_configured

# Returns: true (AI active) or false (AI inactive)
```

---

## Deployment Verification

### Step 1: Verify Backend Health

```bash
curl -s https://your-cmp.com/api/v1/health | jq .status
# Expected: "healthy"
```

### Step 2: Verify Feature Toggle Registration

```bash
curl -s -H "Authorization: Bearer <admin_token>" \
  https://your-cmp.com/api/v1/settings/menu-toggles/known-features | \
  jq '.[] | select(.key == "visual_dag_editor")'

# Expected:
# {
#   "key": "visual_dag_editor",
#   "label": "Visual DAG Editor",
#   "description": "Visual drag-and-drop workflow editor with DAG canvas",
#   "group": "Admin/Developer",
#   "licensed": true
# }
```

### Step 3: Verify Toggle State

```bash
curl -s -H "Authorization: Bearer <admin_token>" \
  https://your-cmp.com/api/v1/settings/feature-toggles/me | \
  jq .visual_dag_editor

# Expected: true (if enabled) or false (if disabled)
```

### Step 4: Verify AI Endpoints (if AI is enabled)

```bash
# Test suggestions endpoint
curl -s -X POST \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  https://your-cmp.com/api/v1/ai/dag-suggestions \
  -d '{
    "type": "suggest_next",
    "context": {
      "current_nodes": [{"step_id": "run_task_1", "action": "run_task", "name": "Deploy"}],
      "current_edges": []
    }
  }'

# Expected: 200 with suggestions array, or 502 if AI service is down
```

### Step 5: Verify Frontend Rendering

1. Log in as an admin or developer
2. Navigate to **Automation → Workflows**
3. Confirm a tab or toggle appears to switch between "Form Builder" and "DAG Editor"
4. Click "DAG Editor" — the canvas should render with toolbar and controls

---

## Troubleshooting

### Feature toggle not appearing in admin UI

**Cause:** The backend may not have the latest `KNOWN_FEATURES` list deployed.

**Fix:** Verify the backend version includes the `visual_dag_editor` entry:
```bash
curl -s -H "Authorization: Bearer <admin_token>" \
  https://your-cmp.com/api/v1/settings/menu-toggles/known-features | \
  jq '[.[] | .key]' | grep visual_dag_editor
```

If missing, redeploy the backend with the latest code.

---

### Toggle enabled but editor not showing

**Possible causes:**
1. User role is not permitted — check the toggle's `roles` restriction
2. Browser cache — hard refresh (Ctrl+Shift+R)
3. Frontend not updated — verify the frontend build includes the DAG editor components

**Diagnosis:**
```bash
# Check what the user's toggle state looks like
curl -s -H "Authorization: Bearer <user_token>" \
  https://your-cmp.com/api/v1/settings/feature-toggles/me | jq .visual_dag_editor
```

---

### AI suggestions not appearing

**Possible causes:**
1. `ai_assistant` license key not present
2. No Gemini API key configured
3. Rate limit exceeded
4. AI service temporarily unavailable

**Diagnosis:**
```bash
# Check AI config
curl -s -H "Authorization: Bearer <admin_token>" \
  https://your-cmp.com/api/v1/ai/config | jq .

# Check if api_key_configured is true and api_key_source is not "none"
```

---

### Rate limit errors (HTTP 429)

**Cause:** A user has exceeded the per-minute request limit.

**Fix:** Wait for the rate limit window to reset (1 minute). If users consistently hit limits, consider:
- Educating users about efficient AI usage
- Reviewing if automated scripts are calling the endpoints

---

### AI returns 502 errors

**Cause:** The Gemini API is unreachable or returning errors.

**Fix:**
1. Verify the API key is valid and has not been revoked
2. Check Google Cloud status for Gemini API outages
3. Verify network connectivity from the backend to `generativelanguage.googleapis.com`
4. Check backend logs for detailed error messages:
   ```bash
   docker logs <backend_container> 2>&1 | grep "AI DAG"
   ```

---

### Users see read-only mode unexpectedly

**Cause:** The user's role is `user` or `readonly`, which only grants view access.

**Fix:** If the user needs edit access, promote their role to `developer` or `admin` via **Administration → Users**.

---

## Related Documentation

- [CMP Feature Catalog](./FEATURE-CATALOG.md) — Licensed feature reference and tier breakdown
- [CMP Complete Functionality Guide](./CMP_COMPLETE_FUNCTIONALITY_GUIDE.md) — Full platform overview
- [Visual DAG Editor — User Guide](./visual-dag-editor-user.md) — End-user guide for the DAG editor
- [Visual DAG Editor — Developer Guide](./visual-dag-editor-developer.md) — Extending the editor with new node types and validators

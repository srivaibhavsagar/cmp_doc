# Inbound Webhooks

## Overview

CMP supports inbound webhook registrations that allow external services to push events into the platform. Each webhook registration creates a unique endpoint URL with optional HMAC signature verification, source filtering, and delivery tracking. Webhooks are tenant-scoped and can receive payloads from supported integrations or generic HTTP sources.

All webhook endpoints — including the inbound receiver — require a valid **API token** (Bearer auth). External systems must include your CMP API token in the `Authorization` header when sending events.

## Supported Sources

| Source | Key | Description |
|--------|-----|-------------|
| GitHub | `github` | Repository events (push, PR, issues, deployments) |
| PagerDuty | `pagerduty` | Incident lifecycle events (triggered, acknowledged, resolved) |
| Jira | `jira` | Issue events (created, updated, transitioned) |
| Generic | `generic` | Any HTTP POST with JSON body — custom integrations |

## Key Concepts

### Webhook Registration

A webhook registration defines a named inbound endpoint belonging to a tenant. Each registration includes:

- **Name** — Human-readable label (1–100 characters)
- **Description** — Optional context (up to 500 characters)
- **Source** — The external service type (`github`, `pagerduty`, `jira`, `generic`)
- **Secret** — Optional HMAC secret for payload signature verification (min 16 characters)
- **Enabled** — Active/disabled toggle

### Security

- **API Token Required** — All webhook endpoints require Bearer token authentication. Generate tokens from **Profile → API Tokens**.
- Secrets are stored as hashes — the raw secret is never persisted or returned in API responses.
- When a secret is configured, inbound payloads must include a valid HMAC signature header in addition to the Bearer token. Requests with invalid signatures are rejected.
- The API response includes a `has_secret` boolean flag but never exposes the secret value.

### Delivery Tracking

Each webhook registration tracks:

- `total_deliveries` — Count of all received payloads
- `failed_deliveries` — Count of payloads that failed processing
- `last_received_at` — Timestamp of the most recent delivery

Individual delivery records are logged with source event type, mapped CMP event, payload size, status, optional error details, and full audit trail information (payload content and the identity of the user who triggered the delivery).

### Health Status

Each webhook has a health indicator derived from its delivery history:

| Status | Badge Color | Meaning |
|--------|-------------|---------|
| Healthy | Green | Receiving events successfully with no recent failures |
| Warning | Yellow | Some failed deliveries detected |
| Inactive | Grey | No events received recently or webhook is disabled |

## Limits

| Limit | Value |
|-------|-------|
| Max registrations per tenant | 25 |
| Max payload size | 1 MB (1,048,576 bytes) |
| Secret length | 16–256 characters |
| Name length | 1–100 characters |
| Description length | Up to 500 characters |

## API Endpoints

### Create Webhook

```
POST /api/v1/webhooks/registrations
```

**Request Body:**

```json
{
  "name": "GitHub Deploy Notifications",
  "description": "Receives deployment status from main repo",
  "source": "github",
  "secret": "my-super-secret-key-1234",
  "enabled": true
}
```

**Response:** `WebhookResponse` object with `webhook_id`, `endpoint_url`, and metadata. The `secret` field is never returned — only `has_secret: true/false`.

### List Webhooks

```
GET /api/v1/webhooks/registrations
```

Returns all webhook registrations for the current tenant.

### Get Webhook

```
GET /api/v1/webhooks/registrations/{webhook_id}
```

Returns a single webhook registration with delivery stats.

### Update Webhook

```
PATCH /api/v1/webhooks/registrations/{webhook_id}
```

**Request Body (all fields optional):**

```json
{
  "name": "Updated Name",
  "description": "New description",
  "secret": "new-secret-at-least-16",
  "enabled": false
}
```

### Delete Webhook

```
DELETE /api/v1/webhooks/registrations/{webhook_id}
```

Removes the webhook registration and invalidates its endpoint URL.

### Receive Payload (Inbound Endpoint)

```
POST /api/v1/webhooks/inbound/{webhook_id}
```

This is the URL given to external services. It accepts the incoming payload, validates the API token and optional HMAC signature, maps the event to a CMP event, and emits it into the event bus.

**Required Header:**

```
Authorization: Bearer YOUR_API_TOKEN
```

**Optional Signature Headers (source-specific):**

| Source | Signature Header |
|--------|-----------------|
| GitHub | `X-Hub-Signature-256` |
| PagerDuty | `X-PagerDuty-Signature` |
| Jira | `X-Atlassian-Webhook-Signature` |
| Generic | `X-Webhook-Signature` |

### Delivery Logs

```
GET /api/v1/webhooks/inbound/logs
```

Returns delivery history across all webhooks for the tenant.

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `webhook_id` | string (optional) | Filter logs to a specific webhook |
| `limit` | integer (optional) | Max records to return (default 100) |

### Webhook Health

```
GET /api/v1/webhooks/inbound/health
```

Returns health status for all webhook registrations in the tenant. Each entry includes `health` field (`healthy`, `warning`, `inactive`, or `new`), delivery counters, and the endpoint URL.

## Delivery Log Fields

| Field | Description |
|-------|-------------|
| `delivery_id` | Unique ID for this delivery |
| `webhook_id` | Parent webhook registration |
| `source` | Source type (github, pagerduty, etc.) |
| `source_event` | Original event type from the source (e.g., `push`, `incident.triggered`) |
| `cmp_event_type` | Mapped CMP event type |
| `cmp_event_id` | ID of the emitted CMP event |
| `received_at` | ISO timestamp when payload was received |
| `payload_size_bytes` | Size of the incoming payload |
| `status` | `delivered` or `failed` |
| `error_message` | Error details if status is `failed` |
| `ip_address` | Source IP of the request |
| `payload` | The full webhook payload content (optional, stored for debugging and audit) |
| `triggered_by_user_id` | User ID of the authenticated user who sent the webhook (audit trail) |
| `triggered_by_username` | Username of the authenticated user who sent the webhook |
| `triggered_by_email` | Email of the authenticated user who sent the webhook |

## How to Use (UI)

The webhook management page is organized into two tabs: **Registrations** and **Delivery Logs**.

### Registrations Tab

1. Navigate to **Settings → Webhooks** (or **Integrations → Inbound Webhooks**)
2. Click **New Webhook** in the top-right corner
3. Fill in the inline creation form:
   - **Name** (required) — e.g., "GitHub Deploy Trigger"
   - **Source** (required) — select from GitHub, PagerDuty, Jira, or Generic
   - **Description** (optional) — contextual notes about this webhook
   - **HMAC Secret** (optional, min 16 characters) — enables payload signature verification
4. Click **Create Webhook**
5. Copy the generated **Endpoint URL** using the copy button on the registration card
6. Configure the external service to send events to that URL with your **API token** in the Authorization header

Each webhook card displays:
- Name and source badge
- Health status badge (Healthy / Warning / Inactive / New)
- "Signed" indicator if an HMAC secret is configured
- Creation date and last event timestamp
- Total deliveries and failed delivery count
- Endpoint URL with copy button
- Toggle button to enable/disable the webhook
- Delete button (with confirmation dialog)

### Delivery Logs Tab

1. Switch to the **Delivery Logs** tab
2. Optionally filter logs by a specific webhook using the dropdown
3. The delivery table shows a summary row for each delivery:
   - Received timestamp
   - Status (Delivered / Failed with error tooltip)
   - Source badge
   - Source event type
   - Mapped CMP event type
   - Triggered by (username or email of the authenticated sender)
   - Payload size in bytes
4. **Click any row to expand it** and view full delivery details:
   - **Delivery ID** — unique identifier for this delivery
   - **CMP Event ID** — the internal event that was emitted
   - **IP Address** — source IP of the request
   - **Triggered By** — full username and email of the authenticated sender
   - **Error message** (if status is failed) — displayed in a red highlight box
   - **Payload** — the full JSON request body rendered in a syntax-highlighted code block (loaded on demand)
5. Click the row again to collapse the detail view

### Refreshing Data

Click the **Refresh** button in the header to reload both registrations and logs.

## Integration Examples

### GitHub

1. Create a webhook with source `github` and a secret
2. In GitHub repo → Settings → Webhooks → Add webhook
3. Set Payload URL to the CMP endpoint URL
4. Set Content type to `application/json`
5. Set Secret to the same value used in CMP
6. Add a custom header for authentication: use a middleware/proxy or configure GitHub's auth to include `Authorization: Bearer YOUR_API_TOKEN`
7. Select events to trigger (e.g., Pushes, Pull requests, Deployments)

**Example curl:**

```bash
curl -X POST \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: sha256=COMPUTED_HMAC" \
  -d '{"action":"completed","deployment":{"id":123}}' \
  https://your-cmp-instance/api/v1/webhooks/inbound/{webhook_id}
```

### PagerDuty

1. Create a webhook with source `pagerduty`
2. In PagerDuty → Integrations → Generic Webhooks (v3)
3. Add the CMP endpoint URL as the webhook destination
4. Include Bearer token in the custom headers configuration
5. Configure event subscriptions (incident lifecycle events)

### Generic

1. Create a webhook with source `generic`
2. POST JSON payloads to the endpoint URL from any system
3. Include `Authorization: Bearer YOUR_API_TOKEN` header (required)
4. Include `X-Webhook-Signature` header if a secret is configured

## Auth Requirements

- All webhook management endpoints (create, list, update, delete) require JWT authentication (`get_current_user`)
- The inbound receive endpoint also requires a valid **API token** (Bearer auth) to identify the tenant and authorize the request
- Per-webhook HMAC signature verification is an additional optional layer when a secret is configured
- Webhook management is typically restricted to `admin` or `developer` roles
- Generate API tokens from **Profile → API Tokens** in the CMP UI

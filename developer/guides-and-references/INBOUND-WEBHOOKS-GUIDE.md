# Inbound Webhooks — Developer Guide

## Architecture

Inbound webhooks follow the standard CMP three-layer separation:

```
models/webhook.py       → Pydantic schemas (WebhookCreate, WebhookUpdate, WebhookInDB, etc.)
crud/webhook.py         → DynamoDB CRUD operations
api/v1/endpoints/webhook.py → FastAPI route handlers
services/webhook.py     → Signature verification, event mapping, payload processing
```

## DynamoDB Schema

### Webhook Registrations

| Attribute | Value |
|-----------|-------|
| PK | `TENANT#{tenant_id}` |
| SK | `WEBHOOK#{webhook_id}` |

### Delivery Logs

| Attribute | Value |
|-----------|-------|
| PK | `TENANT#{tenant_id}` |
| SK | `WHDELIVERY#{webhook_id}#{delivery_id}` |

**Listing webhooks:** `begins_with(SK, "WEBHOOK#")` within the tenant partition.

**Listing deliveries:** `begins_with(SK, "WHDELIVERY#{webhook_id}#")` within the tenant partition.

## Models

### Enums

- `WebhookSource` — `github`, `pagerduty`, `jira`, `generic`
- `WebhookStatus` — `active`, `disabled`

### Schemas

| Model | Purpose |
|-------|---------|
| `WebhookCreate` | POST request body — name, description, source, secret, enabled |
| `WebhookUpdate` | PATCH request body — all fields optional |
| `WebhookInDB` | Stored record — includes `secret_hash`, counters, timestamps |
| `WebhookResponse` | API response — includes `has_secret` flag, never raw secret |
| `WebhookDeliveryLog` | Delivery record — source_event → cmp_event mapping, status |

### Constants

- `MAX_PAYLOAD_SIZE_BYTES = 1_048_576` (1 MB)
- `MAX_WEBHOOKS_PER_TENANT = 25`

## Secret Handling

1. User provides a plaintext secret in `WebhookCreate.secret`
2. Backend hashes it (e.g., SHA-256 or bcrypt) and stores only `secret_hash` in DynamoDB
3. On inbound delivery, compute HMAC of request body using the stored hash for verification
4. API responses return `has_secret: true/false` — never the secret itself

## Signature Verification

Each source has a different header convention:

```python
SIGNATURE_HEADERS = {
    "github": "X-Hub-Signature-256",
    "pagerduty": "X-PagerDuty-Signature",
    "jira": "X-Atlassian-Webhook-Signature",
    "generic": "X-Webhook-Signature",
}
```

Verification flow:
1. Extract signature from the source-specific header
2. Compute expected HMAC-SHA256 of the raw request body using the stored secret
3. Compare using `hmac.compare_digest()` (timing-safe)
4. Reject with 401 if mismatch

## Event Mapping

Inbound source events are mapped to CMP event types and emitted through `BackgroundTasks`:

```python
background_tasks.add_task(
    emit_event, EventType.WEBHOOK_RECEIVED,
    actor={"type": "webhook", "webhook_id": webhook_id},
    resource={"type": "webhook_delivery", "id": delivery_id},
    metadata={"source": source, "source_event": event_type},
    source="webhook"
)
```

## Adding a New Source

1. Add the source key to `WebhookSource` enum in `models/webhook.py`
2. Add the signature header mapping in the webhook service
3. Implement source-specific payload parsing (extract event type, relevant metadata)
4. Map source events to CMP event types
5. Update documentation

## Error Handling

- Payload too large (>1 MB): Return `413 Payload Too Large`
- Invalid signature: Return `401 Unauthorized`
- Webhook disabled: Return `404 Not Found` (don't reveal existence)
- Webhook not found: Return `404 Not Found`
- Tenant limit exceeded on create: Return `409 Conflict` with descriptive message

## Testing

```bash
# Create a webhook
curl -X POST http://localhost:8001/api/v1/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Hook", "source": "generic", "secret": "test-secret-12345678"}'

# Send a test payload
curl -X POST http://localhost:8001/api/v1/webhooks/{webhook_id}/receive \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Signature: sha256=..." \
  -d '{"event": "test", "data": {"key": "value"}}'
```

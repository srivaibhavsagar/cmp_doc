# CMP Agent — Developer Guide

## Overview

The CMP Agent subsystem provides server-side infrastructure for registering lightweight monitoring agents on provisioned VMs and ingesting their system metrics. This guide covers the data models, DynamoDB schema, API patterns, and extension points.

---

## Module Location

```
backend/app/
├── models/agent.py           # Pydantic models (registration, metrics, responses)
├── crud/agent.py             # DynamoDB CRUD operations (to be implemented)
├── api/v1/endpoints/agent.py # API routes (to be implemented)
├── services/agent.py         # Business logic (to be implemented)
```

---

## Data Models

### Registration Models

| Model | Purpose |
|-------|---------|
| `AgentRegistration` | Stored record of a registered agent (DynamoDB item) |
| `AgentRegisterRequest` | Request body when agent registers itself |
| `AgentRegisterResponse` | Response with agent_id, API key, and endpoint info |

### Metric Models

| Model | Purpose |
|-------|---------|
| `CpuMetrics` | CPU usage, cores, load averages |
| `MemoryMetrics` | RAM/swap usage in bytes and percentage |
| `DiskVolume` | Per-volume disk usage stats |
| `NetworkInterface` | Per-interface network stats |
| `OsInfo` | Operating system metadata |
| `ProcessInfo` | Individual process stats |
| `AgentMetricsPayload` | Full metrics push from agent |
| `AgentMetricsStored` | Flattened metrics record stored in DynamoDB |

### Response Models

| Model | Purpose |
|-------|---------|
| `AgentStatusResponse` | Agent health status for frontend |
| `AgentLatestMetrics` | Latest metrics snapshot for frontend |
| `AgentMetricsHistory` | Time-series data points for charts |

---

## DynamoDB Schema

### Agent Registration Table

| Key | Pattern | Example |
|-----|---------|---------|
| PK | `TENANT#{tenant_id}` | `TENANT#default` |
| SK | `AGENT#{agent_id}` | `AGENT#ag_abc123` |

### Agent Metrics Table

| Key | Pattern | Example |
|-----|---------|---------|
| PK | `TENANT#{tenant_id}` | `TENANT#default` |
| SK | `METRIC#{resource_id}#{timestamp}` | `METRIC#res_xyz#2025-01-15T10:30:00Z` |

- Metrics records include a `ttl` field for DynamoDB's Time-to-Live auto-expiry
- Query latest: `begins_with(SK, "METRIC#{resource_id}#")` with `ScanIndexForward=False`, `Limit=1`
- Query history: `begins_with(SK, "METRIC#{resource_id}#")` with time range in `between` condition

---

## Automatic Token Generation in Task Runner

When a workflow task executes (e.g., VM provisioning), the task runner automatically generates an agent registration token and injects it into the task execution context under the `agent` key. This enables tasks to embed agent installation in VM `user_data` scripts without manual token management.

### How It Works

1. The task runner checks the task's input parameters for a `resource_id` or `instance_id` (falls back to `execution_id`)
2. If a resource identifier is found, it calls `create_registration_token()` to generate a one-time token
3. The token, agent endpoint URL, and install script URL are assembled into an `agent` object
4. This object is injected into the task execution context alongside `params`, `credential`, `user`, etc.

### Task Context `agent` Object

```json
{
  "token": "reg_abc123...",
  "endpoint": "https://your-domain.com/api/v1/agent",
  "install_url": "https://your-domain.com/api/v1/agent/install.sh",
  "install_url_windows": "https://your-domain.com/api/v1/agent/install.ps1",
  "resource_id": "res_xyz789",
  "report_interval": 60
}
```

| Field | Description |
|-------|-------------|
| `token` | One-time registration token for the agent to authenticate during registration |
| `endpoint` | Base URL of the CMP agent API |
| `install_url` | URL to the Linux agent install script (bash) |
| `install_url_windows` | URL to the Windows agent install script (PowerShell) |
| `resource_id` | The resource identifier linked to this token |
| `report_interval` | Recommended metrics reporting interval in seconds |

### Using in Task Scripts

In a provisioning task (e.g., Terraform, shell script, or cloud-init), reference the agent context to embed auto-registration:

```bash
#!/bin/bash
# Example user_data script using task context agent info
curl -sSL "{{ context.agent.install_url }}" | bash -s -- \
  --token "{{ context.agent.token }}" \
  --endpoint "{{ context.agent.endpoint }}" \
  --resource-id "{{ context.agent.resource_id }}" \
  --interval {{ context.agent.report_interval }}
```

### Graceful Degradation

If token generation fails (e.g., `create_registration_token` is unavailable or the resource identifier is missing), the `agent` field will be an empty object `{}`. Task scripts should check for the presence of the token before attempting installation:

```bash
{% if context.agent.token %}
# Install CMP agent
curl -sSL "{{ context.agent.install_url }}" | bash -s -- --token "{{ context.agent.token }}"
{% endif %}
```

---

## Pre-Built `user_data` Context Keys

In addition to the raw `agent` object, the task runner provides three ready-to-use startup script keys in the task execution context. Each combines SSH key injection and CMP Agent installation in the format required by its target cloud:

| Key | Format | Use For |
|-----|--------|---------|
| `cmp["user_data"]` | Cloud-init YAML | AWS EC2, Azure VMs (Linux) |
| `cmp["user_data_gcp"]` | Plain bash (`#!/bin/bash`) | GCP Compute Engine VMs |
| `cmp["user_data_windows"]` | PowerShell (`<powershell>`) | Windows VMs (all clouds) |

### Why Separate Keys for GCP?

GCP's guest agent executes the `startup-script` metadata key as a **plain shell script**, not as cloud-init YAML. Passing a cloud-init document to GCP results in the YAML being treated as literal shell commands, which fails. `cmp["user_data_gcp"]` provides a `#!/bin/bash` script with the same SSH + agent payload in the format GCP expects.

### Why Use Pre-Built Keys

Previously, task authors had to manually construct startup scripts by referencing `cmp["agent"]` fields. The pre-built keys handle both SSH access and agent installation in a single string — reducing boilerplate and the risk of misconfiguration.

### Usage in Provisioning Tasks

```python
# AWS EC2 (Linux) — cloud-init YAML
user_data = cmp.get("user_data", "")
instances = ec2.create_instances(
    ImageId=params["ami_id"],
    InstanceType=params["instance_type"],
    MinCount=1, MaxCount=1,
    UserData=user_data,  # SSH keys + agent install combined
)
```

```python
# GCP Compute Engine — plain bash startup-script
user_data_gcp = cmp.get("user_data_gcp", "")
if user_data_gcp:
    metadata_items.append({"key": "startup-script", "value": user_data_gcp})
# Do NOT use cmp["user_data"] here — GCP does not process cloud-init YAML
```

```python
# Azure VM (Linux) — cloud-init YAML
user_data = cmp.get("user_data", "")
if user_data:
    import base64
    vm_params["os_profile"]["custom_data"] = base64.b64encode(user_data.encode()).decode()
```

```python
# Appending custom commands (Linux / GCP)
base = cmp.get("user_data_gcp", "#!/bin/bash\n")  # use user_data for AWS/Azure
custom = "\napt-get install -y nginx\nsystemctl enable nginx\n"
full_script = base + custom
```

### Graceful Degradation

If script generation fails (e.g., no SSH keys and no agent token), the key will be an empty string `""`. Handle this gracefully:

```python
user_data_gcp = cmp.get("user_data_gcp", "")
if user_data_gcp:
    metadata_items.append({"key": "startup-script", "value": user_data_gcp})
```

### Relationship to `agent` Key

| Context Key | Type | Content | Use For |
|-------------|------|---------|---------|
| `cmp["agent"]` | `dict` | Raw agent registration fields (token, endpoint, install_url, etc.) | Fine-grained control over agent install |
| `cmp["user_data"]` | `str` | Cloud-init YAML combining SSH keys + agent install | AWS EC2, Azure VMs (Linux) |
| `cmp["user_data_gcp"]` | `str` | Plain bash startup-script combining SSH keys + agent install | GCP Compute Engine VMs |
| `cmp["user_data_windows"]` | `str` | PowerShell script with agent install + custom commands | Windows VMs (all clouds) |

Use `cmp["user_data"]` for AWS and Azure Linux VMs, `cmp["user_data_gcp"]` for GCP VMs, and `cmp["user_data_windows"]` for Windows. The GCP key exists because GCP's guest agent runs `startup-script` metadata as a plain shell script — it does not process cloud-init YAML. Use `cmp["agent"]` when you need fine-grained control over the agent installation (e.g., custom flags or conditional logic).

### Windows Provisioning

For Windows instances, use the pre-built PowerShell script:

```python
# Windows VM provisioning — use the pre-built PowerShell user_data
user_data_windows = cmp.get("user_data_windows", "")

# Pass to AWS (Windows EC2)
if user_data_windows:
    run_kwargs["UserData"] = user_data_windows

# Or append custom PowerShell commands before the closing tag
if user_data_windows:
    # Insert custom commands before </powershell>
    custom_ps = "Install-WindowsFeature -Name Web-Server\n"
    user_data_windows = user_data_windows.replace("</powershell>", f"{custom_ps}</powershell>")
```

The Windows script wraps all content in `<powershell>...</powershell>` tags as required by AWS and Azure for Windows user_data. It includes the same multi-cloud metadata resolution logic (AWS IMDSv2, Azure IMDS, GCP metadata) to automatically identify the VM's resource ID.

---

## Registration Flow

```
VM Agent                          CMP Backend
   |                                   |
   |-- POST /api/v1/agent/register --->|
   |   { registration_token,           |
   |     resource_id, hostname, ... }  |
   |                                   |-- Validate one-time token
   |                                   |-- Generate agent_id + API key
   |                                   |-- Hash API key (bcrypt)
   |                                   |-- Store AgentRegistration
   |                                   |
   |<-- 201 Created ------------------|
   |   { agent_id, agent_api_key,      |
   |     metrics_endpoint,             |
   |     report_interval_seconds }     |
   |                                   |
```

---

## Metrics Reporting Flow

```
VM Agent                          CMP Backend
   |                                   |
   |-- POST /api/v1/agent/metrics ---->|
   |   Authorization: Bearer <api_key> |
   |   { agent_id, resource_id,        |
   |     timestamp, cpu, memory, ... } |
   |                                   |-- Validate agent API key
   |                                   |-- Sanitize for DynamoDB
   |                                   |-- Store with TTL
   |                                   |-- Update last_seen_at
   |                                   |
   |<-- 202 Accepted ------------------|
   |                                   |
```

---

## API Endpoints (Planned)

### Agent-Facing (called by the agent on the VM)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/agent/register` | Registration token | Register agent and get API key |
| POST | `/api/v1/agent/metrics` | Agent API key | Push metrics payload |

### Install Script Endpoints (no auth — token is passed as script parameter)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/agent/install.sh` | None | Serve Linux agent install script (bash) |
| GET | `/api/v1/agent/install.ps1` | None | Serve Windows agent install script (PowerShell) |

Both install endpoints accept an optional `?endpoint=<url>` query parameter to override the CMP backend URL embedded in the script.

### User-Facing (called by the frontend)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/resources/{id}/agent/status` | JWT | Get agent connection status |
| GET | `/api/v1/resources/{id}/agent/metrics/latest` | JWT | Get latest metrics snapshot |
| GET | `/api/v1/resources/{id}/agent/metrics/history` | JWT | Get metrics time-series |
| POST | `/api/v1/resources/{id}/agent/revoke` | JWT (admin) | Revoke agent token |

---

## Key Implementation Notes

### Sanitization

Always call `_sanitize_for_dynamodb()` before storing metrics — the payload contains `float` values (usage percentages) that must be converted to `Decimal`.

### Async Wrapping

All DynamoDB operations must be wrapped with `run_in_executor` per CMP conventions:

```python
async def store_metrics(payload: AgentMetricsStored):
    loop = asyncio.get_event_loop()
    await loop.run_in_executor(None, _store_metrics_sync, payload)
```

### TTL Strategy

Set `ttl` on metrics records to auto-expire old data:

```python
from datetime import datetime, timedelta
ttl = int((datetime.utcnow() + timedelta(days=7)).timestamp())
```

### Agent Health Detection

An agent is considered "connected" if `last_seen_at` is within `3 * report_interval_seconds` of the current time:

```python
is_connected = (now - last_seen_at).total_seconds() < (3 * 300)
```

### Event Emission

Emit events for agent lifecycle changes:

```python
background_tasks.add_task(
    emit_event, EventType.AGENT_REGISTERED,
    actor={"agent_id": agent_id},
    resource={"resource_id": resource_id},
    metadata={"hostname": hostname},
    source="agent"
)
```

---

## End-to-End Testing

An E2E test script is available at `backend/tests/test_agent_e2e.py` that exercises the full agent lifecycle against a running local backend.

### Prerequisites

- Backend running on `http://localhost:8001`
- A valid admin user (defaults to `admin@cmp.local` / `admin`)

### Running the Test

```bash
cd backend
python tests/test_agent_e2e.py
```

### What It Covers

| Step | Description |
|------|-------------|
| 1 | Generate a registration token (admin action) |
| 2 | Register the agent (simulates VM calling CMP) |
| 3 | Push sample metrics (two data points for history) |
| 4 | Retrieve latest metrics via user-facing API |
| 5 | Verify history endpoint returns multiple data points |
| 6 | Check agent status endpoint |
| 7 | List all agents (admin) |
| 8 | Security: invalid API key correctly rejected (401) |
| 9 | Security: reused/invalid registration token rejected (401) |

### Configuration

Edit the constants at the top of the script to match your local setup:

```python
BASE_URL = "http://localhost:8001"
ADMIN_EMAIL = "admin@cmp.local"
ADMIN_PASSWORD = "admin"
TEST_RESOURCE_ID = "i-test-agent-001"
```

### After Running

The test creates a registered agent for `i-test-agent-001` with sample metrics. You can verify the frontend integration by navigating to a resource detail page and checking the **System Metrics** tab.

---

## Extension Points

- **Custom Metrics**: The `custom_metrics` dict in `AgentMetricsPayload` allows agents to report application-specific metrics without model changes
- **Alerting**: Subscribe to agent events to trigger alerts on disconnect or threshold breaches
- **Multi-Cloud**: The agent is cloud-agnostic — works on any VM regardless of provider

# CMP Agent — Architecture & Data Flow

## End-to-End Flow

This document explains how the CMP monitoring agent connects to the platform and how metrics appear on the System Metrics tab.

---

## Step 1: Registration (One-Time, On First Boot)

When the agent starts for the first time, it registers with CMP using the one-time token from `config.json`.

```
  VM (agent.py starts)                         CMP Backend
  ┌─────────────────┐                         ┌─────────────────┐
  │ Reads config:   │                         │                 │
  │  - endpoint     │   POST /agent/register  │ Validates token │
  │  - token        │ ───────────────────────▶│ (one-time use)  │
  │  - resource_id  │   {                     │                 │
  │                 │     registration_token,  │ Creates record: │
  │                 │     resource_id,         │  PK=TENANT#def  │
  │                 │     hostname,            │  SK=AGENT#i-xxx │
  │                 │     os_info,             │                 │
  │                 │     agent_version        │ Returns:        │
  │                 │   }                      │  - agent_id     │
  │                 │                          │  - agent_api_key│
  │ Saves to        │◀──────────────────────── │  - metrics_url  │
  │ state.json:     │   {agent_id, api_key}   │                 │
  │  agent_id       │                         │                 │
  │  api_key        │                         │                 │
  └─────────────────┘                         └─────────────────┘
```

### What happens:

1. Agent reads `/opt/cmp-agent/config.json` (contains endpoint, registration_token, resource_id)
2. Sends `POST /api/v1/agent/register` with the token + VM metadata (hostname, OS, IP)
3. Backend validates the token (must be unused, not expired — 1 hour TTL)
4. Backend creates an agent record in DynamoDB (`AGENT#<resource_id>`)
5. Backend returns a long-lived `agent_api_key` (SHA-256 hashed before storage)
6. Agent saves the API key to `/opt/cmp-agent/state.json` — never needs the registration token again

### DynamoDB Record Created:

```
PK: TENANT#default
SK: AGENT#i-0057f4823e132b59a
agent_id: agent-a1b2c3d4e5f6
api_key_hash: <sha256 of the api_key>
status: active
registered_at: 2026-06-30T14:15:00Z
hostname: ip-172-31-39-138.ec2.internal
os_info: Amazon Linux 2023
```

---

## Step 2: Metrics Push (Every 60 Seconds, Continuously)

After registration, the agent enters an infinite loop — collect metrics, push to CMP, sleep 60s.

```
  VM (agent.py loop)                           CMP Backend
  ┌─────────────────┐                         ┌─────────────────┐
  │ Collects:       │                         │                 │
  │  psutil.cpu_%   │   POST /agent/metrics   │ Validates       │
  │  psutil.memory  │ ───────────────────────▶│ X-Agent-API-Key │
  │  psutil.disk    │   Header:               │ header          │
  │  psutil.net     │    X-Agent-API-Key: xxx │                 │
  │  /etc/os-release│                         │ Verifies key    │
  │  psutil.procs   │   Body: {               │ matches this    │
  │                 │     agent_id,            │ resource_id     │
  │                 │     resource_id,         │                 │
  │                 │     timestamp,           │ Stores in       │
  │                 │     cpu: {...},          │ DynamoDB:       │
  │                 │     memory: {...},       │  PK=TENANT#def  │
  │                 │     disks: [...],        │  SK=AGENT_METRIC│
  │                 │     network: [...],      │    #i-xxx       │
  │                 │     os_info: {...},      │    #<timestamp> │
  │                 │     processes: [...]     │                 │
  │                 │   }                      │ Updates         │
  │                 │                          │ last_seen_at    │
  │                 │◀─── 202 Accepted ────────│                 │
  │                 │                         │ TTL: 24 hours   │
  │ sleep(60)       │                         │ (auto-expires)  │
  └─────────────────┘                         └─────────────────┘
```

### What the agent collects each cycle:

| Data | Source | Example |
|------|--------|---------|
| CPU usage % | `psutil.cpu_percent(interval=1)` | 23.5% |
| CPU cores | `psutil.cpu_count()` | 4 |
| Load averages | `os.getloadavg()` | 0.82, 0.67, 0.54 |
| Memory total/used/available | `psutil.virtual_memory()` | 8GB total, 3.2GB used |
| Swap usage | `psutil.swap_memory()` | 2GB total, 100MB used |
| Disk volumes | `psutil.disk_partitions()` + `disk_usage()` | /dev/xvda1 mounted at / — 40% used |
| Network interfaces | `psutil.net_if_addrs()` + `net_io_counters()` | eth0: 10.0.1.42, 5GB received |
| OS info | `/etc/os-release` + `platform` module | Ubuntu 22.04, kernel 5.15.0 |
| Uptime | `psutil.boot_time()` | 4 days |
| Top processes | `psutil.process_iter()` sorted by CPU | python3 (15% CPU), postgres (12% mem) |

### DynamoDB Record Created (per push):

```
PK: TENANT#default
SK: AGENT_METRIC#i-0057f4823e132b59a#2026-06-30T14:15:00Z
resource_id: i-0057f4823e132b59a
timestamp: 2026-06-30T14:15:00Z
cpu: {usage_percent: 23.5, core_count: 4, load_avg_1m: 0.82, ...}
memory: {total_bytes: 8589934592, used_bytes: 3435973837, usage_percent: 40.0, ...}
disks: [{device: "/dev/xvda1", mount_point: "/", usage_percent: 40.0, ...}]
network_interfaces: [{name: "eth0", ip_address: "10.0.1.42", ...}]
os_info: {os_type: "linux", distro: "Amazon Linux 2023", kernel: "6.1.0", ...}
processes: [{pid: 1234, name: "python3", cpu_percent: 15.3, ...}]
ttl: 1719842100  (24 hours from now — auto-deleted by DynamoDB)
```

### Authentication:

- Agent sends `X-Agent-API-Key` header with every metrics push
- Backend hashes the key (SHA-256) and looks up the matching agent record
- Verifies the agent's `resource_id` matches the payload's `resource_id`
- If key is invalid or revoked → returns 401 → agent exits

---

## Step 3: Frontend Display (User Opens System Metrics Tab)

The **System Metrics** tab is only shown for VM-type resources (EC2, VM, virtual machine, compute instance, GCE). When a user navigates to a VM resource's detail page and clicks "System Metrics":

```
  Browser (React)                              CMP Backend
  ┌─────────────────┐                         ┌─────────────────┐
  │ ResourceDetail  │                         │                 │
  │ → System Metrics│  GET /agent/{id}/metrics│ Queries DynamoDB│
  │   tab clicked   │ ───────────────────────▶│                 │
  │                 │   Header:               │ 1. Get agent    │
  │                 │    Authorization: JWT    │    record       │
  │                 │                         │    (AGENT#i-xxx)│
  │                 │                         │                 │
  │                 │                         │ 2. Get latest   │
  │                 │                         │    metric       │
  │                 │                         │    (newest SK)  │
  │                 │                         │                 │
  │ Renders:        │◀─── {                   │ 3. Calculate    │
  │  - Agent status │      agent_status: {    │    is_connected │
  │  - CPU gauge    │        is_connected,    │    from         │
  │  - Memory bar   │        last_seen_at,    │    last_seen_at │
  │  - Disk table   │        hostname...},    │                 │
  │  - Network cards│      cpu: {...},        │                 │
  │  - OS info      │      memory: {...},     │                 │
  │  - Processes    │      disks: [...],      │                 │
  │                 │      network: [...],    │                 │
  │ Auto-refreshes  │      os_info: {...},    │                 │
  │ every 60s       │      processes: [...]   │                 │
  │                 │    }                    │                 │
  └─────────────────┘                         └─────────────────┘
```

### Frontend queries:

| API Call | Purpose | Frequency |
|----------|---------|-----------|
| `GET /agent/{resource_id}/metrics` | Latest metrics + agent status | Every 60s (auto-refresh) |
| `GET /agent/{resource_id}/history?limit=30` | Last 30 data points for trends | Every 60s |

### How "Connected" status is determined:

```python
is_connected = (now - last_seen_at) < 900 seconds  # 3 missed reports = disconnected
```

- Agent pushes every 300s (5 minutes) → `last_seen_at` updates on every push
- If `last_seen_at` is less than 15 minutes old → **green "Agent Connected"**
- If `last_seen_at` is older than 15 minutes → **red "Agent Disconnected"**

---

## The Key Link: resource_id

The entire system is connected by `resource_id` (e.g., the EC2 instance ID `i-0057f4823e132b59a`):

| Where | How it's used |
|-------|---------------|
| `config.json` on VM | Agent knows which resource it represents |
| Registration POST | Agent tells CMP "I am resource X" |
| Agent record in DynamoDB | `SK=AGENT#i-0057f4823e132b59a` |
| Every metrics payload | `resource_id` field identifies the source |
| Metrics stored in DynamoDB | `SK=AGENT_METRIC#i-0057f4823e132b59a#<timestamp>` |
| Frontend resource detail page | Calls `GET /agent/i-0057f4823e132b59a/metrics` |
| CMP resource list | Same instance ID shown in cloud resources |

This is how the System Metrics tab knows which metrics belong to which resource — they share the same cloud resource identifier.

---

## Timeline After Installation

```
T+0s    : systemd starts cmp-agent service
T+1s    : Agent reads config.json
T+1s    : Agent calls POST /agent/register with token
T+2s    : Backend validates token, creates agent record, returns api_key
T+2s    : Agent saves api_key to state.json
T+3s    : Agent collects first metrics (psutil calls take ~1s for CPU)
T+4s    : Agent calls POST /agent/metrics → 202 Accepted
T+4s    : ✅ First data point is now in DynamoDB
T+4s    : ✅ System Metrics tab will show data if opened now
T+304s  : Second metrics push
T+604s  : Third metrics push
...continues every 300 seconds (5 minutes) indefinitely...
```

**From install to visible data: ~5 seconds** (assuming CMP backend is deployed and reachable).

---

## Data Retention

- Metrics are stored with a DynamoDB **TTL of 24 hours**
- After 24 hours, DynamoDB automatically deletes old data points
- The frontend history view shows the last 30 data points (most recent 2.5 hours)
- Agent registration records persist indefinitely (no TTL)

---

## What Happens On Failure

| Scenario | Agent Behavior | User Sees |
|----------|---------------|-----------|
| CMP unreachable (network) | Logs warning, retries next cycle (300s) | "Agent Disconnected" after 15 min |
| API key revoked by admin | Agent exits with error, systemd restarts in 30s, fails again | "Agent Disconnected" |
| VM rebooted | systemd auto-starts agent, loads state.json, resumes pushing | Brief disconnect, auto-recovers |
| VM stopped | Agent stops with VM | "Agent Disconnected" |
| VM terminated | Agent gone, record auto-revoked by CMP | Agent record cleaned up automatically |
| DynamoDB throttled | Backend returns 500, agent retries next cycle | Metrics gap in history |
| Agent process crash | systemd restarts in 30s (Restart=always) | ~30s gap, auto-recovers |

---

## Security Summary

```
Registration token ──(one-time)──▶ exchanged for API key ──(long-lived)──▶ used for every push

Token:    cmp-reg-Ex95yhWQ...    (valid 1 hour, single use, stored as SHA-256 hash)
API Key:  cmp-agent-Kx9mP2...   (valid until revoked, stored as SHA-256 hash)
```

- Agent can **only** push metrics for its own `resource_id` (enforced by backend)
- No cross-tenant access (PK scoped to tenant)
- Admin can revoke at any time → agent immediately loses access
- Agent is auto-revoked when its resource is terminated/deleted/destroyed via a resource action
- All communication over HTTPS (TLS 1.2+)
- No secrets stored in plaintext on CMP backend (only hashes)

---

## DynamoDB Table Structure

Table: `agent_metrics`

| Record Type | PK | SK | Purpose |
|-------------|----|----|---------|
| Registration token | `TENANT#<tid>` | `AGENT_REG_TOKEN#<prefix>` | One-time tokens (TTL: 1 hour) |
| Agent record | `TENANT#<tid>` | `AGENT#<resource_id>` | Agent registration + status |
| Metrics data point | `TENANT#<tid>` | `AGENT_METRIC#<resource_id>#<timestamp>` | Time-series metrics (TTL: 24h) |

### Query Patterns:

| Operation | Query |
|-----------|-------|
| Get agent for resource | `PK=TENANT#default, SK=AGENT#i-xxx` |
| Get latest metrics | `PK=TENANT#default, SK begins_with AGENT_METRIC#i-xxx#`, ScanIndexForward=False, Limit=1 |
| Get metrics history | `PK=TENANT#default, SK begins_with AGENT_METRIC#i-xxx#`, ScanIndexForward=False, Limit=30 |
| List all agents | `PK=TENANT#default, SK begins_with AGENT#` (filter out AGENT_METRIC and AGENT_REG_TOKEN) |

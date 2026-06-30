# CMP Agent — VM Monitoring

## Overview

The CMP Agent is a lightweight monitoring service that runs on provisioned VMs (EC2, Azure VMs, GCP VMs) and reports real-time system metrics back to CMP. These metrics are displayed on the **Resource Detail → System Metrics** tab.

## What It Collects

| Metric | Details |
|--------|---------|
| CPU | Usage %, core count, load averages (1m/5m/15m) |
| Memory | Total, used, available, usage %, swap |
| Disk Volumes | Per-mount: device, filesystem, total/used/free, usage % |
| Network | Per-interface: name, IP, MAC, bytes sent/received, status |
| OS Info | Distro, kernel, architecture, hostname, uptime |
| Top Processes | Top 10 by CPU/memory: PID, name, CPU%, memory%, status |

## How It Works

```
┌─────────────────┐     HTTPS POST      ┌───────────────┐
│   VM (Agent)    │ ──── every 60s ────▶ │   CMP Backend │
│                 │                      │               │
│ Collects:       │                      │ Stores in     │
│ - CPU, Memory   │     ◀── API Key ──── │ DynamoDB      │
│ - Disk, Network │                      │ (24h TTL)     │
│ - OS, Processes │                      │               │
└─────────────────┘                      └───────┬───────┘
                                                  │
                                         ┌───────▼───────┐
                                         │   Frontend    │
                                         │ System Metrics│
                                         │     Tab       │
                                         └───────────────┘
```

## Installation (For Task Authors)

### Automatic via User Task

When you write a provisioning task, CMP automatically provides agent installation details in the task context:

```python
# Available in every task execution:
cmp["agent"]["token"]            # One-time registration token (valid 1 hour)
cmp["agent"]["endpoint"]         # CMP agent API URL
cmp["agent"]["install_url"]      # URL to the install shell script
cmp["agent"]["resource_id"]      # Resource identifier
cmp["agent"]["report_interval"]  # Reporting interval in seconds (60)
```

### AWS EC2 Provisioning Task (Python)

```python
import json
import boto3

agent = cmp.get("agent", {})

# Build user_data with agent installation
user_data = f"""#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd

# Install CMP Agent
curl -sSL {agent.get('install_url', '')} | bash -s -- \\
  --endpoint {agent.get('endpoint', '')} \\
  --token {agent.get('token', '')} \\
  --resource-id {{INSTANCE_ID}} \\
  --tenant-id {cmp['execution'].get('tenant_id', 'default')}
"""

ec2 = boto3.client("ec2", region_name=cmp["credential"]["region"],
    aws_access_key_id=cmp["credential"]["aws_access_key_id"],
    aws_secret_access_key=cmp["credential"]["aws_secret_access_key"],
    aws_session_token=cmp["credential"]["aws_session_token"])

response = ec2.run_instances(
    ImageId=params["ami_id"],
    InstanceType=params.get("instance_type", "t3.micro"),
    MinCount=1, MaxCount=1,
    UserData=user_data,
    TagSpecifications=[{"ResourceType": "instance", "Tags": [
        {"Key": "Name", "Value": params.get("instance_name", "cmp-vm")}
    ]}]
)

instance_id = response["Instances"][0]["InstanceId"]
print(json.dumps({"instance_id": instance_id, "resource_id": instance_id}))
```

### Azure VM Provisioning Task (Python)

```python
import json, base64
from azure.identity import ClientSecretCredential
from azure.mgmt.compute import ComputeManagementClient

agent = cmp.get("agent", {})
cred = cmp["credential"]

cloud_init = f"""#!/bin/bash
apt-get update -y && apt-get install -y nginx
curl -sSL {agent.get('install_url', '')} | bash -s -- \\
  --endpoint {agent.get('endpoint', '')} \\
  --token {agent.get('token', '')} \\
  --resource-id {params['vm_name']} \\
  --tenant-id {cmp['execution'].get('tenant_id', 'default')}
"""

credential = ClientSecretCredential(cred["azure_tenant_id"], cred["azure_client_id"], cred["azure_client_secret"])
compute = ComputeManagementClient(credential, cred["azure_subscription_id"])

poller = compute.virtual_machines.begin_create_or_update(
    params["resource_group"], params["vm_name"],
    {"location": params.get("location", "eastus"),
     "hardware_profile": {"vm_size": params.get("vm_size", "Standard_B2s")},
     "os_profile": {"computer_name": params["vm_name"],
                    "admin_username": "azureuser",
                    "custom_data": base64.b64encode(cloud_init.encode()).decode()},
     "storage_profile": {"image_reference": {"publisher": "Canonical",
         "offer": "0001-com-ubuntu-server-jammy", "sku": "22_04-lts-gen2", "version": "latest"}}}
)
vm = poller.result()
print(json.dumps({"instance_id": vm.vm_id, "resource_id": params["vm_name"]}))
```

### GCP Compute Engine Provisioning Task (Python)

```python
import json
from google.oauth2 import service_account
from googleapiclient.discovery import build

agent = cmp.get("agent", {})
cred = cmp["credential"]

startup_script = f"""#!/bin/bash
apt-get update -y && apt-get install -y nginx
curl -sSL {agent.get('install_url', '')} | bash -s -- \\
  --endpoint {agent.get('endpoint', '')} \\
  --token {agent.get('token', '')} \\
  --resource-id {params['instance_name']} \\
  --tenant-id {cmp['execution'].get('tenant_id', 'default')}
"""

sa_info = json.loads(cred.get("secret_key", "{}"))
credentials = service_account.Credentials.from_service_account_info(sa_info,
    scopes=["https://www.googleapis.com/auth/compute"])
compute = build("compute", "v1", credentials=credentials)

config = {
    "name": params["instance_name"],
    "machineType": f"zones/{params['zone']}/machineTypes/{params.get('machine_type', 'e2-micro')}",
    "disks": [{"boot": True, "autoDelete": True,
        "initializeParams": {"sourceImage": "projects/ubuntu-os-cloud/global/images/family/ubuntu-2204-lts"}}],
    "networkInterfaces": [{"network": "global/networks/default", "accessConfigs": [{"type": "ONE_TO_ONE_NAT"}]}],
    "metadata": {"items": [{"key": "startup-script", "value": startup_script}]}
}

compute.instances().insert(project=cred["project_id"], zone=params["zone"], body=config).execute()
print(json.dumps({"instance_id": params["instance_name"], "resource_id": params["instance_name"]}))
```

## Install Agent from Resource Detail Page

The easiest way to install the agent on an existing VM is directly from the Resource Detail page:

1. Navigate to the resource's detail page (e.g., an EC2 instance, Azure VM, or GCP VM)
2. Click the **Install Agent** action button
3. A confirmation dialog ("Install CMP Agent") will appear
4. Click **Install Agent** to confirm

### Automatic Installation (Default)

CMP now attempts **automatic agent installation** using cloud provider management APIs:

| Provider | Mechanism |
|----------|-----------|
| AWS | SSM (Systems Manager) Run Command |
| Azure | VM Run Command (Azure Compute) |
| GCP | Metadata startup-script / OS Config |

When automatic installation succeeds:
- The agent is installed and started on the VM without any SSH access required
- CMP handles token generation, script delivery, and execution transparently
- The result is returned immediately with installation status

### Fallback to Manual Installation

If automatic installation is not possible (e.g., SSM agent not running on the VM, network restrictions, or unsupported OS), CMP falls back to providing a manual install command. In this case:

1. An **Install CMP Agent** modal appears with:
   - The full install command displayed in a dark code block (green text on dark background)
   - A **Copy** button (top-right of the command block) to copy the command to your clipboard
   - A note reminding you the token expires in 1 hour
   - Instructions to SSH into your VM, paste the command, and wait for the agent to start reporting metrics every 60 seconds
2. SSH into the VM and paste the copied command
3. Close the modal when done

### After Installation

Once installed (via either method), the agent automatically registers with CMP, begins collecting system metrics (CPU, memory, disk, network), and reports them every 60 seconds. View the metrics on the **System Metrics** tab of the resource detail page.

### Prerequisites for Automatic Installation

| Provider | Requirement |
|----------|-------------|
| AWS | SSM Agent running on the instance + IAM permissions for `ssm:SendCommand` on the linked credential |
| Azure | VM Agent (waagent) running + `Microsoft.Compute/virtualMachines/runCommand` permission |
| GCP | OS Config agent or Compute API access on the linked service account |

## Manual Installation (Existing VMs)

For VMs already running that weren't provisioned with the agent, you can also use the API directly:

1. Go to **Admin → Agent Management** or use the API:
   ```
   POST /api/v1/agent/generate-token
   Body: { "resource_id": "i-0abc123def456" }
   ```

2. SSH into the VM and run:
   ```bash
   curl -sSL https://<your-cmp>/api/v1/agent/install.sh | bash -s -- \
     --endpoint https://<your-cmp>/api/v1/agent \
     --token <registration-token> \
     --resource-id <instance-id>
   ```

## Agent Service Management

On the VM, the agent runs as a systemd service:

```bash
# Check status
systemctl status cmp-agent

# View logs
journalctl -u cmp-agent -f

# Restart
systemctl restart cmp-agent

# Stop (temporarily)
systemctl stop cmp-agent

# Uninstall
systemctl stop cmp-agent
systemctl disable cmp-agent
rm -rf /opt/cmp-agent /etc/systemd/system/cmp-agent.service
systemctl daemon-reload
```

## Agent Lifecycle & Cleanup

When a resource is terminated, deleted, or destroyed (via a resource action), CMP **automatically revokes** the associated agent. This ensures that:

- No orphaned agent records remain in DynamoDB after a resource is gone
- The agent's API key is invalidated immediately upon resource destruction
- No manual cleanup is needed after terminating VMs

This applies to all destructive resource actions (`terminate`, `delete`, `destroy`) regardless of how they are triggered — manual UI actions, scheduled lease expiry, or API calls.

If no agent was registered for the resource, the cleanup is silently skipped.

## Security

| Aspect | Implementation |
|--------|---------------|
| Registration | One-time token, valid 1 hour, single use |
| Ongoing auth | Scoped API key (SHA-256 hashed in DB) |
| Scope | Agent can only report metrics for its own resource_id |
| Transport | HTTPS only |
| Revocation | Admin can revoke via UI or API (`POST /agent/{resource_id}/revoke`) |
| Auto-revocation | Agent is automatically revoked when its resource is terminated/destroyed |
| Data retention | Metrics auto-expire after 24 hours (DynamoDB TTL) |

## API Reference

### Agent-Facing (no user JWT needed)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/v1/agent/register` | Registration token | Exchange token for API key |
| POST | `/api/v1/agent/metrics` | X-Agent-API-Key header | Push metrics |
| GET | `/api/v1/agent/install.sh` | None | Download install script |

### User-Facing (JWT required)

| Method | Endpoint | Role | Description |
|--------|----------|------|-------------|
| GET | `/api/v1/agent/{resource_id}/metrics` | Any user | Latest metrics |
| GET | `/api/v1/agent/{resource_id}/history` | Any user | Metrics history |
| GET | `/api/v1/agent/{resource_id}/status` | Any user | Agent status |
| POST | `/api/v1/agent/{resource_id}/revoke` | Admin | Revoke agent |
| GET | `/api/v1/agent/list` | Admin | List all agents |
| POST | `/api/v1/agent/generate-token` | Admin/Developer | Manual token generation |

## Viewing Metrics

1. Navigate to any resource's detail page
2. Click the **System Metrics** tab
3. If the agent is installed and connected, you'll see:
   - Connection status (green = connected, red = disconnected)
   - OS information (distro, kernel, architecture, uptime)
   - CPU usage with load averages
   - Memory usage (RAM + swap)
   - Disk volumes table (all mounted volumes with usage)
   - Network interfaces with traffic stats
   - Top 10 processes by resource consumption
   - Historical metrics table (last 30 data points)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Agent not connecting | Check `systemctl status cmp-agent`, ensure CMP URL is reachable |
| "Invalid or expired token" | Generate a new token (tokens expire in 1 hour) |
| Metrics not showing | Wait 60s after install, check agent logs: `journalctl -u cmp-agent` |
| Agent shows "Disconnected" | Agent hasn't reported in 3+ minutes — check VM network/agent service |
| Permission denied | Ensure the install script ran as root (`sudo`) |

## Requirements

**On the VM:**
- Python 3.6+ (auto-installed by the install script if missing)
- `psutil` and `requests` Python packages (auto-installed)
- Outbound HTTPS access to CMP backend
- systemd (for service management)

**On CMP:**
- `agent_metrics` DynamoDB table (auto-created on startup)
- Network accessible from VM (public URL or VPN)

# CMP Agent — Debug & Troubleshooting Guide

## Quick Health Check

SSH into the VM and run:

```bash
sudo systemctl status cmp-agent
```

| Output | Meaning |
|--------|---------|
| `Active: active (running)` | Agent is running normally |
| `Active: activating (auto-restart)` | Agent is crash-looping — check logs |
| `Unit cmp-agent.service could not be found` | Agent is not installed |
| `Active: inactive (dead)` | Agent is stopped — needs restart |

---

## Step-by-Step Diagnosis

### 1. Is the agent installed?

```bash
ls -la /opt/cmp-agent/
```

**Expected output:**
```
-rw-r--r-- 1 root root  ... agent.py
-rw-r--r-- 1 root root  ... config.json
-rw-r--r-- 1 root root  ... state.json   (only after successful registration)
```

If `/opt/cmp-agent/` doesn't exist → agent was never installed. Go to [Manual Installation](#manual-installation).

---

### 2. Is the service running?

```bash
sudo systemctl status cmp-agent --no-pager
```

If stopped:
```bash
sudo systemctl start cmp-agent
```

If not enabled (won't start on boot):
```bash
sudo systemctl enable cmp-agent
```

---

### 3. Check agent logs

```bash
# Last 50 log lines
sudo journalctl -u cmp-agent --no-pager -n 50

# Follow logs in real-time
sudo journalctl -u cmp-agent -f

# Logs from last 5 minutes
sudo journalctl -u cmp-agent --since "5 min ago" --no-pager
```

---

### 4. Common Errors & Fixes

#### `SyntaxError: unexpected character after line continuation character`

**Cause:** The agent Python script has corrupted escape characters (known issue with early install script versions).

**Fix:** Re-download the agent script from CMP:
```bash
sudo curl -sSL https://<your-cmp-url>/api/v1/agent/install.sh | sudo bash -s -- \
  --endpoint https://<your-cmp-url>/api/v1/agent \
  --token <new-token> \
  --resource-id <instance-id>
```
Or generate a new token from CMP UI: Resource Detail → Install Agent button.

---

#### `Registration failed: 404 {"detail":"Not Found"}`

**Cause:** The CMP backend doesn't have the agent endpoints deployed yet, or the endpoint URL is wrong.

**Fix:**
1. Verify the endpoint in config:
   ```bash
   cat /opt/cmp-agent/config.json
   ```
2. Test connectivity:
   ```bash
   curl -s https://<your-cmp-url>/api/v1/agent/install.sh | head -5
   ```
   If this returns HTML or 404, the agent router isn't deployed on the backend.
3. Redeploy the CMP backend with agent endpoints, then restart:
   ```bash
   sudo systemctl restart cmp-agent
   ```

---

#### `Registration failed: 401 {"detail":"Invalid or expired registration token"}`

**Cause:** The one-time registration token has expired (valid for 1 hour) or was already used.

**Fix:** Generate a new token:
1. From CMP UI: go to the resource → click "Install Agent" → get new command
2. Or via API:
   ```bash
   curl -X POST "https://<cmp-url>/api/v1/agent/generate-token?resource_id=<instance-id>" \
     -H "Authorization: Bearer <admin-jwt>"
   ```
3. Update config on the VM:
   ```bash
   sudo python3 -c "
   import json
   with open('/opt/cmp-agent/config.json') as f:
       config = json.load(f)
   config['registration_token'] = '<new-token>'
   with open('/opt/cmp-agent/config.json', 'w') as f:
       json.dump(config, f, indent=4)
   "
   ```
4. Remove old state (forces re-registration):
   ```bash
   sudo rm -f /opt/cmp-agent/state.json
   sudo systemctl restart cmp-agent
   ```

---

#### `Connection refused` / `Connection timed out`

**Cause:** The VM cannot reach the CMP backend over the network.

**Fix:**
1. Test connectivity from the VM:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" https://<your-cmp-url>/api/v1/health
   ```
   - `200` = reachable
   - `000` = can't connect (DNS, firewall, or network issue)

2. Check DNS resolution:
   ```bash
   nslookup <your-cmp-url>
   ```

3. Check outbound HTTPS:
   ```bash
   curl -v https://<your-cmp-url> 2>&1 | grep -i "connected\|refused\|timed out"
   ```

4. If behind a VPC with no internet access, you need a NAT Gateway or VPC endpoint.

---

#### `ModuleNotFoundError: No module named 'psutil'`

**Cause:** Python dependencies not installed. On newer distros (Ubuntu 23.04+, Debian 12+), pip may refuse system-wide installs due to PEP 668 (externally-managed-environment).

**Fix:**
```bash
# Amazon Linux 2023 / RHEL / CentOS
sudo dnf install -y python3-pip
sudo python3 -m pip install psutil requests

# Ubuntu 23.04+ / Debian 12+ (PEP 668 systems)
sudo apt-get install -y python3-pip
sudo python3 -m pip install --break-system-packages psutil requests

# Older Ubuntu / Debian (< 23.04)
sudo apt-get install -y python3-pip
sudo python3 -m pip install psutil requests

# Then restart
sudo systemctl restart cmp-agent
```

> **Note:** The CMP install script automatically handles PEP 668 by attempting `--break-system-packages` first, then falling back to the standard pip install for older systems.

---

#### `Permission denied: '/opt/cmp-agent/state.json'`

**Cause:** Agent service running as wrong user or file permissions issue.

**Fix:**
```bash
sudo chown -R root:root /opt/cmp-agent/
sudo chmod 644 /opt/cmp-agent/*.json
sudo chmod 644 /opt/cmp-agent/agent.py
sudo systemctl restart cmp-agent
```

---

#### Agent reports `API key rejected — agent may have been revoked`

**Cause:** An admin revoked the agent's API key from CMP.

**Fix:**
1. Remove state to force re-registration:
   ```bash
   sudo rm -f /opt/cmp-agent/state.json
   ```
2. Generate a new registration token (see "expired token" fix above)
3. Update config with new token
4. Restart:
   ```bash
   sudo systemctl restart cmp-agent
   ```

---

### 5. Verify agent is pushing metrics

After fixing issues and restarting, verify the agent is actively reporting:

```bash
# Check last few log entries — look for "Metrics pushed successfully"
sudo journalctl -u cmp-agent --no-pager -n 10 --since "2 min ago"
```

**Expected output:**
```
CMP Agent v1.0.0 starting
Registered successfully as agent-xxxxxxxxxxxx
Reporting every 60s to https://<cmp-url>/api/v1/agent/metrics
Metrics pushed successfully
```

From the CMP side:
```bash
curl -s "https://<cmp-url>/api/v1/agent/<resource-id>/status" \
  -H "Authorization: Bearer <jwt>" | python3 -m json.tool
```

Look for `"is_connected": true` and a recent `"last_seen_at"` timestamp.

---

## Manual Installation

If the agent was never installed, or you need to reinstall from scratch:

### Option A: Via CMP UI (recommended)

1. Go to the resource's detail page in CMP
2. Click the **Install Agent** button (purple)
3. If auto-install fails, a modal shows the full command
4. SSH into the VM and run the command with `sudo`

### Option B: Via API

```bash
# 1. Generate a token (requires admin/developer JWT)
TOKEN=$(curl -s -X POST "https://<cmp-url>/api/v1/agent/generate-token?resource_id=<instance-id>" \
  -H "Authorization: Bearer <jwt>" | python3 -c "import sys,json; print(json.load(sys.stdin)['registration_token'])")

# 2. Run on the VM (as root)
sudo bash -c "curl -sSL https://<cmp-url>/api/v1/agent/install.sh | bash -s -- \
  --endpoint https://<cmp-url>/api/v1/agent \
  --token $TOKEN \
  --resource-id <instance-id> \
  --tenant-id default"
```

### Option C: Fully manual (air-gapped / no curl)

If the VM has no internet access to reach CMP:

1. Download the agent script on your local machine:
   ```bash
   curl -o agent.py "https://<cmp-url>/api/v1/agent/install.sh"
   # Or copy from: backend/app/api/v1/endpoints/agent.py (_get_agent_python_code function)
   ```

2. Copy files to the VM (via scp, bastion, etc.):
   ```bash
   scp agent.py config.json user@<vm-ip>:/tmp/
   ```

3. On the VM:
   ```bash
   sudo mkdir -p /opt/cmp-agent
   sudo cp /tmp/agent.py /opt/cmp-agent/
   sudo cp /tmp/config.json /opt/cmp-agent/

   # Create systemd service
   sudo cat > /etc/systemd/system/cmp-agent.service << 'EOF'
   [Unit]
   Description=CMP Monitoring Agent
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/cmp-agent/agent.py
   Restart=always
   RestartSec=30
   Environment=CMP_AGENT_CONFIG=/opt/cmp-agent/config.json

   [Install]
   WantedBy=multi-user.target
   EOF

   sudo systemctl daemon-reload
   sudo systemctl enable --now cmp-agent
   ```

---

## Uninstalling the Agent

```bash
sudo systemctl stop cmp-agent
sudo systemctl disable cmp-agent
sudo rm -rf /opt/cmp-agent
sudo rm -f /etc/systemd/system/cmp-agent.service
sudo systemctl daemon-reload
```

---

## Windows Troubleshooting

### Check if the Scheduled Task is running

```powershell
Get-ScheduledTask -TaskName CMPAgent | Select-Object State
```

| State | Meaning |
|-------|---------|
| `Running` | Agent is active and collecting metrics |
| `Ready` | Task registered but not currently running — may need manual start |
| Not found | Agent was never installed |

### Python download failed during installation

**Symptom:** Agent install log shows:
```
[CMP Agent] ERROR: Failed to download Python after 3 attempts
```

**Cause:** The Windows bootstrap script downloads Python from `python.org` during first boot. If the network isn't ready yet (common on freshly provisioned VMs), the download can fail. The installer retries up to 3 times with a 10-second delay between attempts and a 60-second timeout per attempt. If all 3 attempts fail, the install aborts.

**Fix:**
1. Check network connectivity from the VM:
   ```powershell
   Test-NetConnection -ComputerName www.python.org -Port 443
   ```
2. If the network is now available, re-run the agent install script (the retry logic will handle transient issues).
3. If the VM is behind a restricted firewall, ensure outbound HTTPS access to `python.org` is allowed, or pre-install Python before provisioning.

---

### Agent not starting as a Scheduled Task

**Cause:** The SYSTEM user running the task cannot find `python.exe` because PATH is not inherited from the user session.

**Fix:** The installer now resolves the absolute path to `python.exe` and searches common install locations. If the task still won't start:

1. Verify Python is installed:
   ```powershell
   Test-Path 'C:\Program Files\Python312\python.exe'
   ```

2. Check the task action path:
   ```powershell
   (Get-ScheduledTask -TaskName CMPAgent).Actions | Select-Object Execute, Arguments
   ```

3. If the path is wrong, reinstall the agent or update the task action:
   ```powershell
   $pythonPath = 'C:\Program Files\Python312\python.exe'
   $agentDir = 'C:\ProgramData\CMP-Agent'
   $action = New-ScheduledTaskAction -Execute $pythonPath -Argument "$agentDir\agent.py" -WorkingDirectory $agentDir
   Set-ScheduledTask -TaskName CMPAgent -Action $action
   Start-ScheduledTask -TaskName CMPAgent
   ```

### Task registered but agent process not running

**Cause:** The scheduled task may enter `Ready` state without actually launching the process (common on some Windows Server configurations with restrictive policies).

**Fix:** The installer now includes a fallback that detects this and starts the agent directly. If you need to manually start:

```powershell
$pythonPath = 'C:\Program Files\Python312\python.exe'
$agentDir = 'C:\ProgramData\CMP-Agent'
Start-Process -WindowStyle Hidden -FilePath $pythonPath -ArgumentList "$agentDir\agent.py" -WorkingDirectory $agentDir
```

### Check agent logs (Windows)

The agent writes to stdout which is captured by Task Scheduler's history:

```powershell
# View recent task history events
Get-WinEvent -LogName 'Microsoft-Windows-TaskScheduler/Operational' -MaxEvents 20 |
  Where-Object { $_.Message -match 'CMPAgent' } |
  Format-Table TimeCreated, Message -Wrap
```

### Windows file locations

| File | Purpose |
|------|---------|
| `C:\ProgramData\CMP-Agent\agent.py` | Agent Python script |
| `C:\ProgramData\CMP-Agent\config.json` | Configuration (endpoint, token, resource_id) |
| `C:\ProgramData\CMP-Agent\state.json` | Runtime state (created after registration) |

### Uninstalling (Windows)

```powershell
Unregister-ScheduledTask -TaskName CMPAgent -Confirm:$false
Remove-Item -Recurse -Force 'C:\ProgramData\CMP-Agent'
```

---

## File Locations

| File | Purpose |
|------|---------|
| `/opt/cmp-agent/agent.py` | Agent Python script |
| `/opt/cmp-agent/config.json` | Configuration (endpoint, token, resource_id) |
| `/opt/cmp-agent/state.json` | Runtime state (agent_id, api_key — created after registration) |
| `/etc/systemd/system/cmp-agent.service` | Systemd service unit |

---

## Config File Reference

`/opt/cmp-agent/config.json`:
```json
{
    "endpoint": "https://<cmp-url>/api/v1/agent",
    "registration_token": "cmp-reg-...",
    "resource_id": "i-0abc123def456",
    "tenant_id": "default",
    "report_interval": 60
}
```

`/opt/cmp-agent/state.json` (auto-created after registration):
```json
{
    "agent_id": "agent-xxxxxxxxxxxx",
    "api_key": "cmp-agent-...",
    "metrics_endpoint": "https://<cmp-url>/api/v1/agent/metrics",
    "report_interval": 60,
    "registered": true
}
```

---

## Requirements

| Requirement | Notes |
|-------------|-------|
| Python 3.6+ | Pre-installed on Amazon Linux 2/2023, Ubuntu 20.04+ |
| pip | May need manual install on Amazon Linux 2023: `sudo dnf install python3-pip` |
| psutil | Installed by the install script: `pip install psutil` (uses `--break-system-packages` on PEP 668 systems) |
| requests | Installed by the install script: `pip install requests` (uses `--break-system-packages` on PEP 668 systems) |
| systemd | Required for service management (all modern Linux distros) |
| Outbound HTTPS | VM must be able to reach the CMP backend URL on port 443 |
| Root access | Required for installation (package install, /opt, systemd) |

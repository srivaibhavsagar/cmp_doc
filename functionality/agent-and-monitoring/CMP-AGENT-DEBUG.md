# CMP Agent — Debug & Troubleshooting Guide

This guide covers both **Linux** (systemd service) and **Windows** (Scheduled Task) agent deployments.

Jump to:
- [Linux Troubleshooting](#linux-troubleshooting)
- [Windows Troubleshooting](#windows-troubleshooting)
- [Manual Installation](#manual-installation)
- [File Locations Reference](#file-locations-reference)

---

## Linux Troubleshooting

### Quick Health Check

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

### Step-by-Step Diagnosis

#### 1. Is the agent installed?

```bash
ls -la /opt/cmp-agent/
```

Expected:
```
-rw-r--r-- 1 root root  ... agent.py
-rw-r--r-- 1 root root  ... config.json
-rw-r--r-- 1 root root  ... state.json   (only after successful registration)
```

If `/opt/cmp-agent/` doesn't exist → agent was never installed. Go to [Manual Installation — Linux](#linux-1).

---

#### 2. Is the service running?

```bash
sudo systemctl status cmp-agent --no-pager
```

If stopped:
```bash
sudo systemctl start cmp-agent
```

If not enabled (won't start on reboot):
```bash
sudo systemctl enable cmp-agent
```

---

#### 3. Check agent logs

```bash
# Last 50 lines
sudo journalctl -u cmp-agent --no-pager -n 50

# Follow in real-time
sudo journalctl -u cmp-agent -f

# Last 5 minutes
sudo journalctl -u cmp-agent --since "5 min ago" --no-pager
```

---

### Common Errors — Linux

#### `SyntaxError: unexpected character after line continuation character`

**Cause:** The agent script has corrupted escape characters (known issue with early install script versions).

**Fix:** Reinstall:
```bash
sudo curl -sSL https://<your-cmp-url>/api/v1/agent/install.sh | sudo bash -s -- \
  --endpoint https://<your-cmp-url>/api/v1/agent \
  --token <new-token> \
  --resource-id <instance-id>
```
Generate a new token: Resource Detail → Install Agent button.

---

#### `Registration failed: 404 {"detail":"Not Found"}`

**Cause:** The CMP backend agent endpoints aren't deployed, or the endpoint URL is wrong.

**Fix:**
```bash
# Check config
cat /opt/cmp-agent/config.json

# Test endpoint
curl -s https://<your-cmp-url>/api/v1/agent/install.sh | head -3
```
If this returns HTML or 404, the agent router isn't deployed. Redeploy the CMP backend then:
```bash
sudo systemctl restart cmp-agent
```

---

#### `Registration failed: 401 {"detail":"Invalid or expired registration token"}`

**Cause:** One-time token has expired (valid 1 hour) or was already used.

**Fix:**
1. Generate a new token from CMP UI: Resource Detail → Install Agent
2. Update config on the VM:
```bash
sudo python3 -c "
import json
with open('/opt/cmp-agent/config.json') as f:
    config = json.load(f)
config['registration_token'] = '<new-token>'
with open('/opt/cmp-agent/config.json', 'w') as f:
    json.dump(config, f, indent=4)
"
sudo rm -f /opt/cmp-agent/state.json
sudo systemctl restart cmp-agent
```

---

#### `Connection refused` / `Connection timed out`

**Cause:** VM cannot reach the CMP backend.

**Fix:**
```bash
# Test connectivity
curl -s -o /dev/null -w "%{http_code}" https://<your-cmp-url>/api/v1/health
# 200 = reachable, 000 = unreachable

# Check DNS
nslookup <your-cmp-url>

# Check outbound HTTPS
curl -v https://<your-cmp-url> 2>&1 | grep -i "connected\|refused\|timed out"
```
If behind a VPC with no internet access, you need a NAT gateway or internal endpoint.

---

#### `ModuleNotFoundError: No module named 'psutil'`

**Cause:** Python dependencies not installed. Ubuntu 23.04+ / Debian 12+ enforce PEP 668 (externally-managed-environment).

**Fix:**
```bash
# Amazon Linux 2023 / RHEL / CentOS
sudo dnf install -y python3-pip
sudo python3 -m pip install psutil requests

# Ubuntu 23.04+ / Debian 12+ (PEP 668)
sudo python3 -m pip install --break-system-packages psutil requests

# Older Ubuntu / Debian
sudo apt-get install -y python3-pip
sudo python3 -m pip install psutil requests

sudo systemctl restart cmp-agent
```

> The CMP install script automatically tries `--break-system-packages` first, then falls back for older systems.

---

#### `Permission denied: '/opt/cmp-agent/state.json'`

```bash
sudo chown -R root:root /opt/cmp-agent/
sudo chmod 644 /opt/cmp-agent/*.json /opt/cmp-agent/agent.py
sudo systemctl restart cmp-agent
```

---

#### `API key rejected — agent may have been revoked`

```bash
sudo rm -f /opt/cmp-agent/state.json
# Generate a new token from CMP UI, update config.json, then:
sudo systemctl restart cmp-agent
```

---

### Verify Metrics Are Being Pushed (Linux)

```bash
sudo journalctl -u cmp-agent --no-pager -n 10 --since "2 min ago"
```

Expected:
```
CMP Agent v1.0.0 starting
Registered successfully as agent-xxxxxxxxxxxx
Reporting every 60s to https://<cmp-url>/api/v1/agent/metrics
Metrics pushed successfully
```

From CMP side:
```bash
curl -s "https://<cmp-url>/api/v1/agent/<resource-id>/status" \
  -H "Authorization: Bearer <jwt>" | python3 -m json.tool
```
Look for `"is_connected": true` and a recent `"last_seen_at"`.

---

### Uninstall (Linux)

```bash
sudo systemctl stop cmp-agent
sudo systemctl disable cmp-agent
sudo rm -rf /opt/cmp-agent
sudo rm -f /etc/systemd/system/cmp-agent.service
sudo systemctl daemon-reload
```

---

## Windows Troubleshooting

The Windows agent runs as a **Scheduled Task** named `CMPAgent` under the SYSTEM account, powered by the Python script at `C:\ProgramData\CMP-Agent\agent.py`.

### Quick Health Check

```powershell
Get-ScheduledTask -TaskName CMPAgent | Select-Object State
```

| State | Meaning |
|-------|---------|
| `Running` | Agent is active and collecting metrics |
| `Ready` | Task registered but not running — start manually (see below) |
| Not found | Agent was never installed |

Check if the Python process is actually running:
```powershell
Get-Process -Name python* -ErrorAction SilentlyContinue | Select-Object Id, CPU, CommandLine
```

---

### Step-by-Step Diagnosis

#### 1. Is the agent installed?

```powershell
Test-Path 'C:\ProgramData\CMP-Agent\agent.py'
Test-Path 'C:\ProgramData\CMP-Agent\config.json'
```

If files don't exist → agent was never installed. Go to [Manual Installation — Windows](#windows-1).

---

#### 2. Is the task registered and running?

```powershell
Get-ScheduledTask -TaskName CMPAgent | Select-Object TaskName, State, Description
```

Start it manually if needed:
```powershell
Start-ScheduledTask -TaskName CMPAgent
Start-Sleep -Seconds 5
Get-ScheduledTask -TaskName CMPAgent | Select-Object State
```

If the task state stays `Ready` and won't transition to `Running` (common on restrictive Windows Server policies):
```powershell
$pythonPath = (Get-Command python -ErrorAction SilentlyContinue).Source
if (-not $pythonPath) { $pythonPath = 'C:\Program Files\Python312\python.exe' }
$agentDir = 'C:\ProgramData\CMP-Agent'
Start-Process -WindowStyle Hidden -FilePath $pythonPath -ArgumentList "$agentDir\agent.py" -WorkingDirectory $agentDir
```

---

#### 3. Check agent logs (Windows)

The agent logs to stdout, captured by Task Scheduler's operational log:

```powershell
# Recent task events
Get-WinEvent -LogName 'Microsoft-Windows-TaskScheduler/Operational' -MaxEvents 30 |
  Where-Object { $_.Message -match 'CMPAgent' } |
  Format-Table TimeCreated, Message -Wrap
```

If Task Scheduler history is disabled:
```powershell
# Enable history
wevtutil set-log 'Microsoft-Windows-TaskScheduler/Operational' /e:true
```

You can also check for a log file if the agent was started directly:
```powershell
Get-ChildItem 'C:\ProgramData\CMP-Agent\' | Select-Object Name, LastWriteTime
```

---

### Common Errors — Windows

#### Python not found during installation

**Symptom:** Install log shows:
```
[CMP Agent] ERROR: Failed to download Python after 3 attempts
```

**Cause:** The install script downloads Python from `python.org` on first boot. If the network isn't ready yet (common on freshly provisioned VMs), the download fails. The installer retries up to 3 times with 10-second delays.

**Fix:**
```powershell
# Test network reachability
Test-NetConnection -ComputerName www.python.org -Port 443

# If network is up, re-run the agent install script
# Generate a new token from CMP UI first
```

If the VM is behind a firewall that blocks `python.org`, pre-install Python before provisioning:
```powershell
# Download Python installer manually and run silently
$installer = "$env:TEMP\python-installer.exe"
Invoke-WebRequest -Uri 'https://www.python.org/ftp/python/3.12.0/python-3.12.0-amd64.exe' -OutFile $installer
Start-Process -Wait -FilePath $installer -ArgumentList '/quiet InstallAllUsers=1 PrependPath=1'
```

---

#### Task registered but agent not running

**Cause:** Scheduled task entered `Ready` state without launching (common on restrictive group policy).

**Diagnosis:**
```powershell
# Check the task action — is the Python path correct?
(Get-ScheduledTask -TaskName CMPAgent).Actions | Select-Object Execute, Arguments

# Does that Python executable exist?
$taskAction = (Get-ScheduledTask -TaskName CMPAgent).Actions[0]
Test-Path $taskAction.Execute
```

**Fix:** Update the task action with the correct Python path:
```powershell
$pythonPath = (Get-Command python -ErrorAction SilentlyContinue).Source
if (-not $pythonPath) {
    # Search common install locations
    $candidates = @(
        'C:\Program Files\Python312\python.exe',
        'C:\Program Files\Python311\python.exe',
        'C:\Program Files\Python310\python.exe',
        "$env:LOCALAPPDATA\Programs\Python\Python312\python.exe"
    )
    $pythonPath = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
}
$agentDir = 'C:\ProgramData\CMP-Agent'
$action = New-ScheduledTaskAction -Execute $pythonPath -Argument "$agentDir\agent.py" -WorkingDirectory $agentDir
Set-ScheduledTask -TaskName CMPAgent -Action $action
Start-ScheduledTask -TaskName CMPAgent
```

---

#### `Registration failed: 401 — Invalid or expired registration token`

**Cause:** Token expired (valid 1 hour) or already used.

**Fix:**
```powershell
# Update config with new token (generate from CMP UI: Resource Detail → Install Agent)
$configPath = 'C:\ProgramData\CMP-Agent\config.json'
$config = Get-Content $configPath | ConvertFrom-Json
$config.registration_token = '<new-token>'
$config | ConvertTo-Json | Set-Content $configPath

# Remove old state to force re-registration
Remove-Item 'C:\ProgramData\CMP-Agent\state.json' -ErrorAction SilentlyContinue

# Restart
Stop-ScheduledTask -TaskName CMPAgent -ErrorAction SilentlyContinue
Start-ScheduledTask -TaskName CMPAgent
```

---

#### `Connection refused` / `Connection timed out`

```powershell
# Test CMP backend reachability
$response = Invoke-WebRequest -Uri 'https://<your-cmp-url>/api/v1/health' -UseBasicParsing -ErrorAction SilentlyContinue
$response.StatusCode   # 200 = reachable

# Test DNS
Resolve-DnsName <your-cmp-url>

# Test port 443
Test-NetConnection -ComputerName <your-cmp-url> -Port 443
```

---

#### `ModuleNotFoundError: No module named 'psutil'`

```powershell
# Find Python used by the agent
$taskAction = (Get-ScheduledTask -TaskName CMPAgent).Actions[0]
$pythonPath = $taskAction.Execute

# Install missing packages
& $pythonPath -m pip install psutil requests

# Restart
Stop-ScheduledTask -TaskName CMPAgent -ErrorAction SilentlyContinue
Start-ScheduledTask -TaskName CMPAgent
```

---

### Verify Metrics Are Being Pushed (Windows)

```powershell
# Check Task Scheduler for recent successful runs
Get-WinEvent -LogName 'Microsoft-Windows-TaskScheduler/Operational' -MaxEvents 10 |
  Where-Object { $_.Message -match 'CMPAgent' -and $_.Id -eq 201 } |
  Format-Table TimeCreated, Message -Wrap
```

From CMP side:
```bash
curl -s "https://<cmp-url>/api/v1/agent/<resource-id>/status" \
  -H "Authorization: Bearer <jwt>" | python3 -m json.tool
```
Look for `"is_connected": true` and a recent `"last_seen_at"`.

---

### Uninstall (Windows)

```powershell
Stop-ScheduledTask -TaskName CMPAgent -ErrorAction SilentlyContinue
Unregister-ScheduledTask -TaskName CMPAgent -Confirm:$false
Remove-Item -Recurse -Force 'C:\ProgramData\CMP-Agent'
```

---

## Manual Installation

### Linux {#linux-1}

#### Option A: Via CMP UI (recommended)

1. Go to the resource's detail page in CMP
2. Click **Install Agent** (purple button)
3. If auto-install fails, a modal shows the full command with a copy button
4. SSH into the VM and paste the command

#### Option B: Via the install script directly

```bash
sudo bash -c "curl -sSL https://<cmp-url>/api/v1/agent/install.sh | bash -s -- \
  --endpoint https://<cmp-url>/api/v1/agent \
  --token <registration-token> \
  --resource-id <instance-id> \
  --tenant-id default"
```

#### Option C: Fully manual (air-gapped / no curl)

```bash
# 1. Create directory
sudo mkdir -p /opt/cmp-agent

# 2. Write config.json
sudo tee /opt/cmp-agent/config.json << 'EOF'
{
    "endpoint": "https://<cmp-url>/api/v1/agent",
    "registration_token": "<token>",
    "resource_id": "<instance-id>",
    "tenant_id": "default",
    "report_interval": 60
}
EOF

# 3. Copy agent.py from another machine or download manually
# scp agent.py root@<vm>:/opt/cmp-agent/agent.py

# 4. Create systemd service
sudo tee /etc/systemd/system/cmp-agent.service << 'EOF'
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

### Windows {#windows-1}

#### Option A: Via CMP UI (recommended)

1. Go to the resource's detail page in CMP
2. Click **Install Agent** (purple button)
3. The modal shows the PowerShell command — copy it
4. RDP into the VM, open PowerShell as Administrator, and paste

#### Option B: Via the install script directly

```powershell
# Run as Administrator
powershell -ExecutionPolicy Bypass -Command "
  Invoke-WebRequest -Uri 'https://<cmp-url>/api/v1/agent/install.ps1' -OutFile '$env:TEMP\cmp-install.ps1' -UseBasicParsing;
  & '$env:TEMP\cmp-install.ps1' -Endpoint 'https://<cmp-url>/api/v1/agent' -Token '<token>' -ResourceId '<instance-id>' -TenantId 'default'
"
```

#### Option C: Fully manual (air-gapped)

```powershell
# 1. Create directory
New-Item -ItemType Directory -Force -Path 'C:\ProgramData\CMP-Agent' | Out-Null

# 2. Write config.json
@{
    endpoint           = "https://<cmp-url>/api/v1/agent"
    registration_token = "<token>"
    resource_id        = "<instance-id>"
    tenant_id          = "default"
    report_interval    = 60
} | ConvertTo-Json | Set-Content 'C:\ProgramData\CMP-Agent\config.json'

# 3. Copy agent.py to C:\ProgramData\CMP-Agent\agent.py

# 4. Register scheduled task
$pythonPath = 'C:\Program Files\Python312\python.exe'  # adjust as needed
$agentDir   = 'C:\ProgramData\CMP-Agent'
$action   = New-ScheduledTaskAction -Execute $pythonPath -Argument "$agentDir\agent.py" -WorkingDirectory $agentDir
$trigger  = New-ScheduledTaskTrigger -AtStartup
$settings = New-ScheduledTaskSettingsSet -RestartInterval (New-TimeSpan -Minutes 1) -RestartCount 999 `
              -ExecutionTimeLimit (New-TimeSpan -Days 365) -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName CMPAgent -Action $action -Trigger $trigger -Settings $settings `
  -User 'SYSTEM' -RunLevel Highest -Force | Out-Null
Start-ScheduledTask -TaskName CMPAgent
```

---

## File Locations Reference

### Linux

| File | Purpose |
|------|---------|
| `/opt/cmp-agent/agent.py` | Agent Python script |
| `/opt/cmp-agent/config.json` | Configuration (endpoint, token, resource_id) |
| `/opt/cmp-agent/state.json` | Runtime state — created after successful registration |
| `/etc/systemd/system/cmp-agent.service` | Systemd service unit |

### Windows

| File | Purpose |
|------|---------|
| `C:\ProgramData\CMP-Agent\agent.py` | Agent Python script |
| `C:\ProgramData\CMP-Agent\config.json` | Configuration (endpoint, token, resource_id) |
| `C:\ProgramData\CMP-Agent\state.json` | Runtime state — created after successful registration |
| Scheduled Task `CMPAgent` | Service equivalent — runs at startup under SYSTEM |

---

## Config File Reference

**`config.json`** (same structure on both platforms):
```json
{
    "endpoint": "https://<cmp-url>/api/v1/agent",
    "registration_token": "cmp-reg-...",
    "resource_id": "i-0abc123def456",
    "tenant_id": "default",
    "report_interval": 60
}
```

**`state.json`** (auto-created after registration):
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

### Linux

| Requirement | Notes |
|-------------|-------|
| Python 3.6+ | Pre-installed on Amazon Linux 2/2023, Ubuntu 20.04+ |
| pip | May need install on AL2023: `sudo dnf install python3-pip` |
| psutil | Installed by install script — uses `--break-system-packages` on PEP 668 systems |
| requests | Installed by install script |
| systemd | Required for service management (all modern Linux distros) |
| Outbound HTTPS | VM must reach CMP backend on port 443 |
| Root access | Required for `/opt`, `/etc/systemd`, package install |

### Windows

| Requirement | Notes |
|-------------|-------|
| Python 3.8+ | Downloaded automatically by install script from `python.org` if not present |
| psutil | Installed by install script via `pip` |
| requests | Installed by install script via `pip` |
| PowerShell 5.1+ | Pre-installed on Windows Server 2016+ and Windows 10+ |
| Administrator privileges | Required to create Scheduled Task under SYSTEM |
| Outbound HTTPS | VM must reach CMP backend on port 443 and `python.org` on port 443 (for Python download) |
| Task Scheduler | Required — enabled by default on all Windows Server editions |

# CMP Agent — Manual Installation on Windows

This guide covers manually installing the CMP monitoring agent on a Windows Server when the automated user_data installation didn't run or failed.

## Prerequisites

- Windows Server 2016+ or Windows 10+
- Administrator access (RDP or WinRM)
- Outbound HTTPS access to your CMP endpoint (e.g., `https://cmp-app.yourcompany.com`)

## Step 1: Install Python

Open PowerShell **as Administrator**:

```powershell
# Ensure TLS 1.2
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# Download Python installer
New-Item -ItemType Directory -Force -Path C:\Temp | Out-Null
Invoke-WebRequest -Uri 'https://www.python.org/ftp/python/3.12.4/python-3.12.4-amd64.exe' -OutFile C:\Temp\python-installer.exe -UseBasicParsing

# Install silently (all users, added to PATH, includes pip)
Start-Process -Wait -FilePath C:\Temp\python-installer.exe -ArgumentList '/quiet InstallAllUsers=1 PrependPath=1 Include_pip=1'

# Refresh PATH in current session
$env:PATH = [System.Environment]::GetEnvironmentVariable('PATH', 'Machine') + ';' + $env:PATH

# Verify
python --version
```

## Step 2: Install Dependencies

```powershell
python -m pip install --quiet psutil requests
```

Verify:

```powershell
python -c "import psutil, requests; print('OK')"
```

## Step 3: Generate a Registration Token

In CMP, go to the **Resource Detail** page for this VM and click **Install Agent**. This generates a one-time token.

Alternatively, an admin can generate a token via API:

```
POST /api/v1/agent/generate-token?resource_id=<RESOURCE_ID>
Authorization: Bearer <admin-jwt>
```

## Step 4: Download and Run the Install Script

```powershell
# Download the installer
Invoke-WebRequest -Uri 'https://<CMP_URL>/api/v1/agent/install.ps1' -OutFile C:\Temp\cmp-install.ps1 -UseBasicParsing

# Run it
& C:\Temp\cmp-install.ps1 `
    -Endpoint 'https://<CMP_URL>/api/v1/agent' `
    -Token '<REGISTRATION_TOKEN>' `
    -ResourceId '<RESOURCE_ID>' `
    -TenantId 'default'
```

Replace:
- `<CMP_URL>` — your CMP backend URL (e.g., `cmp-app.yourcompany.com`)
- `<REGISTRATION_TOKEN>` — the one-time token from Step 3
- `<RESOURCE_ID>` — the cloud instance ID (e.g., `i-0abc123` for AWS, VM name for Azure/GCP)

## Step 5 (Alternative): Manual Install Without the Script

If the install script fails, you can do it manually:

```powershell
# Create agent directory
$agentDir = 'C:\ProgramData\CMP-Agent'
New-Item -ItemType Directory -Force -Path $agentDir | Out-Null

# Write config
$config = @{
    endpoint = 'https://<CMP_URL>/api/v1/agent'
    registration_token = '<REGISTRATION_TOKEN>'
    resource_id = '<RESOURCE_ID>'
    tenant_id = 'default'
    report_interval = 300
} | ConvertTo-Json
Set-Content -Path "$agentDir\config.json" -Value $config -Encoding UTF8

# Download agent code
Invoke-WebRequest -Uri 'https://<CMP_URL>/api/v1/agent/install.ps1' -OutFile C:\Temp\cmp-install.ps1 -UseBasicParsing
# Extract the agent.py from the script or download separately — 
# The simplest approach: run the agent directly with the config:

# Create a minimal agent.py (the install script normally writes this)
# Instead, register manually and start:
python -c "
import requests, json, socket, platform
config = json.load(open(r'$agentDir\config.json'))
resp = requests.post(config['endpoint'] + '/register', json={
    'registration_token': config['registration_token'],
    'resource_id': config['resource_id'],
    'hostname': socket.gethostname(),
    'os_info': platform.platform(),
    'agent_version': '1.0.0',
})
print(resp.status_code, resp.json())
data = resp.json()
config['agent_id'] = data['agent_id']
config['api_key'] = data['agent_api_key']
config['metrics_endpoint'] = data['metrics_endpoint']
config['registered'] = True
json.dump(config, open(r'$agentDir\state.json', 'w'))
print('Agent registered:', data['agent_id'])
"
```

Then run the full install script after registration succeeds.

## Step 6: Verify

```powershell
# Check agent files exist
Test-Path C:\ProgramData\CMP-Agent\agent.py
Test-Path C:\ProgramData\CMP-Agent\config.json

# Check scheduled task is running
Get-ScheduledTask -TaskName 'CMPAgent' | Select TaskName, State

# Check agent logs
Get-Content C:\ProgramData\CMP-Agent\agent.log -Tail 20
```

In CMP, go to **Resource Detail → System Metrics** tab. You should see metrics appearing within 5 minutes.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Python was not found` | Ensure Python is installed and in PATH. Run `$env:PATH = [System.Environment]::GetEnvironmentVariable('PATH', 'Machine')` to refresh. |
| pip stderr errors | These are usually warnings, not failures. Verify with `python -c "import psutil, requests; print('OK')"` |
| Agent not reporting | Check `C:\ProgramData\CMP-Agent\agent.log` for errors. Verify outbound HTTPS to CMP endpoint: `Invoke-WebRequest -Uri 'https://<CMP_URL>/api/v1/agent/install.sh' -UseBasicParsing` |
| Token expired | Tokens expire after 1 hour. Generate a new one from the CMP UI or API. |
| Task not starting | Run manually: `python C:\ProgramData\CMP-Agent\agent.py` and check for errors. |
| `C:\Temp` doesn't exist | Run `New-Item -ItemType Directory -Force -Path C:\Temp` before downloading. |

## Uninstall

```powershell
# Stop and remove scheduled task
Unregister-ScheduledTask -TaskName 'CMPAgent' -Confirm:$false -ErrorAction SilentlyContinue

# Remove agent files
Remove-Item -Recurse -Force 'C:\ProgramData\CMP-Agent' -ErrorAction SilentlyContinue

# Optionally revoke in CMP (admin)
# POST /api/v1/agent/<resource_id>/revoke
```

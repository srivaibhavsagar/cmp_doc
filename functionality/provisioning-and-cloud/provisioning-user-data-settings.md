# Provisioning User Data Settings — Admin Guide

Configure SSH keys and custom shell commands that are automatically injected into every VM provisioned via CMP. This is managed from the **Admin Settings → Provisioning** tab.

---

## Overview

When CMP provisions a VM (EC2, GCE, Azure VM), it generates a startup script that runs on first boot. The format differs by cloud provider — AWS and Azure use cloud-init YAML (`#cloud-config`), while GCP uses a plain bash startup-script. The Provisioning User Data settings let admins control what goes into those scripts:

| Component | Purpose | Who configures |
|-----------|---------|----------------|
| SSH Public Keys | Added to `~/.ssh/authorized_keys` on all VMs (Linux only) | Admin (this tab) |
| Custom Commands (Linux) | Shell commands run on boot (security agents, log forwarders, etc.) | Admin (this tab) |
| Custom Commands (Windows) | PowerShell commands run on first boot | Admin (this tab) |
| Windows Admin User | Creates a local administrator account on Windows VMs for remote management | Admin (this tab) |
| CMP Agent Install | Monitoring agent for CPU/memory/disk/network metrics | Automatic (always included) |

The final scripts are accessible in native Python tasks via:
- `cmp["user_data"]` — Linux startup script, format auto-selected by provider:
  - **AWS EC2 / Azure VMs** → `#cloud-config` YAML (cloud-init)
  - **GCP Compute Engine VMs** → plain `#!/bin/bash` script (guest agent startup-script)
- `cmp["user_data_windows"]` — Windows PowerShell script (wrapped in `<powershell>` tags)

> **Note:** There is no longer a separate `cmp["user_data_gcp"]` key. Task authors always use `cmp["user_data"]` regardless of cloud provider — CMP selects the correct format automatically based on the credential's provider.

---

## Accessing the Settings

1. Navigate to **Admin Settings** (gear icon in the sidebar, requires `admin` role)
2. Click the **Provisioning** tab
3. Configure SSH keys, custom commands, and preview the generated script

---

## SSH Public Keys

Add public keys that should be injected into every provisioned VM's `~/.ssh/authorized_keys`.

### Fields

| Field | Description | Example |
|-------|-------------|---------|
| Label | Human-readable name for the key | `ops-team-key` |
| Key | The full public key string | `ssh-ed25519 AAAA... ops@company.com` |

### Supported Key Types

- `ssh-rsa`
- `ssh-ed25519`
- `ecdsa-sha2-nistp256` / `nistp384` / `nistp521`

### Usage

- Click **+ Add Key** to add a new SSH key entry
- Fill in a label and the public key content
- Repeat for as many keys as needed (e.g., ops team key, security team key, CI/CD deploy key)
- Click **Save Provisioning Settings** to persist

All keys are added to the default user's `authorized_keys` file during cloud-init execution.

---

## Custom Commands (Linux)

Additional shell commands that run on Linux VM boot, after the CMP Agent is installed. Useful for:

- Installing additional monitoring agents (Datadog, New Relic, etc.)
- Configuring log forwarding (CloudWatch agent, Fluentd, etc.)
- Applying security hardening scripts
- Setting up custom package repos

### Fields

| Field | Description | Example |
|-------|-------------|---------|
| Label | Descriptive name for the command | `install-datadog-agent` |
| Command | Shell command(s) to execute | `curl -sSL https://example.com/install.sh \| bash` |

### Usage

- Click **+ Add Command** to add a new command entry
- Fill in a label and the shell command(s)
- Commands execute in the order they appear
- Click **Save Provisioning Settings** to persist

### Notes

- Commands run as `root` (cloud-init executes with root privileges)
- Commands run after network is available and after the CMP Agent installs
- Multi-line commands are supported in the command textarea
- If a command fails, subsequent commands still execute (cloud-init default behavior)

---

## Windows Local Admin User

Create a local administrator account on Windows VMs at first boot. This enables remote management (e.g., PowerShell Remoting, RDP) with a known set of credentials managed centrally by the admin.

### Fields

| Field | Description | Example |
|-------|-------------|---------|
| Username | Local account name to create | `cmpadmin` |
| Password | Password for the account | (strong password) |

### What It Does

When both username and password are configured, the Windows user_data script will:

1. **Create the local user** (if it doesn't already exist) with `PasswordNeverExpires` set
2. **Add the user to the local Administrators group** for full management access
3. **Update the password** if the user already exists (idempotent on re-runs)
4. **Enable WinRM** (PowerShell Remoting) with Basic authentication for remote management

If the user already exists (e.g., from a previous provision or a custom image), only the password is updated.

### Generated PowerShell

The following is injected into the Windows user_data script before the CMP Agent install and custom commands:

```powershell
# Create local administrator account
$cmpUser = '<configured_username>'
$cmpPass = ConvertTo-SecureString '<configured_password>' -AsPlainText -Force
try {
    if (-not (Get-LocalUser -Name $cmpUser -ErrorAction SilentlyContinue)) {
        New-LocalUser -Name $cmpUser -Password $cmpPass -FullName 'CMP Admin' -Description 'CMP managed administrator account' -PasswordNeverExpires
        Add-LocalGroupMember -Group 'Administrators' -Member $cmpUser
        Write-Host "[CMP] Created local admin user: $cmpUser"
    } else {
        Set-LocalUser -Name $cmpUser -Password $cmpPass
        Write-Host "[CMP] Updated password for existing user: $cmpUser"
    }
} catch {
    Write-Host "[CMP] Failed to create admin user: $_"
}

# Enable WinRM for remote management
try {
    Enable-PSRemoting -Force -SkipNetworkProfileCheck 2>$null
    Set-Item WSMan:\localhost\Service\Auth\Basic -Value $true
    Set-Item WSMan:\localhost\Service\AllowUnencrypted -Value $true
    Write-Host '[CMP] WinRM enabled for remote management'
} catch {
    Write-Host "[CMP] WinRM setup failed: $_"
}
```

### Execution Order (Windows)

1. Local admin user creation + WinRM setup ← *this feature*
2. CMP Agent install
3. Custom Windows commands

### Security Considerations

- The password is stored encrypted in CMP admin settings (Fernet encryption at rest)
- The password appears in plaintext in the user_data script during VM provisioning — this is standard behavior for cloud-init/user_data across all cloud providers
- Use strong, unique passwords and rotate them periodically
- WinRM with Basic auth + AllowUnencrypted is enabled for initial connectivity; consider hardening WinRM with HTTPS/certificates post-provisioning
- Restrict network access to WinRM ports (5985/5986) via security groups or NSGs

### Notes

- If only the username is configured (no password), the admin user block is skipped entirely
- If neither is configured, the Windows script only includes the CMP Agent and custom commands (same as before)
- The feature is Windows-only — Linux VMs use SSH keys for remote access

---

## Custom Commands (Windows)

PowerShell commands that run on Windows VM first boot, after the CMP Agent is installed. These are stored under the `custom_windows_commands` key in admin settings.

### Fields

| Field | Description | Example |
|-------|-------------|---------|
| Label | Descriptive name for the command | `install-iis` |
| Command | PowerShell command(s) to execute | `Install-WindowsFeature -Name Web-Server -IncludeManagementTools` |

### Usage

- Click **+ Add Windows Command** to add a new Windows command entry
- Fill in a label and the PowerShell command(s)
- Commands execute in the order they appear
- Click **Save Provisioning Settings** to persist

### Notes

- Commands run with administrator privileges
- Commands run inside `<powershell>` tags (AWS/Azure Windows user_data convention)
- `$ErrorActionPreference` is set to `Continue` — a failing command does not block subsequent ones
- The generated script is accessible via `cmp["user_data_windows"]`

---

## Generated cloud-init Preview

The top of the Provisioning tab shows a **read-only preview** of the generated scripts. This is exactly what the context keys will contain at execution time (with placeholder values for per-execution tokens).

- `cmp["user_data"]` — cloud-init YAML for AWS/Azure Linux VMs
- `cmp["user_data_gcp"]` — plain bash startup-script for GCP Compute Engine VMs

The preview shows the execution order:

1. SSH keys injection (Linux)
2. Local admin user creation + WinRM setup (Windows, if configured)
3. CMP Agent installation (automatic) — resolves real instance ID via cloud metadata (AWS/Azure/GCP), then installs agent
4. Custom commands

---

## How It Works at Provisioning Time

When a VM provisioning workflow executes:

```
Admin Settings (SSH keys + Windows admin user + commands)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Execution Engine builds user_data scripts           │
│  1. Reads tenant provisioning settings               │
│  2. Generates one-time agent registration token      │
│  3. Linux AWS/Azure: _build_user_data(provider="")   │
│        → cloud-init YAML → cmp["user_data"]          │
│  4. Linux GCP: _build_user_data(provider="gcp")      │
│        → plain bash script → cmp["user_data_gcp"]   │
│  5. Windows: admin user + WinRM + agent + cmds       │
│        → PowerShell → cmp["user_data_windows"]      │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  Native Task (e.g. aws_ec2_provision.py)             │
│  - Reads cmp["user_data"], cmp["user_data_gcp"],     │
│    or cmp["user_data_windows"]                       │
│  - Passes as UserData / startup-script to cloud      │
│  - VM boots, script runs on first boot               │
└─────────────────────────────────────────────────────┘
```

---

## Using in Native Python Tasks

### Linux VMs (AWS / Azure)

The pre-built cloud-init script is available for AWS and Azure:

```python
# AWS EC2 / Azure (cloud-init YAML)
user_data = cmp["user_data"]

# Pass to AWS
run_kwargs["UserData"] = user_data

# Pass to Azure
vm_params["os_profile"]["custom_data"] = base64.b64encode(user_data.encode()).decode()
```

### GCP VMs (Linux)

GCP runs `startup-script` metadata as a plain shell script, not cloud-init YAML. Use `cmp["user_data_gcp"]`:

```python
# GCP Compute Engine — use user_data_gcp, NOT user_data
user_data_gcp = cmp.get("user_data_gcp", "")
if user_data_gcp:
    instance_body["metadata"]["items"].append({
        "key": "startup-script",
        "value": user_data_gcp   # plain bash #!/bin/bash script
    })
# Do NOT pass cmp["user_data"] here — it is cloud-init YAML and will not execute correctly on GCP
```

### Windows VMs

For Windows instances, use the `cmp["user_data_windows"]` key which provides a PowerShell script wrapped in `<powershell>` tags:

```python
# Windows provisioning — use the pre-built PowerShell user_data
user_data_windows = cmp["user_data_windows"]

# Pass to AWS (Windows EC2 instances)
run_kwargs["UserData"] = user_data_windows

# Pass to Azure (Windows VMs)
vm_params["os_profile"]["custom_data"] = base64.b64encode(user_data_windows.encode()).decode()
```

### Choosing Between Linux and Windows

```python
# Determine OS from task parameters and choose the right user_data
os_type = params.get("os_type", "linux").lower()

if os_type == "windows":
    user_data = cmp.get("user_data_windows", "")
else:
    user_data = cmp.get("user_data", "")

if user_data:
    run_kwargs["UserData"] = user_data
```

If either key is empty (no settings configured), native tasks can fall back to building user_data manually from `cmp["agent"]` context. See the EC2 provisioning sample in `cmp_codes/aws_ec2_provision_with_agent.py`.

---

## Automatic CMP Agent Inclusion

The CMP monitoring agent installation is **always** included in both the Linux and Windows generated user_data, regardless of admin configuration. This provides:

- Live system metrics (CPU, memory, disk, network) on the Resource Detail → System Metrics tab
- Agent registration with a one-time token (generated per execution)
- Automatic association with the provisioned resource using the real cloud instance ID

### Dynamic Resource ID Resolution

The agent install command resolves the VM's resource identifier from the cloud provider's metadata service at boot time. The resolution logic differs by script type:

- **AWS/Azure cloud-init**: probes AWS and Azure metadata endpoints in sequence
- **GCP startup-script**: probes only the GCP metadata endpoint directly

This separation ensures each script targets only the relevant metadata service with no ambiguity, and each returns the identifier format that matches how CMP stores resources internally.

### Linux — AWS / Azure (cloud-init)

The generated cloud-init script probes AWS and Azure metadata:

```bash
# Wait for network to be fully available
sleep 15
CMP_RESOURCE_ID=""

# Try AWS EC2 metadata (IMDSv2)
IMDS_TOKEN=$(curl -s --connect-timeout 2 -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 60" 2>/dev/null)
if [ -n "$IMDS_TOKEN" ]; then
  CMP_RESOURCE_ID=$(curl -s --connect-timeout 2 -H "X-aws-ec2-metadata-token: $IMDS_TOKEN" \
    http://169.254.169.254/latest/meta-data/instance-id 2>/dev/null)
fi

# Try Azure IMDS (use VM name — matches CMP resource naming)
if [ -z "$CMP_RESOURCE_ID" ]; then
  CMP_RESOURCE_ID=$(curl -s --connect-timeout 2 -H "Metadata:true" \
    "http://169.254.169.254/metadata/instance/compute/name?api-version=2021-02-01&format=text" 2>/dev/null)
fi

# Fallback to placeholder
if [ -z "$CMP_RESOURCE_ID" ]; then CMP_RESOURCE_ID="<fallback-name>"; fi

curl -sSL <install_url> | bash -s -- \
  --endpoint <cmp_endpoint> \
  --token <registration_token> \
  --resource-id $CMP_RESOURCE_ID \
  --tenant-id <tenant_id>
```

#### Cloud Provider Detection Order — AWS / Azure cloud-init

| Priority | Cloud | Metadata Endpoint | Returns | Example |
|----------|-------|-------------------|---------|---------|
| 1 | AWS | `http://169.254.169.254/latest/meta-data/instance-id` (IMDSv2) | Instance ID | `i-0057f4823e132b59a` |
| 2 | Azure | `http://169.254.169.254/metadata/instance/compute/name?api-version=2021-02-01` | VM name | `my-web-server` |

Each probe uses a 2-second connect timeout to fail fast on non-matching clouds.

**Fallback behavior:** If neither metadata endpoint responds, the resource ID falls back to the instance name provided in the workflow parameters.

### Linux — GCP (startup-script)

GCP VMs use a separate plain bash startup-script (injected via `cmp["user_data_gcp"]`). This script probes only the GCP metadata endpoint:

```bash
#!/bin/bash
# CMP startup-script — auto-generated by CMP

# Install CMP Agent
sleep 10
CMP_RESOURCE_ID=''
# GCP: use instance name from metadata — matches how CMP stores GCP resources
CMP_RESOURCE_ID=$(curl -s --connect-timeout 5 --retry 3 -H "Metadata-Flavor: Google" \
  "http://metadata.google.internal/computeMetadata/v1/instance/name" 2>/dev/null || true)
if [ -z "$CMP_RESOURCE_ID" ]; then CMP_RESOURCE_ID="<fallback-name>"; fi
curl -sSL "<install_url>" | bash -s -- \
  --endpoint "<cmp_endpoint>" \
  --token "<registration_token>" \
  --resource-id "$CMP_RESOURCE_ID" \
  --tenant-id "<tenant_id>"
```

GCP's guest agent executes `startup-script` metadata as a plain shell script on every boot. Cloud-init YAML (`#cloud-config`) is silently ignored by GCP's guest agent, which is why GCP uses its own dedicated bash script format.

**Why VM name instead of VM ID?** CMP stores Azure and GCP resources by their human-readable name (the VM name or instance name) rather than internal GUIDs or numeric IDs. Using the name from metadata ensures the agent's `resource_id` matches the CMP resource record exactly, enabling automatic correlation on the System Metrics tab.

### Windows (PowerShell)

The generated Windows user_data script (`cmp["user_data_windows"]`) performs the same multi-cloud resource ID resolution using PowerShell.

**Network connectivity wait:** Before resolving cloud metadata or downloading the agent installer, the Windows script actively polls for network connectivity (up to 30 minutes, checking every 10 seconds) using Microsoft's connectivity test endpoint. This extended timeout accommodates Windows VMs in environments where network provisioning is slow (e.g., complex NSG/security group configurations, slow DHCP, or enterprise proxy setups). If the network is not available after 30 minutes, the script aborts with an error.

```powershell
<powershell>
# CMP Provisioning User Data (Windows)
$ErrorActionPreference = 'Continue'

# Wait for network connectivity (up to 30 minutes)
Write-Host '[CMP Agent] Waiting for network connectivity...'
$networkReady = $false
for ($i = 1; $i -le 180; $i++) {
    try {
        $null = Invoke-WebRequest -Uri 'http://www.msftconnecttest.com/connecttest.txt' -UseBasicParsing -TimeoutSec 3 -ErrorAction Stop
        $networkReady = $true
        break
    } catch {
        Start-Sleep -Seconds 10
    }
}
if (-not $networkReady) {
    Write-Host '[CMP Agent] ERROR: Network not available after 30 minutes. Aborting.'
    exit 1
}
Write-Host '[CMP Agent] Network ready.'

# Resolve VM identity from cloud metadata
$CMP_RESOURCE_ID = ''

# Try AWS EC2 metadata (IMDSv2)
try {
    $imdsToken = Invoke-RestMethod -Uri 'http://169.254.169.254/latest/api/token' `
        -Method PUT -Headers @{'X-aws-ec2-metadata-token-ttl-seconds'='60'} -TimeoutSec 2 -ErrorAction Stop
    $CMP_RESOURCE_ID = Invoke-RestMethod -Uri 'http://169.254.169.254/latest/meta-data/instance-id' `
        -Headers @{'X-aws-ec2-metadata-token'=$imdsToken} -TimeoutSec 2 -ErrorAction Stop
} catch { }

# Try Azure IMDS
if (-not $CMP_RESOURCE_ID) {
    try {
        $CMP_RESOURCE_ID = Invoke-RestMethod -Uri 'http://169.254.169.254/metadata/instance/compute/name?api-version=2021-02-01&format=text' `
            -Headers @{'Metadata'='true'} -TimeoutSec 2 -ErrorAction Stop
    } catch { }
}

# Try GCP metadata
if (-not $CMP_RESOURCE_ID) {
    try {
        $CMP_RESOURCE_ID = Invoke-RestMethod -Uri 'http://metadata.google.internal/computeMetadata/v1/instance/name' `
            -Headers @{'Metadata-Flavor'='Google'} -TimeoutSec 2 -ErrorAction Stop
    } catch { }
}

if (-not $CMP_RESOURCE_ID) { $CMP_RESOURCE_ID = '<fallback-name>' }

# Install CMP Agent
try {
    Invoke-WebRequest -Uri '<install_url>/install.ps1' -OutFile C:\Temp\cmp-install.ps1 -UseBasicParsing
    & C:\Temp\cmp-install.ps1 -Endpoint <cmp_endpoint> -Token <registration_token> -ResourceId $CMP_RESOURCE_ID -TenantId <tenant_id>
} catch {
    Write-Host "[CMP Agent] Installation failed: $_"
}

# Custom Windows commands follow here...
</powershell>
```

The Windows script uses the PowerShell `install.ps1` installer (derived from the Linux `install.sh` URL by replacing the extension). The same cloud metadata probing logic applies — AWS IMDSv2, then Azure IMDS, then GCP metadata — with 2-second timeouts per probe.

See [CMP Agent Architecture](../agent-and-monitoring/CMP-AGENT-ARCHITECTURE.md) for full details on the agent lifecycle.

---

## Permissions

| Action | Required Role |
|--------|--------------|
| View provisioning settings | `admin` |
| Edit SSH keys / custom commands | `admin` |
| Edit Windows admin user credentials | `admin` |
| Save provisioning settings | `admin` |

---

## Tenant Scoping

Provisioning user data settings are **per-tenant**. Each tenant can configure their own SSH keys and custom commands independently. The settings are stored in the tenant's DynamoDB partition.

---

## Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|------------|
| Preview shows "No user_data configured" | No SSH keys or commands added yet | Add at least one SSH key or command and save |
| SSH key not working on VM | Key format incorrect or extra whitespace | Verify key starts with `ssh-rsa`, `ssh-ed25519`, or `ecdsa-sha2-*` |
| Custom command not executing | Syntax error in command | Test command manually on a sample VM first |
| Agent not registering | Network/firewall blocking agent endpoint | Ensure VM can reach the CMP agent endpoint (see Agent Debug guide) |
| Windows agent install aborted with "Network not available after 30 minutes" | VM has no outbound internet access within 30 minutes of boot | Check NSGs, route tables, NAT gateways, and proxy settings; the script polls `msftconnecttest.com` to confirm connectivity |
| Windows admin user not created | Only username provided (no password) | Both username and password must be configured |
| Cannot connect via WinRM | Security group blocking port 5985/5986 | Add inbound rule for WinRM ports from your management network |
| WinRM setup failed on VM | OS is pre-Windows Server 2012 or WinRM service disabled | Ensure Windows Server 2012+ and WinRM service is not disabled by GPO |

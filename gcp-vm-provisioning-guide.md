# GCP VM Provisioning Guide — Day-1 Catalog with Secure Credentials

This document explains how GCP Compute Engine VM provisioning works in CMP using the Day-1 catalog pattern, with secure credential handling that never exposes the original service account key.

---

## Architecture Overview

```
User submits catalog form (selects GCP credential + VM config)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Execution Endpoint (POST /api/v1/executions)       │
│  - Validates RBAC, credential access, policies      │
│  - Calls _build_credential_ctx(cred)                │
│  - Generates short-lived OAuth2 token (1hr)         │
│  - Kicks off background flow execution              │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  Orchestrator → Workflow Steps (DAG)                │
│  - Interpolates {{credential.project_id}},          │
│    {{form.instance_name}}, etc.                     │
│  - Passes credential context to Terraform Engine    │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  Terraform Engine                                   │
│  - CredentialInjector writes SA JSON to temp file   │
│  - Sets GOOGLE_APPLICATION_CREDENTIALS env var      │
│  - Runs: init → validate → plan → apply            │
│  - Temp file auto-deleted within 30 seconds         │
└─────────────────────────────────────────────────────┘
```

---

## Credential Security Model

| Layer | What Happens | Why It's Safe |
|-------|-------------|---------------|
| **Storage** | Service account JSON encrypted with Fernet (PBKDF2-derived key) in DynamoDB | At rest, it's just ciphertext |
| **Execution Context** | `_build_credential_ctx()` generates a 1-hour OAuth2 access token via `gcp_auth.py` | Workflow steps only see `temp_access_token` — the raw JSON key is never in the context |
| **Terraform Execution** | `CredentialInjector._get_gcp_env_vars()` writes SA JSON to a temp file, passes path as `GOOGLE_APPLICATION_CREDENTIALS` | File exists only during execution, auto-cleaned in ≤30 seconds |

### What Workflow Steps Can Access

```
{{credential.temp_access_token}}      # Short-lived OAuth2 token (1hr TTL)
{{credential.temp_token_expiration}}  # ISO timestamp when token expires
{{credential.project_id}}            # GCP project ID
{{credential.provider}}              # "gcp"
{{credential.credential_id}}         # CMP credential UUID
```

### What's NEVER Exposed

- The raw service account JSON key
- The `private_key` field
- The `client_email` or `private_key_id`
- The encrypted `secret_key` value from DynamoDB

---

## API Calls to Set Up GCP VM Provisioning

### Step 1: Create a Terraform Template

```http
POST /api/v1/terraform/templates
Authorization: Bearer <admin_token>
Content-Type: application/json
```

```json
{
  "name": "GCP Compute Instance",
  "description": "Provisions a GCP Compute Engine VM with configurable machine type, image, zone, and network.",
  "source_type": "inline",
  "source_config": {
    "hcl_content": "terraform {\n  required_providers {\n    google = {\n      source  = \"hashicorp/google\"\n      version = \"~> 5.0\"\n    }\n  }\n}\n\nprovider \"google\" {\n  project = var.project_id\n  region  = var.region\n  zone    = var.zone\n}\n\nresource \"google_compute_instance\" \"vm\" {\n  name         = var.instance_name\n  machine_type = var.machine_type\n  zone         = var.zone\n\n  boot_disk {\n    initialize_params {\n      image = var.boot_image\n      size  = var.disk_size_gb\n    }\n  }\n\n  network_interface {\n    network    = var.network\n    subnetwork = var.subnetwork\n\n    dynamic \"access_config\" {\n      for_each = var.assign_public_ip ? [1] : []\n      content {}\n    }\n  }\n\n  labels = var.labels\n\n  metadata = {\n    enable-oslogin = \"TRUE\"\n  }\n}\n\noutput \"instance_id\" {\n  value = google_compute_instance.vm.instance_id\n}\n\noutput \"internal_ip\" {\n  value = google_compute_instance.vm.network_interface[0].network_ip\n}\n\noutput \"external_ip\" {\n  value = var.assign_public_ip ? google_compute_instance.vm.network_interface[0].access_config[0].nat_ip : null\n}\n\noutput \"self_link\" {\n  value = google_compute_instance.vm.self_link\n}"
  },
  "input_variables": [
    {"name": "project_id", "type": "string", "description": "GCP Project ID", "required": true},
    {"name": "region", "type": "string", "description": "GCP Region", "default": "us-central1", "required": true},
    {"name": "zone", "type": "string", "description": "GCP Zone", "default": "us-central1-a", "required": true},
    {"name": "instance_name", "type": "string", "description": "VM instance name", "required": true},
    {"name": "machine_type", "type": "string", "description": "Machine type", "default": "e2-medium", "required": true},
    {"name": "boot_image", "type": "string", "description": "Boot disk image", "default": "debian-cloud/debian-12", "required": true},
    {"name": "disk_size_gb", "type": "number", "description": "Boot disk size in GB", "default": 20, "required": true},
    {"name": "network", "type": "string", "description": "VPC network", "default": "default", "required": true},
    {"name": "subnetwork", "type": "string", "description": "Subnetwork", "default": "default"},
    {"name": "assign_public_ip", "type": "bool", "description": "Assign external IP", "default": true},
    {"name": "labels", "type": "map", "description": "Labels to attach", "default": {}}
  ],
  "output_definitions": [
    {"name": "instance_id", "description": "The instance ID of the created VM"},
    {"name": "internal_ip", "description": "Internal IP address"},
    {"name": "external_ip", "description": "External IP address (if assigned)"},
    {"name": "self_link", "description": "Self link URL of the instance"}
  ],
  "required_providers": {"google": "~> 5.0"},
  "supported_providers": ["gcp"],
  "tags": ["gcp", "compute", "vm", "day1"]
}
```

---

### Step 2: Create a Workflow

```http
POST /api/v1/workflows
Authorization: Bearer <admin_token>
Content-Type: application/json
```

```json
{
  "name": "gcp-vm-provision",
  "description": "Provisions a GCP Compute Engine VM via Terraform",
  "steps": [
    {
      "step_id": "terraform_provision",
      "name": "Provision GCP VM",
      "action": "terraform",
      "template_id": "<TEMPLATE_ID_FROM_STEP_1>",
      "inputs": {
        "project_id": "{{credential.project_id}}",
        "region": "{{form.region}}",
        "zone": "{{form.zone}}",
        "instance_name": "{{form.instance_name}}",
        "machine_type": "{{form.machine_type}}",
        "boot_image": "{{form.boot_image}}",
        "disk_size_gb": "{{form.disk_size_gb}}",
        "network": "{{form.network}}",
        "subnetwork": "{{form.subnetwork}}",
        "assign_public_ip": "{{form.assign_public_ip}}"
      },
      "depends_on": [],
      "on_failure": "stop",
      "timeout_seconds": 600
    }
  ],
  "tags": ["gcp", "compute", "terraform"]
}
```

> **Key point:** `"project_id": "{{credential.project_id}}"` pulls the project ID from the credential context automatically. The user never sees or provides the service account key.

---

### Step 3: Create a Flow

```http
POST /api/v1/flows
Authorization: Bearer <admin_token>
Content-Type: application/json
```

```json
{
  "name": "gcp-vm-flow",
  "description": "Flow for GCP VM provisioning",
  "workflows": [
    {
      "workflow_id": "<WORKFLOW_ID_FROM_STEP_2>",
      "order": 1,
      "depends_on": [],
      "input_mapping": {}
    }
  ],
  "tags": ["gcp", "compute"]
}
```

---

### Step 4: Create the Catalog Item (Frontend Form)

```http
POST /api/v1/catalogs
Authorization: Bearer <admin_token>
Content-Type: application/json
```

```json
{
  "name": "GCP Compute Engine VM",
  "description": "Provision a Google Cloud Compute Engine virtual machine. Select your GCP credential, configure VM specs, and deploy.",
  "tags": ["gcp", "compute", "vm", "infrastructure"],
  "catalog_type": "day1",
  "cloud_provider": "gcp",
  "ui_mode": "form_builder",
  "flow_id": "<FLOW_ID_FROM_STEP_3>",
  "allowed_roles": ["admin", "developer", "user"],
  "status": "published",
  "show_cost_estimate": true,
  "form_schema": {
    "fields": [
      {
        "field_id": "instance_name",
        "label": "Instance Name",
        "type": "string",
        "required": true,
        "placeholder": "my-gcp-vm-01",
        "description": "Name for the VM instance (must be unique within project/zone)",
        "validation": {
          "pattern": "^[a-z]([-a-z0-9]*[a-z0-9])?$",
          "maxLength": 63
        }
      },
      {
        "field_id": "machine_type",
        "label": "Machine Type",
        "type": "select",
        "required": true,
        "default": "e2-medium",
        "options": [
          {"label": "e2-micro (0.25 vCPU, 1 GB) — Free tier", "value": "e2-micro"},
          {"label": "e2-small (0.5 vCPU, 2 GB)", "value": "e2-small"},
          {"label": "e2-medium (1 vCPU, 4 GB)", "value": "e2-medium"},
          {"label": "e2-standard-2 (2 vCPU, 8 GB)", "value": "e2-standard-2"},
          {"label": "e2-standard-4 (4 vCPU, 16 GB)", "value": "e2-standard-4"},
          {"label": "e2-standard-8 (8 vCPU, 32 GB)", "value": "e2-standard-8"},
          {"label": "n2-standard-2 (2 vCPU, 8 GB)", "value": "n2-standard-2"},
          {"label": "n2-standard-4 (4 vCPU, 16 GB)", "value": "n2-standard-4"},
          {"label": "n2-standard-8 (8 vCPU, 32 GB)", "value": "n2-standard-8"},
          {"label": "c2-standard-4 (4 vCPU, 16 GB) — Compute optimized", "value": "c2-standard-4"}
        ],
        "description": "GCP machine type determining CPU and memory"
      },
      {
        "field_id": "region",
        "label": "Region",
        "type": "select",
        "required": true,
        "default": "us-central1",
        "options": [
          {"label": "us-central1 (Iowa)", "value": "us-central1"},
          {"label": "us-east1 (South Carolina)", "value": "us-east1"},
          {"label": "us-west1 (Oregon)", "value": "us-west1"},
          {"label": "europe-west1 (Belgium)", "value": "europe-west1"},
          {"label": "europe-west2 (London)", "value": "europe-west2"},
          {"label": "asia-south1 (Mumbai)", "value": "asia-south1"},
          {"label": "asia-east1 (Taiwan)", "value": "asia-east1"},
          {"label": "australia-southeast1 (Sydney)", "value": "australia-southeast1"}
        ],
        "description": "GCP region for the VM"
      },
      {
        "field_id": "zone",
        "label": "Zone",
        "type": "select",
        "required": true,
        "default": "us-central1-a",
        "options": [
          {"label": "us-central1-a", "value": "us-central1-a"},
          {"label": "us-central1-b", "value": "us-central1-b"},
          {"label": "us-central1-c", "value": "us-central1-c"},
          {"label": "us-east1-b", "value": "us-east1-b"},
          {"label": "us-east1-c", "value": "us-east1-c"},
          {"label": "us-west1-a", "value": "us-west1-a"},
          {"label": "europe-west1-b", "value": "europe-west1-b"},
          {"label": "europe-west2-a", "value": "europe-west2-a"},
          {"label": "asia-south1-a", "value": "asia-south1-a"},
          {"label": "australia-southeast1-a", "value": "australia-southeast1-a"}
        ],
        "description": "Availability zone within the region"
      },
      {
        "field_id": "boot_image",
        "label": "Boot Image",
        "type": "select",
        "required": true,
        "default": "debian-cloud/debian-12",
        "options": [
          {"label": "Debian 12 (Bookworm)", "value": "debian-cloud/debian-12"},
          {"label": "Debian 11 (Bullseye)", "value": "debian-cloud/debian-11"},
          {"label": "Ubuntu 22.04 LTS", "value": "ubuntu-os-cloud/ubuntu-2204-lts"},
          {"label": "Ubuntu 24.04 LTS", "value": "ubuntu-os-cloud/ubuntu-2404-lts-amd64"},
          {"label": "CentOS Stream 9", "value": "centos-cloud/centos-stream-9"},
          {"label": "Rocky Linux 9", "value": "rocky-linux-cloud/rocky-linux-9"},
          {"label": "Windows Server 2022", "value": "windows-cloud/windows-2022"},
          {"label": "Container-Optimized OS", "value": "cos-cloud/cos-stable"}
        ],
        "description": "Operating system image for the boot disk"
      },
      {
        "field_id": "disk_size_gb",
        "label": "Boot Disk Size (GB)",
        "type": "number",
        "required": true,
        "default": 20,
        "validation": {"min": 10, "max": 2048},
        "description": "Boot disk size in gigabytes"
      },
      {
        "field_id": "network",
        "label": "VPC Network",
        "type": "string",
        "required": true,
        "default": "default",
        "placeholder": "default",
        "description": "VPC network name (use 'default' for the default VPC)"
      },
      {
        "field_id": "subnetwork",
        "label": "Subnetwork",
        "type": "string",
        "required": false,
        "default": "default",
        "placeholder": "default",
        "description": "Subnetwork name within the VPC (leave empty to auto-select)"
      },
      {
        "field_id": "assign_public_ip",
        "label": "Assign Public IP",
        "type": "boolean",
        "required": false,
        "default": true,
        "description": "Assign an external IP address for internet access"
      }
    ]
  }
}
```

---

### Step 5: Execute Provisioning (User Triggers from Frontend)

```http
POST /api/v1/executions
Authorization: Bearer <user_token>
Content-Type: application/json
```

```json
{
  "catalog_id": "<CATALOG_ID>",
  "flow_id": "<FLOW_ID>",
  "credential_id": "<GCP_CREDENTIAL_ID>",
  "form_data": {
    "instance_name": "web-server-prod-01",
    "machine_type": "e2-standard-2",
    "region": "us-central1",
    "zone": "us-central1-a",
    "boot_image": "ubuntu-os-cloud/ubuntu-2204-lts",
    "disk_size_gb": 50,
    "network": "default",
    "subnetwork": "default",
    "assign_public_ip": true
  },
  "cloud_provider": "gcp"
}
```

---

## Internal Credential Flow (What Happens Behind the Scenes)

```
User submits execution with credential_id
           │
           ▼
┌──────────────────────────────────────────────────┐
│  _build_credential_ctx(cred)                     │
│                                                  │
│  1. Decrypts SA JSON from DynamoDB (server-side) │
│  2. Calls get_gcp_temp_credentials()             │
│  3. Returns ctx = {                              │
│       credential_id: "uuid",                     │
│       provider: "gcp",                           │
│       project_id: "deep-geography-243421",       │
│       temp_access_token: "ya29.xxx...",  ← 1hr   │
│       temp_token_expiration: "2026-06-09T..."    │
│     }                                            │
│  ⛔ secret_key is NOT in the context             │
└──────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────┐
│  Orchestrator → execute_step("terraform")        │
│                                                  │
│  Interpolates: {{credential.project_id}}         │
│            → "deep-geography-243421"             │
│                                                  │
│  Then CredentialInjector.resolve() is called:    │
│  - Fetches cred from DB, decrypts again         │
│  - Writes SA JSON to /tmp/gcp_cred_xxxx.json    │
│  - Sets env: GOOGLE_APPLICATION_CREDENTIALS=... │
│  - Terraform uses this env var for auth         │
│  - Temp file deleted within 30 seconds          │
└──────────────────────────────────────────────────┘
```

---

## Alternative: `run_task` with Python Script (Custom Provisioning)

If you want to use the GCP Python SDK directly instead of Terraform:

```python
# task code uses {{credential.temp_access_token}} and {{credential.project_id}}
import google.oauth2.credentials
from googleapiclient import discovery

# The temp token is injected by the orchestrator
token = "{{credential.temp_access_token}}"
project = "{{credential.project_id}}"

# Use the short-lived token directly (no long-term key exposed)
creds = google.oauth2.credentials.Credentials(token=token)
compute = discovery.build('compute', 'v1', credentials=creds)

# Create instance config
instance_body = {
    "name": "{{form.instance_name}}",
    "machineType": f"zones/{{form.zone}}/machineTypes/{{form.machine_type}}",
    "disks": [{
        "boot": True,
        "autoDelete": True,
        "initializeParams": {
            "sourceImage": f"projects/{{form.boot_image}}",
            "diskSizeGb": "{{form.disk_size_gb}}"
        }
    }],
    "networkInterfaces": [{
        "network": f"global/networks/{{form.network}}",
        "accessConfigs": [{"type": "ONE_TO_ONE_NAT", "name": "External NAT"}]
    }]
}

# Execute
operation = compute.instances().insert(
    project=project,
    zone="{{form.zone}}",
    body=instance_body
).execute()

print(f"VM creation started: {operation['name']}")
```

The task code only receives the ephemeral OAuth2 token, never the long-lived service account key.

---

## Summary

| Aspect | Detail |
|--------|--------|
| **Credential stored as** | Fernet-encrypted SA JSON in DynamoDB |
| **User sees** | Only credential name + project ID in frontend |
| **Workflow steps receive** | `temp_access_token` (1hr TTL) + `project_id` |
| **Terraform receives** | `GOOGLE_APPLICATION_CREDENTIALS` env var pointing to temp file |
| **Temp file lifetime** | Exists only during execution, deleted within 30 seconds |
| **API response** | Never contains private_key or secret_key values |

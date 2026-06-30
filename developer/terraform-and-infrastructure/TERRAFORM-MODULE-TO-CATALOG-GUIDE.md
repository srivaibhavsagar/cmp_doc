# Terraform Module → Template → Catalog: End-to-End Guide

> Step-by-step guide for creating a Terraform module in GitHub, registering it in CMP as a Template, and linking it to a Catalog item for self-service provisioning.

---

## Table of Contents

1. [Overview — How the Pieces Connect](#overview--how-the-pieces-connect)
2. [Step 1: Create Terraform Files in GitHub](#step-1-create-terraform-files-in-github)
3. [Step 2: Register as a Module in CMP](#step-2-register-as-a-module-in-cmp)
4. [Step 3: Create a Template Using the Module](#step-3-create-a-template-using-the-module)
5. [Step 4: Create a Workflow with a Terraform Step](#step-4-create-a-workflow-with-a-terraform-step)
6. [Step 5: Create a Flow Referencing the Workflow](#step-5-create-a-flow-referencing-the-workflow)
7. [Step 6: Create a Catalog Item Linked to the Flow](#step-6-create-a-catalog-item-linked-to-the-flow)
8. [Step 7: Test the End-to-End Provisioning](#step-7-test-the-end-to-end-provisioning)
9. [Quick Reference — API Calls Summary](#quick-reference--api-calls-summary)

---

## Overview — How the Pieces Connect

```
GitHub Repo (Terraform .tf files)
    │
    ▼
CMP Module (reusable building block, optional)
    │
    ▼
CMP Template (deployable Terraform config with variables/outputs)
    │
    ▼
CMP Workflow (step: action="terraform", template_id=<id>)
    │
    ▼
CMP Flow (references one or more workflows in order)
    │
    ▼
CMP Catalog Item (self-service form, linked to flow via flow_id)
    │
    ▼
User Requests → Workspace Created → Infrastructure Provisioned
```

| CMP Concept | Purpose | Analogy |
|-------------|---------|---------|
| Module | Reusable Terraform component | A library/function |
| Template | Complete deployable config | A program |
| Workflow | Ordered steps to execute | A script |
| Flow | Groups workflows together | A pipeline |
| Catalog Item | User-facing request form | An app store listing |
| Workspace | Running instance of a template | A deployed app |

---

## Step 1: Create Terraform Files in GitHub

### 1.1 Create a GitHub Repository

Create a new repo (or use an existing one) with this structure:

```
my-terraform-modules/
├── aws-ec2-instance/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── aws-s3-bucket/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── README.md
```

Each folder is a self-contained module.

### 1.2 Write the Terraform Files

**`aws-ec2-instance/main.tf`**

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

resource "aws_security_group" "instance" {
  name        = "${var.name}-sg"
  description = "Security group for ${var.name}"
  vpc_id      = var.vpc_id

  ingress {
    description = "SSH access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_ssh_cidrs
  }

  dynamic "ingress" {
    for_each = var.additional_ports
    content {
      description = "Port ${ingress.value}"
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = var.allowed_ingress_cidrs
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.tags, { Name = "${var.name}-sg" })
}

resource "aws_instance" "this" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.instance.id]
  key_name               = var.key_name

  root_block_device {
    volume_size = var.root_volume_size
    volume_type = "gp3"
    encrypted   = true
  }

  metadata_options {
    http_tokens   = "required"
    http_endpoint = "enabled"
  }

  tags = merge(var.tags, { Name = var.name })
}
```

**`aws-ec2-instance/variables.tf`**

```hcl
variable "name" {
  description = "Name for the instance and related resources"
  type        = string
}

variable "ami_id" {
  description = "AMI ID to launch"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "vpc_id" {
  description = "VPC ID for the security group"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID to launch the instance in"
  type        = string
}

variable "key_name" {
  description = "SSH key pair name"
  type        = string
  default     = ""
}

variable "root_volume_size" {
  description = "Root EBS volume size in GB"
  type        = number
  default     = 20
}

variable "allowed_ssh_cidrs" {
  description = "CIDR blocks allowed SSH access"
  type        = list(string)
  default     = []
}

variable "additional_ports" {
  description = "Additional ports to open"
  type        = list(number)
  default     = []
}

variable "allowed_ingress_cidrs" {
  description = "CIDR blocks for additional port access"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

**`aws-ec2-instance/outputs.tf`**

```hcl
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.this.id
}

output "private_ip" {
  description = "Private IP address"
  value       = aws_instance.this.private_ip
}

output "public_ip" {
  description = "Public IP address (if assigned)"
  value       = aws_instance.this.public_ip
}

output "security_group_id" {
  description = "Security group ID"
  value       = aws_security_group.instance.id
}
```

### 1.3 Push to GitHub

```bash
cd my-terraform-modules
git init
git add .
git commit -m "Add aws-ec2-instance module"
git remote add origin https://github.com/your-org/my-terraform-modules.git
git push -u origin main
```

---

## Step 2: Register as a Module in CMP

Navigate to **IaC → Terraform → Modules tab** in the CMP UI, or use the API.

### Via API

```bash
curl -X POST "${CMP_URL}/api/v1/terraform/modules" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "aws-ec2-instance",
    "description": "Reusable EC2 instance with security group",
    "source_type": "git",
    "source_config": {
      "repo_url": "your-org/my-terraform-modules",
      "branch": "main",
      "folder_path": "aws-ec2-instance"
    },
    "input_variables": [
      { "name": "name", "type": "string", "description": "Instance name", "required": true },
      { "name": "ami_id", "type": "string", "description": "AMI ID", "required": true },
      { "name": "instance_type", "type": "string", "description": "Instance type", "default": "t3.micro", "required": false },
      { "name": "vpc_id", "type": "string", "description": "VPC ID", "required": true },
      { "name": "subnet_id", "type": "string", "description": "Subnet ID", "required": true },
      { "name": "root_volume_size", "type": "number", "description": "Root volume GB", "default": 20, "required": false }
    ],
    "output_definitions": [
      { "name": "instance_id", "description": "EC2 instance ID" },
      { "name": "private_ip", "description": "Private IP" },
      { "name": "public_ip", "description": "Public IP" }
    ],
    "tags": ["aws", "ec2", "compute"]
  }'
```

### Via UI

1. Go to **IaC → Terraform**
2. Click the **Modules** tab
3. Click **+ Create Module**
4. Fill in:
   - **Name:** `aws-ec2-instance`
   - **Source Type:** Git Repository
   - **Repo URL:** `your-org/my-terraform-modules`
   - **Branch:** `main`
   - **Folder Path:** `aws-ec2-instance`
5. Click **Parse from Git** to auto-detect variables and outputs
6. Review and save

> **Note:** For private repos, add a credential first (Settings → Credentials → GitHub PAT), then select it when creating the module.

### Using "Parse from Git" Feature

CMP can auto-detect variables and outputs from your `.tf` files:

```bash
curl -X POST "${CMP_URL}/api/v1/terraform/templates/parse-git-source" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "repo_url": "your-org/my-terraform-modules",
    "branch": "main",
    "folder_path": "aws-ec2-instance"
  }'
```

Response:
```json
{
  "variables": [
    { "name": "name", "type": "string", "description": "Name for the instance...", "required": true },
    { "name": "ami_id", "type": "string", "description": "AMI ID to launch", "required": true }
  ],
  "outputs": [
    { "name": "instance_id", "description": "EC2 instance ID" },
    { "name": "private_ip", "description": "Private IP address" }
  ],
  "files_scanned": 3
}
```

---

## Step 3: Create a Template Using the Module

A Template is the deployable unit. It can reference your module directly from Git, or use inline HCL that calls the module.

### Option A: Git Source (Recommended)

Point the template directly at your GitHub repo. This is the simplest approach — CMP clones the repo at execution time.

```bash
curl -X POST "${CMP_URL}/api/v1/terraform/templates" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "AWS EC2 Web Server",
    "description": "Provision an EC2 instance with security group for web workloads",
    "source_type": "git",
    "source_config": {
      "repo_url": "your-org/my-terraform-modules",
      "branch": "main",
      "folder_path": "aws-ec2-instance"
    },
    "input_variables": [
      { "name": "name", "type": "string", "description": "Server name", "required": true, "day2_configurable": false },
      { "name": "ami_id", "type": "string", "description": "AMI ID", "required": true },
      { "name": "instance_type", "type": "string", "description": "Instance type", "default": "t3.micro", "required": false, "day2_configurable": true },
      { "name": "vpc_id", "type": "string", "description": "VPC ID", "required": true },
      { "name": "subnet_id", "type": "string", "description": "Subnet ID", "required": true },
      { "name": "root_volume_size", "type": "number", "description": "Root volume size (GB)", "default": 20, "required": false, "day2_configurable": true }
    ],
    "output_definitions": [
      { "name": "instance_id", "description": "EC2 Instance ID" },
      { "name": "private_ip", "description": "Private IP" },
      { "name": "public_ip", "description": "Public IP" }
    ],
    "supported_providers": ["aws"],
    "tags": ["aws", "ec2", "web-server"]
  }'
```

### Option B: Inline HCL (Calls the Module)

If you want the template to compose multiple modules or add extra resources:

```bash
curl -X POST "${CMP_URL}/api/v1/terraform/templates" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "AWS Web App Stack",
    "description": "EC2 instance + S3 bucket for web application",
    "source_type": "inline",
    "source_config": {
      "hcl_content": "provider \"aws\" {\n  region = var.aws_region\n}\n\nmodule \"web_server\" {\n  source = \"git::https://github.com/your-org/my-terraform-modules.git//aws-ec2-instance?ref=main\"\n\n  name          = var.server_name\n  ami_id        = var.ami_id\n  instance_type = var.instance_type\n  vpc_id        = var.vpc_id\n  subnet_id     = var.subnet_id\n}\n\nresource \"aws_s3_bucket\" \"app_data\" {\n  bucket = \"${var.server_name}-data\"\n  tags   = { Name = \"${var.server_name}-data\" }\n}\n\noutput \"instance_id\" {\n  value = module.web_server.instance_id\n}\n\noutput \"bucket_name\" {\n  value = aws_s3_bucket.app_data.id\n}"
    },
    "input_variables": [
      { "name": "aws_region", "type": "string", "description": "AWS Region", "default": "us-east-1", "required": false },
      { "name": "server_name", "type": "string", "description": "Server name", "required": true },
      { "name": "ami_id", "type": "string", "description": "AMI ID", "required": true },
      { "name": "instance_type", "type": "string", "description": "Instance type", "default": "t3.micro", "required": false, "day2_configurable": true },
      { "name": "vpc_id", "type": "string", "description": "VPC ID", "required": true },
      { "name": "subnet_id", "type": "string", "description": "Subnet ID", "required": true }
    ],
    "output_definitions": [
      { "name": "instance_id", "description": "EC2 Instance ID" },
      { "name": "bucket_name", "description": "S3 Bucket Name" }
    ],
    "supported_providers": ["aws"],
    "required_providers": { "aws": "~> 5.0" },
    "tags": ["aws", "ec2", "s3", "web-app"]
  }'
```

**Save the returned `template_id`** — you'll need it in the next step.

### Via UI

1. Go to **IaC → Terraform**
2. Stay on the **Templates** tab
3. Click **+ Create Template**
4. Choose source type (Git or Inline)
5. For Git: enter repo URL, branch, folder path → click **Parse from Git**
6. Review variables/outputs, mark which are `day2_configurable`
7. Set supported providers and tags
8. Save

---

## Step 4: Create a Workflow with a Terraform Step

A Workflow defines the execution steps. For Terraform provisioning, you need a step with `action: "terraform"` and the `template_id`.

### Via API

```bash
curl -X POST "${CMP_URL}/api/v1/workflows" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Provision EC2 Web Server",
    "description": "Runs terraform apply for the EC2 web server template",
    "steps": [
      {
        "step_id": "tf-provision",
        "name": "Terraform Apply",
        "action": "terraform",
        "template_id": "<TEMPLATE_ID_FROM_STEP_3>",
        "inputs": {
          "template_id": "<TEMPLATE_ID_FROM_STEP_3>",
          "operation": "apply",
          "approval_required": false,
          "timeout_seconds": 600
        },
        "depends_on": [],
        "on_failure": "stop",
        "timeout_seconds": 600,
        "retry_count": 0
      }
    ],
    "tags": ["terraform", "ec2", "provisioning"]
  }'
```

**Save the returned `workflow_id`.**

### Key Fields in the Terraform Step

| Field | Purpose |
|-------|---------|
| `action` | Must be `"terraform"` |
| `template_id` | References the template created in Step 3 |
| `inputs.operation` | `"apply"` (provision), `"destroy"` (teardown), `"plan"` (dry-run) |
| `inputs.approval_required` | If `true`, pauses for approval before apply |
| `inputs.timeout_seconds` | Max execution time (default 600s) |

### Variable Passing

Form field values from the catalog submission automatically flow into Terraform variables. The mapping is:

```
User fills form → form_data dict → merged with step inputs → passed as terraform variables
```

So if your template has a variable `instance_type` and your catalog form has a field with `field_name: "instance_type"`, the value flows through automatically.

---

## Step 5: Create a Flow Referencing the Workflow

A Flow groups one or more workflows in execution order.

### Via API

```bash
curl -X POST "${CMP_URL}/api/v1/flows" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "EC2 Web Server Provisioning Flow",
    "description": "Provisions an EC2 web server via Terraform",
    "workflows": [
      {
        "workflow_id": "<WORKFLOW_ID_FROM_STEP_4>",
        "order": 1,
        "depends_on": [],
        "input_mapping": {}
      }
    ],
    "tags": ["terraform", "ec2"]
  }'
```

**Save the returned `flow_id`.**

### Multiple Workflows Example

If you need pre/post steps (e.g., notify Slack after provisioning):

```json
{
  "workflows": [
    { "workflow_id": "<TF_WORKFLOW_ID>", "order": 1, "depends_on": [] },
    { "workflow_id": "<NOTIFY_WORKFLOW_ID>", "order": 2, "depends_on": ["<TF_WORKFLOW_ID>"] }
  ]
}
```

---

## Step 6: Create a Catalog Item Linked to the Flow

The Catalog Item is what users see in the self-service portal. It defines the form fields and links to the flow.

### Via API

```bash
curl -X POST "${CMP_URL}/api/v1/catalog" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "AWS EC2 Web Server",
    "description": "Provision a secure EC2 instance with configurable size and networking",
    "catalog_type": "day1",
    "flow_id": "<FLOW_ID_FROM_STEP_5>",
    "form_schema": {
      "fields": [
        {
          "field_id": "f1",
          "field_name": "name",
          "label": "Server Name",
          "type": "string",
          "required": true,
          "placeholder": "e.g., my-web-server",
          "description": "Unique name for the EC2 instance"
        },
        {
          "field_id": "f2",
          "field_name": "instance_type",
          "label": "Instance Type",
          "type": "select",
          "required": true,
          "default": "t3.micro",
          "options": [
            { "label": "t3.micro (2 vCPU, 1 GB)", "value": "t3.micro" },
            { "label": "t3.small (2 vCPU, 2 GB)", "value": "t3.small" },
            { "label": "t3.medium (2 vCPU, 4 GB)", "value": "t3.medium" },
            { "label": "t3.large (2 vCPU, 8 GB)", "value": "t3.large" }
          ]
        },
        {
          "field_id": "f3",
          "field_name": "ami_id",
          "label": "AMI ID",
          "type": "string",
          "required": true,
          "placeholder": "ami-0c02fb55956c7d316",
          "description": "Amazon Machine Image ID"
        },
        {
          "field_id": "f4",
          "field_name": "vpc_id",
          "label": "VPC",
          "type": "string",
          "required": true,
          "placeholder": "vpc-xxxxxxxx"
        },
        {
          "field_id": "f5",
          "field_name": "subnet_id",
          "label": "Subnet",
          "type": "string",
          "required": true,
          "placeholder": "subnet-xxxxxxxx"
        },
        {
          "field_id": "f6",
          "field_name": "root_volume_size",
          "label": "Root Volume Size (GB)",
          "type": "number",
          "required": false,
          "default": 20,
          "validation": { "min": 8, "max": 500 }
        }
      ]
    },
    "allowed_roles": ["admin", "developer", "user"],
    "requires_approval": false,
    "show_cost_estimate": true,
    "status": "published",
    "tags": ["aws", "ec2", "compute"]
  }'
```

### Important: field_name ↔ Terraform Variable Mapping

The `field_name` in each form field **must match** the Terraform variable name in your template:

| Form Field `field_name` | Terraform Variable | Result |
|-------------------------|-------------------|--------|
| `name` | `variable "name"` | ✅ Auto-mapped |
| `instance_type` | `variable "instance_type"` | ✅ Auto-mapped |
| `server_name` | `variable "name"` | ❌ Won't map (names differ) |

### Via UI

1. Go to **Catalog → Manage**
2. Click **+ Create Catalog Item**
3. Fill in name, description, type = Day 1
4. Build the form using the Form Builder (drag fields)
5. Set each field's `field_name` to match your Terraform variables
6. Under **Execution**, select the Flow created in Step 5
7. Set allowed roles and approval settings
8. Publish

---

## Step 7: Test the End-to-End Provisioning

### Prerequisites

1. **AWS Credential** — Create a credential in CMP (Settings → Credentials) with AWS access key/secret that has permissions to create EC2 instances and security groups.
2. **Terraform Feature Flag** — Ensure Terraform provisioning is enabled for your tenant (Settings → Feature Toggles → `terraform_provisioning`).

### Test via UI

1. Go to **Catalog** (user view)
2. Find "AWS EC2 Web Server"
3. Click **Request**
4. Fill in the form fields
5. Select your AWS credential
6. Submit
7. Monitor execution in **My Requests** or **Executions**

### Test via API

```bash
# Submit a catalog request (triggers the flow → workflow → terraform)
curl -X POST "${CMP_URL}/api/v1/catalog/<CATALOG_ID>/request" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "form_data": {
      "name": "test-web-server",
      "instance_type": "t3.micro",
      "ami_id": "ami-0c02fb55956c7d316",
      "vpc_id": "vpc-abc123",
      "subnet_id": "subnet-def456",
      "root_volume_size": 30
    },
    "credential_id": "<YOUR_AWS_CREDENTIAL_ID>"
  }'
```

### What Happens Behind the Scenes

```
1. Catalog request submitted with form_data
2. Flow starts → Workflow executes
3. Terraform step triggers TerraformEngine:
   a. Resolves credential (decrypts AWS keys)
   b. Clones git repo (or uses inline HCL)
   c. Generates terraform.tfvars from form_data
   d. Runs: terraform init
   e. Runs: terraform plan
   f. (If approval required: pauses for approval)
   g. Runs: terraform apply
   h. Captures outputs (instance_id, private_ip, etc.)
   i. Creates a Workspace record (status: active)
   j. Registers resources in Inventory
4. User sees provisioned resources in their dashboard
```

### Verify the Workspace

```bash
# List workspaces to see the provisioned instance
curl "${CMP_URL}/api/v1/terraform/workspaces" \
  -H "Authorization: Bearer ${TOKEN}"
```

---

## Quick Reference — API Calls Summary

| Step | API Endpoint | Method | Key Fields |
|------|-------------|--------|------------|
| Register Module | `/api/v1/terraform/modules` | POST | name, source_type, source_config, input_variables |
| Parse Git Source | `/api/v1/terraform/templates/parse-git-source` | POST | repo_url, branch, folder_path |
| Create Template | `/api/v1/terraform/templates` | POST | name, source_type, source_config, input_variables, supported_providers |
| Create Workflow | `/api/v1/workflows` | POST | name, steps[].action="terraform", steps[].template_id |
| Create Flow | `/api/v1/flows` | POST | name, workflows[].workflow_id, workflows[].order |
| Create Catalog | `/api/v1/catalog` | POST | name, flow_id, form_schema, status="published" |
| Submit Request | `/api/v1/catalog/{id}/request` | POST | form_data, credential_id |

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| 403 on template/module APIs | Terraform feature flag disabled | Enable `terraform_provisioning` in Feature Toggles |
| "Module not found" during apply | Wrong folder_path in source_config | Verify the path exists in your repo |
| Variables not mapping | field_name ≠ variable name | Ensure catalog form `field_name` matches TF variable names exactly |
| Git clone fails | Private repo without credential | Add a GitHub PAT credential and reference its ID in `source_config.auth_token` |
| Template delete rejected | Active workspaces exist | Destroy workspaces first, then delete template |
| Module delete rejected | Templates reference it | Remove module references from templates first |

---

## Tips

- **Use Parse from Git** to auto-populate variables/outputs instead of typing them manually.
- **Mark `day2_configurable: true`** on variables users should be able to change after initial provisioning (e.g., instance_type, volume_size).
- **Use `required_providers`** on templates to auto-generate `versions.tf` with provider constraints.
- **Tag everything** — modules, templates, and catalog items — for easy filtering.
- **Start with Git source** for templates. It's the most maintainable approach since you can update the repo and existing templates pick up changes on next apply.
- **For private repos**, create a credential of type `github_pat` first, then reference the `credential_id` in the template's `source_config.auth_token` field.

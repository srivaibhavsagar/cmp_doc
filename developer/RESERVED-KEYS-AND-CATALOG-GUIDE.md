# Reserved Keys & Catalog Development Guide

This document covers reserved form field keys, automatic context injection, and best practices for building catalog items (Day 1 provisioning) and resource actions (Day 2 operations) in CMP.

---

## Table of Contents

1. [Region Override Keys](#region-override-keys)
2. [Auto-Injected Context for Resource Actions](#auto-injected-context-for-resource-actions)
3. [Terraform-Based Catalogs](#terraform-based-catalogs)
4. [Native (Task-Based) Catalogs](#native-task-based-catalogs)
5. [Resource Actions (Day 2)](#resource-actions-day-2)
6. [Reserved Form Field Names](#reserved-form-field-names)
7. [Environment Variables Available in Tasks](#environment-variables-available-in-tasks)
8. [Best Practices](#best-practices)
9. [Inventory Record: What Gets Auto-Populated vs. Manual](#inventory-record-what-gets-auto-populated-vs-manual)
10. [Complete Example: Well-Configured Day 1 Catalog](#complete-example-well-configured-day-1-catalog)

---

## Region Override Keys

When a catalog form or resource action includes any of these field names, the platform **automatically overrides** the credential's default region for the execution:

| Field Name | Priority | Description |
|-----------|----------|-------------|
| `region` | 1 (highest) | Generic region field — works for all providers |
| `aws_region` | 2 | AWS-specific region |
| `location` | 3 | Azure-style location (also works for GCP) |
| `azure_region` | 4 | Azure-specific |
| `gcp_region` | 5 | GCP-specific |

**How it works:**

- The first matching non-empty key wins (priority order above).
- For **Terraform catalogs**: overrides `AWS_DEFAULT_REGION` env var and/or sets `TF_VAR_region`, `TF_VAR_location`.
- For **Native tasks**: overrides `CMP_CREDENTIAL_REGION`, `AWS_DEFAULT_REGION`, `AWS_REGION` env vars.
- For **Resource actions**: the resource's actual region is auto-injected as `region` — no user input needed.

**Example — Catalog form field:**
```json
{
  "field_id": "region_field",
  "label": "Deployment Region",
  "type": "cloud_region",
  "field_name": "region",
  "required": true,
  "default": "us-east-1",
  "description": "AWS region where resources will be provisioned"
}
```

---

## Auto-Injected Context for Resource Actions

When a resource action is executed from the resource detail page, the following fields are **automatically injected** into `form_data` — you do NOT need to ask the user for these:

| Key | Value Source | Example |
|-----|-------------|---------|
| `resource_id` | URL path param | `i-0abc123def456` |
| `credential_id` | URL path param | `uuid-of-credential` |
| `resource_type` | URL query param | `EC2`, `RDS`, `S3` |
| `region` | Cloud API lookup | `us-west-2` |
| `resource_region` | Same as `region` (legacy alias) | `us-west-2` |
| `resource_name` | Cloud API lookup | `my-web-server` |
| `resource_tags` | Cloud API lookup | `{"env": "prod", "team": "platform"}` |
| `provider` | Derived from credential | `aws`, `azure`, `gcp` |
| `resource_action_id` | Action definition | `uuid-of-action` |
| `resource_action_name` | Action definition | `Resize Instance` |
| `requested_at` | Server timestamp | `2025-01-15T10:30:00` |

For **flow-based** actions, these are also injected:

| Key | Value |
|-----|-------|
| `credential` | Full credential context (credential_id, provider, region, temp credentials) |
| `requested_by` | User context (user_id, username, email, roles, tenant_id) |

**Important:** Since `region` is auto-injected from the resource, the region override chain activates automatically. Your task/Terraform template will execute in the resource's region without any extra configuration.

---

## Terraform-Based Catalogs

### How Variables Flow

```
User fills form → form_data
                      ↓
Orchestrator merges: variables = {**form_data, **step_inputs}
                      ↓
TerraformEngine:
  1. Resolves credentials → env_vars (AWS_DEFAULT_REGION from credential)
  2. Applies region override from form_data → updates AWS_DEFAULT_REGION
  3. Generates terraform.tfvars.json from variables
  4. Runs: init → validate → plan → apply
```

### Region Handling

| Scenario | What Happens |
|----------|-------------|
| Template uses `provider "aws" { region = var.region }` | Works — `var.region` comes from form_data via tfvars |
| Template uses `provider "aws" {}` (no explicit region) | Works — `AWS_DEFAULT_REGION` env var is overridden from form_data |
| No region in form, credential has `us-east-1` | Uses credential default `us-east-1` |
| Form has `region = "eu-west-1"`, credential has `us-east-1` | Overrides to `eu-west-1` |

### Template Best Practices

```hcl
# RECOMMENDED: Declare a region variable for explicit control
variable "region" {
  description = "AWS region for resource deployment"
  type        = string
  default     = "us-east-1"
}

provider "aws" {
  region = var.region
}
```

Even without this, the platform sets `AWS_DEFAULT_REGION` from form_data, but declaring the variable makes the template self-documenting.

---

## Native (Task-Based) Catalogs

### How Context Flows

```
User fills form → form_data
                      ↓
Orchestrator: merged_inputs = {**form_data, **step_inputs}
                      ↓
TaskRunner builds cmp context:
  - cmp["credential"]["region"] = effective region (form_data override applied)
  - cmp["params"] = merged_inputs (all form values + step inputs)
                      ↓
For Bash: env vars set (AWS_DEFAULT_REGION, CMP_CREDENTIAL_REGION, etc.)
For Python: cmp dict injected as global variable
```

### Package Dependencies (Requirements)

When a task specifies requirements (package dependencies), they are installed before execution:

| Language | Docker Mode | Local Fallback |
|----------|-------------|----------------|
| Python | `pip install -r requirements.txt` inside container | Creates a temporary venv, installs packages, runs task within venv |
| Bash | `apt-get install` (switches to ubuntu:22.04) | Not supported locally |
| TypeScript | `npm install` inside container | Not supported locally |

**Python requirements format:** Standard pip format, one package per line (e.g., `boto3>=1.28.0`, `requests==2.31.0`).

The local fallback for Python uses a temporary `venv` directory that is automatically cleaned up after execution. This ensures tasks with dependencies work consistently whether Docker is available or not.

### Accessing Context in Python Tasks

```python
# cmp is automatically available as a global dict
params = cmp["params"]          # All form inputs
credential = cmp["credential"]  # Credential info (region, provider, temp creds)
user = cmp["user"]              # Who triggered this
catalog = cmp["catalog"]        # Catalog metadata

# Region is already resolved (form_data override applied)
region = credential["region"]   # e.g., "eu-west-1" (from form, not credential default)

# Use temp credentials for AWS
import boto3
ec2 = boto3.client(
    "ec2",
    region_name=credential["region"],
    aws_access_key_id=credential["temp_access_key_id"],
    aws_secret_access_key=credential["temp_secret_access_key"],
    aws_session_token=credential["temp_session_token"],
)
```

### Accessing Context in Bash Tasks

```bash
# Environment variables are pre-set:
echo $AWS_DEFAULT_REGION          # Already set to effective region
echo $AWS_REGION                  # Same as above
echo $CMP_CREDENTIAL_REGION      # Same as above
echo $CMP_CREDENTIAL_ACCESS_KEY  # STS temp access key
echo $CMP_CREDENTIAL_SECRET_KEY  # STS temp secret key
echo $CMP_CREDENTIAL_SESSION_TOKEN

# Full context as JSON:
echo $CMP_PARAMS | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['region'])"

# AWS CLI will automatically use the region from env vars:
aws ec2 describe-instances  # Uses $AWS_DEFAULT_REGION
```

---

## Resource Actions (Day 2)

### Terraform-Based Resource Actions

For Terraform Day 2 actions, the resource's metadata is available via **variable mappings**:

```json
{
  "terraform_variable_mappings": [
    { "variable_name": "instance_id", "source": "resource", "source_key": "resource_id" },
    { "variable_name": "region", "source": "resource", "source_key": "region" },
    { "variable_name": "new_size", "source": "input", "source_key": "instance_type" }
  ]
}
```

The `region` from the resource is also injected into `context["form_data"]`, so `AWS_DEFAULT_REGION` is automatically set to the resource's region.

### Flow-Based Resource Actions

The full resource context is in `form_data`. Access it in your task:

```python
# Python task inside a resource action flow
resource_id = params["resource_id"]
region = params["region"]              # Auto-injected from resource
provider = params["provider"]          # Auto-injected
resource_name = params["resource_name"]  # Auto-injected
resource_tags = params["resource_tags"]  # Auto-injected

# credential["region"] is already overridden to match the resource's region
# AWS_DEFAULT_REGION env var is also set correctly
```

---

## Reserved Form Field Names

These field names have special meaning in CMP. Avoid using them for unrelated purposes in your catalog forms:

### Inventory-Critical Keys (Day 1 Provisioning)

These keys directly control what gets stored in the inventory table after a successful provisioning. Setting them correctly ensures your resources are properly tracked, searchable, and manageable.

| Key | Inventory Field | Description | Required? |
|-----|----------------|-------------|-----------|
| `region` | `region` | Deployment region (also triggers region override) | Highly recommended |
| `aws_region` | `region` | AWS-specific alias for region | Alternative to `region` |
| `location` | `region` | Azure-style alias for region | Alternative to `region` |
| `resource_type` | `resource_type` | Type of resource being provisioned (e.g., `ec2`, `vm`, `rds`, `s3`) | Highly recommended |
| `cloud_provider` | `cloud_provider` | Cloud provider name (`aws`, `azure`, `gcp`) | Recommended (falls back to credential provider) |
| `provider` | `cloud_provider` | Alias for `cloud_provider` | Alternative |
| `resource_name` | `resource_name` | Human-readable name for the resource | Recommended |
| `name` | `resource_name` | Alias for `resource_name` | Alternative |
| `instance_name` | `resource_name` | Alias for `resource_name` | Alternative |
| `vm_name` | `resource_name` | VM-specific alias (checked in step outputs) | Alternative |
| `tags` | `tags` | Key-value tags for the resource (dict/object) | Optional |
| `group_name` | `group_name` | Team/group that owns this resource | Optional (auto-resolved from user's groups) |
| `group` | `group_name` | Alias for `group_name` | Alternative |

**Example — Well-configured catalog form for EC2:**
```json
[
  { "field_name": "name", "label": "Instance Name", "type": "string", "required": true },
  { "field_name": "region", "label": "Region", "type": "cloud_region", "required": true, "default": "us-east-1" },
  { "field_name": "resource_type", "label": "Resource Type", "type": "hidden", "default": "ec2" },
  { "field_name": "cloud_provider", "label": "Provider", "type": "hidden", "default": "aws" },
  { "field_name": "instance_type", "label": "Instance Size", "type": "select", "options": [...] }
]
```

### Step Output Keys (Task/Terraform Outputs That Feed Inventory)

Your task or Terraform template should output these keys in its JSON stdout (tasks) or Terraform outputs. The orchestrator checks them to populate the inventory record:

| Output Key | Inventory Field | Description | Priority |
|-----------|----------------|-------------|----------|
| `instance_id` | `instance_id` | The provisioned resource's cloud ID (e.g., `i-0abc123`) | 1st (highest) |
| `resource_id` | `instance_id` | Alias for instance_id | 2nd |
| `id` | `instance_id` | Generic ID | 3rd |
| `vm_id` | `instance_id` | Azure VM ID | 4th |
| `bucket_name` | `instance_id` | S3 bucket name (also sets resource_type to `s3`) | 5th |
| `arn` | `instance_id` | AWS ARN (also infers region and resource_type) | 6th |
| `resource_name` | `resource_name` | Human-readable name | 1st |
| `name` | `resource_name` | Alias | 2nd |
| `instance_name` | `resource_name` | Alias | 3rd |
| `vm_name` | `resource_name` | Alias | 4th |
| `resource_type` | `resource_type` | Explicit type | 1st |
| `type` | `resource_type` | Alias | 2nd |
| `service_type` | `resource_type` | Alias | 3rd |
| `resource_kind` | `resource_type` | Alias | 4th |
| `region` | `region` | Deployment region | Fallback after form_data |
| `location` | `region` | Azure-style region | Fallback |
| `availability_zone` | `region` | AZ (trailing letter stripped: `us-east-1a` → `us-east-1`) | Fallback |
| `tags` | `tags` | Resource tags dict | Fallback after form_data |
| `workspace_id` | `terraform_workspace_id` | Terraform workspace link | Auto-linked |

**Critical:** If your task does not output at least one of `instance_id`, `resource_id`, `id`, `vm_id`, `bucket_name`, `arn`, or `name` — **no inventory record will be created**. The orchestrator skips inventory creation when it cannot extract an instance ID.

### Resource Type Auto-Inference

If `resource_type` is not set in form_data or step outputs, the platform infers it from:

| Pattern | Inferred Type |
|---------|--------------|
| `instance_id` starts with `i-` | `ec2` |
| `instance_id` starts with `vol-` | `ebs` |
| `instance_id` starts with `sg-` | `security_group` |
| `instance_id` starts with `vpc-` | `vpc` |
| `instance_id` starts with `subnet-` | `subnet` |
| `instance_id` starts with `/subscriptions/` | `vm` (Azure) |
| `bucket_name` is present | `s3` |
| ARN contains service name | Service from ARN (e.g., `ec2`, `rds`, `s3`) |

### Region Override Keys (trigger automatic region override)

| Key | Effect |
|-----|--------|
| `region` | Overrides `AWS_DEFAULT_REGION` / `TF_VAR_region` |
| `aws_region` | Same as `region` (AWS-specific) |
| `location` | Overrides Azure location / `TF_VAR_location` |
| `azure_region` | Same as `location` |
| `gcp_region` | Sets `TF_VAR_gcp_region` / `CLOUDSDK_COMPUTE_REGION` |

### Resource Action Auto-Injected Keys (do not use as form field names)

| Key | Injected By |
|-----|-------------|
| `resource_id` | Resource action executor |
| `credential_id` | Resource action executor |
| `resource_type` | Resource action executor |
| `resource_region` | Resource action executor |
| `resource_name` | Resource action executor |
| `resource_tags` | Resource action executor |
| `provider` | Resource action executor |
| `resource_action_id` | Resource action executor |
| `resource_action_name` | Resource action executor |
| `requested_at` | Resource action executor |
| `credential` | Flow-based action executor (full credential context object) |
| `requested_by` | Flow-based action executor (user context object) |

### Orchestrator Internal Keys (stripped from task params)

| Key | Purpose |
|-----|---------|
| `task_id` | Internal routing — never forwarded to task code |
| `template_id` | Terraform step routing |
| `operation` | Terraform operation type |
| `workspace_id` | Terraform workspace reference |
| `approval_required` | Approval flow control |
| `timeout_seconds` | Execution timeout |

### Catalog Marker Keys (auto-injected by orchestrator)

| Key | Value |
|-----|-------|
| `created_via_catalog` | `true` when execution is from a catalog item |
| `catalog_name` | Name of the catalog item |
| `catalog_id` | ID of the catalog item |

---

## Environment Variables Available in Tasks

### Always Set (All Providers)

| Variable | Description |
|----------|-------------|
| `CMP_TASK_ID` | Current task ID |
| `CMP_TASK_NAME` | Current task name |
| `CMP_PARAMS` | JSON string of all merged params (form_data + step inputs) |
| `CMP_VARS` | JSON string of admin-defined shared variables |
| `CMP_CONTEXT` | Full CMP context as JSON |
| `CMP_CREDENTIAL_PROVIDER` | `aws`, `azure`, or `gcp` |
| `CMP_CREDENTIAL_REGION` | Effective region (with form_data override applied) |
| `CMP_CREDENTIAL_ACCESS_KEY` | STS temp access key (AWS) or client ID (Azure) |
| `CMP_CREDENTIAL_SECRET_KEY` | STS temp secret key (AWS) or client secret (Azure) |
| `CMP_CREDENTIAL_SESSION_TOKEN` | STS session token (AWS only) |

### AWS-Specific (set when provider is `aws`)

| Variable | Description |
|----------|-------------|
| `AWS_DEFAULT_REGION` | Effective region — boto3 and AWS CLI use this automatically |
| `AWS_REGION` | Same as above (some SDKs prefer this) |

### Azure-Specific (set when provider is `azure`)

| Variable | Description |
|----------|-------------|
| `AZURE_LOCATION` | Effective region/location |

### GCP-Specific (set when provider is `gcp`)

| Variable | Description |
|----------|-------------|
| `CLOUDSDK_COMPUTE_REGION` | Effective region — gcloud CLI uses this |

---

## Best Practices

### Catalog Form Design

1. **Always include a `region` field** for Day 1 catalogs that deploy to a specific region. Use type `cloud_region` or `select` with region options.

2. **Don't duplicate auto-injected fields** in resource action input schemas. The platform already provides `resource_id`, `region`, `provider`, etc.

3. **Use `field_name` (not `field_id`)** as the key that flows into `form_data`. The `field_name` must be lowercase with underscores only.

4. **Set sensible defaults** for region fields — use the credential's primary region or the most common deployment region.

### Terraform Templates

1. **Declare a `region` variable** even though the platform sets `AWS_DEFAULT_REGION`. This makes the template portable and self-documenting.

2. **Don't hardcode regions** in provider blocks. Always use a variable or rely on the env var.

3. **For multi-region deployments**, use separate variables (e.g., `primary_region`, `secondary_region`) — only the standard `region` key triggers the automatic override.

4. **Variable mappings for Day 2 actions** should map `region` from the resource attributes so the Terraform operation targets the correct region:
   ```json
   { "variable_name": "region", "source": "resource", "source_key": "region" }
   ```

### Native Tasks (Python/Bash)

1. **Prefer `credential["region"]`** over hardcoding — it already has the form_data override applied.

2. **For AWS, rely on env vars** — `AWS_DEFAULT_REGION` is set automatically. boto3 and AWS CLI pick it up without explicit configuration.

3. **Don't store long-lived credentials** — use `credential["temp_access_key_id"]` and related STS fields. They expire automatically.

4. **Access prior step outputs** via `cmp["task_outputs"]` (Python) or parse `$CMP_CONTEXT` (Bash).

5. **Output JSON to stdout** — the orchestrator parses your task's stdout as JSON and makes it available to downstream steps via `{{steps.<step_id>.<key>}}`.

### Resource Actions

1. **Don't ask for region in the input schema** — it's auto-injected from the resource. Only add a region field if the action deploys to a *different* region than the source resource.

2. **Don't ask for credential_id** — it's auto-injected from the resource's parent account.

3. **Use `default_params`** on the action definition for static values that should always be passed (e.g., `{"force": true}`).

4. **Test with the resource context** — remember that `params` in your task will contain both user inputs AND auto-injected resource metadata.

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Task deploys to wrong region | Ensure form has a `region` field, or rely on auto-injection for resource actions |
| Terraform ignores form region | Template must use `var.region` in provider block OR rely on `AWS_DEFAULT_REGION` (don't set region to a hardcoded value in provider) |
| Resource action can't find resource | Don't override `resource_id` in your input schema — it's auto-injected |
| Credential region doesn't match resource | For resource actions, the resource's region auto-overrides the credential default |
| Task can't authenticate | Use temp credentials from `cmp["credential"]`, not raw access/secret keys |
| Form field value not reaching task | Check `field_name` is set on the form field (not just `field_id`) |
| Step output not available downstream | Task must print valid JSON to stdout — non-JSON output is ignored |
| **No inventory record created** | Task must output at least one of: `instance_id`, `resource_id`, `id`, `vm_id`, `bucket_name`, `arn`, or `name` |
| **Inventory shows "unknown" resource_type** | Set `resource_type` in form_data (hidden field) or output it from your task |
| **Inventory shows "unknown" cloud_provider** | Set `cloud_provider` in form_data (hidden field) — falls back to credential provider |
| **Inventory shows "unknown" resource_name** | Set `name` or `resource_name` in form_data, or output it from your task |
| **Resource not in correct group** | Set `group_name` in form_data, or ensure the user belongs to a group |

---

## Inventory Record: What Gets Auto-Populated vs. Manual

| Field | Auto-Populated? | Source |
|-------|----------------|--------|
| `request_id` | ✅ | execution_id |
| `resource_name` | ✅ | form_data or step outputs |
| `resource_type` | ✅ | form_data > step outputs > auto-infer |
| `cloud_provider` | ✅ | form_data > catalog_info > credential |
| `region` | ✅ | form_data > step outputs > ARN inference |
| `instance_id` | ✅ | step outputs (required for inventory creation) |
| `owner_user_id` | ✅ | Current user |
| `owner_username` | ✅ | Current user |
| `group_name` | ✅ | form_data > user's first group |
| `tags` | ✅ | form_data > step outputs > catalog tags |
| `catalog_id` | ✅ | Catalog metadata |
| `catalog_name` | ✅ | Catalog metadata |
| `inputs_provided` | ✅ | Entire form_data dict |
| `execution_payload` | ✅ | form_data + step_logs |
| `terraform_workspace_id` | ✅ | Linked from step outputs |
| `ip_address` | ❌ | Must be set via Day 2 update or resource sync |
| `os_type` | ❌ | Must be set manually or via Day 2 action |
| `os_version` | ❌ | Must be set manually or via Day 2 action |
| `middleware_type` | ❌ | Must be set manually or via Day 2 action |
| `middleware_version` | ❌ | Must be set manually or via Day 2 action |
| `lease_date` | ❌ | Set via lease management |
| `expire_date` | ❌ | Set via lease management |
| `group_id` | ❌ | Auto-resolved from group_name during creation |

---

## Complete Example: Well-Configured Day 1 Catalog

### Form Schema (ensures all inventory fields are populated)

```json
[
  {
    "field_name": "name",
    "label": "Resource Name",
    "type": "string",
    "required": true,
    "placeholder": "e.g., prod-web-server-01"
  },
  {
    "field_name": "region",
    "label": "Deployment Region",
    "type": "cloud_region",
    "required": true,
    "default": "us-east-1"
  },
  {
    "field_name": "resource_type",
    "label": "Resource Type",
    "type": "hidden",
    "default": "ec2"
  },
  {
    "field_name": "cloud_provider",
    "label": "Cloud Provider",
    "type": "hidden",
    "default": "aws"
  },
  {
    "field_name": "instance_type",
    "label": "Instance Size",
    "type": "select",
    "required": true,
    "options": [
      { "label": "Small (t3.small)", "value": "t3.small" },
      { "label": "Medium (t3.medium)", "value": "t3.medium" },
      { "label": "Large (t3.large)", "value": "t3.large" }
    ]
  },
  {
    "field_name": "group_name",
    "label": "Owning Team",
    "type": "select",
    "required": false,
    "description": "Team that owns this resource (auto-detected if not set)"
  }
]
```

### Task Output (ensures inventory record is created)

```python
import json

# ... provisioning logic ...

# Output MUST include at least instance_id for inventory creation
print(json.dumps({
    "instance_id": "i-0abc123def456789",   # REQUIRED — triggers inventory creation
    "name": "prod-web-server-01",           # Populates resource_name
    "region": "us-east-1",                  # Confirms region in inventory
    "resource_type": "ec2",                 # Explicit type
    "tags": {"env": "production", "team": "platform"},
    "ip_address": "10.0.1.42",             # Bonus: useful for reference
}))
```

### Terraform Output (ensures inventory record is created)

```hcl
output "instance_id" {
  value       = aws_instance.main.id
  description = "EC2 instance ID — required for CMP inventory"
}

output "name" {
  value = aws_instance.main.tags["Name"]
}

output "region" {
  value = var.region
}
```

---

## Quick Reference: Region Resolution Priority

```
For Day 1 Catalogs:
  form_data["region"] > form_data["aws_region"] > form_data["location"] > credential.region > "us-east-1"

For Resource Actions:
  resource.region (auto-injected as form_data["region"]) > credential.region > "us-east-1"

For Terraform Day 2:
  resource_attributes["region"] (injected into context.form_data) > credential.region > "us-east-1"
```

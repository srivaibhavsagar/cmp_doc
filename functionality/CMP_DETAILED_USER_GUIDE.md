# Cloud Management Platform — Detailed User Guide

> **Version:** 1.0  
> **Audience:** Platform users, developers, and administrators  
> **Purpose:** Complete reference for every feature, field, and workflow in the Cloud Management Platform (CMP)

---

## Table of Contents

1. [Service Catalog](#1-service-catalog)
2. [Tasks](#2-tasks)
3. [Workflows](#3-workflows)
4. [Flows](#4-flows)
5. [Executions](#5-executions)
6. [Credentials (Cloud Accounts)](#6-credentials-cloud-accounts)
7. [Resources](#7-resources)
8. [Resource Actions](#8-resource-actions)
9. [Shared Variables (Context)](#9-shared-variables-context)
10. [Terraform Templates](#10-terraform-templates)
11. [Terraform Workspaces](#11-terraform-workspaces)
12. [Policies](#12-policies)
13. [Quotas](#13-quotas)
14. [Approvals](#14-approvals)
15. [Cost Models](#15-cost-models)
16. [Budgets](#16-budgets)
17. [Event Automation](#17-event-automation)
18. [Users & Groups (Admin)](#18-users--groups-admin)
19. [SSO Configuration](#19-sso-configuration)
20. [Tenants](#20-tenants)
21. [Scheduled Jobs](#21-scheduled-jobs)
22. [Notifications & Channels](#22-notifications--channels)
23. [Webhooks](#23-webhooks)
24. [Feature Toggles](#24-feature-toggles)
25. [Catalog Components (Reusable)](#25-catalog-components-reusable)
26. [Reports & Analytics](#26-reports--analytics)
27. [Profile & Settings](#27-profile--settings)
28. [AI Assistant](#28-ai-assistant)

---


## 1. Service Catalog

The Service Catalog is the primary self-service interface for all users. It presents a curated collection of pre-configured cloud services, infrastructure templates, and operational actions that users can order with a few clicks.

---

### 1.1 Browsing the Catalog

When you navigate to **Service Catalog** from the main navigation, you see a grid of catalog item cards.

**How to find items:**
- **Search bar** — Type keywords to filter items by name or description
- **Tag filters** — Click tags to narrow results (e.g., "aws", "networking", "database")
- **Status filter** — Only "Published" items are visible to regular users; Admins/Developers can also see Draft and Archived items

**Understanding catalog item cards:**

Each card displays:

| Element | Description |
|---------|-------------|
| Name | The service or action name (e.g., "AWS EC2 Instance") |
| Description | A brief summary of what the item provisions or does |
| Tags | Color-coded labels for categorization (e.g., "aws", "compute", "production") |
| Status Badge | Draft (gray), Published (green), or Archived (yellow) |
| Status Message | Optional warning/info banner (e.g., "Maintenance window: Saturdays 2-4 AM") |
| Cost Indicator | Shows if cost estimation is available |

---

### 1.2 Creating a Catalog Item (Admin/Developer)

To create a new catalog item, navigate to **Service Catalog → Create New** (or click the "+" button). The creation form is divided into several sections.

---

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Display name for the catalog item. Maximum 200 characters. Example: "AWS EC2 Instance" |
| **Description** | No | Detailed explanation of what this item does. Maximum 2000 characters. Supports plain text. |
| **Tags** | No | Categorization labels. Add up to 50 tags. Type and press Enter to add each tag. Examples: "aws", "compute", "production" |
| **Catalog Type** | Yes | Choose between **Day 1** or **Day 2** (see below) |
| **Status** | Yes | Controls visibility: Draft, Published, or Archived |
| **Status Message** | No | Optional banner message shown to users viewing this item |
| **Status Message Severity** | No | "warning" (yellow banner) or "info" (blue banner) |

**Catalog Type explained:**

- **Day 1 (Provisioning)** — Used for creating new resources. When a user orders this item, they will be prompted to select a cloud credential (AWS account, Azure subscription, etc.) to provision against.
- **Day 2 (Operational Actions)** — Used for actions on existing resources (e.g., restart, scale, backup). No credential selection is needed because the action targets an already-provisioned resource.

**Status lifecycle:**

1. **Draft** — Item is being configured. Only visible to Admins and Developers.
2. **Published** — Item is live and visible to all allowed users.
3. **Archived** — Item is retired. Not orderable but preserved for audit history.

---

#### UI Mode

Choose how the order form is presented to users:

| Mode | Description |
|------|-------------|
| **Form Builder** | Use the built-in drag-and-drop visual form designer. Recommended for most use cases. |
| **BYOUI (Bring Your Own UI)** | Provide custom HTML/JavaScript that renders your own form. Must call a submit contract to pass data back to CMP. |

**BYOUI Mode details:**
- Paste your custom HTML/JS into the "Custom UI HTML" editor
- Define a **Submit Contract** — a mapping of expected field names and their types that your custom UI will submit
- Your custom UI must call the CMP submit function with a JSON object matching the contract

---

#### Form Builder — Field Types

When using Form Builder mode, you construct the order form by adding fields. Each field type serves a specific purpose:

| Field Type | Description | Use Case Example |
|------------|-------------|-----------------|
| **String** | Single-line text input | Instance name, project ID |
| **Number** | Numeric input with optional min/max constraints | CPU count, disk size in GB |
| **Boolean** | Checkbox or toggle switch | "Enable public IP", "Auto-backup" |
| **Select** | Single-choice dropdown menu | Instance type, region |
| **Multiselect** | Multi-choice dropdown (select multiple values) | Security groups, availability zones |
| **Radio** | Radio button group (single choice, all options visible) | Environment: Dev/Staging/Prod |
| **Date** | Date picker calendar widget | Lease expiry date, maintenance window |
| **Email** | Text input with email format validation | Notification email, owner email |
| **URL** | Text input with URL format validation | Callback URL, repository URL |
| **Password** | Masked text input (dots/asterisks) | Database password, API key |
| **Range** | Slider control with min/max values | Memory allocation (1-64 GB) |
| **File** | File upload control | SSH key file, configuration file |
| **Hidden** | Invisible field with a pre-set value | Internal tracking IDs, fixed parameters |
| **Readonly** | Visible but non-editable field | Calculated values, system-assigned IDs |
| **Secret** | Encrypted value field (stored securely) | Sensitive credentials, tokens |
| **Cloud Region** | Special dropdown pre-populated with cloud provider regions | AWS/Azure/GCP region selection |
| **Textarea** | Multi-line text input | User data scripts, descriptions, notes |
| **Table** | Dynamic table where users can add/remove rows with configurable columns | Tag key-value pairs, port mappings, environment variables |

---

#### Form Field Configuration

Every field you add to the form has the following configuration options:

| Setting | Description |
|---------|-------------|
| **Field ID** | Unique identifier within this form (auto-generated, editable). Used internally to reference this field. |
| **Label** | The display name shown to users above the field (e.g., "Instance Type") |
| **Field Name** | The backend key used when submitting data. Must be lowercase letters and underscores only (e.g., "instance_type") |
| **Required** | Yes/No — Whether the user must fill this field before submitting |
| **Default Value** | Pre-filled value when the form loads. Can be any valid value for the field type. |
| **Placeholder** | Gray hint text shown inside an empty field (e.g., "Enter instance name...") |
| **Description / Help Text** | Explanatory text shown below the field to guide users |
| **Reusable Component** | Reference a pre-built Catalog Component instead of configuring from scratch |

**Validation Rules:**

| Rule | Applies To | Description |
|------|-----------|-------------|
| **min** | Number, Range, String (length) | Minimum allowed value or character count |
| **max** | Number, Range, String (length) | Maximum allowed value or character count |
| **pattern** | String, Email, URL | Regular expression the value must match |

**Conditional Display (depends_on):**

Make a field appear or disappear based on other fields' values. The `depends_on` field accepts a JSON object with several supported formats:

**Single Condition:**

```json
{"field": "environment", "operator": "equals", "value": "prod"}
```

This shows the field only when the "environment" field equals "prod".

**Multiple Conditions — ALL (AND logic):**

Use the `all` wrapper when every condition must be true:

```json
{
  "all": [
    {"field": "environment", "operator": "equals", "value": "prod"},
    {"field": "region", "operator": "equals", "value": "us-east-1"}
  ]
}
```

The field is shown only when environment is "prod" AND region is "us-east-1".

**Multiple Conditions — ANY (OR logic):**

Use the `any` wrapper when at least one condition must be true:

```json
{
  "any": [
    {"field": "environment", "operator": "equals", "value": "prod"},
    {"field": "environment", "operator": "equals", "value": "staging"}
  ]
}
```

The field is shown when environment is either "prod" OR "staging".

**Shorthand Object Syntax (implicit AND with equals):**

```json
{"environment": "prod", "region": "us-east-1"}
```

This is a shorthand equivalent to "all equals" — the field is shown only when environment is "prod" AND region is "us-east-1".

**Supported Operators:**

| Operator | Description | Example Value |
|----------|-------------|---------------|
| `equals` | Exact match (default if operator is omitted) | `"prod"` |
| `not_equals` | Value does not match | `"dev"` |
| `in` | Value is in an array | `["prod", "staging"]` |
| `not_in` | Value is not in an array | `["test", "sandbox"]` |
| `contains` | String/array contains the value | `"prod"` |
| `exists` | Field has any non-null value | (no value needed) |
| `not_empty` | Field is not empty or blank | (no value needed) |

**Examples:**

- Show "Subnet ID" only when "Enable VPC" is checked:
  ```json
  {"field": "enable_vpc", "operator": "equals", "value": true}
  ```

- Show "Production Approval Ticket" only when environment is "prod" AND region is a US region:
  ```json
  {
    "all": [
      {"field": "environment", "operator": "equals", "value": "prod"},
      {"field": "region", "operator": "in", "value": ["us-east-1", "us-east-2", "us-west-1", "us-west-2"]}
    ]
  }
  ```

- Show "Backup Schedule" when environment is either "prod" or "staging":
  ```json
  {
    "any": [
      {"field": "environment", "operator": "equals", "value": "prod"},
      {"field": "environment", "operator": "equals", "value": "staging"}
    ]
  }
  ```

---

#### Options Configuration (Select / Multiselect / Radio)

For dropdown and radio fields, you need to define the available choices. Two sources are supported:

**Static Options:**

Manually define label/value pairs:

| Label (shown to user) | Value (submitted) |
|-----------------------|-------------------|
| Small (t3.micro) | t3.micro |
| Medium (t3.medium) | t3.medium |
| Large (t3.large) | t3.large |

**API-Sourced Options:**

Fetch options dynamically from an external API:

| Setting | Description |
|---------|-------------|
| **URL** | The API endpoint to call (e.g., `https://api.example.com/regions`) |
| **Method** | HTTP method: GET, POST, PUT, PATCH |
| **Response Path** | Dot-notation path to the array in the API response (e.g., `data.items`) |
| **Label Key** | Which field in each item to use as the display label (default: "label") |
| **Value Key** | Which field in each item to use as the submitted value (default: "value") |
| **Label Template** | Template for complex labels using placeholders: `{{item.name}} ({{item.id}})` |
| **Value Template** | Template for complex values using placeholders: `{{item.id}}` |
| **Authentication** | None, Bearer Token, Basic Auth, or API Key |
| **Auth Credential** | Reference to a saved credential for authentication |
| **Body** | JSON body for POST/PUT/PATCH requests |
| **Response Transform** | JavaScript expression to transform the response into label/value pairs |

---

#### Form Layout

Organize your form fields into a structured layout:

**Sections:**
- Group related fields under collapsible section headers
- Each section has a **Title** (e.g., "Network Configuration")
- Sections can be **collapsible** (user can expand/collapse)
- Sections can be **collapsed by default** (starts minimized)

**Tabs:**
- Organize fields into tabbed panels for complex forms
- Each tab has a **Title** (e.g., "Basic", "Advanced", "Networking")
- Users click tab headers to switch between panels

**Rows:**
- Within sections or tabs, fields are arranged in rows
- Each row can contain **1 field** (full width) or **2 fields** (side by side)
- Use 2-field rows for related short fields (e.g., "Min Count" and "Max Count")

---

#### Access Control

Control who can see and order this catalog item:

| Setting | Description |
|---------|-------------|
| **Allowed Roles** | Which roles can access this item. Options: admin, developer, user, readonly. Default: admin, developer, user |
| **Allowed Groups** | Specific group IDs that can access this item. Leave empty for no group restriction. |

If both are configured, a user must match at least one role OR belong to at least one allowed group.

---

#### Cost Settings

| Setting | Description |
|---------|-------------|
| **Show Cost Estimate** | Toggle on/off. When enabled, users see an estimated cost before ordering based on Cost Models. |
| **Show Live Pricing** | Toggle on/off. When enabled, real-time cloud provider pricing is displayed. If Live Pricing is enabled but Cost Estimate is disabled, the widget appears with a "Live Pricing" header instead of "Cost Estimate". |

> **Note:** The pricing/cost widget appears whenever *either* setting is enabled. If only Live Pricing is on, the widget shows live cloud prices without the CMP cost model breakdown. If only Cost Estimate is on, the widget shows the cost model calculation. If both are on, the widget shows both sections under the "Cost Estimate" header.

---

#### Approval Configuration

| Setting | Description |
|---------|-------------|
| **Requires Approval** | Toggle. When enabled, orders go to an approval queue before execution. |
| **Approver User IDs** | Specific users who can approve requests for this item |
| **Approver Group IDs** | Groups whose members can approve requests |
| **Approver Roles** | Roles that can approve (e.g., "admin") |

**Conditional Approval:**

Instead of requiring approval for every order, you can set conditions. Approval is only required when the conditions match.

Each condition has:

| Setting | Description |
|---------|-------------|
| **Field Name** | The form field to evaluate (e.g., "instance_type") |
| **Operator** | Comparison: equals, not_equals, contains, greater_than, less_than, in |
| **Value** | The comparison value (e.g., "x1.16xlarge") |

**Condition Logic:**
- **ANY (OR)** — Approval required if ANY condition matches
- **ALL (AND)** — Approval required only if ALL conditions match

Example: Require approval only when `instance_type` equals "x1.16xlarge" OR `disk_size` is greater than 500.

**Approval Action Flows:**

| Setting | Description |
|---------|-------------|
| **On Approve Flow** | A Flow to trigger automatically when the request is approved |
| **On Reject Flow** | A Flow to trigger when the request is rejected |
| **On Cancel Flow** | A Flow to trigger when the request is cancelled |

---

#### Linked Flow

| Setting | Description |
|---------|-------------|
| **Flow ID** | Select the provisioning Flow that executes when this catalog item is ordered. This is the automation that actually creates/configures the cloud resources. |

---

### 1.3 Ordering from the Catalog (User)

**Step-by-step process:**

1. Navigate to **Service Catalog** from the main menu
2. Browse or search for the service you need
3. Click on the catalog item card to open it
4. Review the description and any status messages
5. Fill in the order form:
   - Complete all required fields (marked with *)
   - For Day 1 items: Select a cloud credential (AWS account, Azure subscription, etc.)
   - Review the cost estimate (if shown)
6. Click **Submit Order**
7. If approval is required:
   - Your request enters the approval queue
   - You receive a notification when approved/rejected
   - Once approved, execution begins automatically
8. If no approval needed:
   - Execution starts immediately
   - Track progress in the **Executions** section

---

### 1.4 Example: Creating an "AWS EC2 Instance" Catalog Item

**Basic Information:**
- Name: `AWS EC2 Instance`
- Description: `Provision a new EC2 instance in your AWS account with customizable instance type, storage, and networking options.`
- Tags: `aws`, `compute`, `ec2`, `virtual-machine`
- Catalog Type: Day 1 (Provisioning)
- Status: Published

**Form Builder Fields:**

| Field | Type | Required | Default | Options/Validation |
|-------|------|----------|---------|-------------------|
| Instance Name | String | Yes | — | Placeholder: "my-web-server" |
| Environment | Radio | Yes | dev | dev, staging, production |
| Instance Type | Select | Yes | t3.medium | t3.micro, t3.small, t3.medium, t3.large, t3.xlarge |
| Region | Cloud Region | Yes | us-east-1 | (auto-populated) |
| Storage Size (GB) | Number | Yes | 50 | Min: 20, Max: 16384 |
| Enable Public IP | Boolean | No | false | — |
| Key Pair Name | String | Yes | — | Pattern: `^[a-zA-Z0-9-_]+$` |
| Tags | Table | No | — | Columns: Key (string), Value (string) |

**Layout:**
- Section 1: "Instance Configuration" — Instance Name, Environment, Instance Type, Region
- Section 2: "Storage & Networking" — Storage Size, Enable Public IP, Key Pair Name
- Section 3: "Metadata" — Tags

**Access Control:**
- Allowed Roles: admin, developer, user

**Cost Settings:**
- Show Cost Estimate: Yes
- Show Live Pricing: Yes

**Approval:**
- Requires Approval: Conditional
- Condition: `instance_type` in ["t3.xlarge", "t3.2xlarge"] OR `storage_size` greater_than 500
- Approver Roles: admin

**Linked Flow:** `ec2-provisioning-flow`

---


## 2. Tasks

### 2.1 What are Tasks?

Tasks are the smallest unit of automation in CMP. A Task is a single script or action that performs one specific operation — such as creating an S3 bucket, running a database query, or calling an external API. Tasks are the building blocks that get assembled into Workflows and Flows.

---

### 2.2 Creating a Task

Navigate to **Provisioning → Tasks → Create New Task**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Descriptive name for the task. Max 200 characters. Example: "Create S3 Bucket" |
| **Description** | No | Explanation of what the task does. Max 2000 characters. |
| **Tags** | No | Categorization labels. Up to 50 tags. |

#### Language

Select the programming language for your task script:

| Language | Description | Best For |
|----------|-------------|----------|
| **Python** | Python 3.x script | AWS SDK (boto3), data processing, complex logic |
| **TypeScript** | TypeScript/Node.js script | API integrations, JSON processing |
| **Bash** | Shell script | CLI commands, file operations, quick automations |
| **Go** | Go program | High-performance operations, concurrent tasks |
| **SQL** | SQL query | Database operations, data queries |
| **REST** | HTTP request definition | Simple API calls without scripting |

#### Code Source

| Option | Description |
|--------|-------------|
| **Local (Inline)** | Write code directly in the built-in editor. Max 500KB. |
| **GitHub-Linked** | Pull code from a GitHub repository at execution time. |

**GitHub-Linked Settings:**

| Field | Description |
|-------|-------------|
| Repository | Full repo path (e.g., `org/repo-name`) |
| Branch | Branch to pull from (default: main) |
| File Path | Path to the script file within the repo |
| GitHub Credential | Select a saved GitHub credential for private repos |

#### Input Parameters (Input Schema)

Define parameters that can be passed to the task at runtime:

| Setting | Description |
|---------|-------------|
| **Parameter Name** | The variable name used in your script (e.g., `bucket_name`) |
| **Type** | Data type: string, number, boolean, object, array |
| **Default Value** | Value used if no input is provided |
| **Required** | Whether this parameter must be supplied |
| **Description** | Help text explaining the parameter |

#### Additional Settings

| Setting | Description |
|---------|-------------|
| **Requirements** | Package dependencies (e.g., `boto3==1.28.0` for Python). One per line. Max 10,000 characters. |
| **Python Version** | Specific Python version if needed. Supported versions: 3.10, 3.11, 3.12, 3.13, 3.14 (default: host Python) |
| **Write Output to Payload** | When enabled, task output is stored in the execution payload for downstream steps to use |

---

### 2.3 Running a Task

**Manual execution:**
1. Navigate to **Provisioning → Tasks**
2. Find your task in the list
3. Click the **Run** button (play icon)
4. Fill in any required input parameters
5. Click **Execute**
6. View real-time output in the execution log

**Automated execution:**
- Tasks run automatically when triggered by a Workflow step
- Tasks can be scheduled via Scheduled Jobs
- Tasks can be triggered by Event Automation rules

---

### 2.4 Example: Creating a "Create S3 Bucket" Task

**Basic Information:**
- Name: `Create S3 Bucket`
- Description: `Creates a new S3 bucket with versioning enabled and standard encryption.`
- Language: Python
- Tags: `aws`, `s3`, `storage`

**Input Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| bucket_name | string | Yes | — | Name for the new S3 bucket |
| region | string | Yes | us-east-1 | AWS region for the bucket |
| enable_versioning | boolean | No | true | Enable object versioning |
| tags | object | No | {} | Key-value tags to apply |

**Code:** (Python script using boto3 to create the bucket, enable versioning, and apply tags)

**Requirements:** `boto3>=1.28.0`

---

## 3. Workflows

### 3.1 What are Workflows?

A Workflow is an ordered sequence of steps that together accomplish a larger automation goal. Each step can run a Task, make an HTTP request, perform a cloud operation, or execute Terraform. Steps can run in parallel or sequentially based on dependencies, and you can configure failure handling for each step.

---

### 3.2 Creating a Workflow

Navigate to **Provisioning → Workflows → Create New Workflow**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Descriptive name (e.g., "EC2 Provisioning Workflow") |
| **Description** | No | Explanation of the workflow's purpose |
| **Tags** | No | Categorization labels |

#### Steps Configuration

Each step in a workflow has the following settings:

| Setting | Required | Description |
|---------|----------|-------------|
| **Step ID** | Yes | Unique identifier within this workflow (e.g., "create_vpc", "launch_instance") |
| **Name** | Yes | Human-readable step name |
| **Action** | Yes | What this step does (see Action Types below) |
| **Task ID** | Conditional | Required when action is "run_task" — selects which Task to execute |
| **Template ID** | Conditional | Required when action is "terraform" — selects which Terraform Template to use |
| **Inputs** | No | Key-value pairs passed to the step. Supports variable interpolation: `{{form.field_name}}`, `{{steps.step_id.output_key}}`, `{{credential.access_key}}` |
| **Depends On** | No | List of Step IDs that must complete before this step runs. Creates a DAG (directed acyclic graph) execution order. |
| **On Failure** | No | What happens if this step fails: **stop** (halt workflow), **continue** (skip and proceed), **rollback** (run rollback steps) |
| **Timeout (seconds)** | No | Maximum time this step can run before being killed |
| **Retry Count** | No | Number of times to retry on failure (default: 0) |
| **Output Key** | No | Custom key name for storing this step's output. Other steps reference it via `{{steps.output_key.field}}`. Defaults to the Step ID. |

#### Action Types

| Action | Description |
|--------|-------------|
| **run_task** | Execute a CMP Task (Python, TypeScript, Bash, Go, SQL, REST) |
| **http_request** | Make an HTTP request to any URL (GET, POST, PUT, DELETE) |
| **aws.ec2** | Perform AWS EC2 operations (run instances, stop, terminate, describe) |
| **aws.s3** | Perform AWS S3 operations (create bucket, put object, delete) |
| **aws.rds** | Perform AWS RDS operations (create DB, snapshot, modify) |
| **aws.lambda** | Invoke AWS Lambda functions |
| **azure.vm** | Perform Azure VM operations (create, start, stop, deallocate) |
| **gcp.compute** | Perform GCP Compute Engine operations (create instance, stop, delete) |
| **builtin.delay** | Wait for a specified duration (useful for propagation delays) |
| **builtin.condition** | Evaluate a condition and branch execution |
| **terraform** | Execute a Terraform plan/apply/destroy using a template |

#### Variable Interpolation in Inputs

Use double-curly-brace syntax to reference dynamic values:

| Pattern | Description | Example |
|---------|-------------|---------|
| `{{form.field_name}}` | Value from the catalog order form | `{{form.instance_type}}` |
| `{{steps.step_id.field}}` | Output from a previous step | `{{steps.create_vpc.vpc_id}}` |
| `{{credential.access_key}}` | Credential field from the selected cloud account | `{{credential.secret_key}}` |
| `{{credential.temp_access_token}}` | Short-lived OAuth2 token (GCP) or STS temp key (AWS) | `{{credential.project_id}}` |
| `{{context.variable_name}}` | Shared variable value | `{{context.default_region}}` |

**Provider-specific credential template variables:**

| Variable | AWS | Azure | GCP |
|----------|-----|-------|-----|
| `{{credential.temp_access_key_id}}` | STS access key | — | — |
| `{{credential.temp_secret_access_key}}` | STS secret key | — | — |
| `{{credential.temp_session_token}}` | STS session token | — | — |
| `{{credential.temp_access_token}}` | — | — | OAuth2 access token (1-hour TTL) |
| `{{credential.temp_token_expiration}}` | — | — | Token expiry (ISO timestamp) |
| `{{credential.project_id}}` | — | — | GCP project ID |
| `{{credential.region}}` | Effective region | Effective location | Effective region |

---

### 3.3 Example: Multi-step EC2 Provisioning Workflow

**Name:** `EC2 Full Provisioning`  
**Description:** `Creates a security group, launches an EC2 instance, and tags it.`

**Steps:**

| # | Step ID | Name | Action | Depends On | On Failure |
|---|---------|------|--------|-----------|------------|
| 1 | create_sg | Create Security Group | aws.ec2 | — | stop |
| 2 | launch_instance | Launch EC2 Instance | aws.ec2 | create_sg | rollback |
| 3 | tag_instance | Tag Instance | aws.ec2 | launch_instance | continue |
| 4 | notify_owner | Send Notification | run_task | tag_instance | continue |

**Step 1 Inputs:**
- group_name: `{{form.instance_name}}-sg`
- vpc_id: `{{form.vpc_id}}`
- ingress_rules: `[{"port": 22, "cidr": "10.0.0.0/8"}]`

**Step 2 Inputs:**
- instance_type: `{{form.instance_type}}`
- ami_id: `{{form.ami_id}}`
- security_group_id: `{{steps.create_sg.group_id}}`
- key_name: `{{form.key_pair_name}}`

**Step 3 Inputs:**
- instance_id: `{{steps.launch_instance.instance_id}}`
- tags: `{"Name": "{{form.instance_name}}", "Environment": "{{form.environment}}"}`

---

## 4. Flows

### 4.1 What are Flows?

A Flow chains multiple Workflows together into a complete end-to-end automation. While a Workflow handles a specific set of related steps, a Flow orchestrates the bigger picture — for example, first running a "Network Setup" workflow, then a "Compute Provisioning" workflow, then a "Monitoring Setup" workflow.

Flows are what you link to Catalog Items. When a user orders from the catalog, the linked Flow executes.

---

### 4.2 Creating a Flow

Navigate to **Provisioning → Flows → Create New Flow**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Flow name (e.g., "Complete VM Provisioning") |
| **Description** | No | What this flow accomplishes end-to-end |
| **Tags** | No | Categorization labels |

#### Workflow Chain

Add one or more workflows to the flow. For each workflow reference:

| Setting | Description |
|---------|-------------|
| **Workflow ID** | Select which workflow to include |
| **Order** | Execution order (1, 2, 3...). Workflows with the same order run in parallel. |
| **Depends On** | Other workflow IDs that must complete first (alternative to numeric ordering) |
| **Input Mapping** | Map values from the catalog form or previous workflow outputs into this workflow's inputs |

**Input Mapping examples:**
- Map form field directly: `instance_type` → `{{form.instance_type}}`
- Map from previous workflow output: `vpc_id` → `{{workflows.network_setup.vpc_id}}`
- Map credential values: `aws_region` → `{{credential.region}}`

#### Linking to Catalog Items

After creating a Flow, go to the Catalog Item's settings and set the **Linked Flow** field to this Flow's ID. When users order that catalog item, this Flow executes with the form data and selected credential.

---

### 4.3 Example: Complete VM Provisioning Flow

**Name:** `Full EC2 Provisioning Flow`  
**Description:** `End-to-end provisioning: network setup, instance launch, monitoring configuration.`

**Workflow Chain:**

| Order | Workflow | Input Mapping |
|-------|----------|---------------|
| 1 | Network Setup Workflow | vpc_cidr → `{{form.vpc_cidr}}`, region → `{{form.region}}` |
| 2 | EC2 Launch Workflow | subnet_id → `{{workflows.network_setup.subnet_id}}`, instance_type → `{{form.instance_type}}` |
| 3 | Monitoring Setup Workflow | instance_id → `{{workflows.ec2_launch.instance_id}}`, alert_email → `{{form.owner_email}}` |

---

## 5. Executions

### 5.1 Viewing Executions

Navigate to **Executions** from the main menu (or **Provisioning → Executions** for Admin/Developer).

The executions list shows all your provisioning runs with:
- Catalog item name
- Status badge (Pending, Running, Success, Failed, Cancelled)
- Submitted by (username)
- Start time and duration
- Cloud provider icon

**Filtering options:**
- By status (Pending, Running, Success, Failed, Cancelled)
- By catalog item
- By date range
- By submitter (Admin only)

---

### 5.2 Execution Detail View

Click any execution to see its full details:

**Header Information:**
- Execution ID
- Catalog item name
- Status with timestamp
- Submitted by (username and email)
- Cloud provider and credential used
- Total duration

**Step-by-Step Progress:**

Each workflow step is displayed with:
- Step name and status icon (pending ○, running ◐, success ✓, failed ✗)
- Start time and duration
- Expandable output/logs section
- Error message (if failed)

**Cost Tracking:**
- Estimated CMP charges (management fees, markups)
- Estimated total cost (cloud + CMP charges)
- Cost breakdown by rule
- Actual cloud cost (populated after resource termination)

**Dry-Run Mode:**

When an execution is submitted with dry-run enabled:
- No actual cloud operations are performed
- A preview shows what would happen at each step
- Useful for validating configurations before committing

---

### 5.3 Retrying Failed Executions

When an execution fails:
1. Open the execution detail view
2. Review the failed step's error message and logs
3. Click **Retry** to re-run from the failed step
4. The system resumes from where it left off (successful steps are not re-run)

Retry history is tracked — you can see how many times each step was attempted.

---

### 5.4 Cancelling Executions

For running or pending executions:
1. Open the execution detail view
2. Click **Cancel Execution**
3. Confirm the cancellation
4. The execution status changes to "Cancelled"
5. Any in-progress cloud operations may need manual cleanup

---


## 6. Credentials (Cloud Accounts)

Credentials represent your cloud provider accounts and authentication tokens. They are used when provisioning resources, syncing inventory, and performing cloud operations.

---

### 6.1 Adding a Credential

Navigate to **Infrastructure → Accounts & Credentials → Add Credential** (Admin/Developer) or **Credentials → Add** (User).

> **Tip:** You can also ask the AI Assistant to add a credential (e.g., "Add a new AWS credential"). The assistant will open the credential form securely in the main panel — it will never ask you to type secrets in chat. See [Section 28.6](#286-secure-credential-creation-via-chat) for details.

Select your provider type and fill in the required fields:

#### AWS

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Friendly name for this account (e.g., "Production AWS", "Dev Account"). Max 200 characters. |
| **Access Key** | Yes | AWS Access Key ID (starts with AKIA...) |
| **Secret Key** | Yes | AWS Secret Access Key (encrypted at rest) |
| **Region** | No | Default region for this account (e.g., "us-east-1") |
| **Regions** | No | List of all regions this account operates in |
| **Description** | No | Notes about this account. Max 1000 characters. |

#### Azure

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Friendly name (e.g., "Azure Production Subscription") |
| **Client ID** | Yes | Azure AD Application (client) ID |
| **Client Secret** | Yes | Azure AD Application secret (encrypted at rest) |
| **Tenant ID** | Yes | Azure AD Directory (tenant) ID |
| **Subscription ID** | Yes | Azure Subscription ID |
| **Resource Groups** | No | List of resource groups to manage |
| **Description** | No | Notes about this account |

#### GCP

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Friendly name (e.g., "GCP Analytics Project") |
| **Project ID** | Yes | GCP Project ID (e.g., `my-gcp-project-123`). Found in GCP Console → Project Settings. |
| **Service Account JSON Key** | Yes | Service account key in JSON format. Can be pasted directly or uploaded as a `.json` file. Private keys are encrypted before storage and never displayed after save. |
| **Environment** | No | Deployment environment label: Development, Test, UAT, or Production |
| **Default Region** | No | Default GCP region for operations. Regions are fetched dynamically when the dropdown is focused. |
| **Resource Discovery** | No | Enable automatic discovery of cloud resources in this project (checkbox) |
| **Cost Collection** | No | Enable cost data collection for this project (checkbox) |
| **Governance Scans** | No | Enable policy governance scanning for this project (checkbox) |
| **Description** | No | Notes about this account |

**Service Account JSON input modes:**
- **Paste JSON** — Paste the full JSON key content directly into the text area.
- **Upload File** — Click to upload a `.json` key file from your local machine.

**Test Connection:** Before saving, click **Test Connection** to verify the service account credentials. The button is enabled once Project ID and Service Account JSON are provided. A successful test confirms authentication and checks access to Compute Engine, Cloud Storage, and IAM APIs. The test passes if the service account can access at least one of these APIs, even if it lacks the `resourcemanager.projects.get` permission.

> **How to create a Service Account Key:** GCP Console → IAM & Admin → Service Accounts → Select or create an account → Keys tab → Add Key → Create new key (JSON). The service account needs at least one of: `Viewer`, `Compute Viewer`, `Storage Object Viewer`, or `Service Account User` role. For full project metadata in test results, also grant the `Browser` role (which provides `resourcemanager.projects.get`).

> **Security: Workflow Execution** — When a GCP credential is used in workflow executions, a short-lived OAuth2 access token (1-hour TTL) is generated from the service account key. Only this temporary token is forwarded to workflow steps via `{{credential.temp_access_token}}` and `{{credential.project_id}}`. The raw service account JSON key is never exposed to task code or step outputs.

#### GitHub

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Friendly name (e.g., "GitHub Org Token") |
| **Token** | Yes | Personal Access Token or Fine-Grained Token |
| **Description** | No | Notes about scope and permissions |

#### Bearer Token

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Friendly name (e.g., "Datadog API Token") |
| **Token** | Yes | Bearer token value (encrypted at rest) |
| **Description** | No | Notes about the service this token accesses |

#### Basic Auth

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Friendly name (e.g., "Jenkins Admin") |
| **Access Key** | Yes | Username |
| **Secret Key** | Yes | Password (encrypted at rest) |
| **Description** | No | Notes about the service |

---

### 6.2 Visibility Controls

Control who can see and use each credential:

| Setting | Description |
|---------|-------------|
| **Visible Roles** | Only users with these roles can see this credential. Leave empty for all roles. |
| **Visible User IDs** | Specific users who can see this credential |
| **Visible Group IDs** | Members of these groups can see this credential |

If no visibility restrictions are set, the credential is visible to all users in the tenant.

---

### 6.3 Validating Credentials

After adding a credential:
1. Click the **Validate** button on the credential card
2. CMP attempts to authenticate with the provider using the stored credentials
3. Results:
   - ✓ **Valid** — Connection successful, credentials are working
   - ✗ **Failed** — Authentication failed (check keys, permissions, or expiry)

Validation checks:
- **AWS:** Calls STS GetCallerIdentity to verify access key and secret
- **Azure:** Authenticates the service principal via OAuth2 token and verifies subscription access
- **GCP:** Parses the service account JSON, authenticates with Google OAuth2, verifies project access via Cloud Resource Manager, and reports API access status for Compute Engine, Cloud Storage, and IAM
- **GitHub:** Calls the authenticated user endpoint to verify the token and confirm username match

---

### 6.4 Example: Adding an AWS Account

1. Navigate to **Infrastructure → Accounts & Credentials**
2. Click **Add Credential**
3. Select Provider: **AWS**
4. Fill in:
   - Name: `Production AWS - US East`
   - Access Key: `AKIAIOSFODNN7EXAMPLE`
   - Secret Key: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`
   - Region: `us-east-1`
   - Regions: `us-east-1`, `us-west-2`, `eu-west-1`
   - Description: `Production account for US East workloads. IAM user: cmp-automation`
5. Set Visibility:
   - Visible Roles: `admin`, `developer`
   - Visible Groups: `platform-team`
6. Click **Save**
7. Click **Validate** to confirm the credentials work

---

### 6.5 Example: Adding a GCP Account

1. Navigate to **Infrastructure → Accounts & Credentials**
2. Click **Add Credential**
3. Select Provider: **GCP**
4. Fill in:
   - Name: `GCP Analytics - Production`
   - Project ID: `analytics-prod-123456`
   - Environment: `Production`
   - Default Region: `us-central1`
   - Enable: ☑ Resource Discovery, ☑ Cost Collection, ☑ Governance Scans
5. Provide the Service Account JSON:
   - Choose **Paste JSON** and paste the full key content, or
   - Choose **Upload File** and select your `.json` key file
6. Click **Test Connection** — wait for the green "Connection Verified ✓" confirmation
7. Set Visibility:
   - Visible Roles: `admin`, `developer`
   - Visible Groups: `data-engineering`
8. Click **Save**

---

## 7. Resources

### 7.1 Viewing Resources

Navigate to **Infrastructure → Resources** (Admin/Developer) or **Resources** (User).

The Resources page shows all cloud resources across your connected accounts:

**List View Features:**
- Sortable columns: Name, Type, Status, Region, Provider, Account
- Filter by: Provider, Resource Type, Status, Region
- Search by name or resource ID
- Pagination with configurable page size

**Resource Information Displayed:**

| Column | Description |
|--------|-------------|
| Name | Resource name or identifier |
| Type | Resource type (e.g., EC2 Instance, S3 Bucket, RDS Database) |
| Status | Current state (running, stopped, available, etc.) |
| Region | Cloud region where the resource exists |
| Provider | AWS, Azure, or GCP icon |
| Account | Which credential/account owns this resource |
| Created | When the resource was first detected |
| Tags | Key-value metadata tags |

---

### 7.2 Resource Detail

Click any resource to view its full details:

- **Overview** — Name, ID, type, status, region, account
- **Properties** — All resource-specific attributes (instance type, storage, networking, etc.)
- **Tags** — All metadata tags applied to the resource
- **Available Actions** — Native and custom actions you can perform (see Resource Actions)
- **Execution History** — Past provisioning and action executions related to this resource
- **Lease Information** — If the resource has a lease, shows expiry date and warnings

---

### 7.3 Performing Actions on Resources

From the resource detail page:
1. View the **Available Actions** section
2. Actions are divided into:
   - **Native Actions** — Built-in cloud operations (Start, Stop, Reboot, Terminate)
   - **Automation Actions** — Custom actions defined by admins (Scale Up, Backup, Patch)
3. Click an action button
4. Fill in any required parameters (if the action has an input form)
5. Select a credential (if required)
6. Confirm and execute
7. Track the action's progress in Executions

---

### 7.4 Resource Inventory & Leases

**Inventory** tracks all resources provisioned through CMP with additional metadata:

| Field | Description |
|-------|-------------|
| Resource Name | Name assigned during provisioning |
| Resource Type | Type of cloud resource |
| Cloud Provider | AWS, Azure, or GCP |
| Owner | User who provisioned the resource |
| Group | Team/group the resource belongs to |
| Instance ID | Cloud provider's resource identifier |
| IP Address | Assigned IP (if applicable) |
| Region | Deployment region |
| Status | Active, Stopped, Expired, or Destroyed |
| Catalog Name | Which catalog item was used to provision |

> **Note:** If a resource is marked "Destroyed" but a cloud provider sync later confirms it is actually running or stopped, CMP automatically recovers the inventory record to reflect the correct status. This can occur with GCP VMs whose TERMINATED state (meaning stopped) was previously misinterpreted as deleted.

**Lease Management:**

| Field | Description |
|-------|-------------|
| Lease Date | When the resource lease started |
| Expire Date | When the resource lease expires |
| Days Until Expiry | Countdown to expiration |

**Lease Warnings:**
- **3-day warning** — Notification sent 3 days before expiry
- **1-day warning** — Urgent notification sent 1 day before expiry
- **Expired** — Resource marked as expired; may be auto-cleaned depending on policy

---

## 8. Resource Actions

### 8.1 What are Resource Actions?

Resource Actions are custom operations that can be performed on existing cloud resources. They extend the built-in actions (start, stop, terminate) with organization-specific automations like "Scale Up", "Create Snapshot", "Apply Security Patch", or "Rotate Credentials".

---

### 8.2 Creating a Resource Action

Navigate to **Infrastructure → Resource Actions → Create New Action**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Action name shown to users (e.g., "Scale Up Instance") |
| **Description** | No | Explanation of what this action does |
| **Enabled** | Yes | Toggle to activate/deactivate this action |

#### Status Message

| Field | Description |
|-------|-------------|
| **Status Message** | Optional warning/info banner shown when users view this action |
| **Status Message Severity** | "warning" (yellow) or "info" (blue) |

#### Backend Type

| Type | Description |
|------|-------------|
| **Native** | Uses built-in cloud provider operations (start, stop, reboot, terminate) |
| **Flow** | Triggers a CMP Flow for complex multi-step operations |
| **Terraform** | Executes a Terraform operation (plan, apply, destroy, targeted-apply) |

#### Provider and Resource Type Targeting

| Field | Description |
|-------|-------------|
| **Providers** | Which cloud providers this action applies to (e.g., ["aws"]) |
| **Resource Types** | Which resource types this action appears on (e.g., ["ec2_instance", "rds_instance"]) |

#### Conditions

Define when this action should be available on a resource:

| Condition | Description |
|-----------|-------------|
| **Status In** | Only show when resource status is one of these (e.g., ["running"]) |
| **Status Not In** | Hide when resource status is one of these (e.g., ["terminated"]) |
| **Region In** | Only show for resources in these regions |
| **Name Contains** | Only show for resources whose name contains these strings |
| **Tag Equals** | Only show when resource has specific tag key-value pairs |
| **Tag Exists** | Only show when resource has these tag keys (any value) |

#### Input Schema (Custom Form)

Define a form that users fill in when executing this action:

| Field Setting | Description |
|---------------|-------------|
| **Key** | Parameter name (e.g., "new_instance_type") |
| **Label** | Display label (e.g., "New Instance Type") |
| **Type** | text, textarea, number, boolean, select, password |
| **Required** | Whether the field must be filled |
| **Placeholder** | Hint text |
| **Description** | Help text |
| **Default Value** | Pre-filled value |
| **Options** | For select type: list of label/value pairs |
| **Reusable Component** | Reference a Catalog Component |

#### Visibility

| Field | Description |
|-------|-------------|
| **Visible Roles** | Which roles can see and execute this action (default: user, developer, admin) |

#### Approval Settings

| Field | Description |
|-------|-------------|
| **Requires Approval** | Toggle — when enabled, action execution requires approval |
| **Approver User IDs** | Specific approvers |
| **Approver Group IDs** | Approver groups |
| **Approver Roles** | Roles that can approve |
| **Approval Conditions** | Conditional approval (same as catalog items) |
| **Approval Condition Logic** | ANY (OR) or ALL (AND) |

#### Cost Settings

| Field | Description |
|-------|-------------|
| **Show Cost Estimate** | Display estimated cost before execution |
| **Show Live Pricing** | Display real-time pricing information |

#### Terraform-Specific Settings

When Backend Type is "Terraform":

| Field | Description |
|-------|-------------|
| **Terraform Template ID** | Which template to use |
| **Workspace Resolution** | How to find the workspace: "inventory_link" (from resource's linked workspace) or "explicit" (specify workspace ID) |
| **Workspace ID** | Explicit workspace ID (when resolution is "explicit") |
| **Variable Mappings** | Map action inputs to Terraform variables. Up to 50 mappings. |
| **Operation** | What Terraform operation to perform: plan, apply, destroy, or targeted-apply |

---

### 8.3 Example: "Scale Up Instance" Resource Action

**Basic Information:**
- Name: `Scale Up Instance`
- Description: `Change the instance type to a larger size. Requires a brief restart.`
- Enabled: Yes

**Targeting:**
- Providers: `aws`
- Resource Types: `ec2_instance`

**Conditions:**
- Status In: `running`, `stopped`
- Status Not In: `terminated`, `pending`

**Backend Type:** Flow  
**Flow ID:** `ec2-resize-flow`

**Input Schema:**

| Key | Label | Type | Required | Options |
|-----|-------|------|----------|---------|
| new_instance_type | New Instance Type | select | Yes | t3.medium, t3.large, t3.xlarge, t3.2xlarge |
| confirm_restart | I understand this requires a restart | boolean | Yes | — |

**Visibility:** admin, developer  
**Requires Approval:** Yes (when new_instance_type in ["t3.xlarge", "t3.2xlarge"])

---

## 9. Shared Variables (Context)

### 9.1 What are Shared Variables?

Shared Variables (also called Context) are tenant-wide key-value pairs that can be referenced in Tasks, Workflows, and Flows. They store common configuration values like default regions, standard naming prefixes, team email addresses, or API endpoints — avoiding hardcoding these values in every automation.

---

### 9.2 Creating Variables

Navigate to **Infrastructure → Shared Variables**.

Click **Add Variable** and configure:

| Field | Description |
|-------|-------------|
| **Key** | Variable name (e.g., "default_region", "alert_email", "naming_prefix") |
| **Value** | The variable's value |
| **Description** | Optional explanation of what this variable is for |
| **Type** | **string** (plain text, visible) or **secret** (encrypted, masked in UI with •••) |

**String type:**
- Value is stored as plain text
- Visible to all users with access to Shared Variables
- Displayed in full in the UI

**Secret type:**
- Value is encrypted at rest
- Displayed as `••••••••` in the UI
- Only the actual value is injected at runtime (never exposed in logs)
- Use for API keys, passwords, tokens that shouldn't be visible

---

### 9.3 Using Variables in Tasks and Workflows

Reference shared variables in workflow step inputs using the interpolation syntax:

```
{{context.variable_key}}
```

**Examples:**
- `{{context.default_region}}` → resolves to "us-east-1"
- `{{context.alert_email}}` → resolves to "ops-team@company.com"
- `{{context.api_endpoint}}` → resolves to "https://internal-api.company.com"

Variables are resolved at execution time, so updating a variable immediately affects all future executions.

---

## 10. Terraform Templates

### 10.1 Creating a Template

Navigate to **Provisioning → Terraform Templates → Create New Template**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Template name. Max 128 characters. (e.g., "VPC with Public/Private Subnets") |
| **Description** | No | What infrastructure this template provisions. Max 1024 characters. |
| **Tags** | No | Categorization labels. Max 20 tags. |
| **Terraform Version** | No | Version constraint (e.g., ">= 1.5.0", "~> 1.6") |

#### Source Type

| Source | Description |
|--------|-------------|
| **Inline** | Paste HCL code directly. Max 1 MB. Good for simple templates. |
| **Git** | Pull from a Git repository. Best for version-controlled templates. |
| **Upload** | Upload a .zip archive containing Terraform files. |
| **Registry** | Reference a module from the Terraform Registry (e.g., "hashicorp/consul/aws"). |

**Git Source Settings:**

| Field | Description |
|-------|-------------|
| Repository URL | Full Git URL (e.g., `https://github.com/org/terraform-modules.git`) |
| Branch | Branch to use (default: "main") |
| Folder Path | Path within the repo to the Terraform files (default: ".") |
| Auth Token | Authentication token for private repos (encrypted) |

**Registry Source Settings:**

| Field | Description |
|-------|-------------|
| Registry Module | Module identifier (e.g., "hashicorp/consul/aws") |
| Registry Version | Specific version or constraint |

#### Input Variables

Define the variables your Terraform template accepts:

| Field | Description |
|-------|-------------|
| **Name** | Variable name as defined in your Terraform code (e.g., "vpc_cidr") |
| **Type** | string, number, bool, list, map, object |
| **Description** | Help text explaining this variable |
| **Default** | Default value (if any) |
| **Required** | Whether a value must be provided |
| **Sensitive** | Whether this variable contains secrets (masked in UI and logs) |
| **Day 2 Configurable** | Whether this variable can be modified after initial deployment (for Day 2 operations) |

#### Output Definitions

Define the outputs your Terraform template produces:

| Field | Description |
|-------|-------------|
| **Name** | Output name as defined in your Terraform code (e.g., "vpc_id") |
| **Description** | What this output represents |
| **Sensitive** | Whether this output contains secrets |

#### Required Providers

Specify which Terraform providers are needed:

| Provider | Version |
|----------|---------|
| aws | ">= 5.0" |
| azurerm | "~> 3.0" |
| google | ">= 4.0" |

---

### 10.2 Example: VPC Template from Git

**Basic Information:**
- Name: `AWS VPC with Public/Private Subnets`
- Description: `Creates a VPC with configurable CIDR, public and private subnets across multiple AZs, NAT gateway, and route tables.`
- Tags: `aws`, `networking`, `vpc`
- Terraform Version: `>= 1.5.0`

**Source:**
- Type: Git
- Repository URL: `https://github.com/company/terraform-modules.git`
- Branch: `main`
- Folder Path: `modules/aws-vpc`

**Input Variables:**

| Name | Type | Required | Default | Sensitive | Day 2 |
|------|------|----------|---------|-----------|-------|
| vpc_cidr | string | Yes | "10.0.0.0/16" | No | No |
| environment | string | Yes | — | No | No |
| availability_zones | list | Yes | — | No | No |
| enable_nat_gateway | bool | No | true | No | Yes |
| single_nat_gateway | bool | No | false | No | Yes |
| tags | map | No | {} | No | Yes |

**Outputs:**

| Name | Description | Sensitive |
|------|-------------|-----------|
| vpc_id | The ID of the created VPC | No |
| public_subnet_ids | List of public subnet IDs | No |
| private_subnet_ids | List of private subnet IDs | No |
| nat_gateway_ip | Elastic IP of the NAT gateway | No |

**Required Providers:**
- aws: ">= 5.0"

---

## 11. Terraform Workspaces

### 11.1 Deploying a Workspace

A Terraform Workspace represents a deployed instance of a Terraform Template. To deploy:

1. Navigate to **Infrastructure → Workspace Management**
2. Click **Deploy New Workspace**
3. Select a **Terraform Template** and version
4. Select a **Cloud Credential** for authentication
5. Fill in the **Variable Values** (the template's input variables)
6. Click **Deploy**

The workspace goes through these statuses:
- **Initializing** → Terraform init and plan running
- **Active** → Successfully deployed and operational
- **Errored** → Deployment failed (review logs and retry)

---

### 11.2 Day 2 Operations

Once a workspace is active, you can perform Day 2 operations to modify the deployed infrastructure:

| Operation | Description |
|-----------|-------------|
| **Variable Update** | Change the values of variables marked as "Day 2 Configurable" and re-apply |
| **Resource Addition** | Add new resources to the existing stack |
| **Resource Removal** | Remove specific resources from the stack |
| **Stack Destruction** | Completely destroy all resources managed by this workspace |

**To perform a Day 2 operation:**
1. Open the workspace detail page
2. Click **Day 2 Operations**
3. Select the operation type
4. For Variable Update: modify the configurable variables
5. Add a description explaining the change
6. Click **Apply**
7. Monitor the execution progress

---

### 11.3 Drift Detection

Drift detection identifies when the actual cloud infrastructure has diverged from the Terraform state (e.g., someone manually changed a resource outside of CMP).

**Drift Status Values:**
- **Unknown** — Drift check has not been run
- **No Drift** — Infrastructure matches the Terraform state
- **Drift Detected** — Differences found between actual and expected state

**To check for drift:**
1. Open the workspace detail page
2. Click **Check Drift**
3. Review the drift report showing what changed
4. Choose to either:
   - **Reconcile** — Update Terraform state to match reality
   - **Re-apply** — Force infrastructure back to the desired state

---

### 11.4 Workspace Lifecycle

| Status | Description | Available Actions |
|--------|-------------|-------------------|
| **Initializing** | First deployment in progress | Wait, or cancel |
| **Active** | Successfully deployed | Day 2 ops, drift check, destroy |
| **Updating** | Day 2 operation in progress | Wait |
| **Destroying** | Destruction in progress | Wait |
| **Destroyed** | All resources removed | Archive or delete workspace record |
| **Errored** | Operation failed | Retry, force-destroy, or force-transition (Admin) |

**Workspace Locking:**
- Only one operation can run on a workspace at a time
- The lock holder and acquisition time are displayed
- Admins can force-release locks if needed

---


## 12. Policies

### 12.1 Creating a Policy

Policies enforce governance rules on catalog orders. They can block (deny) or warn users when their requested configuration violates organizational standards.

Navigate to **Administration → Policies → Create New Policy**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Policy name (e.g., "Block Large Instances in Dev") |
| **Description** | No | Explanation of the policy's purpose |
| **Version** | No | Version string for tracking changes (e.g., "1.0", "2.1") |
| **Enabled** | Yes | Toggle to activate/deactivate the policy |

#### Scope

Define which catalog items this policy applies to:

| Scope Option | Description |
|--------------|-------------|
| **All Catalogs** | Leave both catalog_ids and catalog_tags empty — policy applies to every catalog item |
| **Specific Catalog IDs** | List specific catalog item IDs this policy targets |
| **Catalog Tags** | Apply to any catalog item that has these tags (e.g., "production", "aws") |

#### Rules

Each policy contains one or more rules. Each rule defines:

| Field | Description |
|-------|-------------|
| **Rule ID** | Unique identifier within this policy (e.g., "block_xlarge") |
| **Name** | Human-readable rule name (e.g., "Block XLarge Instances") |
| **Description** | What this rule checks |
| **Effect** | **Deny** (blocks the order) or **Warn** (shows warning but allows proceeding) |
| **Message** | Text shown to the user when the rule triggers (e.g., "XLarge instances are not allowed in development environments") |

**Rule Conditions:**

Each rule has one or more conditions that are evaluated against the catalog order form data:

| Field | Description |
|-------|-------------|
| **field** | The form field name to check (dot-path supported, e.g., "instance_type") |
| **operator** | Comparison operator |
| **value** | The value to compare against |

**Available Operators:**

| Operator | Description | Example |
|----------|-------------|---------|
| equals | Exact match | field: "environment", value: "production" |
| not_equals | Does not match | field: "environment", value: "dev" |
| in | Value is in a list | field: "instance_type", value: ["t3.xlarge", "t3.2xlarge"] |
| not_in | Value is not in a list | field: "region", value: ["us-gov-west-1"] |
| greater_than | Numeric greater than | field: "disk_size", value: 1000 |
| less_than | Numeric less than | field: "instance_count", value: 1 |
| contains | String contains substring | field: "name", value: "prod" |

**Condition Logic:**
- **ALL (AND)** — All conditions must be true for the rule to trigger
- **ANY (OR)** — Any single condition being true triggers the rule

---

### 12.2 Example: "Block Large Instances in Dev" Policy

**Basic Information:**
- Name: `Block Large Instances in Dev`
- Description: `Prevents provisioning of expensive instance types in development environments to control costs.`
- Version: `1.0`
- Enabled: Yes

**Scope:**
- Catalog Tags: `aws`, `compute`

**Rules:**

**Rule 1: Block XLarge in Dev**
- Rule ID: `block_xlarge_dev`
- Name: `Block XLarge Instances in Development`
- Effect: Deny
- Message: `XLarge and larger instances are not permitted in development environments. Please use t3.medium or smaller, or request a production catalog item.`
- Condition Logic: ALL
- Conditions:
  - field: `environment`, operator: `equals`, value: `dev`
  - field: `instance_type`, operator: `in`, value: `["t3.xlarge", "t3.2xlarge", "m5.xlarge", "m5.2xlarge"]`

**Rule 2: Warn on Large Storage**
- Rule ID: `warn_large_storage`
- Name: `Large Storage Warning`
- Effect: Warn
- Message: `You are requesting more than 500 GB of storage. Please confirm this is necessary for your use case.`
- Condition Logic: ALL
- Conditions:
  - field: `disk_size`, operator: `greater_than`, value: `500`

---

## 13. Quotas

### 13.1 Creating a Quota

Quotas limit how many resources of a specific type can be provisioned by a user, group, or tenant.

Navigate to **Administration → Quotas → Create New Quota**.

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Quota name (e.g., "Max EC2 per Developer") |
| **Scope** | Yes | Who this quota applies to: **User**, **Group**, or **Tenant** |
| **Scope ID** | Yes | The specific user ID, group ID, or tenant ID |
| **Resource Type** | Yes | What resource type to limit (e.g., "ec2_instance", "s3_bucket", "catalog_request") |
| **Max Count** | Yes | Maximum number of this resource type allowed |

---

### 13.2 How Quotas are Enforced

When a user submits a catalog order:
1. CMP checks all applicable quotas for the user, their groups, and the tenant
2. The current count of the resource type is compared against the max count
3. If any quota would be exceeded:
   - The order is **blocked**
   - The user sees a message: "Quota exceeded: You have X of Y allowed [resource_type]"
4. If within limits, the order proceeds normally

**Quota Check Result includes:**
- Whether the action is allowed
- Current count vs. maximum count
- Which quota is blocking (if denied)

---

### 13.3 Example: "Max 5 EC2 per Developer" Quota

- Name: `Max 5 EC2 Instances per Developer`
- Scope: User
- Scope ID: `user_dev_001` (or apply to each developer individually)
- Resource Type: `ec2_instance`
- Max Count: `5`

When this developer tries to provision their 6th EC2 instance, the order is blocked with: "Quota exceeded: You have 5 of 5 allowed ec2_instance resources."

---

## 14. Approvals

### 14.1 Submitting a Request

When you order a catalog item that requires approval:

1. Fill in the order form as usual
2. Click **Submit Order**
3. Instead of immediate execution, you see: "Your request has been submitted for approval"
4. The request appears in the **Approvals** section with status "Pending"
5. You can add a **Justification** explaining why you need this resource
6. Designated approvers receive a notification

**What approvers see:**
- Requester name and email
- Catalog item name
- All form data submitted
- Cost estimate (if available)
- Justification text
- Comments thread

---

### 14.2 Approving/Rejecting Requests

If you are a designated approver:

1. Navigate to **Approvals** (or click the notification)
2. Review the pending request details
3. Check the form data, cost estimate, and justification
4. Choose an action:
   - **Approve** — The linked Flow executes automatically
   - **Reject** — The request is denied; requester is notified
5. Optionally add a **Comment** explaining your decision

---

### 14.3 Adding Comments

Both requesters and approvers can add comments to an approval request:

1. Open the approval request
2. Scroll to the **Comments** section
3. Type your message
4. Click **Add Comment**

Comments create a discussion thread useful for:
- Asking for clarification before approving
- Explaining why a request was rejected
- Documenting approval rationale for audit

---

### 14.4 Approval Lifecycle

| Status | Description |
|--------|-------------|
| **Pending** | Awaiting approver action |
| **Approved** | Approved by an authorized approver; execution begins |
| **Rejected** | Denied by an approver; no execution |
| **Cancelled** | Withdrawn by the requester before a decision |
| **Timed Out** | No action taken within the configured timeout period |

**After approval:**
- If an "On Approve Flow" is configured, it triggers automatically
- The main provisioning Flow executes with the original form data and credential

**After rejection:**
- If an "On Reject Flow" is configured, it triggers (e.g., to notify the requester's manager)
- The requester receives a notification with the rejection reason

---

## 15. Cost Models

### 15.1 Creating a Cost Model

Cost Models define how CMP calculates management fees, markups, and surcharges on top of cloud provider costs.

Navigate to **Administration → Cost Models → Create New Cost Model**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Model name (e.g., "Standard Management Fee") |
| **Description** | No | Explanation of this cost model |
| **Scope** | Yes | Where this model applies: Global, Catalog-specific, or Resource Action-specific |
| **Priority** | No | When multiple models apply, higher priority (lower number) takes precedence |
| **Active** | Yes | Toggle to enable/disable |

**Scope Options:**

| Scope | Description |
|-------|-------------|
| **Global** | Applies to all catalog items and resource actions |
| **Catalog** | Only applies to specific catalog item IDs |
| **Resource Action** | Only applies to specific resource action IDs |

When scope is "Catalog", specify the **Catalog IDs** this model targets.  
When scope is "Resource Action", specify the **Resource Action IDs**.

#### Rules

Each cost model contains rules that define charges:

| Field | Description |
|-------|-------------|
| **Rule ID** | Unique identifier (e.g., "mgmt_fee") |
| **Name** | Rule name (e.g., "Management Fee") |
| **Description** | What this charge represents |
| **Charge Type** | **Percentage** (% of cloud cost) or **Fixed** (flat amount) |
| **Charge Value** | The percentage (e.g., 15 for 15%) or fixed amount (e.g., 50.00) |
| **Charge Label** | Label shown in the cost breakdown (e.g., "Platform Management Fee") |
| **Priority** | Order of evaluation within this model |
| **Active** | Toggle to enable/disable this specific rule |

**Rule Conditions:**

Rules can be conditional — only apply the charge when certain form data matches:

| Field | Description |
|-------|-------------|
| **field** | Form field name or "cloud_cost" for cost-based conditions |
| **operator** | equals, not_equals, contains, greater_than, less_than, greater_than_or_equal, less_than_or_equal, in, not_in |
| **value** | Comparison value |

**Condition Logic:** ANY (OR) or ALL (AND)

---

### 15.2 How Cost Estimation Works

When a user fills in a catalog order form:

1. CMP identifies all applicable cost models (based on scope and priority)
2. For each model's rules:
   - Evaluates conditions against the form data
   - If conditions match (or no conditions exist), calculates the charge
3. **Percentage charges:** `cloud_cost × (charge_value / 100)`
4. **Fixed charges:** Added as-is
5. The cost breakdown is displayed to the user:
   - Cloud cost (estimated from provider pricing)
   - Each CMP charge with its label
   - Total CMP charges
   - Final cost (cloud + CMP charges)

---

### 15.3 Example: "15% Management Fee" Cost Model

**Basic Information:**
- Name: `Standard Management Fee`
- Description: `15% management fee applied to all provisioning requests. Reduced to 10% for development environments.`
- Scope: Global
- Priority: 1
- Active: Yes

**Rules:**

**Rule 1: Production Fee (15%)**
- Rule ID: `prod_fee`
- Name: `Production Management Fee`
- Charge Type: Percentage
- Charge Value: `15`
- Charge Label: `Platform Management Fee (15%)`
- Conditions:
  - field: `environment`, operator: `not_equals`, value: `dev`
- Condition Logic: ALL

**Rule 2: Development Fee (10%)**
- Rule ID: `dev_fee`
- Name: `Development Management Fee`
- Charge Type: Percentage
- Charge Value: `10`
- Charge Label: `Platform Management Fee (Dev - 10%)`
- Conditions:
  - field: `environment`, operator: `equals`, value: `dev`
- Condition Logic: ALL

**Rule 3: Support Surcharge**
- Rule ID: `support_surcharge`
- Name: `Premium Support Surcharge`
- Charge Type: Fixed
- Charge Value: `25.00`
- Charge Label: `Premium Support Fee`
- Conditions:
  - field: `support_tier`, operator: `equals`, value: `premium`

---

## 16. Budgets

### 16.1 Creating a Budget

Budgets help you track and control cloud spending with alerts when thresholds are reached.

Navigate to **Administration → Budgets → Create New Budget**.

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Budget name (e.g., "AWS Monthly Production") |
| **Amount** | Yes | Budget limit (e.g., 10000) |
| **Currency** | Yes | Currency code (default: "USD") |
| **Period** | Yes | Budget period: Monthly, Quarterly, or Annual |
| **Cloud Provider** | No | Limit to a specific provider (AWS, Azure, GCP). Leave empty for all providers. |
| **Credential ID** | No | Limit to a specific cloud account |
| **Group ID** | No | Limit to a specific team/group's spending |

#### Alert Thresholds

Add one or more alert thresholds:

| Field | Description |
|-------|-------------|
| **Percentage** | At what % of budget to trigger alert (e.g., 50, 80, 100) |
| **Notification Channels** | Where to send the alert: "email", "slack", "teams" |

Example thresholds:
- 50% → Email notification
- 80% → Email + Slack notification
- 100% → Email + Slack + Teams notification

---

### 16.2 Budget Monitoring

The Budgets page shows:
- Budget name and period
- Current spend vs. budget amount (progress bar)
- Percentage used
- Remaining amount
- Alert status (which thresholds have been triggered)
- Trend indicator (spending rate)

---

### 16.3 Example: "$10,000 Monthly AWS Budget"

- Name: `AWS Production Monthly Budget`
- Amount: `10000`
- Currency: `USD`
- Period: Monthly
- Cloud Provider: `aws`
- Credential ID: `prod-aws-credential-id`
- Group ID: (empty — applies to all groups using this credential)

**Thresholds:**

| Percentage | Channels |
|-----------|----------|
| 50% | email |
| 75% | email, slack |
| 90% | email, slack, teams |
| 100% | email, slack, teams |

---


## 17. Event Automation

### 17.1 Creating an Automation Rule

Event Automation lets you define rules that automatically trigger actions when specific events occur in the platform — such as sending a Slack notification when a provisioning fails, or triggering a cleanup workflow when a resource is terminated.

Navigate to **Provisioning → Event Automation → Create New Rule**.

#### Basic Information

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Rule name (e.g., "Notify on Failed Provisioning") |
| **Description** | No | Explanation of what this rule does |
| **Priority** | No | Lower number = higher priority. Default: 100. Rules are evaluated in priority order. |
| **Enabled** | Yes | Toggle to activate/deactivate |

#### Event Type

Select which platform event triggers this rule. Available event types (60+):

**Catalog Lifecycle:**
- catalog.requested — User submits a catalog order
- catalog.approval.requested — Order sent for approval
- catalog.approved — Order approved
- catalog.rejected — Order rejected
- catalog.cancelled — Order cancelled

**Resource Lifecycle:**
- resource.provision.started — Provisioning begins
- resource.provision.completed — Provisioning succeeds
- resource.provision.failed — Provisioning fails
- resource.created — New resource detected
- resource.updated — Resource modified
- resource.deleted — Resource removed
- resource.started — Resource started
- resource.stopped — Resource stopped
- resource.terminated — Resource terminated

**Credential Lifecycle:**
- credential.added — New credential added
- credential.updated — Credential modified
- credential.deleted — Credential removed
- credential.validated — Credential validation succeeded
- credential.failed — Credential validation failed

**User Lifecycle:**
- user.created — New user account created
- user.updated — User profile updated
- user.deleted — User account deleted
- user.login — User logged in
- user.logout — User logged out
- user.password_changed — Password changed

**SSO Authentication:**
- sso.login.success — SSO login succeeded
- sso.login.failure — SSO login failed
- sso.user.provisioned — New user auto-created via SSO

**Group Lifecycle:**
- group.created / group.updated / group.deleted

**Approval Lifecycle:**
- approval.pending — New approval request
- approval.approved — Request approved
- approval.rejected — Request rejected
- approval.timed_out — Approval timed out
- approval.cancelled — Approval cancelled

**Workflow / Execution Lifecycle:**
- workflow.created / workflow.updated / workflow.deleted
- workflow.started / workflow.completed / workflow.failed
- flow.created / flow.updated / flow.deleted
- execution.started / execution.completed / execution.failed / execution.retried / execution.cancelled
- task.created / task.updated / task.deleted
- task.started / task.completed / task.failed

**Cost Events:**
- cost.threshold.exceeded — Budget threshold breached
- cost.estimate.generated — Cost estimate calculated
- cost.anomaly.detected — Unusual spending pattern detected

**Terraform Events:**
- terraform.action.started / terraform.action.completed / terraform.action.failed

**System Events:**
- system.health.degraded / system.health.restored
- notification.sent / notification.failed
- webhook.delivered / webhook.failed
- automation.rule.triggered / automation.rule.failed

#### Conditions

Optionally filter events further with conditions on the event data:

| Field | Description |
|-------|-------------|
| **field** | Dot-path into the event data (e.g., "metadata.cost", "actor.role", "resource.type") |
| **operator** | equals, not_equals, contains, not_contains, greater_than, less_than, greater_or_equal, less_or_equal, exists, not_exists, in |
| **value** | Comparison value |

**Condition Logic:** AND (all must match) or OR (any can match)

Example: Trigger only when `resource.type` equals "ec2_instance" AND `metadata.region` equals "us-east-1"

#### Actions

Define what happens when the rule triggers. Multiple actions can be configured per rule:

**Send Notification (Email):**

| Setting | Description |
|---------|-------------|
| recipients | List of email addresses |
| subject_template | Email subject with placeholders: `{{event.resource.name}}`, `{{event.actor.username}}` |
| body_template | Email body with placeholders |

**Send Slack:**

| Setting | Description |
|---------|-------------|
| subject_template | Message title with placeholders |
| body_template | Message body with placeholders |

(Uses the Slack webhook URL configured in Notification Channels)

**Send Teams:**

| Setting | Description |
|---------|-------------|
| subject_template | Message title with placeholders |
| body_template | Message body with placeholders |

(Uses the Teams webhook URL configured in Notification Channels)

**Trigger Workflow:**

| Setting | Description |
|---------|-------------|
| workflow_id | Which workflow to execute |
| credential_id | Cloud credential to use |
| form_data_template | Input data with placeholders from the event |

**Call Webhook:**

| Setting | Description |
|---------|-------------|
| url | Webhook endpoint URL |
| method | HTTP method (POST, PUT, etc.) |
| headers | Custom headers (key-value pairs) |
| body_template | JSON body with event placeholders |

**Call API:**

| Setting | Description |
|---------|-------------|
| url | API endpoint URL |
| method | HTTP method |
| headers | Custom headers |
| body_template | JSON body with placeholders |
| auth | Authentication configuration |

**Execute Task:**

| Setting | Description |
|---------|-------------|
| task_id | Which task to run |
| params | Parameters to pass (can include event placeholders) |

**Publish Event:**

| Setting | Description |
|---------|-------------|
| event_type | New event type to emit |
| metadata_template | Metadata for the new event |

---

### 17.2 Example: "Notify Slack on Failed Provisioning" Rule

**Basic Information:**
- Name: `Notify Slack on Failed Provisioning`
- Description: `Sends a Slack alert to the ops channel when any resource provisioning fails.`
- Priority: 10
- Enabled: Yes

**Event Type:** `resource.provision.failed`

**Conditions:**
- field: `resource.type`, operator: `not_equals`, value: `test_resource`
  (Skip test resources)

**Actions:**

**Action 1: Send Slack**
- subject_template: `🚨 Provisioning Failed: {{event.resource.name}}`
- body_template: `Resource provisioning failed.\n\n• Resource: {{event.resource.name}} ({{event.resource.type}})\n• Requested by: {{event.actor.username}}\n• Error: {{event.metadata.error}}\n• Time: {{event.timestamp}}`

**Action 2: Send Notification (Email)**
- recipients: `["ops-team@company.com"]`
- subject_template: `[CMP Alert] Provisioning Failed: {{event.resource.name}}`
- body_template: `A resource provisioning has failed. Please investigate.\n\nResource: {{event.resource.name}}\nType: {{event.resource.type}}\nRequested by: {{event.actor.username}} ({{event.actor.role}})`

---

## 18. Users & Groups (Admin)

### 18.1 Creating Users

Navigate to **Administration → Users & Groups → Create User**.

| Field | Required | Description |
|-------|----------|-------------|
| **Username** | Yes | Unique login name. 3-50 characters. Allowed: letters, numbers, underscore, dot, hyphen, @. |
| **First Name** | Yes | User's first name. 1-100 characters. |
| **Last Name** | Yes | User's last name. 1-100 characters. |
| **Email** | Yes | Valid email address |
| **Password** | Yes | Must be 8+ characters with at least one uppercase letter, one lowercase letter, and one digit |
| **Roles** | Yes | One or more roles: admin, developer, user, readonly |
| **Active** | Yes | Whether the account is enabled (default: Yes) |

**Managing existing users:**
- View all users in a searchable, sortable table
- Edit user details (name, email, roles)
- Activate/deactivate accounts
- Reset passwords

---

### 18.2 Managing Groups

Navigate to **Administration → Users & Groups → Groups tab**.

Groups organize users into teams and can be used for access control throughout CMP.

**Creating a Group:**

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Group name. 1-100 characters. (e.g., "Platform Engineering") |
| **Description** | No | Group purpose. Max 500 characters. |
| **Role** | Yes | Role linked to this group: admin, developer, user, or readonly |
| **Members** | No | List of user IDs to add as members |

**Group uses throughout CMP:**
- Catalog item access control (Allowed Groups)
- Credential visibility (Visible Group IDs)
- Approval routing (Approver Group IDs)
- Budget scoping (Group ID)
- Quota scoping (Group scope)

---

### 18.3 Role Hierarchy

CMP uses a four-level role hierarchy:

| Role | Level | Capabilities |
|------|-------|-------------|
| **Admin** | Highest | Full platform access. Manage users, groups, policies, quotas, budgets, cost models, tenants, SSO, feature toggles. Access all provisioning and infrastructure features. Approve requests. |
| **Developer** | High | Create and manage Tasks, Workflows, Flows, Catalog Items. Manage credentials and resources. View executions. Cannot manage users, policies, or platform settings. |
| **User** | Standard | Browse and order from the Service Catalog. View own executions and resources. Manage own credentials. Submit approval requests. |
| **Readonly** | Lowest | View-only access. Can browse the catalog and view resources but cannot order, create, or modify anything. |

A user can have multiple roles. The highest role determines their access level.

---

## 19. SSO Configuration

### 19.1 Setting Up an SSO Provider

Navigate to **Administration → SSO Configuration**.

CMP supports Single Sign-On with these identity providers:

| Provider | Protocol |
|----------|----------|
| Microsoft (Entra ID / Azure AD) | OIDC |
| Google Workspace | OIDC |
| Okta | OIDC |
| AWS Identity Center | OIDC |
| Ping Identity | OIDC |

**Configuration Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| **Provider Type** | Yes | Select from: Microsoft, Google, Okta, AWS Identity Center, Ping Identity |
| **Client ID** | Yes | OAuth2 Client ID from your identity provider |
| **Client Secret** | Yes | OAuth2 Client Secret (encrypted at rest) |
| **Issuer URL** | Yes | OIDC Issuer URL (must be HTTPS). Example: `https://login.microsoftonline.com/{tenant-id}/v2.0` |
| **Redirect URL** | Yes | CMP callback URL (must be HTTPS). Example: `https://your-cmp.com/sso/callback` |
| **Display Name** | No | Custom name shown on the login button (e.g., "Sign in with Company SSO") |
| **Protocol Type** | Yes | OIDC (default) or SAML |

**Advanced Settings:**

| Field | Description |
|-------|-------------|
| **Allowed Domains** | Only allow SSO login from these email domains (e.g., ["company.com", "subsidiary.com"]) |
| **Auto User Creation** | Automatically create CMP accounts for new SSO users (default: No) |
| **Group Sync Enabled** | Sync group memberships from the identity provider (default: Yes) |
| **Single Logout Enabled** | When user logs out of CMP, also log out of the identity provider (default: No) |

---

### 19.2 Group Mappings

Navigate to **Administration → SSO Group Mappings**.

Group mappings connect your identity provider's groups to CMP roles. When a user logs in via SSO, their group memberships determine their CMP role.

**Creating a Group Mapping:**

| Field | Required | Description |
|-------|----------|-------------|
| **Provider Type** | Yes | Which SSO provider this mapping is for |
| **External Group ID** | Yes | The group identifier from your identity provider (e.g., Azure AD group object ID) |
| **Group Name** | Yes | Human-readable name for this group (e.g., "Cloud Admins") |
| **CMP Role** | Yes | The CMP role to assign: admin, developer, user, or readonly |
| **Priority** | Yes | 1-100. When a user belongs to multiple groups, the mapping with the lowest priority number wins. |

**How it works:**
1. User logs in via SSO
2. Identity provider sends the user's group memberships
3. CMP matches each group against configured mappings
4. The highest-priority matching mapping determines the user's role
5. Unmatched groups are logged but don't affect access

---

### 19.3 Example: Setting Up Microsoft Entra ID

**Step 1: Register an Application in Azure Portal**
- Go to Azure Portal → Entra ID → App Registrations → New Registration
- Set Redirect URI to: `https://your-cmp-domain.com/sso/callback`
- Note the Application (client) ID and Directory (tenant) ID
- Create a Client Secret under Certificates & Secrets

**Step 2: Configure in CMP**
- Provider Type: Microsoft
- Client ID: `12345678-abcd-efgh-ijkl-123456789012`
- Client Secret: `your-client-secret-value`
- Issuer URL: `https://login.microsoftonline.com/your-tenant-id/v2.0`
- Redirect URL: `https://your-cmp-domain.com/sso/callback`
- Allowed Domains: `company.com`
- Auto User Creation: Yes
- Group Sync Enabled: Yes

**Step 3: Configure Group Mappings**

| External Group ID | Group Name | CMP Role | Priority |
|-------------------|-----------|----------|----------|
| `aad-group-id-admins` | Cloud Platform Admins | admin | 1 |
| `aad-group-id-devs` | Cloud Developers | developer | 2 |
| `aad-group-id-users` | Cloud Users | user | 3 |
| `aad-group-id-viewers` | Cloud Viewers | readonly | 4 |

**Step 4: Test**
- Click "Sign in with Company SSO" on the login page
- Verify you're redirected to Microsoft login
- After authentication, verify correct role assignment

---

## 20. Tenants

### 20.1 Creating a Tenant

Tenants provide complete isolation between organizations sharing the same CMP instance. Each tenant has its own users, credentials, catalog items, resources, and configurations.

Navigate to **Administration → Tenants → Create New Tenant**.

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Organization name (e.g., "Acme Corporation") |
| **Slug** | Yes | URL-safe unique identifier, used in paths (e.g., "acme-corp"). Becomes the tenant_id. |
| **Admin Email** | Yes | Email for the tenant's initial admin user |
| **Admin Username** | Yes | Username for the initial admin |
| **Admin Password** | Yes | Password for the initial admin (8+ chars, uppercase, lowercase, digit) |
| **Plan** | No | Subscription plan (default: "standard") |
| **Max Users** | No | Maximum number of users allowed (default: 50) |
| **Max Credentials** | No | Maximum cloud accounts allowed (default: 20) |
| **Branding** | No | Custom branding settings (see below) |

**Branding Options:**

| Setting | Description |
|---------|-------------|
| Logo URL | URL to the organization's logo image |
| Company Name | Displayed in the navigation bar |
| Top Bar Color | Hex color for the navigation bar (e.g., "#1d4ed8") |
| Primary Color | Brand accent color used throughout the UI |

---

### 20.2 Tenant Switching

Users who belong to multiple tenants can switch between them:

1. Look for the **tenant selector** dropdown in the top navigation bar
2. Select the tenant you want to switch to
3. The page reloads with the new tenant's context
4. All data (catalog, resources, credentials) now reflects the selected tenant

The URL updates to include the tenant slug: `/t/acme-corp/dashboard`

---

### 20.3 Tenant Status

| Status | Description |
|--------|-------------|
| **Active** | Fully operational. All features available. |
| **Trial** | Trial period. May have feature or usage limitations. |
| **Suspended** | Access restricted. Users can log in but cannot perform operations. Contact platform admin. |

---

## 21. Scheduled Jobs

### 21.1 Creating a Scheduled Job

Scheduled Jobs run Tasks automatically on a recurring schedule using cron expressions.

Navigate to **Provisioning → Scheduled Jobs → Create New Job**.

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Job name (e.g., "Daily Cleanup", "Weekly Report") |
| **Description** | No | What this job does |
| **Cron Expression** | Yes | Standard cron format defining the schedule |
| **Task** | Yes | Select which Task to execute |
| **Parameters** | No | Key-value parameters to pass to the task |
| **Enabled** | Yes | Toggle to activate/pause the job |
| **Max Runs** | No | Maximum number of times to run. Leave empty for unlimited. |

**Cron Expression Format:**

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
│ │ │ │ │
* * * * *
```

**Common Examples:**

| Expression | Schedule |
|-----------|----------|
| `0 2 * * *` | Every day at 2:00 AM |
| `0 8 * * 1-5` | Weekdays at 8:00 AM |
| `*/15 * * * *` | Every 15 minutes |
| `0 0 1 * *` | First day of every month at midnight |
| `0 9 * * 1` | Every Monday at 9:00 AM |
| `0 */6 * * *` | Every 6 hours |
| `30 4 * * 0` | Every Sunday at 4:30 AM |

---

### 21.2 Monitoring Job History

Each scheduled job tracks its execution history:

| Field | Description |
|-------|-------------|
| Run ID | Unique identifier for each execution |
| Status | success or failed |
| Started At | When the run began |
| Completed At | When the run finished |
| Duration | How long it took (milliseconds) |
| Exit Code | Process exit code (0 = success) |
| Output | Standard output from the task |
| Errors | Standard error output (if any) |

**Job Status:**

| Status | Description |
|--------|-------------|
| **Active** | Running on schedule |
| **Paused** | Temporarily disabled (can be re-enabled) |
| **Completed** | Reached max_runs limit |
| **Failed** | Last run failed (still scheduled for next run) |

**Additional Information:**
- Run Count: Total number of times the job has executed
- Last Run At: Timestamp of the most recent execution
- Next Run At: When the job will run next

---

### 21.3 Example: "Daily Cleanup at 2 AM" Job

- Name: `Daily Resource Cleanup`
- Description: `Removes expired resources and sends summary report to ops team.`
- Cron Expression: `0 2 * * *` (every day at 2:00 AM)
- Task: `cleanup-expired-resources` (a Python task that checks lease dates)
- Parameters:
  - `dry_run`: `false`
  - `notify_owners`: `true`
  - `grace_period_hours`: `24`
- Enabled: Yes
- Max Runs: (empty — runs indefinitely)

---

## 22. Notifications & Channels

### 22.1 In-App Notifications

CMP provides a built-in notification center accessible via the **bell icon** in the top navigation bar.

**Notification types you receive:**
- Approval requests (when you're an approver)
- Approval decisions (when your request is approved/rejected)
- Execution completions and failures
- Budget threshold alerts
- Lease expiry warnings
- System announcements

**Managing notifications:**
- Click the bell icon to see recent notifications
- Unread count shown as a badge on the bell
- Click a notification to navigate to the relevant item
- Mark individual notifications as read
- Mark all as read

---

### 22.2 Configuring Notification Channels

Navigate to **Administration → Settings** (or configure within Event Automation rules).

#### Email (SMTP)

| Setting | Description |
|---------|-------------|
| SMTP Host | Mail server hostname (e.g., "smtp.company.com") |
| SMTP Port | Port number (typically 587 for TLS, 465 for SSL) |
| Username | SMTP authentication username |
| Password | SMTP authentication password |
| From Address | Sender email address (e.g., "cmp-noreply@company.com") |
| TLS/SSL | Enable encryption |

#### Slack

| Setting | Description |
|---------|-------------|
| Webhook URL | Slack Incoming Webhook URL (e.g., `https://hooks.slack.com/services/T.../B.../xxx`) |

To set up:
1. Go to your Slack workspace → Apps → Incoming Webhooks
2. Create a new webhook for your desired channel
3. Copy the webhook URL into CMP

#### Microsoft Teams

| Setting | Description |
|---------|-------------|
| Webhook URL | Teams Incoming Webhook URL |

To set up:
1. In Teams, go to the channel → Connectors → Incoming Webhook
2. Create and name the webhook
3. Copy the webhook URL into CMP

#### PagerDuty

| Setting | Description |
|---------|-------------|
| Integration Key | PagerDuty Events API integration key |

To set up:
1. In PagerDuty, create a new service or use an existing one
2. Add an "Events API v2" integration
3. Copy the integration key into CMP

---


## 23. Webhooks

### 23.1 Creating Inbound Webhooks

Inbound Webhooks allow external systems to trigger actions in CMP by sending HTTP requests to a unique endpoint.

Navigate to **Administration → Inbound Webhooks → Create New Webhook**.

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Webhook name (e.g., "GitHub Deploy Trigger") |
| **Description** | No | What this webhook is used for |
| **Enabled** | Yes | Toggle to activate/deactivate |

After creation, CMP generates a unique webhook URL that external systems can POST to.

**Webhook URL format:** `https://your-cmp-domain.com/api/v1/webhooks/inbound/{webhook_id}`

**Security:**
- Each webhook has a unique, unguessable ID
- Optionally configure a secret token for signature verification
- Requests are logged for audit

---

### 23.2 Use Cases

| Use Case | Description |
|----------|-------------|
| **CI/CD Integration** | Trigger a CMP workflow when a GitHub Actions pipeline completes |
| **Monitoring Alerts** | Auto-scale resources when Datadog/CloudWatch sends an alert |
| **External Approvals** | Receive approval decisions from external ticketing systems (ServiceNow, Jira) |
| **Scheduled Triggers** | External cron services triggering CMP automations |
| **Cross-Platform** | Other platforms notifying CMP of events (e.g., Terraform Cloud run completed) |

**Example: GitHub Deploy Trigger**
1. Create a webhook named "GitHub Deploy Trigger"
2. Copy the generated webhook URL
3. In your GitHub repository → Settings → Webhooks → Add webhook
4. Paste the CMP webhook URL
5. Select events: "Workflow run completed"
6. Configure an Event Automation rule to react to incoming webhook events

---

## 24. Feature Toggles

### 24.1 Managing Feature Toggles

Feature Toggles allow administrators to enable or disable platform features globally or for specific roles, groups, or users.

Navigate to **Administration → Settings → Feature Toggles** (Admin only).

Each toggle has:

| Setting | Description |
|---------|-------------|
| **Enabled** | Master switch — when off, the feature is hidden for everyone |
| **Roles** | If specified, only these roles see the feature when enabled. Empty = all roles. |
| **Groups** | If specified, only members of these groups see the feature. Empty = all groups. |
| **Users** | If specified, only these user IDs see the feature. Empty = all users. |

---

### 24.2 Available Features

| # | Feature Key | Label | Group | Description |
|---|-------------|-------|-------|-------------|
| 1 | service_catalog | Service Catalog | General | Browse and order from the service catalog |
| 2 | executions | Executions | General | View and manage task/workflow executions |
| 3 | approvals | Approvals | General | Submit and manage approval requests |
| 4 | resources | Resources | General | View cloud resources across accounts |
| 5 | credentials | Credentials | General | Access cloud account credentials |
| 6 | notifications | Notifications | General | User notification center |
| 7 | chatbot | AI Chatbot | General | AI assistant chatbot in the navigation bar |
| 8 | self_registration | Self Registration | Authentication | Allow new users to register from the login page |
| 9 | forgot_password | Forgot Password | Authentication | Show forgot/reset password option on login |
| 10 | provisioning | Provisioning | Admin/Developer | Tasks, Workflows, Flows, Executions, Catalog Components, Scheduled Jobs |
| 11 | infrastructure | Infrastructure | Admin/Developer | Cloud accounts, Resources, Resource Actions, Shared Variables |
| 12 | event_automation | Event Automation | Admin/Developer | Configure automated event-driven workflows |
| 13 | reports | Reports & Analytics | Admin | Platform reports and analytics dashboards |
| 14 | cost_models | Cost Models | Admin | Define and manage cost allocation models |
| 15 | budgets | Budgets | Admin | Set and track spending budgets |
| 16 | policies | Policies | Admin | Cloud governance policies |
| 17 | quotas | Quotas | Admin | Resource quota limits per team or project |
| 18 | tenants | Tenants | Admin | Manage tenant organizations |
| 19 | webhooks | Inbound Webhooks | Admin | Configure inbound webhook endpoints |
| 20 | event_log | Event Log | Admin | System-wide event audit log |
| 21 | logging_monitoring | Logging & Monitoring | Admin | Platform logs and monitoring dashboards |
| 22 | cost_analytics | Cost Analytics | Admin | Aggregated cost summaries by user, group, catalog, and provider |
| 23 | live_cost_projections | Live Cost Projections | General | Real-time cost projections for active cloud resources |
| 24 | terraform_provisioning | Terraform Provisioning | Admin/Developer | Terraform templates, workspaces, Day 2 operations, drift detection |

**Example: Enabling AI Chatbot only for Developers and Admins**
- Feature: `chatbot`
- Enabled: Yes
- Roles: `admin`, `developer`
- Groups: (empty)
- Users: (empty)

---

## 25. Catalog Components (Reusable)

### 25.1 Creating a Reusable Component

Catalog Components are pre-built, reusable form elements that can be shared across multiple catalog items and resource actions. They save time and ensure consistency.

Navigate to **Provisioning → Catalog Components → Create New Component**.

| Field | Required | Description |
|-------|----------|-------------|
| **Name** | Yes | Component name (e.g., "AWS Region Selector", "Environment Picker") |
| **Description** | No | What this component provides |
| **Component Type** | Yes | The type of component (see below) |
| **Tags** | No | Categorization labels |
| **Data** | Yes | The component's configuration data (varies by type) |

**Component Types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **option_list** | A reusable set of options (label/value pairs) | Instance types, regions, environments |
| **field_template** | A complete field configuration that can be dropped into any form | Standard "Environment" radio button, "Region" dropdown |
| **input** | A reusable input field with validation | Email field with company domain validation |

---

### 25.2 Using Components in Catalog Items

When building a catalog item form:

1. Add a new field to your form
2. Instead of configuring it from scratch, set the **Reusable Component ID** field
3. Select the component from the dropdown
4. The field inherits all configuration from the component (options, validation, defaults)
5. You can still override specific settings if needed

Similarly, in Resource Action input schemas, set the `reusable_component_id` on any input field.

---

### 25.3 Example: "Region Selector" Component

**Component Details:**
- Name: `AWS Region Selector`
- Description: `Standard AWS region dropdown with all commonly used regions. Pre-selects us-east-1.`
- Component Type: `option_list`
- Tags: `aws`, `region`, `standard`

**Data:**
```
Options:
- US East (N. Virginia) → us-east-1
- US East (Ohio) → us-east-2
- US West (N. California) → us-west-1
- US West (Oregon) → us-west-2
- EU (Ireland) → eu-west-1
- EU (London) → eu-west-2
- EU (Frankfurt) → eu-central-1
- Asia Pacific (Tokyo) → ap-northeast-1
- Asia Pacific (Singapore) → ap-southeast-1
- Asia Pacific (Sydney) → ap-southeast-2
```

**Usage:** When creating any AWS catalog item, reference this component for the region field instead of manually typing all options each time.

---

## 26. Reports & Analytics

### 26.1 Available Reports

Navigate to **Administration → Reports & Analytics**.

The Reports section provides insights into platform usage, costs, and operations:

| Report | Description |
|--------|-------------|
| **Execution Summary** | Total executions by status, catalog item, and time period |
| **Resource Inventory** | Count of resources by type, provider, status, and region |
| **User Activity** | Login frequency, orders placed, resources owned per user |
| **Cost Summary** | Spending by provider, credential, group, and catalog item |
| **Approval Metrics** | Average approval time, approval/rejection rates |
| **Policy Violations** | Most triggered policies and rules |
| **Quota Utilization** | How close users/groups are to their limits |

---

### 26.2 Cost Analytics

The Cost Analytics dashboard provides:

- **Spending by Provider** — Pie chart showing AWS vs. Azure vs. GCP spend
- **Spending by Group** — Which teams are spending the most
- **Spending by Catalog Item** — Which services cost the most
- **Spending by User** — Individual user cost attribution
- **Trend Over Time** — Line chart showing daily/weekly/monthly spend
- **Live Cost Projections** — Estimated end-of-month spend based on current rate

---

### 26.3 Dashboard Overview

The main **Dashboard** (accessible to all users) shows:

- **Quick Stats** — Total resources, active executions, pending approvals
- **Recent Executions** — Last 5 executions with status
- **My Resources** — Resources you own with status indicators
- **Cost Overview** — Your spending this period (if cost analytics enabled)
- **Notifications** — Recent unread notifications
- **Quick Actions** — Shortcuts to frequently used catalog items

---

## 27. Profile & Settings

### 27.1 User Profile

Navigate to your **Profile** by clicking your avatar/initials in the top-right corner.

**Profile Information:**
- First Name and Last Name
- Email address
- Username
- Current Role(s)
- Tenant membership(s)
- Account creation date

---

### 27.2 Password Management

From your Profile page:

1. Click **Change Password**
2. Enter your current password
3. Enter your new password (must meet requirements):
   - Minimum 8 characters
   - At least one uppercase letter
   - At least one lowercase letter
   - At least one digit
4. Confirm the new password
5. Click **Save**

---

### 27.3 Developer Mode

Available to Admin and Developer roles:

- Toggle **Developer Mode** using the "DEV" button in the top navigation bar
- When enabled (shows "DEV ON" in amber):
  - Additional technical details are shown in execution logs
  - Raw form data is visible in approval requests
  - Debug information appears in error messages
  - AI Assistant provides more technical responses
- When disabled:
  - Standard user-friendly interface
  - Simplified error messages

---

## 28. AI Assistant

### 28.1 Accessing the Assistant

The AI Assistant is available via:
- **Bot icon** in the top navigation bar
- **Keyboard shortcut:** `⌘K` (Mac) or `Ctrl+K` (Windows/Linux)

The assistant opens as a side panel with two modes:
- **Command Mode** — Quick actions, navigation shortcuts, and AI queries (Linear-style palette)
- **Chat Mode** — Full conversational AI with context-aware intelligence

---

### 28.2 Context-Aware Intelligence

The AI Assistant is a **context-aware copilot** that dynamically adapts based on:

| Dimension | What It Detects | How It Helps |
|-----------|----------------|--------------|
| **User Role** | admin, developer, user, readonly | Only suggests actions you can perform |
| **Current Page** | Dashboard, Catalog, Resources, etc. | Tailors suggestions to your current workflow |
| **Selected Resource** | Resource type, status, provider, region | Enables "restart this" without asking "which resource?" |
| **Recent Activity** | Last 5 pages visited | Understands your workflow pattern |
| **Permissions** | Role-based permission set | Hides unauthorized actions entirely |

**Example:** When viewing a specific EC2 instance, the assistant automatically knows the resource details and offers relevant actions like "Stop this VM", "Check cost", or "Apply a lease" — without asking you to specify which resource.

---

### 28.3 What You Can Ask

The AI Assistant can help with:

| Category | Example Questions |
|----------|-------------------|
| **Platform Navigation** | "How do I create a new catalog item?" |
| **Resource Actions** | "Restart this VM", "Stop this resource", "Show cost" |
| **Troubleshooting** | "Why did my execution fail?", "Explain this error" |
| **Analytics** | "Show my budget status", "How many resources are running?" |
| **Provisioning** | "Provision an AWS EC2 instance", "Open the VM catalog" |
| **Credential Management** | "Add a new AWS credential", "Set up an Azure account", "Create a GCP credential" |
| **Cost Questions** | "How much does a t3.large cost in us-east-1?" |
| **Policy Guidance** | "What policies are active?", "Show governance rules" |
| **API Execution** | "Run GET /api/v1/catalog" (Developer/Admin only) |
| **General Cloud** | "Compare Azure vs AWS VM pricing" |

---

### 28.4 Page-Specific Suggestions

The assistant provides different suggestions based on your current page:

| Page | User Suggestions | Admin Suggestions |
|------|-----------------|-------------------|
| **Dashboard** | Resource summary, recent executions | Platform health, budget alerts, pending approvals |
| **Service Catalog** | Browse catalogs, provision resources | Catalog usage, approval requirements |
| **Resources** | Show running resources, stop/start | Sync errors, expensive resources |
| **Executions** | Recent runs, explain failures | Execution summary, failure analytics |
| **Approvals** | My pending requests | All pending, approve/reject |

---

### 28.5 Resource Context Actions

When viewing a specific resource, the assistant understands:
- Resource type (EC2, Azure VM, RDS, etc.)
- Current status (running, stopped, terminated)
- Cloud provider and region
- Associated credential and inventory

You can say things like:
- "Restart this" → Restarts the currently viewed resource
- "What's the cost?" → Looks up pricing for this instance type
- "Apply a 30-day lease" → Sets a destroy date on this resource
- "Show execution history" → Shows provisioning history for this resource

---

### 28.6 Secure Credential Creation via Chat

When you ask the AI Assistant to add, create, or set up a new cloud credential, it will **never** ask you to type sensitive information (access keys, secret keys, tokens) directly in the chat. Instead, it opens the credential creation form in the main UI panel where you can fill in secrets securely.

**How it works:**
1. Ask the assistant something like "Add a new AWS credential" or "Set up an Azure account"
2. The assistant opens the credential form in the main panel
3. If you specified a provider or name, the form fields are pre-populated
4. Fill in the remaining details (including secrets) in the secure form
5. Submit the form as normal

**Supported pre-fill fields:**
- Provider (aws, azure, gcp, github, bearer_token, basic_auth)
- Name
- Description
- Region

**Example prompts:**
- "Add a new AWS credential called Production US" → Opens form with provider=AWS and name pre-filled
- "Create a GCP credential" → Opens form with GCP pre-selected
- "Set up a new cloud account" → Opens the form for you to choose the provider

This ensures that sensitive credentials are never exposed in chat history or logs.

---

### 28.7 Tips for Best Results

- **Be specific** — "How do I add an AWS credential?" works better than "help with credentials"
- **Use pronouns** — When viewing a resource, "stop this" or "restart it" works naturally
- **Ask follow-ups** — The assistant remembers conversation context (last 20 messages)
- **Click suggestions** — Use the clickable suggestion chips for quick follow-ups
- **Developer Mode** — When enabled, the assistant provides more technical detail
- **API Execution** — Developers and admins can run any CMP API endpoint through the assistant

---

## Appendix: Quick Reference

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `⌘K` / `Ctrl+K` | Toggle AI Assistant |

### Status Color Codes

| Color | Meaning |
|-------|---------|
| 🟢 Green | Success, Active, Published, Approved |
| 🔵 Blue | Running, Processing, Info |
| 🟡 Yellow/Amber | Warning, Pending, Draft, Trial |
| 🔴 Red | Failed, Error, Denied, Expired |
| ⚪ Gray | Cancelled, Archived, Destroyed, Unknown |

### Role Access Summary

| Feature | Admin | Developer | User | Readonly |
|---------|-------|-----------|------|----------|
| Service Catalog (browse/order) | ✓ | ✓ | ✓ | View only |
| Tasks/Workflows/Flows | ✓ | ✓ | — | — |
| Credentials (manage) | ✓ | ✓ | Own only | — |
| Resources (view) | ✓ | ✓ | ✓ | ✓ |
| Resource Actions (execute) | ✓ | ✓ | ✓ | — |
| Approvals (approve) | ✓ | — | — | — |
| Policies/Quotas/Budgets | ✓ | — | — | — |
| Users & Groups | ✓ | — | — | — |
| SSO Configuration | ✓ | — | — | — |
| Tenants | ✓ | — | — | — |
| Feature Toggles | ✓ | — | — | — |
| Cost Models | ✓ | — | — | — |
| Event Automation | ✓ | — | — | — |
| Scheduled Jobs | ✓ | ✓ | — | — |
| Terraform Templates | ✓ | ✓ | — | — |
| Terraform Workspaces | ✓ | ✓ | — | — |
| Reports & Analytics | ✓ | — | — | — |

---

*End of Document*

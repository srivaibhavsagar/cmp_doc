# Infrastructure-as-Conversation (Plan Generator)

> Describe your desired infrastructure in plain English — CMP generates a complete plan with cost estimates, resource dependencies, and exportable Terraform code.

---

## Overview

Infrastructure-as-Conversation lets users create infrastructure plans by describing what they need in natural language. The AI interprets the request, selects appropriate resources (compute, database, storage, networking), estimates monthly costs, and generates Terraform HCL ready for deployment.

This feature is part of the **Conversational Infrastructure Management** suite (Feature 7) and is accessed through the AI Assistant chat panel.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Plan Generation | Describe infrastructure needs → get a full resource plan |
| Cost Estimation | Per-resource and total monthly cost breakdown |
| Plan Modification | Refine plans by adding/removing/scaling resources via chat |
| Terraform Export | Generate production-ready HCL code from any plan |
| Multi-Cloud | Supports AWS, Azure, and GCP resource selection |
| Budget Awareness | Warns when estimated costs exceed specified budgets |

---

## How to Use

### Prerequisites

1. AI Assistant must be enabled (Admin Settings → Feature Toggles → `AI Chatbot`)
2. Open the AI panel: press `⌘J` (Mac) or `Ctrl+J` (Windows/Linux)

### Generate a Plan

Describe what you need in natural language:

```
"I need a web application with 3 servers, a PostgreSQL database, 
 load balancer, and S3 storage. Budget is $500/month, 
 must be highly available."
```

The AI will return:
- A list of planned resources with types and sizes
- Resource dependency graph
- Estimated monthly cost
- Budget warnings if applicable

### Modify a Plan

After generating a plan, refine it conversationally:

```
"Remove the storage and scale up the servers to larger instances"
"Add a database to the plan"
"Make the servers bigger for production workloads"
```

### Get Cost Breakdown

Request a detailed cost analysis:

```
"Show me the cost breakdown for that plan"
```

Returns per-resource monthly and annual costs.

### Export Terraform

When satisfied with the plan:

```
"Export this plan as Terraform code"
```

Generates complete Terraform HCL with:
- Provider configuration
- Resource definitions (EC2, RDS, S3, VPC, ALB, etc.)
- Tags (`ManagedBy = "CMP"`)
- Random suffix for unique naming

---

## Supported Resources

### Compute (Instances/VMs)

| Provider | Instance Types | Use Case |
|----------|---------------|----------|
| AWS | t3.micro, t3.small, t3.medium, t3.large, m5.large, m5.xlarge | Dev → Production |
| Azure | Standard_B1s, Standard_B2s, Standard_D2s_v3, Standard_D4s_v3 | Dev → Production |
| GCP | e2-micro, e2-small, e2-medium, n2-standard-2, n2-standard-4 | Dev → Production |

Instance selection is automatic based on description keywords:
- "small", "dev", "test" → smaller instances
- "medium" → mid-tier instances
- "large", "production" → larger instances

### Database

| Type | When Selected | Multi-AZ Support |
|------|---------------|------------------|
| db.t3.medium | "small" or "dev" in description | Optional |
| db.r5.large | Production workloads (default) | Yes (doubles cost) |

Engine defaults to PostgreSQL. Multi-AZ enabled when `high_availability` constraint is set.

### Networking

| Resource | Auto-Included | Details |
|----------|--------------|---------|
| VPC | When compute exists | CIDR 10.0.0.0/16, 3 subnets, NAT gateway |
| Load Balancer | When multiple compute or explicitly requested | Application LB, HTTP/HTTPS listeners |

### Storage

| Type | Estimated Cost |
|------|--------------|
| S3 Standard (100GB) | $2.30/month |

Includes encryption and versioning by default.

---

## Constraints

Specify constraints to guide resource selection:

| Constraint | Effect |
|-----------|--------|
| `budget_monthly` | Warns if plan exceeds budget |
| `region` | Deploys all resources in specified region |
| `provider_preference` | Forces a specific cloud (aws/azure/gcp) |
| `high_availability` | Enables multi-AZ for databases, increases instance count |
| `encryption_required` | Ensures storage encryption (default: true) |
| `compliance` | Tags for compliance frameworks (hipaa, pci, soc2) |

---

## Plan Lifecycle

```
DRAFT → REVIEWED → APPROVED → EXPORTED → APPLIED
                                    ↘ CANCELLED
```

| Status | Description |
|--------|-------------|
| Draft | Initial plan generated from description |
| Reviewed | Plan has been examined (cost breakdown viewed) |
| Approved | User has confirmed the plan |
| Exported | Terraform code has been generated |
| Applied | Infrastructure has been provisioned |
| Cancelled | Plan was abandoned |

---

## Example Conversation

```
User: "I need infrastructure for a medium-sized web app on AWS in us-west-2, 
       budget $300/month"

AI: Infrastructure plan generated with 5 resources. 
    Estimated cost: $186.34/month.
    
    Resources:
    • app-server-1 (t3.large) — $60.74/mo
    • primary-database (db.t3.medium) — $49.64/mo
    • application-load-balancer — $16.20/mo
    • app-vpc (NAT gateway) — $32.40/mo
    • app-storage (S3 100GB) — $2.30/mo
    
    ✅ Within budget ($300/mo)

User: "Make it highly available with 3 servers"

AI: Plan updated. Now has 7 resources. 
    New estimated cost: $329.06/month.
    ⚠️ Estimated cost ($329.06/mo) exceeds budget ($300.00/mo)

User: "Export as Terraform"

AI: Terraform code generated for 7 resources. 
    Copy the code to your Terraform workspace.
    
    [Terraform HCL output...]
```

---

## Cost Reference

Approximate monthly costs used for estimation:

| Resource | Monthly Cost (USD) |
|----------|-------------------|
| AWS t3.micro | $7.59 |
| AWS t3.medium | $30.37 |
| AWS m5.large | $70.08 |
| AWS m5.xlarge | $140.16 |
| AWS db.t3.medium | $49.64 |
| AWS db.r5.large | $175.20 |
| Load Balancer | $16.20 |
| NAT Gateway | $32.40 |
| S3 (100GB) | $2.30 |
| EBS gp3 (100GB) | $8.00 |
| Elastic IP | $3.60 |

Costs are estimates for planning purposes. Actual costs may vary based on usage, data transfer, and current provider pricing.

---

## Limitations

- Plan generation uses pattern matching and heuristics (not full AI model inference for resource selection)
- Cost estimates are approximate and based on on-demand pricing
- Terraform export supports AWS resources fully; Azure and GCP generate provider blocks but resource definitions are AWS-focused
- Plans are stored per-tenant in DynamoDB; no cross-tenant access
- Maximum 10 compute instances per plan request

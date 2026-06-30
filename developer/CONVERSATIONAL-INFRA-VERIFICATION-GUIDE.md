# Conversational Infrastructure Management — UI Verification Guide

This document describes how to verify each implemented feature from the CMP UI.

---

## Prerequisites (All Features)

1. **Start the stack**: `docker compose up -d` (or run backend + frontend separately)
2. **Login** as an admin user (has all permissions)
3. **Enable feature toggles**: Go to **Admin Settings → Feature Toggles** and enable:
   - `AI Chatbot` (if not already on)
   - `Cloud Operations via Chat` (for Feature 1)
   - `Deployments via Chat` (for Feature 2)
4. **Configure AI**: Go to **Admin Settings → AI Assistant** and ensure an API key is configured (Gemini or OpenAI)
5. **Add a credential**: Go to **Credentials** and add an AWS credential with valid access/secret keys
6. **Open AI Panel**: Press `⌘J` (Mac) or `Ctrl+J` (Windows/Linux) to open the AI assistant

---

## Feature 1: Direct Cloud Operations via Chat

### Where to Verify
- **AI Assistant Panel** (⌘J from any page)
- **Best tested from**: Resources page or Inventory page (AI gets resource context)

### Test Scenarios

#### 1.1 Resize an EC2 Instance
```
You: "Resize instance i-0abc123def to t3.large"
```
**Expected behavior:**
- AI identifies the resource and credential
- Shows a plan: current type → new type, cost impact, risk level (MEDIUM)
- Asks for confirmation
- On confirm: executes resize (stop → modify → start)
- Shows success message with new state

**What to look for:**
- ✅ Risk assessment shows "MEDIUM" level
- ✅ Cost impact calculated (e.g., "+$30/mo")
- ✅ Confirmation flow (not executed immediately)
- ✅ After confirmation: success message with duration

#### 1.2 Scale an Auto Scaling Group
```
You: "Scale my-web-asg to 6 instances"
```
**Expected:**
- Shows current capacity vs proposed
- Cost estimation per additional instance
- Asks for confirmation
- On confirm: modifies ASG desired capacity

#### 1.3 Modify Security Group (Admin Only)
```
You: "Open port 443 on sg-0abc123 for 10.0.0.0/8"
```
**Expected:**
- AI calls with appropriate SG params
- Risk: HIGH (security modification)
- Shows exactly what rule will be added
- Requires confirmation

#### 1.4 Security Group - Critical Risk (0.0.0.0/0)
```
You: "Add inbound SSH on sg-prod-web from 0.0.0.0/0"
```
**Expected:**
- Risk: CRITICAL (opens to internet + production name)
- Warning about 0.0.0.0/0 exposure
- Requires approval (non-admins blocked)
- Admin sees warning but can proceed after confirmation

#### 1.5 Create Snapshot (Low Risk)
```
You: "Create a snapshot of volume vol-0abc123"
```
**Expected:**
- Risk: LOW (non-destructive)
- Minimal confirmation required
- Quick execution

#### 1.6 Dry-Run Preview
```
You: "What would happen if I resize i-0abc123 to m5.xlarge?"
```
**Expected:**
- AI uses `cloud_operation_dry_run` tool
- Shows current state, proposed state, cost impact
- Does NOT execute anything
- Offers to proceed if user wants

#### 1.7 Role Enforcement
Login as a **regular user** (not developer/admin):
```
You: "Resize instance i-0abc123 to t3.large"
```
**Expected:**
- AI returns: "Cloud operations require DEVELOPER or ADMIN role."
- No operation executed

#### 1.8 Feature Toggle Off
Disable `Cloud Operations via Chat` in Feature Toggles, then try:
```
You: "Scale my-asg to 4"
```
**Expected:**
- AI returns: "Cloud Operations via Chat is not enabled..."
- Directs user to ask admin

### Verification Checklist (Feature 1)
- [ ] Resize instance works end-to-end (with real AWS credential)
- [ ] Scale group shows current/proposed capacity
- [ ] Security group modification blocked for non-admins
- [ ] 0.0.0.0/0 triggers CRITICAL risk
- [ ] Production resources get escalated risk
- [ ] Dry-run shows plan without executing
- [ ] Regular users are blocked
- [ ] Feature toggle off blocks all operations
- [ ] Cost impact shown for resize and scale operations
- [ ] Events visible in Event Log after execution


---

## Feature 2: Conversational Deployment Pipelines

### Where to Verify
- **AI Assistant Panel** (⌘J from any page)
- **Best tested from**: Dashboard or Executions page

### Test Scenarios

#### 2.1 Deploy a Service (Basic)
```
You: "Deploy order-service v2.3.1 to staging"
```
**Expected behavior:**
- AI asks for credential_id (or infers from context)
- Shows deployment plan: service, version, environment, strategy (rolling default)
- Shows previous version if a prior deployment exists
- Asks for confirmation
- On confirm: creates deployment record, starts execution
- Returns deployment_id for tracking

**What to look for:**
- ✅ Confirmation shows: service, version, environment, strategy, previous version
- ✅ After confirmation: deployment_id returned
- ✅ "Use get_deployment_status to monitor progress" suggested

#### 2.2 Deploy with Strategy
```
You: "Deploy payment-api v1.5.0 to production using blue/green strategy"
```
**Expected:**
- AI recognizes "blue/green" → sets strategy to `blue_green`
- Production deployment requires ADMIN role
- Plan shows blue/green specific steps:
  1. Deploy to inactive environment
  2. Health check inactive
  3. Switch traffic
  4. Verify active environment
  5. Cleanup

#### 2.3 Canary Deployment
```
You: "Deploy auth-service v3.0.0 to staging with 20% canary"
```
**Expected:**
- Strategy: canary
- Canary percentage: 20%
- Steps include: deploy canary, route 20% traffic, monitor, promote, verify

#### 2.4 Check Deployment Status
```
You: "How is my deployment going?"
```
or after deploying:
```
You: "Check status of deployment abc-123"
```
**Expected:**
- Shows step-by-step progress:
  ```
  ✓ Pre-deployment health check (2s)
  ✓ Pull/build artifact (3s)
  ⏳ Rolling update (running...)
  ○ Post-deployment health check
  ○ Cleanup
  Progress: 2/5 steps
  ```
- Status: running/completed/failed
- Duration, error details if failed

#### 2.5 List Deployments
```
You: "Show my recent deployments"
```
or:
```
You: "List failed deployments for order-service"
```
**Expected:**
- Table of deployments with: service, version, environment, status, time
- Filtered by ownership (non-admins see only their own)
- Admins see all deployments

#### 2.6 Rollback a Deployment
```
You: "Something is wrong with order-service, roll it back"
```
**Expected:**
- AI finds the latest deployment for order-service
- Shows: "Rollback order-service on staging from v2.3.1 → v2.3.0. Confirm?"
- On confirm: creates a new rollback deployment
- Uses IN_PLACE strategy for speed
- Original deployment marked as "rolled_back"

#### 2.7 Rollback by ID
```
You: "Rollback deployment abc-123"
```
**Expected:**
- Finds specific deployment
- Shows version change
- Asks for confirmation

#### 2.8 No Previous Version
```
You: "Rollback deployment xyz-first-ever"
```
**Expected:**
- "No previous version available for rollback."

#### 2.9 Production Restriction
Login as **developer** (not admin):
```
You: "Deploy my-api v2.0 to production"
```
**Expected:**
- "Production deployments require ADMIN role."
- Operation blocked

#### 2.10 Feature Toggle Off
Disable `Deployments via Chat` in Feature Toggles:
```
You: "Deploy order-service v1.0 to staging"
```
**Expected:**
- "Deployments via Chat is not enabled..."

### Verification Checklist (Feature 2)
- [ ] Basic deployment creates record and starts execution
- [ ] Rolling strategy shows correct steps
- [ ] Blue/green strategy shows correct steps
- [ ] Canary strategy with percentage
- [ ] Deployment status shows step-by-step progress
- [ ] List deployments filters by service/environment/status
- [ ] Non-admins only see their own deployments in list
- [ ] Rollback creates new deployment with previous version
- [ ] Rollback marks original as "rolled_back"
- [ ] "No previous version" handled gracefully
- [ ] Production deployments blocked for non-admins
- [ ] Feature toggle off blocks all deployment operations
- [ ] Events emitted (check Event Log for deployment.started, deployment.completed)
- [ ] Deployment with AWS ECS credential triggers real ECS update

---

## General Verification (Both Features)

### Event Log Verification
1. Go to **Event Log** page (Admin)
2. After running any operation, verify events appear:
   - `cloud_operation.executed` / `cloud_operation.failed`
   - `deployment.started` / `deployment.completed` / `deployment.failed` / `deployment.rolled_back`
3. Check event details include: actor, resource, metadata (operation type, risk level, etc.)

### Audit Trail Verification
1. Go to **Audit Logs** page (Admin)
2. Verify cloud operations and deployments create audit entries
3. Source should show "cloud_operations_chat" or "deployment"

### AI Conversation History
1. Open AI panel, perform an operation
2. Close and reopen the panel — conversation preserved
3. Click **Conversation History** — see the operation conversation listed

### Multi-Tenant Isolation
1. Switch to a different tenant
2. Verify deployments from other tenants are NOT visible
3. Verify operations only work with credentials accessible in current tenant

### Error Handling
Test with invalid inputs:
```
You: "Resize instance INVALID-ID to t3.large"
You: "Deploy to staging using invalid-credential"
You: "Scale non-existent-asg to 10"
```
**Expected:** Clear error messages, no crashes, graceful failures

---

## Troubleshooting

| Issue | Check |
|-------|-------|
| AI doesn't recognize cloud ops tools | Verify `chat_cloud_operations` toggle is ON |
| AI doesn't recognize deployment tools | Verify `chat_deployments` toggle is ON |
| "Cloud operations require DEVELOPER or ADMIN role" | Login as developer or admin |
| "You don't have access to this credential" | Ensure credential is accessible to your user/role |
| "AWS only" message | Cloud operations currently only support AWS credentials |
| Deployment created but stays "pending" | Check backend logs for background task errors |
| No events in Event Log | Verify EventBus is running (check backend startup logs) |
| DynamoDB errors | Ensure `deployments` table exists (auto-created on startup) |


---

## Feature 7: Infrastructure-as-Conversation (Plan Generator)

### Where to Verify
- **AI Assistant Panel** (⌘J from any page)
- **Best tested from**: Any page — no resource context needed

### Prerequisites
- AI chat tool registration for `plan_infrastructure`, `modify_infra_plan`, `estimate_plan_cost`, `export_terraform` must be complete
- DynamoDB table `infra_plans` must be created at startup

### Test Scenarios

#### 7.1 Generate a Basic Plan
```
You: "I need a web server with a PostgreSQL database on AWS in us-east-1"
```
**Expected behavior:**
- AI generates a plan with compute + database + VPC + load balancer resources
- Shows resource list with names, types, and individual costs
- Shows total estimated monthly cost
- Returns a plan_id for further operations

**What to look for:**
- ✅ Resources include: compute instance, database, VPC, load balancer (auto-added)
- ✅ Each resource has estimated_monthly_cost
- ✅ Provider is "aws", region is "us-east-1"
- ✅ Dependencies created (compute depends on network)

#### 7.2 Generate Plan with Budget Constraint
```
You: "I need 3 production servers with a database, budget $200/month"
```
**Expected:**
- Plan generated with multiple compute instances + database
- Budget warning: "⚠️ Estimated cost exceeds budget"
- Plan still generated (warning only, not rejected)

#### 7.3 Multi-Instance Extraction
```
You: "Set up 5 web servers behind a load balancer"
```
**Expected:**
- Exactly 5 compute instances created (app-server-1 through app-server-5)
- Load balancer included
- VPC included (auto-added with compute)
- Dependencies: compute → VPC, load balancer → compute

#### 7.4 High Availability
```
You: "I need a highly available database setup"
```
**Expected (when HA constraint set):**
- Database with multi-AZ enabled
- Cost doubled for multi-AZ database
- At least 3 compute instances if "ha" in description

#### 7.5 Azure Provider
```
You: "I need a medium production server on Azure in westus2"
```
**Expected:**
- Provider: "azure"
- Instance type: Standard_D2s_v3 (medium) or Standard_D4s_v3 (production)
- Region: "westus2"

#### 7.6 GCP Provider
```
You: "Set up a small dev server on GCP"
```
**Expected:**
- Provider: "gcp"
- Instance type: e2-medium (small/dev)

#### 7.7 Modify Plan — Remove Resources
```
You: "Remove the database from that plan"
```
**Expected:**
- Resources of type "database" removed
- Total cost recalculated
- Message confirms the update

#### 7.8 Modify Plan — Add Resources
```
You: "Add a database to plan <plan_id>"
```
**Expected:**
- New database resource appended (db.t3.medium, $49.64/mo)
- Total cost updated

#### 7.9 Modify Plan — Scale Up
```
You: "Make the servers bigger"
```
**Expected:**
- Compute resources upgraded to m5.large ($70.08/mo each)
- Total cost recalculated

#### 7.10 Cost Breakdown
```
You: "Show me the cost breakdown for plan <plan_id>"
```
**Expected:**
- Per-resource breakdown with name, type, monthly cost, annual cost
- Total monthly and annual costs
- Currency: "USD"

#### 7.11 Export Terraform
```
You: "Export plan <plan_id> as Terraform"
```
**Expected:**
- Complete HCL code with:
  - Provider block with region
  - `aws_instance` for compute resources
  - `aws_db_instance` for databases (with multi_az, engine)
  - `aws_s3_bucket` for storage (with random suffix)
  - `aws_vpc` for networking (with CIDR)
  - `aws_lb` for load balancers
  - `random_id` resource for unique naming
  - Tags: `Name` and `ManagedBy = "CMP"` on all resources
- Plan status updated to "exported"
- Header comment with plan ID and generation timestamp

#### 7.12 Default Fallback
```
You: "Set up some infrastructure"
```
**Expected:**
- Even vague descriptions produce at least 1 resource (basic compute)
- No errors or empty plans returned

#### 7.13 Plan Not Found
```
You: "Export terraform for plan non-existent-id"
```
**Expected:**
- Error response: "Plan 'non-existent-id' not found"
- No crash

### Verification Checklist (Feature 7)
- [ ] Basic plan generation returns correct resources for description
- [ ] AWS instance types match description keywords (small → t3.small, production → m5.xlarge)
- [ ] Azure instance types selected correctly
- [ ] GCP instance types selected correctly
- [ ] Count extraction works ("3 servers" → 3 instances, capped at 10)
- [ ] "ha" / "high availability" → 3 instances
- [ ] Budget warning when cost exceeds constraint
- [ ] VPC auto-included when compute exists
- [ ] Load balancer auto-included when multiple compute or explicitly requested
- [ ] Dependencies: compute → network, LB → compute
- [ ] Modify: remove by resource type works
- [ ] Modify: add database/server works
- [ ] Modify: scale up changes instance types
- [ ] Cost breakdown shows per-resource and totals
- [ ] Terraform export generates valid-looking HCL
- [ ] Terraform includes all resource types
- [ ] Plan persisted to DynamoDB (check `infra_plans` table)
- [ ] Tenant isolation: plans scoped to tenant_id
- [ ] Plan not found returns clear error (no crash)


---

## Feature 3: Scaling & Auto-Scaling Configuration

### Where to Verify
- **AI Assistant Panel** (⌘J), best from Resources or Inventory page
- **Enable toggle**: `Scaling via Chat`

### Test Scenarios

#### 3.1 Get Scaling Config
```
You: "Show the scaling config for my-web-asg"
```
**Expected:** Shows min/max/desired capacity, target metric, cooldowns

#### 3.2 Update Scaling Policy
```
You: "Set my-web-asg to min 2, max 10, target CPU 70%"
```
**Expected:** Shows current vs proposed, asks confirmation, updates ASG

#### 3.3 Create Scaling Schedule
```
You: "Scale my-web-asg to 6 instances every weekday at 8am"
```
**Expected:** Creates a scheduled scaling action

#### 3.4 Get Recommendations
```
You: "Are any of my resources over-provisioned?"
```
**Expected:** Analyzes metrics and suggests optimizations with savings estimates

---

## Feature 4: Configuration Management via Chat

### Where to Verify
- **AI Assistant Panel**, best from Resources page with a resource selected
- **Enable toggle**: `Config Management via Chat`

### Test Scenarios

#### 4.1 Get Resource Config
```
You: "Show tags on instance i-0abc123"
```
**Expected:** Lists all tags/env vars for the resource

#### 4.2 Modify Config (Tag)
```
You: "Add tag environment=staging to instance i-0abc123"
```
**Expected:** Preview → confirm → applied

#### 4.3 Bulk Tag
```
You: "Tag all untagged EC2 instances with team=platform"
```
**Expected:** Finds matching resources, shows count, asks confirmation

#### 4.4 Rollback Config Change
```
You: "Undo that tag change"
```
**Expected:** Reverts to previous values using stored changeset

---

## Feature 5: Proactive AI Recommendations

### Where to Verify
- **AI Assistant Panel** from any page
- **Enable toggle**: `Proactive AI Recommendations`

### Test Scenarios

#### 5.1 Get Recommendations
```
You: "Do you have any recommendations for me?"
```
**Expected:** Shows categorized list (cost, security, lifecycle, performance)

#### 5.2 Generate Fresh Analysis
```
You: "Analyze my resources for optimizations"
```
**Expected:** Runs fresh analysis, finds idle resources, expiring leases, budget proximity

#### 5.3 Act on Recommendation
```
You: "Dismiss the first recommendation"
```
**Expected:** Marks as dismissed, won't show again

---

## Feature 6: Multi-Step Guided Conversations (Wizards)

### Where to Verify
- **AI Assistant Panel** from any page
- **Enable toggle**: `Guided Conversations (Wizards)`

### Test Scenarios

#### 6.1 List Wizards
```
You: "What guided workflows are available?"
```
**Expected:** Lists available wizards appropriate for user's role

#### 6.2 Start a Wizard
```
You: "Help me provision a new environment"
```
**Expected:** Starts wizard, asks first question (e.g., "Which cloud provider?")

#### 6.3 Step Through Wizard
```
You: "AWS"  (answering provider question)
You: "us-east-1"  (answering region question)
You: "t3.medium"  (answering instance type question)
```
**Expected:** Each answer advances to next step, shows progress, ends with summary

#### 6.4 Wizard Summary
After all steps:
**Expected:** Shows complete plan with all collected answers, asks "Execute this plan?"

---

## Feature 7: Infrastructure-as-Conversation

### Where to Verify
- **AI Assistant Panel** from any page
- **Enable toggle**: `Infrastructure Planning via Chat`

### Test Scenarios

#### 7.1 Generate Plan
```
You: "I need a 3-tier web application with load balancer, 2 web servers, and a PostgreSQL database"
```
**Expected:** Generates plan with resources, dependencies, cost estimate

#### 7.2 Modify Plan
```
You: "Make the database bigger and add a Redis cache"
```
**Expected:** Updates plan with new resources, shows revised cost

#### 7.3 Cost Estimate
```
You: "How much will this plan cost?"
```
**Expected:** Detailed per-resource cost breakdown (monthly/annual)

#### 7.4 Export Terraform
```
You: "Export this plan as Terraform"
```
**Expected:** Generates HCL code for all resources in the plan

---

## Feature 8: Conversational Terraform Operations

### Where to Verify
- **AI Assistant Panel**, best from Terraform Workspaces page
- **Enable toggle**: `Terraform Operations via Chat`

### Test Scenarios

#### 8.1 Plan
```
You: "Run a terraform plan on my networking workspace"
```
**Expected:** Summarizes what will be created/changed/destroyed in plain language

#### 8.2 Apply
```
You: "Apply the changes"
```
**Expected:** Asks for confirmation (HIGH risk), then triggers apply

#### 8.3 Drift Detection
```
You: "Check for drift in my production workspace"
```
**Expected:** Explains any drift in human-readable format with remediation options

#### 8.4 Workspace Status
```
You: "Show my terraform workspace status"
```
**Expected:** Lists workspaces with last run status, resource count, state

#### 8.5 Set Variable
```
You: "Set the instance_count variable to 4 in my web workspace"
```
**Expected:** Updates workspace variable

---

## Feature 9: Real-Time Operation Streaming

### Where to Verify
- **API endpoint**: `GET /api/v1/chat/stream/{operation_id}`
- **Enable toggle**: `Real-Time Operation Streaming`

### Test Scenarios

#### 9.1 Subscribe to Stream
After starting any long-running operation (deployment, terraform apply):
```bash
curl -N -H "Authorization: Bearer <token>" \
  http://localhost:8001/api/v1/chat/stream/<operation_id>
```
**Expected:** Receives SSE events as operation progresses

#### 9.2 Cancel Operation
```bash
curl -X POST -H "Authorization: Bearer <token>" \
  http://localhost:8001/api/v1/chat/stream/<operation_id>/cancel
```
**Expected:** Operation cancelled, stream closes with cancelled event

---

## Feature 10: ChatOps Integration (Slack/Teams)

### Where to Verify
- **API endpoints**: `/api/v1/chatops/*`
- **Enable toggle**: `ChatOps Integration`

### Test Scenarios

#### 10.1 Get Config
```bash
curl -H "Authorization: Bearer <token>" \
  http://localhost:8001/api/v1/chatops/config
```
**Expected:** Returns current ChatOps configuration

#### 10.2 Update Config (Slack)
```bash
curl -X PUT -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"slack_enabled": true, "slack_bot_token": "xoxb-..."}' \
  http://localhost:8001/api/v1/chatops/config
```
**Expected:** Saves encrypted configuration

#### 10.3 Link Identity
```bash
curl -X POST -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"platform": "slack", "external_user_id": "U12345", "external_username": "john"}' \
  http://localhost:8001/api/v1/chatops/link
```
**Expected:** Creates identity mapping

#### 10.4 Slack Event Webhook
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"type": "event_callback", "event": {"type": "app_mention", "text": "show my resources", "user": "U12345"}}' \
  http://localhost:8001/api/v1/chatops/slack/events
```
**Expected:** Routes message to chat engine, returns formatted response

---

## Complete Implementation Status

| # | Feature | Tools | Events | Toggle | Status |
|---|---------|-------|--------|--------|--------|
| 1 | Cloud Operations | 7 | 3 | `chat_cloud_operations` | ✅ |
| 2 | Deployments | 4 | 4 | `chat_deployments` | ✅ |
| 3 | Scaling | 4 | 3 | `chat_scaling` | ✅ |
| 4 | Config Management | 5 | 4 | `chat_config_management` | ✅ |
| 5 | Proactive AI | 3 | 3 | `chat_proactive_ai` | ✅ |
| 6 | Guided Wizards | 4 | 3 | `chat_guided_flows` | ✅ |
| 7 | Infra Planning | 4 | 3 | `chat_infra_planning` | ✅ |
| 8 | Terraform Chat | 5 | 3 | `chat_terraform_ops` | ✅ |
| 9 | Streaming (SSE) | — | 3 | `chat_streaming` | ✅ |
| 10 | ChatOps | — | 3 | `chatops_integration` | ✅ |
| **Total** | | **36 new** | **32** | **10** | |

Total chat tools after implementation: **88** (52 original + 36 new)

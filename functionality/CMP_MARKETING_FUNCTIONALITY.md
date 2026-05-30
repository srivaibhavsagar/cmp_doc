# Cloud Management Platform — Product Capabilities

**Tagline:** One platform to provision, manage, and govern your entire multi-cloud estate.

---

## What is CMP?

The Cloud Management Platform (CMP) is an enterprise-grade, multi-tenant platform that gives organizations complete control over their cloud infrastructure across AWS, Azure, and GCP — from a single, unified interface.

CMP replaces fragmented tooling with a self-service portal where teams can order pre-approved cloud services, automate provisioning workflows, enforce governance policies, and track costs — all without writing tickets or waiting on ops teams.

---

## Core Capabilities

### 1. Self-Service Cloud Catalog

Give your teams a curated storefront of pre-approved cloud services they can order in minutes.

- **Visual Form Builder** — Design rich request forms with dropdowns, conditional fields, validations, and multi-tab layouts without writing code
- **Custom UI Support** — Bring your own HTML/JS interface for advanced ordering experiences
- **Role-Based Visibility** — Control which teams and roles can see and order each service
- **Real-Time Cost Estimates** — Show users the projected cost before they submit, including platform charges and live cloud pricing
- **Approval Workflows** — Route high-risk or high-cost requests through configurable approval chains
- **Conditional Approvals** — Only require approval when specific conditions are met (e.g., production environment, large instance types)

### 2. Multi-Cloud Provisioning Automation

Automate any cloud operation with a powerful, composable automation engine.

- **Code Tasks** — Write automation in Python, TypeScript, Bash, Go, SQL, or REST API calls
- **Multi-Step Workflows** — Orchestrate complex provisioning with DAG-based workflows supporting parallel execution, conditional branching, retries, and rollback
- **Flow Orchestration** — Chain multiple workflows into end-to-end provisioning pipelines
- **Dry-Run Mode** — Preview what an execution will do before committing real cloud changes
- **Step-Level Retry** — Resume failed executions from the exact point of failure
- **Scheduled Automation** — Run recurring tasks on cron schedules (backups, cleanups, reports)
- **Reusable Components** — Build once, use everywhere — shared form fields and automation blocks

### 3. Multi-Cloud Resource Management

Discover, view, and operate on cloud resources across all your accounts from one place.

- **Unified Resource View** — See EC2 instances, S3 buckets, RDS databases, Azure VMs, GCP Compute, and more in a single pane
- **Multi-Account Support** — Connect unlimited cloud accounts across AWS, Azure, and GCP
- **Day 2 Operations** — Define custom actions (scale, patch, backup, migrate) that users can execute on any resource
- **Resource Lifecycle Tracking** — Full inventory with lease management, expiry warnings, and auto-cleanup
- **Execution History** — Complete audit trail of every action performed on every resource
- **Native Actions** — Start, stop, restart, and terminate resources directly from the UI

### 4. Terraform Infrastructure-as-Code

Manage your Terraform lifecycle end-to-end without leaving the platform.

- **Template Library** — Store and version Terraform configurations from inline code, Git repos, file uploads, or the Terraform Registry
- **Workspace Management** — Deploy, update, and destroy infrastructure with full state tracking
- **Day 2 Variable Updates** — Modify configurable variables on live infrastructure and re-apply safely
- **Drift Detection** — Automatically detect when real infrastructure diverges from desired state
- **Drift Remediation** — One-click remediation to bring infrastructure back into compliance
- **Flexible State Backends** — Store state in the built-in backend, AWS S3, or Terraform Cloud
- **Module Registry** — Maintain a library of reusable Terraform modules for your organization

### 5. Governance & Compliance

Enforce organizational standards automatically — before resources are provisioned, not after.

- **Policy Engine** — Define rules that block or warn on non-compliant requests (e.g., "deny metal instances in dev environments")
- **Resource Quotas** — Set limits per user, team, or organization to prevent sprawl
- **Approval Workflows** — Multi-approver chains with comments, justification, and full audit trail
- **Conditional Policies** — Apply rules based on form data, tags, catalog type, or any custom field
- **Real-Time Evaluation** — Policies checked at request time with instant user feedback

### 6. Cost Management & FinOps

Full visibility and control over cloud spending across your organization.

- **Cost Models** — Define platform charges (percentage-based or fixed fees) applied automatically to every request
- **Budget Management** — Set monthly, quarterly, or annual budgets with configurable alert thresholds
- **Live Cloud Pricing** — Show real-time pricing from AWS, Azure, and GCP during the ordering process
- **Cost Analytics** — Aggregated spending views by user, team, service, and cloud provider
- **Budget Alerts** — Automatic notifications via email, Slack, or Teams when spending approaches limits
- **Cost Breakdown** — Transparent itemized view of cloud costs plus platform charges before submission
- **Reports & Dashboards** — Visual analytics for spend trends, utilization, and optimization opportunities

### 7. Event-Driven Automation

React to any platform event in real time with configurable automation rules.

- **60+ Event Types** — Every significant action emits a structured event (provisioning, approvals, cost thresholds, user activity, drift detection)
- **Automation Rules** — "When X happens, do Y" — with conditions, priorities, and multiple action types
- **Action Library** — Send notifications, post to Slack/Teams, trigger workflows, call webhooks, execute tasks, or chain events
- **Conditional Logic** — Filter events by any field using operators (equals, contains, greater than, exists, etc.)
- **Full Audit Trail** — Every event logged with actor, resource, timestamp, and metadata for compliance

### 8. Identity & Access Management

Enterprise-grade security with flexible authentication and fine-grained access control.

- **Role-Based Access Control** — Four-tier role hierarchy (Admin, Developer, User, Readonly) with granular permissions
- **Single Sign-On (SSO)** — Connect your existing identity provider (Microsoft Entra ID, Google Workspace, Okta, AWS Identity Center, Ping Identity)
- **Automatic Group Sync** — Map external IdP groups to platform roles automatically
- **Auto-Provisioning** — New users created automatically on first SSO login
- **Multi-Factor via IdP** — Leverage your IdP's MFA policies seamlessly
- **API Tokens** — Long-lived tokens for CI/CD and programmatic access
- **Session Management** — Token-based sessions with configurable expiry and revocation

### 9. Multi-Tenancy

Complete organizational isolation for enterprises, MSPs, and SaaS providers.

- **Tenant Isolation** — Each organization gets fully isolated data, users, configurations, and resources
- **Custom Branding** — Per-tenant logo, company name, and color scheme
- **Tenant Switching** — Users belonging to multiple organizations switch context instantly
- **Independent Configuration** — Each tenant has its own policies, budgets, feature toggles, and approval chains
- **Scalable Plans** — Configurable user and resource limits per tenant

### 10. AI-Powered Cloud Copilot

An enterprise-grade AI copilot that understands your entire cloud environment and acts as your intelligent assistant.

- **Context-Aware Intelligence** — Automatically adapts to your current page, selected resource, and role. Say "restart this" while viewing a VM — it knows exactly which one
- **Natural Language Provisioning** — "Provision a t3.medium EC2 in us-west-2" opens the form pre-filled with your specifications
- **Proactive Insights** — Dashboard cards showing expiring leases, failed executions, budget alerts, and sync errors before you ask
- **Catalog Recommendations** — "I need a cheap Linux VM for testing" returns ranked catalog suggestions with cost estimates
- **Execution Troubleshooting** — "Why did my deployment fail?" analyzes step logs and explains errors in plain language with fix suggestions
- **Live Cloud Pricing** — Compare instance costs across AWS, Azure, and GCP in real-time
- **Role-Based Filtering** — Full platform awareness with strict RBAC: users see only their own data, admins see everything
- **API Execution** — Developers can run any platform API endpoint through natural language
- **Ownership-Aware Queries** — "Show my resources" automatically filters to your inventory without configuration
- **Policy Explanation** — Understand why a request was blocked and how to comply
- **Quota Checking** — Verify resource availability before provisioning to prevent failures
- **One-Click Actions** — Start, stop, restart, lease, or terminate resources directly from conversation
- **Contextual Inline AI Suggestions** — AI action buttons appear directly in the UI where issues occur (e.g., "Ask AI to explain this error" on failed executions, "Suggest optimizations" on resource pages, "Generate Terraform" on resource details). No need to open the AI panel or context-switch
- **AI-Powered Catalog Recommendations** — The catalog page proactively suggests relevant items based on your usage patterns, role, and stated intent. Includes "users like you also provisioned" recommendations and intent-matched search results
- **Persistent Conversation History** — Every AI conversation is automatically saved per user in DynamoDB. Resume past conversations, reference previous answers, and build on prior context across sessions without starting over

### 11. Notifications & Integrations

Keep everyone informed through the channels they already use.

- **In-App Notifications** — Real-time notification center with unread badges
- **Email Notifications** — SMTP-based email delivery for approvals, alerts, and updates
- **Slack Integration** — Post to Slack channels for team visibility
- **Microsoft Teams** — Deliver notifications to Teams channels
- **PagerDuty** — Escalate critical alerts to on-call teams
- **Inbound Webhooks** — Accept events from external systems to trigger CMP actions
- **Outbound Webhooks** — Call external APIs when platform events occur

### 12. Feature Management

Control exactly which capabilities are available to which users.

- **24 Toggleable Features** — Enable or disable any platform module independently
- **Granular Targeting** — Restrict features by role, team, or individual user
- **Progressive Rollout** — Enable new features for pilot groups before organization-wide release
- **Zero Downtime** — Toggle features on or off without restarting or redeploying

---

## Who Is CMP For?

### Platform Engineering Teams
Build golden paths for developers. Create self-service catalogs that enforce standards while giving teams the speed they need.

### Cloud Operations Teams
Automate repetitive provisioning tasks. Replace manual runbooks with repeatable, auditable workflows.

### FinOps Teams
Get visibility into cloud spending. Set budgets, define cost models, and catch overruns before they happen.

### Security & Compliance Teams
Enforce policies at the point of request. Every action is logged, every resource is tracked, every approval is auditable.

### Managed Service Providers (MSPs)
Serve multiple customers from a single platform with complete tenant isolation, custom branding, and independent configurations.

### Development Teams
Order the infrastructure you need in minutes, not days. No tickets, no waiting — just fill a form and go.

---

## Key Differentiators

| Capability | CMP | Traditional ITSM | Cloud-Native Tools |
|-----------|-----|-------------------|-------------------|
| Multi-cloud (AWS + Azure + GCP) | ✅ | ❌ | Partial |
| Self-service catalog with forms | ✅ | ✅ | ❌ |
| Terraform lifecycle management | ✅ | ❌ | Partial |
| Policy-as-code governance | ✅ | ❌ | ✅ |
| Built-in cost models & budgets | ✅ | ❌ | ❌ |
| Event-driven automation | ✅ | ❌ | Partial |
| Multi-tenant with custom branding | ✅ | ❌ | ❌ |
| SSO with 5 providers | ✅ | ✅ | Partial |
| AI assistant | ✅ | ❌ | ❌ |
| Drift detection & remediation | ✅ | ❌ | Partial |
| No vendor lock-in | ✅ | ❌ | ❌ |

---

## Platform at a Glance

| Metric | Value |
|--------|-------|
| Supported Cloud Providers | 3 (AWS, Azure, GCP) |
| SSO Providers | 5 (Microsoft, Google, Okta, AWS IC, Ping) |
| Automation Languages | 6 (Python, TypeScript, Bash, Go, SQL, REST) |
| Event Types | 60+ |
| Toggleable Features | 24 |
| Form Field Types | 18 |
| Terraform Source Types | 4 (Inline, Git, Upload, Registry) |
| Notification Channels | 5 (In-app, Email, Slack, Teams, PagerDuty) |
| Role Levels | 4 (Admin, Developer, User, Readonly) |

---

## Use Cases

### Self-Service VM Provisioning
Users select instance type, region, and OS from a catalog form. CMP evaluates policies, checks quotas, calculates cost, routes for approval if needed, then provisions via Terraform — all in minutes.

### Automated Environment Teardown
Scheduled jobs automatically destroy development environments every Friday evening. Lease management warns owners before expiry and cleans up forgotten resources.

### Cost Guardrails
Policies block requests for oversized instances in non-production environments. Budgets alert finance teams at 80% spend. Cost models add a 15% platform fee transparently.

### Compliance-Ready Provisioning
Every request goes through policy evaluation, approval workflows, and event logging. Auditors get a complete trail from request to resource with who, what, when, and why.

### Multi-Team Governance
Each team gets their own quotas, budgets, and credential visibility. Platform-wide policies ensure consistency while team-level controls provide flexibility.

### Terraform at Scale
Manage hundreds of Terraform workspaces with drift detection, Day 2 variable updates, and centralized state management — without giving every developer direct Terraform access.

---

## Getting Started

1. **Deploy** — Run CMP via Docker in minutes or deploy to your cloud of choice
2. **Connect** — Add your AWS, Azure, or GCP credentials
3. **Build** — Create your first catalog item with the visual form builder
4. **Publish** — Make it available to your team
5. **Order** — Users self-serve from the catalog immediately

---

*CMP — Cloud management that works the way your teams do.*

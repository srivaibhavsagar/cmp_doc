# Proactive AI Recommendations

> CMP's AI generates actionable recommendations based on resource patterns, cost anomalies, and health signals — delivered as conversational suggestions you can act on immediately.

---

## Overview

Proactive AI Recommendations transforms the AI Assistant from a reactive tool into an intelligent advisor. The system continuously analyzes your infrastructure and surfaces recommendations for cost savings, security improvements, performance optimizations, and compliance fixes.

This feature requires the `chat_proactive_ai` feature toggle to be enabled.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Cost Recommendations | Identify idle resources, right-sizing opportunities, savings plans |
| Security Alerts | Open ports, public access, expired certificates |
| Performance Insights | High utilization, scaling opportunities |
| Lifecycle Warnings | Lease expiry, unused resources, drift detection |
| One-Click Actions | Act on recommendations directly from the chat |

---

## Prerequisites

1. **AI Assistant** enabled (Settings → Feature Toggles → `AI Chatbot`)
2. **Proactive AI** enabled (Settings → Feature Toggles → `chat_proactive_ai`)
3. Resources must be discoverable via stored credentials

---

## How to Use

### View Recommendations

```
"What recommendations do you have for me?"
"Show me cost optimization suggestions"
"Any security issues I should know about?"
```

### Filter by Category or Severity

```
"Show critical recommendations only"
"What performance issues exist?"
"Give me cost recommendations"
```

Available categories: `cost`, `security`, `health`, `drift`, `lifecycle`, `performance`, `compliance`

Available severities: `info`, `warning`, `critical`

### Act on a Recommendation

```
"Apply recommendation rec-123"
"Dismiss the idle instance recommendation"
"Snooze that alert for a week"
```

### Generate Fresh Recommendations

```
"Run a new analysis of my infrastructure"
"Refresh recommendations"
```

Triggers a new scan and returns updated recommendations.

---

## Recommendation Categories

| Category | Examples | Typical Severity |
|----------|----------|-----------------|
| **Cost** | Idle instances, oversized resources, unused EBS volumes | Warning |
| **Security** | Open security groups, public S3 buckets, weak encryption | Critical |
| **Health** | High error rates, degraded services, unhealthy targets | Warning–Critical |
| **Drift** | Terraform state drift, config drift from baseline | Warning |
| **Lifecycle** | Expiring leases, stale snapshots, unused credentials | Info–Warning |
| **Performance** | CPU saturation, memory pressure, scaling needed | Warning |
| **Compliance** | Missing tags, policy violations, audit findings | Warning–Critical |

---

## Recommendation Lifecycle

```
pending → seen → acted / dismissed / snoozed
```

| Status | Meaning |
|--------|---------|
| `pending` | New recommendation, not yet viewed |
| `seen` | User has viewed the recommendation |
| `acted` | User applied the suggested action |
| `dismissed` | User chose to ignore this recommendation |
| `snoozed` | Temporarily hidden (will resurface later) |

---

## Example Conversation

```
AI:   "💡 I found 3 recommendations for your infrastructure:

       1. COST (Warning): Instance i-0abc123 has been idle for 7 days 
          (avg CPU: 2%). Estimated savings: $52/month if stopped.
          → [Stop Instance] [Dismiss]

       2. SECURITY (Critical): Security group sg-prod-api allows 
          inbound traffic on port 22 from 0.0.0.0/0.
          → [Restrict to VPC CIDR] [Review]

       3. LIFECYCLE (Info): 3 EBS snapshots older than 90 days 
          consuming 450 GB. Estimated savings: $11/month.
          → [Delete Old Snapshots] [Dismiss]"

User: "Restrict the SSH access to our VPC only"

AI:   "I'll update sg-prod-api to allow port 22 only from 10.0.0.0/16.
       This is a HIGH risk operation. Confirm?"

User: "Yes"

AI:   "Done. SSH access restricted to VPC CIDR. Recommendation marked as resolved."
```

---

## Permissions

All users can view recommendations relevant to their accessible resources. Acting on recommendations follows the same permission model as the underlying operation (e.g., modifying a security group requires Admin role).

---

## Admin Configuration

Admins can configure proactive AI behavior:

| Setting | Description | Default |
|---------|-------------|---------|
| Enabled categories | Which recommendation types to generate | All |
| Severity threshold | Minimum severity to surface to users | Info |
| Scan frequency | How often to run analysis | Hourly |
| Digest mode | Send daily/weekly summary instead of real-time | Off |

---

## Inline Resource AI Actions

In addition to the chat-based recommendations above, CMP surfaces contextual AI-suggested actions directly on resource detail pages. These appear as compact action buttons beneath the resource header, letting you act on common operations without opening the AI panel first.

### Prerequisites

- **AI Assistant** enabled (`chatbot` feature toggle)
- **AI Inline Suggestions** enabled (`ai_inline_suggestions` feature toggle)
- Both toggles fall under the `ai_assistant` license key

### How It Works

The system combines three tiers of actions, sorted by priority, and displays up to 4 at a time:

#### Tier 1: Native State-Based Actions (Highest Priority)

These appear only when the resource's `available_actions` list includes them AND the resource is in the correct state:

| Action | Condition | Suggestion Label |
|--------|-----------|-----------------|
| `start` | Resource is stopped | "Start this resource" |
| `stop` | Resource is running | "Stop to save costs" |
| `restart` | Resource is running | "Restart this resource" |
| `lease` | Available in actions list | "Set a lease (auto-expire)" |
| `unlease` | Available in actions list | "Remove lease (keep running)" |
| `terminate` | Resource is stopped | "Terminate (stopped & unused?)" |

Native actions are only suggested when they make contextual sense. For example, "stop" only appears for running resources, and "terminate" only appears for stopped resources to avoid accidental destruction.

#### Tier 2: Automation Actions (Medium Priority)

Admin-defined automation actions attached to the resource appear as purple "Workflow" buttons. Up to 2 automation actions display, each showing the custom action name configured by the admin. If the automation requires approval, the AI prompt notes this.

#### Tier 3: Lifecycle & Governance Suggestions (Lower Priority)

These fill remaining slots (up to the 4-button cap) based on resource state:

| Condition | Suggestion | Example Prompt |
|-----------|-----------|----------------|
| No tags on resource | "Add tags (untagged)" | Add standard tags (environment, team, project) to my-instance. It currently has no tags. |
| Monthly cost exceeds $100 | "~$X/mo — right-sized?" | This EC2 (my-instance) costs ~$150/month. Is it right-sized? Can I downsize to save money? |
| Running resource | "Security check" | Check the security posture of my-instance. Are there open ports, public access, or missing encryption? |

### Behavior

- Suggestions are generated client-side based on resource metadata (status, available actions, automation actions, tags, cost).
- Clicking any suggestion opens the AI chat panel with a context-aware pre-filled prompt — no typing needed.
- A maximum of 4 suggestions display at once, sorted by priority (native actions first, automation second, lifecycle third).
- Suggestions update automatically when resource state changes (e.g., after starting a stopped resource, the "Start" button disappears and "Stop" may appear).
- Automation actions show whether approval is required in the generated AI prompt.

### Priority Order

Actions are displayed in this priority order (lowest number = shown first):

1. Start (for stopped resources)
2. Stop (for running resources)
3. Restart / Automation actions
4. Lease management
5. Terminate
6. Add tags
7. Right-sizing
8. Security check

### Supported Resource Types

Inline action suggestions appear on all resource types. Native actions (start, stop, restart, terminate) are driven by the backend's `available_actions` field, so they only appear for resource types that support them. Lifecycle suggestions (tags, cost, security) apply broadly to any resource matching their conditions.

---

## Limitations

- Recommendations are based on available CloudWatch/monitoring metrics
- Cost estimates are approximate (based on on-demand pricing)
- Analysis limited to resources accessible via stored credentials
- Learning from user dismiss/snooze patterns is planned but not yet active
- Inline resource suggestions are client-side heuristics — they do not run a live infrastructure scan
- Native action suggestions depend on the backend providing `available_actions` in the resource response
- Automation action suggestions depend on the backend providing `automation_actions` in the resource response


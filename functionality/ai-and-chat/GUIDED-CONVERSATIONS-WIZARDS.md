# Guided Conversations (Wizards)

> Multi-step conversational wizards that walk users through complex operations — collecting inputs step by step, skipping what can be inferred, and summarizing a full plan before execution.

---

## Overview

Guided Conversations provide structured, multi-step workflows within the AI chat interface. Instead of requiring users to know all parameters upfront, the AI asks questions one at a time, provides intelligent defaults, and skips steps it can infer from context.

This feature requires the `chat_guided_flows` feature toggle to be enabled.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Step-by-Step Guidance | Complex operations broken into manageable questions |
| Context-Aware Skipping | AI auto-fills steps it can infer from your environment |
| Progress Tracking | See which step you're on and what's left |
| Resume Later | Save progress and return to an incomplete wizard |
| Plan Summary | Full review of all choices before execution |
| Role-Based Templates | Wizards available based on your role |

---

## Prerequisites

1. **AI Assistant** enabled (Settings → Feature Toggles → `AI Chatbot`)
2. **Guided Conversations** enabled (Settings → Feature Toggles → `chat_guided_flows`)

---

## How to Use

### List Available Wizards

```
"What wizards are available?"
"Show me guided setup options"
"Help me set up something new"
```

Returns a list of wizard templates available for your role.

### Start a Wizard

```
"Start the microservice setup wizard"
"I want to set up a new environment"
"Guide me through database migration"
```

### Respond to Steps

The AI will ask questions one at a time. Simply answer each:

```
AI:   "Which cloud provider? (AWS / Azure / GCP)"
User: "AWS"

AI:   "Which region?"
User: "us-west-2"

AI:   "What instance size do you need? (small / medium / large)"
User: "medium"
```

### Check Progress

```
"Where am I in the wizard?"
"What's my current step?"
```

### Resume a Previous Session

```
"Resume my last wizard"
"Continue where I left off"
```

---

## Built-in Wizard Templates

| Template | Description | Required Role |
|----------|-------------|---------------|
| Setup Microservice Environment | Provider → Region → Compute → DB → DNS | Developer |
| Secure a Public Resource | Identify → Assess → Fix | Developer |
| Migrate Database | Source → Target → Strategy → Schedule | Admin |
| Set Up CI/CD Pipeline | Repo → Build → Test → Deploy | Developer |
| Onboard New Team Member | Name → Role → Groups → Credentials | Admin |
| Cost Optimization Review | Scan → Review → Apply savings | Developer |

---

## Wizard Session Lifecycle

```
active → completed
       → abandoned (timeout or user cancels)
```

| Status | Meaning |
|--------|---------|
| `active` | Wizard is in progress, waiting for next answer |
| `completed` | All steps answered and plan executed |
| `abandoned` | Session expired or user explicitly cancelled |

---

## Example Conversation

```
User: "Help me set up a new microservice"

AI:   "I'll guide you through setting up a microservice environment.
       Step 1 of 6: Which cloud provider?
       • AWS
       • Azure  
       • GCP"

User: "AWS"

AI:   "Step 2 of 6: Which region?
       (Default: us-east-1)"

User: "us-west-2"

AI:   "Step 3 of 6: What compute size?
       • Small (t3.small, good for dev/test)
       • Medium (t3.medium, suitable for staging)
       • Large (m5.large, production-ready)"

User: "Medium"

AI:   "Step 4 of 6: Do you need a database?
       • Yes (PostgreSQL on RDS)
       • No"

User: "Yes"

AI:   "Step 5 of 6: Do you need a load balancer?"

User: "Yes"

AI:   "Step 6 of 6: Here's your plan summary:
       
       🏗️ Microservice Environment Plan
       ├── Provider: AWS (us-west-2)
       ├── Compute: 1× t3.medium EC2 instance
       ├── Database: RDS PostgreSQL (db.t3.medium)
       ├── Load Balancer: Application LB
       └── Estimated cost: ~$120/month
       
       Ready to provision? (Yes / Modify / Cancel)"

User: "Yes"

AI:   "Provisioning started. I'll update you when it's complete."
```

---

## Context-Aware Intelligence

The wizard engine automatically:
- **Skips steps** when the answer can be inferred (e.g., if you're viewing an AWS resource, it skips the provider question)
- **Provides defaults** based on your usage history and current context
- **Validates inputs** against available options (e.g., only shows regions where you have credentials)

---

## Permissions

| Action | Admin | Developer | User | Readonly |
|--------|-------|-----------|------|----------|
| List wizards | ✓ | ✓ | ✓ | ✓ |
| Start wizard | ✓ | ✓ | ✓* | — |
| Execute wizard plan | ✓ | ✓ | — | — |

*Users can start wizards but execution of the resulting plan requires Developer or Admin role for infrastructure changes.

---

## Limitations

- Sessions expire after 24 hours of inactivity
- Maximum of 5 active wizard sessions per user
- Custom wizard templates are admin-configurable but not yet editable via chat
- Going back to a previous step resets all subsequent answers


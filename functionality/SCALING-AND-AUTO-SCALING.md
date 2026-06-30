# Scaling & Auto-Scaling

## Overview

The Scaling & Auto-Scaling service provides programmatic management of cloud scaling policies, scheduled scaling actions, and AI-driven optimization recommendations. It integrates with AWS Auto Scaling Groups (ASGs) and is accessible through CMP's conversational cloud operations interface.

**Access:** Developers and Admins (via chat commands or internal service calls)  
**Cloud Support:** AWS (ASGs) — Azure VMSS and GCP Instance Groups planned

---

## Key Concepts

### Scaling Policies

A scaling policy defines the capacity boundaries and target-tracking configuration for a scalable resource:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `min_capacity` | Minimum instance count | 1 |
| `max_capacity` | Maximum instance count | 10 |
| `desired_capacity` | Target instance count | 2 |
| `target_metric` | Metric to track for auto-scaling decisions | CPU Utilization |
| `target_value` | Target metric threshold (%) | 70.0 |
| `scale_in_cooldown` | Seconds to wait after scale-in before next action | 300 |
| `scale_out_cooldown` | Seconds to wait after scale-out before next action | 60 |

### Scaling Schedules

Scheduled scaling actions allow predictive capacity management — scaling resources up before anticipated demand and down during off-hours.

| Field | Description | Example |
|-------|-------------|---------|
| `cron_expression` | Standard cron schedule | `0 8 * * MON-FRI` |
| `desired_capacity` | Target capacity at trigger time | 5 |
| `min_capacity` | Optional minimum override | 3 |
| `max_capacity` | Optional maximum override | 10 |
| `timezone` | Timezone for cron evaluation | `UTC` |

### Scaling Recommendations

The system analyzes CloudWatch metrics (CPU utilization over the last 7 days) and generates right-sizing recommendations:

- **Scale Down** — Average CPU < 20% and max CPU < 40% indicates over-provisioning
- **Scale Up** — Average CPU > 75% indicates saturation risk
- **No Action** — Healthy utilization patterns, no changes needed

Each recommendation includes a confidence score (0.0–1.0) and estimated savings where applicable.

---

## Supported Metrics

| Metric | Key | Description |
|--------|-----|-------------|
| CPU Utilization | `cpu_utilization` | Average processor usage |
| Memory Utilization | `memory_utilization` | RAM consumption percentage |
| Request Count | `request_count` | Incoming request volume |
| Network In | `network_in` | Inbound network traffic |
| Network Out | `network_out` | Outbound network traffic |
| Custom | `custom` | User-defined CloudWatch metric |

---

## Service Operations

### Get Scaling Configuration

Retrieves the current auto-scaling setup for a resource, including:
- Min/max/desired capacity
- Target tracking policies (metric type, target value, cooldowns)
- Current instance count and health status (up to 10 instances)

### Update Scaling Policy

Modifies an ASG's capacity boundaries. Accepts partial updates — only specified fields are changed:
- `min_capacity` — New minimum instance count
- `max_capacity` — New maximum instance count
- `desired_capacity` — New desired instance count

### Create Scaling Schedule

Creates a scheduled scaling action using a cron expression. Useful for:
- Scaling up before business hours
- Scaling down overnight or on weekends
- Preparing for known traffic spikes (marketing campaigns, releases)

### Get Scaling Recommendations

Analyzes up to 5 resources per call using 7-day CloudWatch CPU metrics. Returns:
- Current utilization summary (average and peak CPU)
- Recommended action (scale up, scale down, or no change)
- Confidence score and estimated monthly savings

---

## Usage via Conversational Interface

The scaling service is accessible through CMP's AI chat. It requires the `chat_scaling` feature toggle to be enabled (Settings → Feature Toggles).

Example commands:

| User Intent | Example Phrase |
|-------------|----------------|
| View config | "Show me the scaling config for web-tier-asg" |
| Update policy | "Set auto-scaling to target 70% CPU, max 8 instances" |
| Scale immediately | "Scale my-asg to 5 instances" |
| Create schedule | "Scale up to 6 instances every weekday at 8am" |
| Get recommendations | "What are the scaling recommendations for my ASGs?" |

The chat system routes scaling intents to the dedicated `_handle_scaling` handler in the chat endpoint, which delegates to the scaling service layer. Operations that modify scaling policies use a two-step confirmation pattern — the AI shows proposed changes and waits for user approval before applying.

### Chat Tools (Feature Toggle: `chat_scaling`)

| Tool | Description | Confirmation Required |
|------|-------------|----------------------|
| `get_scaling_config` | Retrieve current scaling setup | No |
| `update_scaling_policy` | Modify min/max/desired/target | Yes |
| `create_scaling_schedule` | Create cron-based scheduled scaling | No |
| `get_scaling_recommendations` | AI-powered right-sizing advice | No |

---

## Credential Requirements

The service requires valid AWS credentials with the following IAM permissions:

| Permission | Purpose |
|------------|---------|
| `autoscaling:DescribeAutoScalingGroups` | Read ASG configuration |
| `autoscaling:DescribePolicies` | Read scaling policies |
| `autoscaling:UpdateAutoScalingGroup` | Modify capacity settings |
| `autoscaling:PutScheduledUpdateGroupAction` | Create scheduled actions |
| `cloudwatch:GetMetricStatistics` | Fetch utilization metrics for recommendations |

Credentials are resolved from CMP's credential store using `credential_id` and decrypted at runtime.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│              Chat / Operations Layer             │
│         (api/v1/endpoints/chat.py)               │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│          Scaling Service Layer                   │
│   (services/scaling/scaling_service.py)          │
│                                                  │
│  • get_scaling_config()                          │
│  • update_scaling_policy()                       │
│  • create_scaling_schedule()                     │
│  • get_scaling_recommendations()                 │
│                                                  │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│            AWS Auto Scaling API                  │
│            AWS CloudWatch API                    │
└─────────────────────────────────────────────────┘
```

All boto3 calls are wrapped with `run_in_executor` for async compatibility.

---

## Permissions

| Action | Admin | Developer | User | Readonly |
|--------|-------|-----------|------|----------|
| View scaling config | ✓ | ✓ | — | — |
| Update scaling policy | ✓ | ✓ | — | — |
| Create scaling schedule | ✓ | ✓ | — | — |
| View recommendations | ✓ | ✓ | — | — |

---

## Data Models

### ScalingPolicyConfig

| Field | Type | Description |
|-------|------|-------------|
| `resource_id` | string | AWS ASG name or resource identifier |
| `provider` | string | Cloud provider (`aws`, `azure`, `gcp`) |
| `resource_type` | string | Resource type (`asg`, `vmss`, `instance_group`) |
| `min_capacity` | int | Minimum instances |
| `max_capacity` | int | Maximum instances |
| `desired_capacity` | int | Target instance count |
| `target_metric` | enum | Metric for auto-scaling decisions |
| `target_value` | float | Target metric percentage |
| `scale_in_cooldown` | int | Seconds before next scale-in |
| `scale_out_cooldown` | int | Seconds before next scale-out |
| `region` | string | AWS region |
| `credential_id` | string | Reference to stored credential |

### ScalingScheduleCreate

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `resource_id` | string | ✓ | ASG name |
| `cron_expression` | string | ✓ | Cron schedule |
| `desired_capacity` | int | ✓ | Target capacity |
| `min_capacity` | int | — | Optional min override |
| `max_capacity` | int | — | Optional max override |
| `timezone` | string | — | Default: `UTC` |
| `description` | string | — | Human-readable description |
| `credential_id` | string | ✓ | Credential for AWS access |
| `region` | string | — | Default: `us-east-1` |

### ScalingRecommendation

| Field | Type | Description |
|-------|------|-------------|
| `resource_id` | string | Resource identifier |
| `resource_name` | string | Human-friendly name |
| `current_config` | object | Current metrics (avg/max CPU) |
| `recommended_config` | object | Suggested action and parameters |
| `reason` | string | Explanation of the recommendation |
| `estimated_savings` | float | Projected monthly cost reduction ($) |
| `confidence` | float | Confidence score (0.0–1.0) |

---

## Limitations

- Currently supports **AWS Auto Scaling Groups** only
- Recommendations based on CPU utilization; memory and custom metrics planned
- Analyzes a maximum of 5 resources per recommendation call
- Health status limited to first 10 instances per ASG
- No built-in approval workflow for scaling changes (executes immediately)

# CMP Documentation

All documentation for the Cloud Management Platform, organized by audience and topic.

---

## Structure

```
cmp_doc/
├── customer/                        # Customer-facing documentation
│   ├── LICENSE.md
│   ├── installation-and-setup/      # Installation, configuration, promoter setup
│   └── operations/                  # Upgrades, rollbacks, troubleshooting, air-gapped ops
│
├── developer/                       # Internal developer documentation
│   ├── architecture-and-deployment/ # Deployment patterns, packaging, zero-downtime, registries
│   ├── guides-and-references/       # Feature dev guides, hooks, webhooks, scheduled jobs
│   ├── planning-and-roadmap/        # UX roadmap, Spacelift comparison, TTL planning
│   └── terraform-and-infrastructure/# Terraform modules, conversational infra, pipelines
│
├── functionality/                   # Feature & capability documentation
│   ├── CMP_COMPLETE_FUNCTIONALITY_GUIDE.md   # Full feature overview
│   ├── CMP_DETAILED_USER_GUIDE.md            # Detailed user guide
│   ├── CMP_MARKETING_FUNCTIONALITY.md        # Marketing feature list
│   ├── agent-and-monitoring/        # CMP Agent architecture, debugging, monitoring, promoter
│   ├── ai-and-chat/                 # AI chat ops, knowledge sources, guided conversations
│   ├── catalog-and-workflows/       # Service catalog, DAG editor, webhooks
│   ├── infrastructure-and-terraform/# Terraform chat operations
│   ├── provisioning-and-cloud/      # Cloud VM provisioning, scaling
│   └── reporting-and-insights/      # Dashboards, report scheduling & export
│
└── vendor/                          # Vendor/internal operations
    ├── CUSTOMER-ONBOARDING.md
    ├── LICENSE-FEATURES.md
    ├── LICENSE-MANAGEMENT.md
    ├── PROMOTER-OPERATIONS.md
    ├── RELEASE-OPERATIONS.md
    ├── SUPPORT-GUIDE.md
    └── shipping_product_steps.md
```

## Audience Guide

| Folder | Who It's For | Content |
|--------|-------------|---------|
| `customer/` | End customers deploying CMP | Installation, config, upgrades, troubleshooting |
| `developer/` | CMP developers & contributors | Architecture, dev guides, planning docs |
| `functionality/` | Product managers, users, stakeholders | Feature descriptions, capabilities, user guides |
| `vendor/` | CMP vendor operations team | Licensing, releases, customer onboarding, support |

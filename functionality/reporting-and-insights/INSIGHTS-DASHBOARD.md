# Insights Dashboard

## Overview

The Insights Dashboard provides customizable, data-driven analytics views across cost, operations, and compliance domains. Users can build personalized dashboards with drag-and-drop widgets, choose from pre-built templates, and share dashboards with their team. The module supports multiple chart types, data sources, auto-refresh, anomaly detection, and export capabilities.

**Route:** `/t/{tenant}/insights`  
**Access:** All authenticated users (widgets filter data by role and tenant)

---

## Key Concepts

### Dashboards

A dashboard is a named collection of widgets arranged in a responsive 12-column grid. Each user can have multiple dashboards, mark one as default, and optionally share dashboards with all tenant users.

### Widgets

Each widget visualizes a single metric or data series. A widget configuration includes:

- **Widget Type** — The chart visualization (bar, line, pie, etc.)
- **Data Source** — Where the data comes from (cost analytics, inventory, logs, etc.)
- **Dimension / Measure** — What fields to group by and aggregate
- **Period** — Time window for the data (e.g., 30 days)
- **Layout** — Grid position and size (react-grid-layout compatible)

### Templates

Pre-built dashboard templates let users get started quickly without manual widget configuration. Templates are categorized by domain (Cost, Operations, Compliance).

---

## Supported Widget Types

| Widget Type | Key | Description |
|-------------|-----|-------------|
| Bar Chart | `bar` | Vertical bar chart for categorical comparisons |
| Line Chart | `line` | Time-series trend visualization |
| Area Chart | `area` | Line chart with filled area below |
| Stacked Area | `stacked_area` | Multiple series stacked for cumulative view |
| Pie Chart | `pie` | Proportional breakdown (circular) |
| Donut Chart | `donut` | Pie chart with center cutout |
| Treemap | `treemap` | Hierarchical area-based visualization |
| Funnel | `funnel` | Stage-based conversion/pipeline view |
| Radar | `radar` | Multi-axis comparison chart |
| Heatmap | `heatmap` | Matrix with color intensity for values |
| Gauge | `gauge` | Single-value meter (e.g., budget utilization) |
| KPI Card | `kpi_card` | Single metric with trend indicator |
| Table | `table` | Tabular data with sorting |
| Composed | `composed` | Bar + line combination chart |

---

## Data Sources

| Source | Key | Description |
|--------|-----|-------------|
| Cost Analytics | `cost_analytics` | Aggregated cost data by provider, user, catalog, etc. |
| Cost Trend | `cost_trend` | Time-series cost data for trend analysis |
| Inventory | `inventory` | Provisioned resource inventory across clouds |
| Login Logs | `login_logs` | User authentication activity |
| Audit Logs | `audit_logs` | Platform audit trail events |
| Executions | `executions` | Workflow and provisioning execution records |
| Approvals | `approvals` | Approval workflow status and history |
| Budgets | `budgets` | Budget allocation and utilization |
| Policies | `policies` | Policy engine rules and violations |
| Compliance | `compliance` | Compliance posture and scores |

---

## Dashboard Categories

| Category | Key | Typical Use |
|----------|-----|-------------|
| Cost | `cost` | Spend tracking, budgets, anomalies, forecasting |
| Operations | `operations` | Executions, provisioning funnel, resource lifecycle |
| Compliance | `compliance` | Policy violations, compliance scores |
| Custom | `custom` | User-defined dashboards (default category) |

---

## Auto-Refresh

Dashboards support configurable auto-refresh intervals:

| Interval | Key |
|----------|-----|
| Off | `off` |
| 30 seconds | `30s` |
| 1 minute | `1m` |
| 5 minutes | `5m` |
| 15 minutes | `15m` |

---

## Analytics Capabilities

### Cost Trend Analysis

View cost trends over time broken down by cloud provider (AWS, Azure, GCP). Includes:
- Historical cost data points by date
- Percentage change from the previous period
- Cost forecast projections

### Cost Anomaly Detection

Automatically identifies unusual spending patterns:
- Compares actual cost against expected cost (statistical baseline)
- Reports deviation percentage
- Categorizes severity (low, medium, high)
- Optionally filters by resource type or cloud provider
- Configurable threshold (default: 30% deviation)

### Top Cost Consumers

Ranks the highest-spending entities by a chosen dimension:
- By user, group, catalog, resource, or cloud provider
- Shows each consumer's percentage of total spend
- Indicates trend direction (up, down, stable)

### Budget vs. Actual

Compares allocated budgets against actual spend:
- Per-budget utilization percentage
- Remaining budget calculation
- Overall portfolio utilization

### Provisioning Funnel

Tracks requests through the provisioning pipeline:
- Stages: Requested → Approved → Provisioning → Active
- Count and percentage at each stage
- Identifies bottlenecks in the process

### Resource Lifecycle

Shows the distribution of resources by lifecycle status:
- Active, stopped, terminated, pending, failed
- Count and percentage for each status

---

## API Endpoints

### Dashboard CRUD

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/insights/dashboards` | List saved dashboards for current user |
| `POST` | `/api/v1/insights/dashboards` | Create a new dashboard |
| `GET` | `/api/v1/insights/dashboards/{id}` | Get a specific dashboard |
| `PUT` | `/api/v1/insights/dashboards/{id}` | Update a dashboard |
| `DELETE` | `/api/v1/insights/dashboards/{id}` | Delete a dashboard |

### Templates

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/insights/templates` | List available dashboard templates |
| `POST` | `/api/v1/insights/templates/{id}/instantiate` | Create a dashboard from a template |

### Analytics Data

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/insights/cost-trend` | Cost trend time series |
| `GET` | `/api/v1/insights/cost-anomalies` | Detected cost anomalies |
| `GET` | `/api/v1/insights/top-consumers` | Top N cost consumers |
| `GET` | `/api/v1/insights/budget-vs-actual` | Budget utilization comparison |
| `GET` | `/api/v1/insights/provisioning-funnel` | Provisioning pipeline funnel |
| `GET` | `/api/v1/insights/resource-lifecycle` | Resource lifecycle distribution |

### Export

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/insights/export` | Export dashboard or widgets as PNG, PDF, CSV, or XLSX |

**Export Formats:** `png`, `pdf`, `csv`, `xlsx`

---

## Widget Configuration Reference

Each widget is configured with:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `widget_id` | string | — | Unique identifier |
| `title` | string | — | Display title |
| `widget_type` | enum | — | Chart type (see table above) |
| `data_source` | enum | — | Data source (see table above) |
| `dimension` | string | null | Field to group by |
| `measure` | string | null | Field to aggregate |
| `aggregation` | string | `count` | Aggregation function |
| `filters` | object | `{}` | Key-value filter criteria |
| `period` | string | `30d` | Time window |
| `group_by` | string | null | Secondary grouping |
| `limit` | int | `10` | Top N items to display |
| `colors` | array | `[]` | Custom color palette |
| `layout` | object | `{x:0, y:0, w:6, h:4}` | Grid position/size |
| `show_legend` | bool | `true` | Display chart legend |
| `show_labels` | bool | `true` | Display data labels |
| `stacked` | bool | `false` | Stack series (bar/area) |
| `annotations` | array | `[]` | Chart annotations |

### Layout Grid

The dashboard uses a 12-column responsive grid:
- `x` — Column position (0–11)
- `y` — Row position (0+)
- `w` — Width in columns (min: 3)
- `h` — Height in grid units (min: 2)

---

## User Workflows

### Creating a Custom Dashboard

1. Navigate to **Insights** in the side navigation
2. Click **New Dashboard**
3. Enter a name, optional description, and select a category
4. Add widgets using the **Add Widget** button
5. Configure each widget's data source, chart type, and filters
6. Drag and resize widgets to arrange the layout
7. Click **Save**

### Using a Template

1. Navigate to **Insights → Templates**
2. Browse available templates by category
3. Click **Use Template** on the desired template
4. The template creates a new dashboard pre-configured with widgets
5. Customize widget settings or layout as needed

### Sharing a Dashboard

1. Open a saved dashboard
2. Click the **Share** toggle in dashboard settings
3. Shared dashboards are visible to all users in the same tenant
4. Only the dashboard creator can edit shared dashboards

### Exporting Data

1. Open a dashboard or select specific widgets
2. Click **Export**
3. Choose format: PNG (image), PDF (document), CSV (raw data), or XLSX (spreadsheet)
4. The export includes data from all selected widgets

---

## Permissions

| Action | Admin | Developer | User | Readonly |
|--------|-------|-----------|------|----------|
| View own dashboards | ✓ | ✓ | ✓ | ✓ |
| View shared dashboards | ✓ | ✓ | ✓ | ✓ |
| Create/edit dashboards | ✓ | ✓ | ✓ | — |
| Delete dashboards | ✓ | ✓ | Own only | — |
| Share dashboards | ✓ | ✓ | — | — |
| Export dashboards | ✓ | ✓ | ✓ | ✓ |
| Access all data sources | ✓ | ✓ | Limited | Limited |

---

## DynamoDB Storage

Dashboards are stored using the standard CMP single-table design:

- **PK:** `TENANT#{tenant_id}`
- **SK:** `INSIGHTS_DASHBOARD#{dashboard_id}`
- Widgets are stored inline as a list within the dashboard item
- The `created_by` field links back to the user who created the dashboard

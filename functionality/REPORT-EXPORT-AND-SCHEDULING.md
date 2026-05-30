# Report Export (CSV/PDF) & Scheduled Report Delivery

## Overview

CMP provides comprehensive report export capabilities and automated scheduled report delivery. Finance teams and administrators can export any report as CSV or PDF on-demand, and configure recurring email delivery of reports on a cron schedule.

## Feature 23: Report Export (CSV / PDF)

### Supported Report Types

| Report Type | Description |
|-------------|-------------|
| `cost_summary` | Aggregated cost data by user, group, catalog, or cloud provider |
| `login_logs` | User authentication events with session details |
| `audit_logs` | Complete audit trail of all platform actions |
| `inventory` | Provisioned resource inventory across cloud providers |
| `approvals` | Approval workflow records |

### Export Formats

- **CSV** — Comma-separated values, ideal for spreadsheet import (Excel, Google Sheets)
- **PDF** — Portable document format, ideal for sharing and archiving

### How to Export (UI)

1. Navigate to **Reports & Analytics** page
2. Select the desired tab (Login Logs, Audit Logs, Inventory, or Approvals)
3. Apply any date filters as needed
4. Click the **CSV** button for spreadsheet export or **PDF** button for document export
5. The file downloads automatically

### API Endpoints

```
GET /api/v1/reports/export/{report_type}?format={csv|pdf}
```

**Query Parameters:**
- `format` — `csv` or `pdf` (default: `csv`)
- `group_by` — For cost_summary: `user`, `group`, `catalog`, `cloud_provider`
- `period` — For cost_summary: `7d`, `30d`, `90d`, `all`
- `status` — Filter by status
- `start_date` / `end_date` — ISO date range filter
- `group_id`, `resource_type`, `cloud_provider` — Inventory filters

**Response:** Binary file download with appropriate Content-Type and Content-Disposition headers.

**Auth:** Requires admin role.

---

## Feature 24: Scheduled Report Delivery

### Overview

Administrators can configure recurring report delivery via email. Reports are generated automatically based on a cron schedule and sent as email attachments to specified recipients.

### Configuration

Each scheduled report has:
- **Name** — Human-readable identifier
- **Report Type** — One of: cost_summary, login_logs, audit_logs, inventory, approvals
- **Format** — CSV or PDF
- **Schedule** — 5-field cron expression (minute, hour, day-of-month, month, day-of-week)
- **Recipients** — List of email addresses
- **Filters** — Optional filters applied to the report data
- **Enabled** — Toggle to pause/resume delivery

### Cron Schedule Presets

| Preset | Cron Expression | Description |
|--------|----------------|-------------|
| Daily | `0 8 * * *` | Every day at 8:00 AM UTC |
| Weekly | `0 8 * * 1` | Every Monday at 8:00 AM UTC |
| Monthly | `0 8 1 * *` | 1st of each month at 8:00 AM UTC |
| Quarterly | `0 8 1 1,4,7,10 *` | 1st of Jan/Apr/Jul/Oct at 8:00 AM UTC |

### How to Configure (UI)

1. Navigate to **Reports & Analytics** → **Scheduled Reports** tab
2. Click **+ New Schedule**
3. Fill in the form:
   - Name and optional description
   - Select report type and format
   - Choose a schedule preset
   - Enter recipient email addresses (comma-separated)
4. Click **Create Schedule**
5. Use **Pause/Resume** to temporarily disable delivery
6. Use **Delete** to remove a schedule

### API Endpoints

```
POST   /api/v1/reports/scheduled          — Create a new scheduled report
GET    /api/v1/reports/scheduled          — List all scheduled reports
GET    /api/v1/reports/scheduled/{id}     — Get a specific scheduled report
PUT    /api/v1/reports/scheduled/{id}     — Update a scheduled report
DELETE /api/v1/reports/scheduled/{id}     — Delete a scheduled report
```

**Create Request Body:**
```json
{
  "name": "Monthly Cost Report",
  "description": "Cost summary for finance team",
  "report_type": "cost_summary",
  "format": "csv",
  "cron_expression": "0 8 1 * *",
  "recipients": ["finance@company.com", "cfo@company.com"],
  "filters": {"group_by": "user", "period": "30d"},
  "enabled": true
}
```

**Auth:** Requires admin role for all endpoints.

### Prerequisites

- **SMTP Configuration** — Email delivery requires SMTP to be configured either via:
  - Admin UI: Settings → Notification Channels → SMTP
  - Environment variables: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM`

### Architecture

- Reports are evaluated every 60 seconds by the background scheduler
- The cron expression is parsed using a built-in 5-field parser (no external dependencies)
- Report data is fetched fresh at delivery time (not cached)
- Email is sent with the report as an attachment (CSV or PDF)
- Run history is tracked per scheduled report (success/failure, timestamps, recipient count)
- Failed deliveries are logged and the report status is updated to "failed"

### Events

The following events are emitted:
- `report.exported` — When a report is exported on-demand
- `report.scheduled` — When a new scheduled report is created
- `report.delivered` — When a scheduled report is successfully delivered
- `report.delivery.failed` — When a scheduled report delivery fails

### DynamoDB Schema

Table: `scheduled_reports`
- PK: `TENANT#{tenant_id}`
- SK: `SCHEDULED_REPORT#{report_id}`

Stores: name, description, report_type, format, cron_expression, recipients, filters, enabled, status, run_count, last_run_at, run_history, created_by, created_at, updated_at.

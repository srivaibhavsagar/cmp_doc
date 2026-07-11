# CMP UX Enhancement Opportunities

> Living document tracking UX gaps, improvement opportunities, and feature ideas.
> Last updated: May 2026

---

## ✅ Previously Identified — Now Implemented

The following items from the original audit have been fully implemented and are no longer gaps:

| # | Feature | Implementation |
|---|---------|---------------|
| 1 | Dashboard Real-Time Stat Cards | `/api/v1/dashboard/stats/realtime` feeds live stat cards (cost, credentials, workflows, approvals, executions, budget %) |
| 2 | Global Search / Command Palette | `CommandPalette.tsx` with ⌘K shortcut, navigation, backend search via `/api/v1/search`, role/feature gating |
| 3 | Execution Detail / Log Viewer | Full `ExecutionDetail` with step-by-step logs, live polling, retry/resubmit/cancel, Terraform plan diffs |
| 5 | Notification Preferences | Per-user per-category per-channel preferences with digest modes, platform defaults, admin config |
| 6 | Approval SLA & Escalation | `approval_sla.py` — 5-min scheduler, 50% reminder + escalation, 100% timeout, event emission |
| 7 | Unread Notification Badge | Red badge on Bell icon with count, animated ring, "99+" cap, real-time updates |
| 11 | Contextual Inline AI Suggestions | `InlineAISuggestions.tsx` — AI action buttons on failed executions, resources, catalogs, policies, budgets |
| 12 | AI-Powered Catalog Recommendations | `CatalogRecommendations.tsx` — intent matching, usage patterns, discovery suggestions |
| 13 | Persistent AI Conversation History | `ConversationHistory.tsx` + DynamoDB storage — list, resume, rename, delete past conversations |
| 14 | Visual Execution Step Progress | `ExecutionStepTracker.tsx` — horizontal/vertical layouts, color-coded status, connector lines, duration |
| 15 | Scheduled Jobs Calendar View | `ScheduledJobsCalendar.tsx` — month/week views, cron parsing, color-coded status, navigation |
| 16 | Workflow Clone / Start from Template | Clone button on workflows calling `POST /api/v1/workflows/{id}/clone` |
| 17 | Compliance Dashboard | `ComplianceDashboard.tsx` — KPI cards, policy breakdown, quotas-at-risk, compliance score |
| 18 | Quota Usage Visual Progress Bars | `QuotaBar` component with color-coded bars, percentage, "FULL"/"HIGH" badges |
| 19 | Audit Log UI Page | `AuditLogs.tsx` — summary cards, search, filters, expandable details, pagination |
| 21 | Breadcrumb Navigation | `Breadcrumb.tsx` — tenant-aware, used in ResourceDetail, Executions, Terraform pages |
| 23 | Report Export (CSV / PDF) | Client-side CSV + backend `/api/v1/reports/export/{type}?format=csv|pdf` |
| 24 | Scheduled Report Delivery | `report_scheduler.py` + CRUD endpoints + cron evaluation + email delivery |
| 25 | API Token Self-Service | Profile page "API Tokens" tab with personal token creation/management |
| 26 | Terraform Plan Diff Viewer | Color-coded diff (green/yellow/red) with resource addresses in execution detail |

---

## ⚠️ Partially Implemented — Needs Improvement

### 1. Live Resource Status Polling
**Current state:** `ResourceDetail.tsx` polls every 60 seconds with a toggle.
**Gap:** Interval-based polling is sluggish for state transitions (pending → running). Users who just provisioned a VM wait up to 60s to see the change.
**Improvement:** Reduce polling interval to 5s when resource is in a transitional state (pending, stopping, starting), or implement Server-Sent Events for real-time push updates.

### 2. Cost Anomaly Detection ✅ Implemented
**Current state:** Budget sync runs statistical anomaly detection comparing current spend against expected linear spend. Detects three anomaly types: spend spike (≥30% above expected), burn rate acceleration (≥40% ahead of schedule), and projected overrun (≥120% of budget at period end). A 6-hour cooldown window per budget+type prevents duplicate notifications from the 10-minute sync loop.
**Gap:** ~~No statistical anomaly detection.~~ — Resolved.
**Remaining:** Consider adding per-resource anomaly detection (individual resource cost spikes beyond budget-level analysis).

### 3. Per-Resource Cost Attribution
**Current state:** `LiveCostWidget` shows per-resource hourly projections. `ResourceDetail` shows cost estimates for actions.
**Gap:** No historical cost breakdown per individual resource (e.g., "this EC2 instance cost $47 this month"). No cost tab on resource detail.
**Improvement:** Add a "Cost" tab to ResourceDetail showing daily/monthly cost history for that specific resource, sourced from cost analytics data.

### 4. Keyboard Shortcuts Reference
**Current state:** Command Palette footer shows ↑↓/Enter/Esc hints. AI panel has ⌘J shortcut.
**Gap:** No dedicated shortcuts reference modal listing all available shortcuts across the platform.
**Improvement:** Add a `?` key shortcut that opens a modal listing all keyboard shortcuts (like GitHub's shortcuts overlay).

---

## 🆕 New Enhancement Opportunities

### User Experience


### 5. Onboarding / First-Run Setup Wizard
**Impact:** High | **Effort:** Medium
New tenants land on an empty dashboard with no guidance. A setup wizard would dramatically reduce time-to-value.
**Proposed flow:**
1. Welcome screen with platform overview
2. Connect your first cloud account (AWS/Azure/GCP)
3. Create or import your first catalog item
4. Invite a team member
5. Submit your first provisioning request

Show a dismissible progress checklist on the dashboard until all steps are complete. Track completion per user in DynamoDB.

### 6. Budget Burn Rate Forecasting
**Impact:** High | **Effort:** Low
Budgets show current spend vs. limit but don't project end-of-month spend.
**Proposed implementation:**
- Calculate daily burn rate from the last 7 days of spend data
- Project end-of-period spend: `current_spend + (daily_rate × remaining_days)`
- Show a forecast line on the budget card: "At this rate, you'll hit 100% by the 22nd"
- Add a "Projected Overspend" warning badge when forecast exceeds budget
- Backend: Add a `/api/v1/budgets/{id}/forecast` endpoint

### 7. Bulk Resource Actions
**Impact:** High | **Effort:** Medium
Users can only act on one resource at a time. For fleet operations (stop all dev VMs on Friday, tag 20 resources), they must repeat the action individually.
**Proposed implementation:**
- Add multi-select checkboxes to the resource list view
- Show a floating action bar when resources are selected: "3 selected — Stop | Tag | Delete | Lease"
- Execute actions in parallel with a progress indicator
- Support bulk tag assignment and bulk lease application

### 8. Resource Tagging & Tag-Based Views
**Impact:** Medium | **Effort:** Medium
Resources can be tagged but there's no way to filter/group by tags across the platform.
**Proposed implementation:**
- Add a "Tags" filter dropdown to the Resources page (multi-select)
- Add a "Group by Tag" toggle that clusters resources by a selected tag key
- Show tag-based cost breakdown in Cost Analytics
- Allow bulk tag editing from the resource list

### 9. Favorites / Pinned Items
**Impact:** Medium | **Effort:** Low
Power users repeatedly navigate to the same catalogs, resources, or dashboards. There's no way to pin frequently-used items.
**Proposed implementation:**
- Add a star/pin icon on catalog items, resources, and dashboard widgets
- Show a "Favorites" section at the top of the Command Palette
- Store favorites per user in DynamoDB (lightweight — just IDs and types)
- Show pinned catalogs at the top of the catalog page

### 10. Dark Mode
**Impact:** Medium | **Effort:** Medium
The platform is light-mode only. Many developers prefer dark mode, especially for long sessions.
**Proposed implementation:**
- Add a theme toggle in the user profile dropdown (Light / Dark / System)
- Use Tailwind's `dark:` variant classes (already supported by the framework)
- Store preference in localStorage + user settings
- Ensure all custom components respect the dark mode classes

---

### Developer Experience

### 11. Code Task Live Preview / REPL
**Impact:** High | **Effort:** Medium
Developers write Code Tasks but can only test them by running a full execution. A live preview/REPL would speed up development.
**Proposed implementation:**
- Add a "Test Run" button in the task editor that executes the code in a sandboxed environment
- Show stdout/stderr output inline below the editor
- Support passing mock `form_data` and `context` variables for testing
- Time-limit test runs to 30 seconds

### 12. Workflow Visual DAG Editor
**Impact:** High | **Effort:** High
Workflows are configured via JSON/form but there's no visual representation during creation.
**Proposed implementation:**
- Add a visual DAG editor (drag-and-drop nodes for tasks, connect with edges)
- Show parallel paths, conditional branches, and retry indicators visually
- Allow editing step properties by clicking nodes
- Generate the workflow JSON from the visual representation
- Use a library like ReactFlow or dagre for layout

### 13. Execution Comparison View
**Impact:** Medium | **Effort:** Low
When an execution fails after a previous success, users want to compare what changed.
**Proposed implementation:**
- Add a "Compare with previous" button on execution detail
- Show a side-by-side diff of form_data, step outputs, and errors
- Highlight what changed between the two runs
- Useful for debugging intermittent failures

### 14. API Playground / Swagger Integration
**Impact:** Medium | **Effort:** Low
Developers can run APIs through the AI assistant, but there's no dedicated API playground.
**Proposed implementation:**
- Add an "API Explorer" page (admin/developer only) that embeds Swagger UI or a custom API tester
- Pre-fill authentication headers from the current session
- Show request/response history for the current session
- Link from the AI assistant's API execution results to the playground

### 15. Git-Linked Task Versioning
**Impact:** Medium | **Effort:** Medium
Code Tasks are stored in DynamoDB with no version history. Developers can't see what changed or roll back.
**Proposed implementation:**
- Add version history to tasks (store previous versions as a list in DynamoDB)
- Show a "History" tab on the task editor with diff view between versions
- Allow rollback to any previous version
- Optionally link tasks to a Git repository for source-of-truth management

---

### Admin Experience

### 16. Tenant Health Dashboard
**Impact:** High | **Effort:** Medium
Admins managing multiple tenants have no cross-tenant overview.
**Proposed implementation:**
- Add a "Tenant Health" page showing all tenants with key metrics (users, resources, spend, executions)
- Color-coded health indicators (green/yellow/red) based on failed executions, budget overruns, sync errors
- Drill-down to individual tenant dashboards
- Show tenant growth trends over time

### 17. User Activity Timeline
**Impact:** Medium | **Effort:** Low
Admins can see audit logs but there's no per-user activity timeline.
**Proposed implementation:**
- Add an "Activity" tab to the user detail page in Admin
- Show a chronological timeline of the user's actions (logins, provisioning requests, resource actions, approvals)
- Filter by action type and date range
- Useful for security investigations and usage analysis

### 18. Bulk User Management
**Impact:** Medium | **Effort:** Medium
Adding/modifying multiple users requires individual operations.
**Proposed implementation:**
- Add CSV import for bulk user creation (username, email, roles, groups)
- Add multi-select on the users list with bulk actions (change role, add to group, disable)
- Add CSV export of current user list
- Show import progress and error summary

### 19. Policy Simulation / What-If Analysis
**Impact:** High | **Effort:** Medium
Admins create policies but can't test them without submitting real requests.
**Proposed implementation:**
- Add a "Simulate" button on the policy editor
- Allow admins to input mock form_data and see which policies would trigger
- Show block/warn/pass results with explanations
- Useful for validating policy changes before publishing

### 20. Credential Health Monitoring
**Impact:** Medium | **Effort:** Low
Credentials can fail silently (expired tokens, revoked permissions).
**Proposed implementation:**
- Add a "Health" column to the credentials list showing last sync status and time
- Show a warning badge on credentials that haven't synced in 24+ hours
- Add a "Test Connection" button that validates the credential without full sync
- Surface credential health issues on the dashboard stat cards

---

### Performance & Polish

### 21. Optimistic UI Updates
**Impact:** Medium | **Effort:** Medium
Actions like start/stop/delete show a loading state and wait for the API response before updating the UI.
**Proposed implementation:**
- Immediately update the UI to reflect the expected state (optimistic update)
- Revert if the API call fails
- Reduces perceived latency for common actions
- Apply to: resource start/stop, approval approve/reject, notification mark-as-read

### 22. Skeleton Loading States Consistency
**Impact:** Low | **Effort:** Low
Some pages use `SkeletonLoader`, others show a spinner, and some show nothing during load.
**Proposed implementation:**
- Audit all pages and standardize on `SkeletonLoader` for initial page loads
- Use inline spinners only for action buttons (retry, submit)
- Ensure every page has a loading state that matches the final layout shape

### 23. Mobile-Responsive Improvements
**Impact:** Medium | **Effort:** Medium
The platform is desktop-first. Some pages break on tablet/mobile.
**Proposed implementation:**
- Audit all pages at 768px and 375px breakpoints
- Fix overflow issues on data tables (horizontal scroll or card layout on mobile)
- Make the AI panel full-screen on mobile instead of side panel
- Ensure the Command Palette is usable on touch devices
- Collapse sidebar navigation into a hamburger menu on mobile

### 24. Empty State Illustrations
**Impact:** Low | **Effort:** Low
Empty states (no resources, no executions, no catalogs) show plain text.
**Proposed implementation:**
- Add lightweight SVG illustrations for empty states
- Include a clear call-to-action button ("Add your first credential", "Create a catalog item")
- Make empty states educational — briefly explain what the feature does

---

## Priority Recommendations

Top 5 by impact vs. effort:

| Priority | Feature | Rationale |
|----------|---------|-----------|
| 1 | **Onboarding / First-Run Wizard** | Biggest time-to-value improvement for new users. No backend exists yet. |
| 2 | **Budget Burn Rate Forecasting** | Low effort, high value — simple math on existing data. |
| 3 | **Bulk Resource Actions** | Fleet operations are painful without this. Multi-select pattern is well-understood. |
| 4 | **Policy Simulation** | Admins create policies blind. Simulation prevents production incidents. |
| 5 | **Code Task Live Preview** | Developers iterate slowly without a test mode. Reduces execution waste. |

---

## Implementation Notes

- All new pages: `frontend/src/pages/MyPage.tsx` + lazy-loaded in `App.tsx`
- New backend domains: `models/` → `crud/` → `endpoints/` → register in `api/v1/api.py`
- Gate new features behind a toggle key in `models/toggle_settings.py`
- Use `AdvancedDataTable` for list views, `SkeletonLoader` for loading states, `Pagination` for paging
- Emit events via `BackgroundTasks.add_task(emit_event, ...)` for lifecycle actions
- Always scope DynamoDB queries to `tenant_id` in PK
- Call `_sanitize_for_dynamodb()` before every write

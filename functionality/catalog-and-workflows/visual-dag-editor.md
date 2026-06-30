# Visual DAG Editor — User Guide

> Build workflows visually using a drag-and-drop canvas with directed acyclic graph (DAG) representation, AI-powered suggestions, and real-time validation.

---

## Overview

The Visual DAG Editor is a canvas-based graphical interface for creating and editing workflows in the Cloud Management Platform. Instead of configuring workflows through forms and linear step lists, you can visually compose workflow steps as nodes, connect them with edges to define execution order, and see parallel paths and conditional branches at a glance.

**Key capabilities:**
- Drag-and-drop node creation from a toolbar of action types
- Visual edge connections to define step dependencies
- Inline property editing via a side panel
- AI-powered suggestions for next steps and natural language workflow generation
- Automatic graph layout with dagre algorithm
- Undo/redo support (up to 50 actions)
- Real-time validation before saving
- Minimap for navigating large workflows

---

## Getting Started

### Prerequisites

The Visual DAG Editor is gated behind a feature toggle. An admin must enable it before it becomes available.

1. **Feature toggle:** The `visual_dag_editor` toggle must be enabled by an admin under Settings → Feature Toggles
2. **Role requirement:** The editor is available to users with **admin** or **developer** roles. Users with "user" or "readonly" roles see the graph in read-only mode (no editing capabilities)

### Accessing the Editor

1. Navigate to **Workflows** from the main navigation
2. When the feature toggle is enabled, you'll see a tab or toggle to switch between the **Form Builder** (existing) and the **DAG Editor**
3. Click the DAG Editor tab to open the visual canvas
4. Existing workflows created in the Form Builder load seamlessly in the DAG Editor and vice versa — the underlying data format is identical

---

## Editor Layout

The DAG Editor interface is organized into several panels:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Editor Header                               │
│  [Undo] [Redo] [Fit View] [Auto Layout]          [Validate] [Save] │
├────────┬────────────────────────────────────────────┬───────────────┤
│        │                                            │               │
│        │                                            │   Node Panel  │
│  Tool  │                                            │   (Properties │
│  bar   │              Canvas                        │    Editor)    │
│        │         (ReactFlow Area)                   │               │
│ ┌────┐ │                                            │  ┌─────────┐  │
│ │Task│ │    ┌──────┐       ┌──────┐                 │  │  Name   │  │
│ ├────┤ │    │Node A│──────▶│Node B│                 │  │  Action │  │
│ │HTTP│ │    └──────┘       └──────┘                 │  │  Task   │  │
│ ├────┤ │         │                                  │  │  Timeout│  │
│ │Wait│ │         ▼                                  │  │  Retry  │  │
│ ├────┤ │    ┌──────┐                                │  │  Failure│  │
│ │Cond│ │    │Node C│                                │  └─────────┘  │
│ ├────┤ │    └──────┘                                │               │
│ │ TF │ │                                            ├───────────────┤
│ └────┘ │                         ┌────────┐         │  Suggestion   │
│        │                         │Minimap │         │    Panel      │
│        │                         │ ┌──┐   │         │  (AI Assist)  │
│        │                         │ └──┘   │         │               │
│        │                         └────────┘         │  [NL Prompt]  │
├────────┴────────────────────────────────────────────┴───────────────┤
│                        Status Bar / Validation Errors                │
└─────────────────────────────────────────────────────────────────────┘
```

| Component | Purpose |
|-----------|---------|
| **Toolbar** | Draggable node templates for each action type |
| **Canvas** | Main interactive area where nodes and edges are rendered |
| **Node Panel** | Property editor for the selected node |
| **Suggestion Panel** | AI-generated recommendations and natural language input |
| **Minimap** | Scaled overview of the full graph (appears when >8 nodes) |
| **Editor Controls** | Zoom, fit view, undo, redo, auto-layout buttons |

---

## Canvas Navigation

### Panning

- **Mouse drag** on empty canvas area (not on a node or edge) to pan the view
- There is no fixed boundary — you can pan freely in any direction

### Zooming

- **Scroll wheel** to zoom in/out toward the cursor position
- Zoom range: **25%** (zoomed out) to **200%** (zoomed in)
- Default zoom: **100%**
- Step increment: 10% per scroll tick
- Zoom stops at the min/max boundaries automatically

### Fit View

- Click the **Fit View** button in the editor controls
- Adjusts zoom and pan to fit all nodes within the visible viewport with padding
- If the canvas is empty, resets to 100% zoom centered on the origin

### Minimap

- Appears automatically when the workflow has more than 8 nodes
- Shows a scaled overview of the entire graph in a small panel (200×150px max)
- Click within the minimap to navigate to that area of the canvas

### Selection

- **Click** a node to select it (opens the Node Panel)
- **Click and drag** on empty canvas to draw a selection rectangle — selects all nodes whose boundaries intersect the rectangle
- **Click** an edge to select it (highlights with a distinct color and shows a delete action)

---

## Creating Nodes

Nodes represent individual workflow steps. Each node has an action type that determines what it does when executed.

### Drag-and-Drop from Toolbar

1. Locate the **Toolbar** on the left side of the editor
2. Click and drag one of the five action type templates onto the Canvas
3. Release at the desired position — a new node is created at the drop coordinates
4. The **Node Panel** opens automatically so you can configure the node's properties

### Supported Action Types

| Action Type | Icon | Description |
|-------------|------|-------------|
| **run_task** | Task | Execute a predefined task from the task library |
| **http_request** | HTTP | Make an HTTP request to an external endpoint |
| **builtin.delay** | Wait | Pause execution for a specified duration |
| **builtin.condition** | Cond | Evaluate a condition to branch execution (rendered as diamond shape) |
| **builtin.loop** | Loop | Iterate over an array and execute body steps once per item |
| **terraform** | TF | Execute a Terraform template (plan/apply) |

### Node Identification

- Each new node receives an auto-generated `step_id` in the format `{action_type}_{number}` (e.g., `run_task_1`, `http_request_2`)
- If a collision exists, a numeric suffix is appended (e.g., `run_task_1_2`)
- The `step_id` is read-only after creation

### Visual Indicators on Nodes

Nodes display badges and icons to communicate configuration at a glance:

| Indicator | Condition | Appearance |
|-----------|-----------|------------|
| **Retry badge** | `retry_count > 0` | Numeric badge showing retry count |
| **Continue badge** | `on_failure = "continue"` | Icon indicating execution continues on failure |
| **Timeout warning** | `timeout_seconds > 0` and `< 60` | Warning icon for short timeout |
| **Diamond shape** | `action = "builtin.condition"` | Node rendered as diamond with distinct color |
| **Color-coded badge** | Always | Each action type has a unique badge color |

Node labels are truncated to 40 characters with ellipsis for long names.

### Dropping Outside the Canvas

If you release a drag outside the canvas boundaries, the drop is cancelled — no node is created and no state changes.

---

## Creating Edges (Dependencies)

Edges define execution order. A directed edge from Node A to Node B means "Node B depends on Node A" — B will not execute until A completes.

### Drawing an Edge

1. Hover over a node to reveal its **connection handles** (small circles on the node border)
2. Click and drag from the **output handle** (bottom) of the source node
3. A visual connection line follows your cursor
4. Drop onto the **input handle** (top) of the target node
5. The edge is created and the source node's `step_id` is added to the target's `depends_on` array

### Parallel Paths

Nodes that share the same upstream dependencies (or have no mutual dependency) execute in parallel. The auto-layout positions parallel nodes side by side with at least 40px spacing.

```
        ┌──────────┐
        │  Start   │
        └────┬─────┘
             │
      ┌──────┴──────┐
      ▼              ▼
┌──────────┐  ┌──────────┐
│  Task A  │  │  Task B  │    ← These run in parallel
└────┬─────┘  └────┬─────┘
      │              │
      └──────┬──────┘
             ▼
        ┌──────────┐
        │   End    │
        └──────────┘
```

### Edge Constraints

The editor enforces DAG integrity:

| Constraint | Behavior |
|------------|----------|
| **Cycle detection** | If adding an edge would create a circular dependency, the connection is rejected with a toast: "Cannot create dependency — would create a circular loop" |
| **Self-loops** | Connecting a node to itself is silently rejected |
| **Duplicate edges** | Creating a second edge in the same direction between two nodes is silently rejected |

### Deleting an Edge

1. Click on an edge to select it (it highlights with a distinct color)
2. A delete button or action appears on the edge
3. Click delete to remove the edge — the source node's `step_id` is removed from the target's `depends_on` array

---

## Editing Node Properties (Node Panel)

Click any node to open the **Node Panel** on the right side of the editor. The panel displays all configurable properties for the selected step.

### Common Fields (All Action Types)

| Field | Type | Description |
|-------|------|-------------|
| **step_id** | Read-only | Auto-generated unique identifier |
| **name** | Text input | Step name (1–100 characters, required) |
| **action** | Dropdown | Action type (changing this updates available fields) |
| **inputs** | Key-value editor | Input parameters for the step |
| **depends_on** | Read-only list | Shows current upstream dependencies (managed via edges) |
| **on_failure** | Dropdown | Behavior on failure: "stop", "continue", or "rollback" |
| **timeout_seconds** | Number | Execution timeout in seconds (1–86,400) |
| **retry_count** | Number | Number of retry attempts (0–10) |

### Action-Specific Fields

| Action Type | Additional Fields |
|-------------|-------------------|
| **run_task** | `task_id` — dropdown of available tasks from the task library |
| **terraform** | `template_id` — dropdown of Terraform templates; approval checkbox |
| **http_request** | Common fields only (URL, method, headers configured via inputs) |
| **builtin.delay** | Common fields only (duration configured via inputs) |
| **builtin.condition** | Common fields only (condition expression via inputs) |
| **builtin.loop** | `loop_over` — expression resolving to an array (e.g., `{{steps.list_instances.data}}`); `loop_variable` — variable name for the current item (default: `"item"`), accessible as `{{loop.<variable>}}` or `{{steps.loop.<variable>}}`; `execution_mode` — `"sequential"` or `"parallel"` |

### Real-Time Updates

- Changes in the Node Panel update the node on the canvas within 200ms
- Name changes immediately reflect in the node label
- Indicator badges update within 1 second when relevant properties change (retry_count, on_failure, timeout_seconds)

### Error Handling

- If the tasks API or templates API fails to load, an inline error with a **Retry** button appears in the dropdown area
- Invalid values for timeout_seconds or retry_count show a validation error adjacent to the field and are not applied to the node

---

## AI Features

The DAG Editor integrates with the platform's AI assistant (Gemini 2.5 Flash Lite) to help you build workflows faster.

### Suggestion Panel

Located below the Node Panel on the right side, the Suggestion Panel provides context-aware recommendations.

**Automatic suggestions:**
- When you add a node, the AI analyzes the action type and existing graph structure
- Up to 3 suggested next-step node types appear within 3 seconds
- Example: After adding a "terraform" node, suggestions might include "builtin.condition" for approval gating or "builtin.delay" for cooldown

**Optimization recommendations:**
- When your workflow has 3+ nodes, the AI analyzes for optimization opportunities
- Example: "Steps X and Y have no mutual dependency and can execute in parallel"

**Accepting a suggestion:**
- Click **Accept** on any suggestion to apply it to the graph
- The change (add node, add edge, or restructure) is applied and the previous state is pushed to the undo stack
- You can always undo an accepted suggestion with Ctrl+Z

**Dismissing a suggestion:**
- Click **Dismiss** to remove a suggestion
- Dismissed suggestions do not reappear during the current editing session

### Natural Language Workflow Generation

1. Find the **NL Prompt** text input at the bottom of the Suggestion Panel
2. Type a description of the workflow you want to create (e.g., "Deploy a Terraform template, wait 30 seconds, then run a health check task. If the health check fails, rollback the Terraform.")
3. Press Enter or click Submit
4. The AI generates a complete DAG structure (nodes and edges) on the canvas within 10 seconds
5. All generated nodes have valid step_ids, supported action types, and proper dependency relationships

### AI Error Handling

| Scenario | Behavior |
|----------|----------|
| AI service unavailable | "AI suggestions unavailable. Try again later." shown in panel |
| NL generation timeout (>10s) | "AI generation timed out. Try a simpler description." |
| API error | Inline error with retry option |

---

## Saving Workflows

### Validation Before Save

When you click **Save**, the editor validates the entire graph before persisting:

| Validation Rule | Condition |
|-----------------|-----------|
| **Required name** | Every node must have a name (1–100 characters, whitespace-only names are invalid) |
| **Required task_id** | Nodes with action "run_task" must have a task selected |
| **Required template_id** | Nodes with action "terraform" must have a template selected |
| **No orphaned nodes** | In workflows with 2+ nodes, every node must have at least one incoming or outgoing edge |
| **Unique names** | No two nodes can share the same name (case-insensitive) |

### When Validation Fails

- Invalid nodes are highlighted with a **red border** on the canvas
- An error summary lists all validation errors grouped by node
- The Node Panel shows which specific rule failed for the selected node
- **Saving is blocked** until all errors are resolved

### Fixing Validation Errors

1. Click on a highlighted node to open its properties
2. Correct the issue (add a name, select a task, connect the node, etc.)
3. The red border clears within 1 second of the correction
4. Once all errors are resolved, save becomes available

### Serialization

The editor serializes the graph into the standard `WorkflowStep[]` JSON format:
- Every node becomes a WorkflowStep with all fields preserved
- Edges are converted to `depends_on` arrays on each target step
- The output is identical to what the Form Builder produces — the backend execution engine handles both seamlessly
- An empty canvas serializes to an empty steps array

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| **Ctrl+Z** (Cmd+Z on Mac) | Undo last action |
| **Ctrl+Shift+Z** (Cmd+Shift+Z on Mac) | Redo last undone action |
| **Delete** or **Backspace** | Delete selected node(s) and their connected edges |

### Undo/Redo Details

- Every graph modification (add/delete node, add/delete edge, move node, edit property) is tracked
- Undo stack holds up to **50 actions** per editing session
- When the stack is full, the oldest entry is discarded
- Performing a new action after undoing clears the redo stack
- Undo/redo buttons are disabled when their respective stacks are empty

---

## Auto-Layout

The editor uses the **dagre** algorithm to automatically position nodes in a readable arrangement.

- **Direction:** Top-to-bottom (TB) by default
- **Spacing:** At least 40px between parallel nodes, 80px between ranks (vertical levels)
- **No overlapping:** Nodes never overlap after layout is applied
- **When applied:** Automatically on first load; manually via the "Auto Layout" button
- **Performance:** Layout completes within 2 seconds on initial load, 1 second on re-layout

---

## Tips and Best Practices

| Tip | Why |
|-----|-----|
| Start with the main execution path | Build the happy path first, then add error handling and conditions |
| Use meaningful step names | Names appear on the canvas — descriptive names make the graph self-documenting |
| Leverage parallel paths | Steps without mutual dependencies run concurrently, reducing total execution time |
| Use AI suggestions | The AI learns from your graph context and suggests relevant next steps |
| Save frequently | The editor validates on save, catching issues early |
| Use Fit View after major changes | Recenters the viewport to show all nodes |
| Check the minimap for large workflows | Helps you navigate and understand the overall structure |

---

## Compatibility

- Workflows created in the **Form Builder** load in the DAG Editor without modification
- Workflows created in the **DAG Editor** load in the Form Builder without modification
- The underlying JSON format (`WorkflowStep[]`) is identical regardless of which editor created it
- You can switch between editors at any time without data loss

---

## Related Documentation

- [CMP Complete Functionality Guide](./CMP_COMPLETE_FUNCTIONALITY_GUIDE.md) — Full platform overview
- [Feature Catalog](./FEATURE-CATALOG.md) — Licensed feature reference
- [CMP Detailed User Guide](./CMP_DETAILED_USER_GUIDE.md) — General platform usage

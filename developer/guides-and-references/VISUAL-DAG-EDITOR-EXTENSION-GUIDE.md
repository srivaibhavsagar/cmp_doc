# Visual DAG Editor — Developer Extension Guide

This document covers the architecture, extension points, and step-by-step instructions for extending the Workflow Visual DAG Editor in CMP.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [File Structure](#file-structure)
4. [Adding a New Action Type](#adding-a-new-action-type)
5. [Creating a Custom Node Renderer](#creating-a-custom-node-renderer)
6. [Creating Custom Validators](#creating-custom-validators)
7. [Extending the AI Integration](#extending-the-ai-integration)
8. [Testing](#testing)
9. [Checklist](#checklist)

---

## Overview

The Visual DAG Editor is a canvas-based graphical interface for composing and editing workflows as Directed Acyclic Graphs. It uses ReactFlow for the canvas, dagre for auto-layout, and integrates with the existing backend API, feature toggle system, RBAC, and AI services (Gemini 2.5 Flash Lite).

The editor operates entirely on the frontend — all DAG manipulation (layout, cycle detection, validation) runs client-side. The backend API remains unchanged for workflow storage. A separate AI endpoint provides intelligent suggestions.

Key technologies:
- **ReactFlow** — Canvas rendering, node/edge interaction
- **dagre** — Automatic graph layout (top-to-bottom)
- **React Context** — State management (DAGEditorContext)
- **Gemini 2.5 Flash Lite** — AI-powered suggestions and NL generation

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DAG Editor Module                            │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  NodeToolbar  │  │  DAGCanvas   │  │  NodePropertyPanel       │ │
│  │  (drag src)   │  │  (ReactFlow) │  │  + ActionSpecificFields  │ │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘ │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │EditorControls│  │AISuggestion  │  │  DAGEditorContext         │ │
│  │(zoom/undo)   │  │Panel         │  │  (state + history)        │ │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
          │                     │                      │
          ▼                     ▼                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Utilities                                    │
│  stepIdGenerator │ cycleDetector │ graphSerializer │ dagValidator   │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Hierarchy

```
DAGEditor (orchestrator)
├── DAGEditorContext.Provider (state management)
│   ├── NodeToolbar (draggable action templates)
│   ├── DAGCanvas (ReactFlow wrapper)
│   │   ├── WorkflowNode (default node renderer)
│   │   ├── ConditionNode (diamond-shaped condition node)
│   │   ├── DependencyEdge (custom edge with delete)
│   │   └── MiniMap (shown when >8 nodes)
│   ├── NodePropertyPanel (selected node editor)
│   │   └── ActionSpecificFields (action-type-specific form fields)
│   ├── AISuggestionPanel (AI suggestions + NL prompt)
│   └── EditorControls (zoom, fit, undo/redo)
```

---

## File Structure

```
frontend/src/components/dag-editor/
├── DAGEditor.tsx              # Main orchestrator component
├── DAGCanvas.tsx              # ReactFlow canvas wrapper + node type registration
├── NodeToolbar.tsx            # Draggable action type templates (NODE_TEMPLATES)
├── NodePropertyPanel.tsx      # Selected node property editor
├── ActionSpecificFields.tsx   # Action-type-specific form fields
├── AISuggestionPanel.tsx      # AI suggestions + NL prompt
├── EditorControls.tsx         # Zoom, fit, undo/redo buttons
├── types.ts                   # Core TypeScript interfaces (StepAction, DAGNode, etc.)
├── index.ts                   # Public exports
├── nodes/
│   ├── WorkflowNode.tsx       # Default node renderer (ACTION_BADGE_COLORS, ACTION_LABELS)
│   ├── ConditionNode.tsx      # Diamond-shaped condition node
│   └── NodeIndicators.tsx     # Retry, failure, timeout badges
├── edges/
│   └── DependencyEdge.tsx     # Custom edge with delete button
├── hooks/
│   ├── useDAGEditor.ts        # Context consumer hook
│   ├── useGraphLayout.ts      # Dagre layout logic
│   ├── useUndoRedo.ts         # Undo/redo stack management
│   ├── useGraphValidation.ts  # Validation rules
│   ├── useCycleDetection.ts   # Cycle detection + toast
│   └── useAISuggestions.ts    # AI integration hook
├── utils/
│   ├── graphSerializer.ts     # WorkflowStep[] ↔ ReactFlow nodes/edges
│   ├── stepIdGenerator.ts     # Unique step_id generation
│   ├── cycleDetector.ts       # DFS-based cycle detection
│   └── dagValidator.ts        # Validation rule engine
└── context/
    └── DAGEditorContext.tsx    # Graph state + history context provider

backend/app/
├── models/ai_dag.py           # DAGAIRequest, DAGAIResponse Pydantic models
├── services/ai_dag_service.py # Prompt building, response parsing, Gemini calls
└── api/v1/endpoints/ai_dag.py # POST /api/v1/ai/dag-suggestions, dag-generate
```

---

## Adding a New Action Type

To add a new action type (e.g., `"notification"` or `"approval"`), update the following files in order:

### Step 1: Register in `StepAction` type union

**File:** `frontend/src/components/dag-editor/types.ts`

Add the new action to the `StepAction` type union:

```typescript
export type StepAction =
  | 'run_task'
  | 'http_request'
  | 'builtin.delay'
  | 'builtin.condition'
  | 'terraform'
  | 'notification';  // ← New action type
```

### Step 2: Add badge colors and label

**File:** `frontend/src/components/dag-editor/nodes/WorkflowNode.tsx`

Add an entry to `ACTION_BADGE_COLORS` and `ACTION_LABELS`:

```typescript
const ACTION_BADGE_COLORS: Record<StepAction, { bg: string; text: string }> = {
  // ... existing entries ...
  notification: { bg: 'bg-pink-100', text: 'text-pink-700' },
};

const ACTION_LABELS: Record<StepAction, string> = {
  // ... existing entries ...
  notification: 'Notification',
};
```

Each action type must have a distinct badge color to satisfy the visual differentiation requirement.

### Step 3: Add toolbar template

**File:** `frontend/src/components/dag-editor/NodeToolbar.tsx`

Add a new entry to the `NODE_TEMPLATES` array:

```typescript
import { Bell } from 'lucide-react';  // Choose an appropriate Lucide icon

const NODE_TEMPLATES: NodeTemplate[] = [
  // ... existing templates ...
  {
    action: 'notification',
    label: 'Notification',
    icon: Bell,
    colorClass: 'text-pink-600',
    bgClass: 'bg-pink-50 border-pink-200 hover:bg-pink-100',
  },
];
```

### Step 4: Add validation rules

**File:** `frontend/src/components/dag-editor/utils/dagValidator.ts`

If the new action type requires specific fields, add validation logic inside `validateGraph()`:

```typescript
// Rule: notification requires a channel
if (data.action === 'notification' && !data.inputs?.channel) {
  errors.push({
    nodeId: node.id,
    field: 'inputs.channel',
    message: 'Notification channel is required',
    rule: 'required_channel' as ValidationRule,
  });
}
```

If you add a new `ValidationRule` identifier, also update the `ValidationRule` type in `types.ts`:

```typescript
export type ValidationRule =
  | 'required_task_id'
  | 'required_template_id'
  | 'required_name'
  | 'name_too_long'
  | 'orphaned_node'
  | 'duplicate_name'
  | 'required_channel';  // ← New rule
```

### Step 5: Add action-specific fields

**File:** `frontend/src/components/dag-editor/ActionSpecificFields.tsx`

Add a new rendering branch for the action type:

```typescript
// Render: notification action
if (action === 'notification') {
  return (
    <div className="space-y-2">
      <label
        htmlFor={`channel-select-${nodeId}`}
        className="block text-sm font-medium text-gray-700"
      >
        Notification Channel
      </label>
      <select
        id={`channel-select-${nodeId}`}
        value={(nodeData.inputs?.channel as string) || ''}
        onChange={(e) => onUpdate({ inputs: { ...nodeData.inputs, channel: e.target.value } })}
        className="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
      >
        <option value="">Select channel...</option>
        <option value="email">Email</option>
        <option value="slack">Slack</option>
        <option value="webhook">Webhook</option>
      </select>
    </div>
  );
}
```

### Step 6: Update graph serializer (if needed)

**File:** `frontend/src/components/dag-editor/utils/graphSerializer.ts`

If the new action type uses a custom node type (not `'workflow'`), update the `deserializeWorkflow` function:

```typescript
// In deserializeWorkflow:
type: step.action === 'builtin.condition' ? 'condition'
    : step.action === 'notification' ? 'notification'  // ← Custom node type
    : 'workflow',
```

### Step 7: Update DAGCanvas node types (if custom renderer)

**File:** `frontend/src/components/dag-editor/DAGCanvas.tsx`

If you created a custom node renderer (see next section), register it:

```typescript
const nodeTypes = {
  workflow: WorkflowNode,
  condition: ConditionNode,
  notification: NotificationNode,  // ← New custom renderer
};
```

---

## Creating a Custom Node Renderer

Custom node renderers allow you to change how specific action types appear on the canvas (e.g., diamond shape for conditions, hexagon for approvals).

### Step 1: Create the component

**File:** `frontend/src/components/dag-editor/nodes/NotificationNode.tsx`

```typescript
import { memo } from 'react';
import { Handle, Position, type NodeProps } from 'reactflow';
import type { WorkflowNodeData } from '../types';
import NodeIndicators from './NodeIndicators';

function NotificationNode({ data }: NodeProps<WorkflowNodeData>) {
  const { name } = data;
  const displayName = name.length > 40 ? name.slice(0, 39) + '…' : name;

  return (
    <div className="relative rounded-full border-2 border-pink-300 bg-pink-50 px-4 py-3 shadow-sm min-w-[160px] text-center">
      {/* Input handle (top) */}
      <Handle
        type="target"
        position={Position.Top}
        className="!w-3 !h-3 !bg-pink-400 !border-2 !border-white"
      />

      {/* Badge */}
      <div className="mb-1">
        <span className="inline-block rounded px-1.5 py-0.5 text-xs font-medium bg-pink-100 text-pink-700">
          Notification
        </span>
      </div>

      {/* Label */}
      <div className="text-sm font-medium text-gray-900" title={name}>
        {displayName}
      </div>

      {/* Visual indicators */}
      <NodeIndicators data={data} />

      {/* Output handle (bottom) */}
      <Handle
        type="source"
        position={Position.Bottom}
        className="!w-3 !h-3 !bg-pink-400 !border-2 !border-white"
      />
    </div>
  );
}

export default memo(NotificationNode);
```

**Key requirements for custom node renderers:**
- Must accept `NodeProps<WorkflowNodeData>` from ReactFlow
- Must include `Handle` components for input (top) and output (bottom) connections
- Must truncate labels to 40 characters maximum
- Must render `<NodeIndicators data={data} />` for retry/failure/timeout badges
- Must be wrapped in `memo()` for performance

### Step 2: Register in DAGCanvas

**File:** `frontend/src/components/dag-editor/DAGCanvas.tsx`

```typescript
import NotificationNode from './nodes/NotificationNode';

const nodeTypes = {
  workflow: WorkflowNode,
  condition: ConditionNode,
  notification: NotificationNode,  // ← Register here
};
```

### Step 3: Handle in deserializeWorkflow

**File:** `frontend/src/components/dag-editor/utils/graphSerializer.ts`

Update the `type` assignment in `deserializeWorkflow` to map the action to your custom node type:

```typescript
const nodes: DAGNode[] = steps.map(step => ({
  id: step.step_id,
  type: step.action === 'builtin.condition' ? 'condition'
      : step.action === 'notification' ? 'notification'
      : 'workflow',
  // ... rest of node definition
}));
```

### Step 4: Update DAGNode type (if needed)

**File:** `frontend/src/components/dag-editor/types.ts`

Extend the `type` field to include your new node type:

```typescript
export interface DAGNode {
  id: string;
  type: 'workflow' | 'condition' | 'notification';  // ← Add here
  position: { x: number; y: number };
  data: WorkflowNodeData;
}
```

---

## Creating Custom Validators

The validation engine in `dagValidator.ts` runs before every save operation. You can extend it with new rules.

### Step 1: Add a new ValidationRule type

**File:** `frontend/src/components/dag-editor/types.ts`

```typescript
export type ValidationRule =
  | 'required_task_id'
  | 'required_template_id'
  | 'required_name'
  | 'name_too_long'
  | 'orphaned_node'
  | 'duplicate_name'
  | 'max_parallel_paths'    // ← New rule
  | 'timeout_too_short';    // ← New rule
```

### Step 2: Add rule logic in validateGraph

**File:** `frontend/src/components/dag-editor/utils/dagValidator.ts`

Add your validation logic inside the `validateGraph` function. Rules can be per-node or graph-wide:

```typescript
export function validateGraph(ctx: ValidationContext): ValidationError[] {
  const errors: ValidationError[] = [];

  // ... existing per-node rules ...

  for (const node of ctx.nodes) {
    const { data } = node;

    // Custom rule: warn if timeout is dangerously short
    if (data.timeout_seconds > 0 && data.timeout_seconds < 10) {
      errors.push({
        nodeId: node.id,
        field: 'timeout_seconds',
        message: 'Timeout under 10 seconds may cause premature failures',
        rule: 'timeout_too_short',
      });
    }
  }

  // ... existing graph-wide rules (orphan detection) ...

  // Custom graph-wide rule: limit parallel paths
  if (ctx.nodes.length > 1) {
    // Find nodes at the same depth with shared parents
    const rootNodes = ctx.nodes.filter(
      n => !ctx.edges.some(e => e.target === n.id)
    );
    if (rootNodes.length > 5) {
      for (const root of rootNodes.slice(5)) {
        errors.push({
          nodeId: root.id,
          field: '_graph',
          message: 'Maximum 5 parallel entry points exceeded',
          rule: 'max_parallel_paths',
        });
      }
    }
  }

  return errors;
}
```

### Validation Architecture

The validation flow works as follows:

1. **User clicks Save** → `DAGEditor.tsx` calls `validate()` from context
2. **Context calls** `validateGraph({ nodes, edges })` from `dagValidator.ts`
3. **Errors are stored** in `DAGEditorState.validationErrors`
4. **NodePropertyPanel** reads errors for the selected node and displays them adjacent to fields
5. **DAGCanvas** applies red border to nodes with errors
6. **On correction** — `useGraphValidation` hook clears errors within 1 second

### Best Practices for Custom Validators

- Use descriptive `message` strings — they're shown directly to users
- Use the `field` property to associate errors with specific form fields (or `'_graph'` for graph-wide issues)
- Keep validation synchronous — the `validateGraph` function must return immediately
- Don't duplicate existing rules — check what's already validated before adding

---

## Extending the AI Integration

The AI subsystem has two extension points: backend prompts (for new suggestion types) and frontend handling (for new suggestion actions).

### Backend: Updating Prompts

**File:** `backend/app/services/ai_dag_service.py`

The `build_dag_prompt()` function constructs prompts for three request types:
- `suggest_next` — Suggests 1-3 next steps after a node is added
- `optimize` — Analyzes the graph for optimization opportunities
- `generate_dag` — Generates a full DAG from natural language

To add a new request type (e.g., `"security_audit"`):

```python
# In build_dag_prompt():
elif request_type == "security_audit":
    return (
        f"You are a cloud security expert reviewing a workflow DAG.\n\n"
        f"{context_block}\n\n"
        f"Analyze this workflow for security concerns:\n"
        f"- Missing approval steps before destructive operations\n"
        f"- Terraform applies without condition checks\n"
        f"- HTTP requests without error handling\n\n"
        f"Respond with ONLY a JSON array of findings...\n"
    )
```

### Backend: Adding New Suggestion Types

**File:** `backend/app/models/ai_dag.py`

Update the `DAGAIRequest` model to accept the new type:

```python
class DAGAIRequest(BaseModel):
    type: str  # "suggest_next" | "optimize" | "generate_dag" | "security_audit"
    context: DAGContext
    prompt: Optional[str] = None
```

Update the `DAGSuggestion` model if you need new suggestion types:

```python
class DAGSuggestion(BaseModel):
    id: str
    type: str  # "next_step" | "optimization" | "parallel" | "security"
    title: str
    description: str
    action: SuggestionAction
```

### Backend: Response Parsing

**File:** `backend/app/services/ai_dag_service.py`

Update `parse_ai_dag_response()` to handle the new request type:

```python
def parse_ai_dag_response(response_text: str, request_type: str) -> DAGAIResponse:
    # ... existing parsing logic ...

    if request_type in ("suggest_next", "optimize", "security_audit"):
        return _parse_suggestions(parsed, request_type)
    elif request_type == "generate_dag":
        return _parse_generated_dag(parsed)
```

### Frontend: Handling New Suggestion Actions

**File:** `frontend/src/components/dag-editor/hooks/useAISuggestions.ts`

The `acceptSuggestion` handler in the hook processes suggestion actions. To handle a new action type:

```typescript
// When accepting a suggestion with a custom action type:
case 'remove_node':
  // Remove the specified node from the graph
  removeNodes([suggestion.action.nodeId]);
  break;
```

### Frontend: Adding New Suggestion Types

**File:** `frontend/src/components/dag-editor/types.ts`

Extend the `SuggestionAction` type:

```typescript
export type SuggestionAction =
  | { type: 'add_node'; action: StepAction; connectTo: string }
  | { type: 'add_edge'; source: string; target: string }
  | { type: 'restructure'; changes: GraphChange[] }
  | { type: 'remove_node'; nodeId: string };  // ← New action type
```

### AI Integration Architecture

```
Frontend                              Backend
────────                              ───────
useAISuggestions hook                  POST /api/v1/ai/dag-suggestions
  │                                     │
  ├─ On node added → suggest_next      ├─ build_dag_prompt()
  ├─ On 3+ nodes  → optimize           ├─ call Gemini API
  └─ On NL submit → generate_dag       ├─ parse_ai_dag_response()
                                        └─ Return DAGAIResponse
```

The AI uses the existing `AIContext` provider and the platform's Gemini 2.5 Flash Lite configuration. No separate API key is needed — it shares the key configured in Platform Settings.

---

## Testing

### Running Existing Tests

```bash
# Frontend unit tests
cd frontend && npm run test -- --run

# Backend tests
cd backend && python -m pytest tests/ -v

# Property-based tests (if configured)
cd frontend && npm run test -- --run dag-editor
```

### Writing Tests for New Extensions

When adding a new action type or validator, write tests covering:

1. **Serialization round-trip** — Ensure the new action type survives `serialize → deserialize` without data loss
2. **Validation** — Test that your new rules produce errors for invalid states and pass for valid states
3. **Node rendering** — Verify the custom renderer displays correctly with various data configurations

Example test for a new validator rule:

```typescript
import { validateGraph } from './dagValidator';

describe('custom validation: timeout_too_short', () => {
  it('should flag nodes with timeout under 10 seconds', () => {
    const ctx = {
      nodes: [{
        id: 'task_1',
        type: 'workflow' as const,
        position: { x: 0, y: 0 },
        data: {
          step_id: 'task_1',
          name: 'Quick Task',
          action: 'run_task' as const,
          task_id: 'some-task',
          template_id: null,
          inputs: {},
          depends_on: [],
          on_failure: 'stop' as const,
          timeout_seconds: 5,
          retry_count: 0,
          output_key: null,
        },
      }],
      edges: [],
    };

    const errors = validateGraph(ctx);
    expect(errors.some(e => e.rule === 'timeout_too_short')).toBe(true);
  });
});
```

---

## Checklist

Use this checklist when adding a new action type to ensure nothing is missed:

- [ ] Add to `StepAction` type union in `types.ts`
- [ ] Add badge colors in `ACTION_BADGE_COLORS` (WorkflowNode.tsx)
- [ ] Add label in `ACTION_LABELS` (WorkflowNode.tsx)
- [ ] Add toolbar template in `NODE_TEMPLATES` (NodeToolbar.tsx)
- [ ] Add validation rules in `validateGraph()` (dagValidator.ts)
- [ ] Add `ValidationRule` type if new rule identifiers are needed (types.ts)
- [ ] Add action-specific fields in `ActionSpecificFields.tsx`
- [ ] Update `deserializeWorkflow()` if custom node type is used (graphSerializer.ts)
- [ ] Register custom node renderer in `nodeTypes` (DAGCanvas.tsx) — if applicable
- [ ] Update AI prompts to include the new action type (ai_dag_service.py)
- [ ] Add `SUPPORTED_ACTIONS` entry in backend (ai_dag_service.py)
- [ ] Write unit tests for new validation rules
- [ ] Write serialization round-trip test for the new action type
- [ ] Update this documentation to reflect the new extension point

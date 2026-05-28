# BYOUI (Bring Your Own UI) — Custom HTML/CSS/JS Catalog Guide

This document covers how to build catalog items using the BYOUI mode, which allows fully custom HTML/CSS/JS forms while still leveraging CMP platform features like reusable components, cost estimation, approvals, and credential selection.

---

## Table of Contents

1. [Overview](#overview)
2. [How BYOUI Works](#how-byoui-works)
3. [Platform Features Available to BYOUI](#platform-features-available-to-byoui)
4. [PostMessage API Reference](#postmessage-api-reference)
5. [Using Reusable Components](#using-reusable-components)
6. [Complete Example: Basic Form](#complete-example-basic-form)
7. [Complete Example: With Reusable Components](#complete-example-with-reusable-components)
8. [Styling Guidelines](#styling-guidelines)
9. [API Call to Create a BYOUI Catalog](#api-call-to-create-a-byoui-catalog)
10. [Best Practices](#best-practices)

---

## Overview

BYOUI (Bring Your Own UI) is a catalog UI mode that lets you write the entire input form in raw HTML, CSS, and JavaScript. The form is rendered inside a sandboxed iframe on the "Request Service" tab.

**When to use BYOUI:**

- You need custom UI elements not supported by the form builder (e.g., color pickers, drag-and-drop, canvas, charts)
- You want to embed a third-party widget or library
- You need complex client-side logic (live validation, conditional rendering, multi-step wizards)
- You want full control over layout and styling

**When to use Form Builder instead:**

- Standard forms with text, select, multiselect, radio, toggle, date, etc.
- You want zero-code form creation via the admin UI
- You need built-in field validation, conditional visibility, and layout sections/tabs

---

## How BYOUI Works

1. Admin creates a catalog item with `ui_mode: "byoui"` and provides HTML in `custom_ui_html`
2. When a user opens the catalog's "Request Service" tab, the HTML is rendered in a sandboxed iframe
3. The iframe communicates with CMP via `window.parent.postMessage()`
4. When the form is submitted, CMP receives the data and triggers the linked flow/workflow

```
┌─────────────────────────────────────────────────┐
│  CMP Platform (parent page)                     │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │  Recent Requests / Pre-fill               │  │
│  ├───────────────────────────────────────────┤  │
│  │  Credential Selector (Day 1 catalogs)     │  │
│  ├───────────────────────────────────────────┤  │
│  │  ┌─────────────────────────────────────┐  │  │
│  │  │  BYOUI iframe (your custom HTML)    │  │  │
│  │  │                                     │  │  │
│  │  │  postMessage ←→ CMP                 │  │  │
│  │  └─────────────────────────────────────┘  │  │
│  ├───────────────────────────────────────────┤  │
│  │  Cost Estimate Widget                     │  │
│  ├───────────────────────────────────────────┤  │
│  │  Approval Info                            │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

---

## Platform Features Available to BYOUI

BYOUI catalogs get the same platform features as form_builder catalogs:

| Feature | How it works |
|---------|-------------|
| **Recent Requests & Pre-fill** | Shown above the iframe — users can click to pre-fill from past executions |
| **Credential Selector** | For `day1` catalogs, the cloud credential dropdown appears above the iframe |
| **Cost Estimate** | If cost models are configured, the cost widget appears below the iframe |
| **Approval Workflow** | If `requires_approval` is set, approval info is shown below the iframe |
| **Reusable Components** | Accessible via the postMessage bridge (see below) |
| **Flow Execution** | Form data is passed to the linked flow just like form_builder mode |

---

## PostMessage API Reference

Your custom HTML communicates with CMP via `window.parent.postMessage()`. CMP responds back to the iframe via the same mechanism.

### Sending Data (iframe → CMP)

#### Submit Form Data

```javascript
window.parent.postMessage({
  type: 'form_submit',
  data: {
    submitted_data: JSON.stringify({
      project_name: 'my-project',
      environment: 'production'
    })
  }
}, '*');
```

This triggers the linked flow with the provided form data.

#### Request All Reusable Components

```javascript
window.parent.postMessage({ type: 'get_components' }, '*');
```

#### Request a Specific Component

```javascript
window.parent.postMessage({
  type: 'get_component',
  component_id: 'your-component-uuid'
}, '*');
```

#### Fetch Options from an Option List Component

```javascript
window.parent.postMessage({
  type: 'fetch_options',
  component_id: 'your-option-list-component-uuid'
}, '*');
```

This handles API calls for API-backed option lists — CMP makes the request server-side and returns the resolved options.

---

### Receiving Data (CMP → iframe)

Listen for messages in your iframe:

```javascript
window.addEventListener('message', function(event) {
  var msg = event.data;

  switch (msg.type) {
    case 'components_data':
      // msg.components = [{component_id, name, description, component_type, tags, data}, ...]
      console.log('All components:', msg.components);
      break;

    case 'component_data':
      // msg.component_id = requested ID
      // msg.component = {...} or null if not found
      console.log('Component:', msg.component);
      break;

    case 'options_data':
      // msg.component_id = requested option list ID
      // msg.options = [{label: "US East", value: "us-east-1"}, ...]
      // msg.error = string if fetch failed (optional)
      console.log('Options:', msg.options);
      break;
  }
});
```

---

## Using Reusable Components

### Option List Components

Option list components store dropdown options (static or API-backed). Your BYOUI form can fetch these to populate `<select>` elements dynamically.

**Step 1:** Create an option list component in the admin UI (Service Catalog → Components) or via API:

```bash
curl -X POST http://localhost:8001/api/v1/catalog-components/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "name": "AWS Regions",
    "component_type": "option_list",
    "tags": ["aws", "regions"],
    "data": {
      "option_source": "static",
      "options": [
        {"label": "US East (N. Virginia)", "value": "us-east-1"},
        {"label": "US West (Oregon)", "value": "us-west-2"},
        {"label": "EU (Ireland)", "value": "eu-west-1"},
        {"label": "AP (Singapore)", "value": "ap-southeast-1"}
      ]
    }
  }'
```

**Step 2:** Use the returned `component_id` in your BYOUI form:

```javascript
// Request options for the "AWS Regions" component
window.parent.postMessage({
  type: 'fetch_options',
  component_id: 'abc123-your-component-id'
}, '*');

// Handle the response
window.addEventListener('message', function(e) {
  if (e.data.type === 'options_data' && e.data.component_id === 'abc123-your-component-id') {
    populateSelect('region_select', e.data.options);
  }
});

function populateSelect(selectId, options) {
  var sel = document.getElementById(selectId);
  sel.innerHTML = '<option value="">Select a region...</option>';
  options.forEach(function(opt) {
    var o = document.createElement('option');
    o.value = opt.value;
    o.textContent = opt.label;
    sel.appendChild(o);
  });
}
```

### Input Components

Input components define reusable field configurations (type, placeholder, validation, default value, etc.). Your BYOUI form can fetch these to dynamically configure fields.

```javascript
window.parent.postMessage({
  type: 'get_component',
  component_id: 'input-component-uuid'
}, '*');

window.addEventListener('message', function(e) {
  if (e.data.type === 'component_data' && e.data.component) {
    var cfg = e.data.component.data;
    // cfg.input_type = 'string' | 'number' | 'select' | etc.
    // cfg.placeholder = 'Enter value...'
    // cfg.required = true/false
    // cfg.default_value = 'some default'
    // cfg.validation = { pattern: '^[a-z]+$', message: 'Lowercase only' }
    // cfg.min_value, cfg.max_value (for number/range)

    var input = document.getElementById('my_field');
    if (cfg.placeholder) input.placeholder = cfg.placeholder;
    if (cfg.default_value) input.value = cfg.default_value;
    if (cfg.required) input.required = true;
  }
});
```

### API-Backed Option Lists

For option lists that fetch from external APIs, CMP handles the API call for you (including authentication). Just use `fetch_options`:

```javascript
// CMP will call the configured API endpoint, apply response_path,
// and return [{label, value}] pairs — no CORS issues since CMP makes the call server-side
window.parent.postMessage({
  type: 'fetch_options',
  component_id: 'api-backed-option-list-id'
}, '*');
```

---

## Complete Example: Basic Form

A minimal BYOUI form that submits data to CMP:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<style>
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: transparent; color: #1e293b; line-height: 1.5; padding: 16px 0;
  }
  .form-fields { display: flex; flex-direction: column; gap: 20px; }
  .field-group { display: flex; flex-direction: column; gap: 5px; }
  .field-group label { font-size: 14px; font-weight: 500; color: #374151; }
  .field-group label .required { color: #ef4444; }
  input[type="text"], select, textarea {
    width: 100%; border: 1px solid #d1d5db; border-radius: 8px;
    padding: 9px 13px; font-size: 14px; color: #1e293b; background: #fff;
  }
  input:focus, select:focus, textarea:focus {
    outline: none; border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59,130,246,0.08);
  }
  select {
    appearance: none; padding-right: 36px;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='20' height='20' fill='none' stroke='%236b7280' stroke-width='1.5' viewBox='0 0 24 24'%3E%3Cpath d='M6 9l6 6 6-6'/%3E%3C/svg%3E");
    background-repeat: no-repeat; background-position: right 10px center;
  }
  .btn-submit {
    display: inline-flex; align-items: center; gap: 8px;
    background: #16a34a; color: #fff; border: none; border-radius: 8px;
    padding: 9px 20px; font-size: 14px; font-weight: 600; cursor: pointer;
  }
  .btn-submit:hover { background: #15803d; }
</style>
</head>
<body>
<form id="cmpForm" class="form-fields">
  <div class="field-group">
    <label>Instance Name <span class="required">*</span></label>
    <input id="instance_name" name="instance_name" type="text" placeholder="my-server-01" required />
  </div>
  <div class="field-group">
    <label>Instance Type</label>
    <select id="instance_type" name="instance_type">
      <option value="t3.micro">t3.micro (1 vCPU, 1 GB)</option>
      <option value="t3.small">t3.small (2 vCPU, 2 GB)</option>
      <option value="t3.medium">t3.medium (2 vCPU, 4 GB)</option>
      <option value="m5.large">m5.large (2 vCPU, 8 GB)</option>
    </select>
  </div>
  <div class="field-group">
    <label>Description</label>
    <textarea id="description" name="description" rows="3" placeholder="Purpose of this instance..."></textarea>
  </div>
  <div>
    <button type="submit" class="btn-submit">▶ Provision</button>
  </div>
</form>

<script>
  document.getElementById('cmpForm').addEventListener('submit', function(e) {
    e.preventDefault();
    var data = {};
    new FormData(e.target).forEach(function(v, k) { data[k] = v; });
    window.parent.postMessage({
      type: 'form_submit',
      data: { submitted_data: JSON.stringify(data) }
    }, '*');
  });
</script>
</body>
</html>
```

---

## Complete Example: With Reusable Components

A BYOUI form that dynamically loads options from CMP reusable components:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<style>
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: transparent; color: #1e293b; line-height: 1.5; padding: 16px 0;
  }
  .form-fields { display: flex; flex-direction: column; gap: 20px; }
  .field-group { display: flex; flex-direction: column; gap: 5px; }
  .field-group label { font-size: 14px; font-weight: 500; color: #374151; }
  .field-group label .required { color: #ef4444; }
  .field-group .hint { font-size: 12px; color: #9ca3af; }
  input[type="text"], select, textarea {
    width: 100%; border: 1px solid #d1d5db; border-radius: 8px;
    padding: 9px 13px; font-size: 14px; color: #1e293b; background: #fff;
  }
  input:focus, select:focus, textarea:focus {
    outline: none; border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59,130,246,0.08);
  }
  select {
    appearance: none; padding-right: 36px;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='20' height='20' fill='none' stroke='%236b7280' stroke-width='1.5' viewBox='0 0 24 24'%3E%3Cpath d='M6 9l6 6 6-6'/%3E%3C/svg%3E");
    background-repeat: no-repeat; background-position: right 10px center;
  }
  .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
  .btn-submit {
    display: inline-flex; align-items: center; gap: 8px;
    background: #16a34a; color: #fff; border: none; border-radius: 8px;
    padding: 9px 20px; font-size: 14px; font-weight: 600; cursor: pointer;
  }
  .btn-submit:hover { background: #15803d; }
  .loading { font-size: 12px; color: #6b7280; font-style: italic; }
  .component-tag {
    display: inline-block; font-size: 10px; color: #6366f1; background: #eef2ff;
    padding: 1px 6px; border-radius: 3px; margin-left: 6px;
  }
</style>
</head>
<body>
<form id="cmpForm" class="form-fields">
  <div class="field-group">
    <label>Application Name <span class="required">*</span></label>
    <input id="app_name" name="app_name" type="text" placeholder="my-app" required />
  </div>

  <div class="form-row">
    <div class="field-group">
      <label>Region <span class="component-tag">from component</span></label>
      <select id="region" name="region">
        <option value="">Loading regions...</option>
      </select>
      <span class="hint">Powered by CMP reusable option list</span>
    </div>
    <div class="field-group">
      <label>Team <span class="component-tag">from component</span></label>
      <select id="team" name="team">
        <option value="">Loading teams...</option>
      </select>
    </div>
  </div>

  <div class="field-group">
    <label>Instance Size</label>
    <select id="instance_size" name="instance_size">
      <option value="small">Small (1 vCPU, 2 GB)</option>
      <option value="medium">Medium (2 vCPU, 4 GB)</option>
      <option value="large">Large (4 vCPU, 8 GB)</option>
    </select>
  </div>

  <div>
    <button type="submit" class="btn-submit">▶ Provision</button>
  </div>
</form>

<script>
  // ============================================================
  // CONFIGURATION: Replace these with your actual component IDs
  // ============================================================
  var REGION_COMPONENT_ID = 'your-region-option-list-component-id';
  var TEAM_COMPONENT_ID = 'your-team-option-list-component-id';

  // ============================================================
  // Helper: populate a <select> with options
  // ============================================================
  function populateSelect(selectId, options, placeholder) {
    var sel = document.getElementById(selectId);
    if (!sel) return;
    sel.innerHTML = '';
    var defaultOpt = document.createElement('option');
    defaultOpt.value = '';
    defaultOpt.textContent = placeholder || 'Select...';
    sel.appendChild(defaultOpt);
    options.forEach(function(opt) {
      var o = document.createElement('option');
      o.value = opt.value;
      o.textContent = opt.label;
      sel.appendChild(o);
    });
  }

  // ============================================================
  // Listen for CMP responses
  // ============================================================
  window.addEventListener('message', function(e) {
    if (!e.data || !e.data.type) return;

    if (e.data.type === 'options_data') {
      if (e.data.component_id === REGION_COMPONENT_ID) {
        populateSelect('region', e.data.options, 'Select a region...');
      }
      if (e.data.component_id === TEAM_COMPONENT_ID) {
        populateSelect('team', e.data.options, 'Select a team...');
      }
    }

    // You can also use 'components_data' to discover all available components
    if (e.data.type === 'components_data') {
      console.log('[BYOUI] Available components:', e.data.components);
      // Auto-discover option lists by name or tag:
      e.data.components.forEach(function(c) {
        if (c.component_type === 'option_list' && c.tags.indexOf('regions') !== -1) {
          window.parent.postMessage({ type: 'fetch_options', component_id: c.component_id }, '*');
        }
      });
    }
  });

  // ============================================================
  // On load: request options from CMP reusable components
  // ============================================================
  window.parent.postMessage({ type: 'fetch_options', component_id: REGION_COMPONENT_ID }, '*');
  window.parent.postMessage({ type: 'fetch_options', component_id: TEAM_COMPONENT_ID }, '*');

  // ============================================================
  // Form submission
  // ============================================================
  document.getElementById('cmpForm').addEventListener('submit', function(e) {
    e.preventDefault();
    var data = {};
    new FormData(e.target).forEach(function(v, k) { data[k] = v; });
    window.parent.postMessage({
      type: 'form_submit',
      data: { submitted_data: JSON.stringify(data) }
    }, '*');
  });
</script>
</body>
</html>
```

---

## Styling Guidelines

To make your BYOUI form look native to CMP, follow these conventions:

| Property | Value |
|----------|-------|
| Body background | `transparent` (the parent card provides the white background) |
| Font family | `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif` |
| Text color | `#1e293b` (primary), `#374151` (labels), `#9ca3af` (hints) |
| Input border | `1px solid #d1d5db`, border-radius `8px` |
| Input focus | `border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59,130,246,0.08)` |
| Input padding | `9px 13px` |
| Font size | `14px` for inputs, `14px` for labels, `12px` for hints |
| Submit button | `background: #16a34a` (green, matching CMP's Provision button) |
| Required indicator | `<span style="color: #ef4444">*</span>` |
| Field gap | `20px` between fields, `5px` between label and input |
| Grid layout | `display: grid; grid-template-columns: 1fr 1fr; gap: 16px` for two-column rows |

---

## API Call to Create a BYOUI Catalog

```bash
curl -X POST http://localhost:8001/api/v1/catalog \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "name": "My Custom Form Catalog",
    "description": "A catalog item with a fully custom HTML form.",
    "tags": ["custom-ui", "byoui"],
    "catalog_type": "day2",
    "ui_mode": "byoui",
    "status": "published",
    "custom_ui_html": "<YOUR_HTML_HERE>",
    "submit_contract": {
      "app_name": "string",
      "region": "string",
      "team": "string",
      "instance_size": "string"
    },
    "flow_id": "<YOUR_FLOW_ID>",
    "requires_approval": false,
    "show_cost_estimate": true,
    "allowed_roles": ["user", "developer", "admin"]
  }'
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `ui_mode` | Must be `"byoui"` |
| `custom_ui_html` | Your full HTML document as a string |
| `submit_contract` | Documents the expected fields from the form (for reference) |
| `flow_id` | The flow to execute when the form is submitted |
| `catalog_type` | `"day1"` (shows credential selector) or `"day2"` (no credential needed) |

---

## Best Practices

1. **Use `transparent` background** — The CMP page provides the white card container. Don't add your own card/shadow wrapper.

2. **Match CMP's input styles** — Use the same border radius (8px), colors, and focus states so the form feels native.

3. **Keep the submit button green** — Use `#16a34a` to match the platform's "Provision" button. Label it "Provision" or "Submit Request".

4. **Always stringify submitted data** — CMP expects `{ submitted_data: JSON.stringify({...}) }` in the postMessage payload.

5. **Request components on load** — Call `postMessage({ type: 'get_components' })` or `fetch_options` immediately so dropdowns are populated by the time the user interacts.

6. **Handle loading states** — Show "Loading..." in selects while waiting for component data.

7. **Use `submit_contract`** — Document what fields your form submits so other admins understand the data shape.

8. **Test with the iframe sandbox** — CMP uses `sandbox="allow-scripts allow-forms"`. External network requests from the iframe are blocked — use `fetch_options` to let CMP make API calls on your behalf.

9. **Don't duplicate platform features** — Cost estimation, approval info, and credential selection are handled by CMP outside the iframe. Focus your HTML on the form fields only.

10. **Use component IDs, not names** — When referencing reusable components, always use the `component_id` UUID. Names can change; IDs are stable.

# Event Modeling AI: UI Exploration via Chrome DevTools

**Read CLAUDE.md first** - defines core concepts, slice types, and JSON schema.

## Mission
Execute ONE state machine iteration per run. Explore a running application via Chrome DevTools MCP to discover business processes and build Event Models.

---

## State Machine (execute in order, stop after first match)

### State 1: High-Level Analysis Missing
**If:** `analysis/ui-high-level-analysis.json` doesn't exist

**Do:**
1. **Quick navigation scan:** Explore menus, sidebar, header links - identify all reachable screens
2. **For each screen:** Identify forms (→ Commands), data displays (→ ReadModels), action buttons
3. Create MULTIPLE slices (5-20+ typical):
   - Forms/buttons that modify data → `STATE_CHANGE` (Command + Event)
   - Data displays/lists/tables → `STATE_VIEW` (ReadModel)
   - Background indicators (status updates, notifications) → `AUTOMATION` (Processor + Command + Events)
4. High-level mode: `fields: []`, `specifications: []`
5. **Validate:** sliceType ∈ {STATE_CHANGE, STATE_VIEW, AUTOMATION}, status ∈ {Created, Done, InProgress}
6. Write `analysis/ui-high-level-analysis.json` → **STOP**

---

### State 2: Flows Catalog Missing
**If:** Any folder has `ui-high-level-analysis.json` but no `ui-flows.json`

**Do:**
1. Find first such folder
2. Group slices into logical flows based on:
   - Navigation structure
   - Business process sequences (e.g., Browse → Add to Cart → Checkout → Payment)
   - User journeys observed in UI
3. Write `ui-flows.json`:
```json
{ "flows": [{ "name": "Flow Name", "folder": "flow-name", "status": "open", "depth": 1 }] }
```
Or `{ "flows": [] }` if no sub-flows → **STOP**

---

### State 3: Flow High-Level Analysis
**If:** Any flow has `status: "open"` and no `ui-high-level-analysis.json` in its folder

**Do:**
1. Find first such flow
2. **Quick scan:** Navigate to relevant screens, identify UI patterns - no detailed field extraction
3. Create MULTIPLE slices with empty fields/specs
4. **Validate:** sliceType/status enums (see State 1)
5. Write `analysis/{path}/ui-high-level-analysis.json` → **STOP**

---

### State 4: Detailed Analysis Missing
**If:** Any folder has `ui-flows.json` but no `ui-config.json`

**Do:**
1. Find first such folder
2. **Deep exploration using Chrome DevTools MCP:**

   **For each screen in the flow:**
   - Navigate to screen
   - **Inspect forms:** Extract field names, types, validation patterns
   - **Monitor Network tab:** Capture API requests/responses for:
     - Endpoint URLs → `apiEndpoint` field
     - Request payloads → Command fields
     - Response payloads → Event/ReadModel fields
   - **Test interactions:** Fill forms, click buttons, observe changes
   - **Capture field examples** from actual data displayed

3. **Build detailed elements:**
   - Commands with fields from form inputs + API request bodies
   - Events with fields from API responses
   - ReadModels with fields from displayed data + API responses
   - Screens with actual UI component references

4. **Extract specifications from observed behavior:**
   - GIVEN: What state must exist (displayed data)
   - WHEN: User action (button click, form submit)
   - THEN: Expected result (new data displayed, navigation, message)

5. Write `analysis/{path}/ui-config.json` → **STOP**

---

### State 5: Mark Flow Completed
**If:** Flow has both `ui-high-level-analysis.json` AND `ui-config.json`, AND (`ui-flows.json` is empty OR all sub-flows completed), AND parent shows `status: "open"`

**Do:** Update parent's `ui-flows.json`: `"open"` → `"completed"` → **STOP**

---

### State 6: All Complete
**If:** All flows recursively have `status: "completed"`

**Do:** Output `<promise>COMPLETE</promise>` → **STOP**

---

## Analysis Modes

| Mode | Scope | Fields | Specs | Output |
|------|-------|--------|-------|--------|
| High-Level | Screen navigation, UI patterns only | `[]` | `[]` | `ui-high-level-analysis.json` |
| Detailed | Forms, Network requests, interactions | types/examples | From observed behavior | `ui-config.json` |

---

## Chrome DevTools MCP Usage

### Discovery Phase (States 1-3)
```
1. mcp_navigate(url)           → Go to screen
2. mcp_get_elements("nav")     → Find navigation elements
3. mcp_get_elements("form")    → Find forms
4. mcp_get_elements("button")  → Find action triggers
5. mcp_get_elements("table")   → Find data displays
```

### Detail Phase (State 4)
```
1. mcp_network_enable()        → Start capturing network
2. mcp_click(selector)         → Trigger action
3. mcp_type(selector, value)   → Fill form field
4. mcp_submit(selector)        → Submit form
5. mcp_network_get_requests()  → Get captured API calls
6. mcp_evaluate(js)            → Extract DOM data
```

### Key Selectors to Inspect
```css
/* Forms & Inputs */
form, input, select, textarea, [type="submit"]

/* Actions */
button, [role="button"], a[href], [onclick]

/* Data Display */
table, [role="grid"], .list, .card, [data-*]

/* Navigation */
nav, [role="navigation"], .menu, .sidebar

/* State Indicators */
.status, .badge, .alert, .notification, [role="status"]
```

---

## Mapping UI to Event Model Elements

| UI Pattern | Slice Type | Elements |
|------------|------------|----------|
| Form with Submit | `STATE_CHANGE` | Screen(Form) → Command → Event |
| Data Table/List | `STATE_VIEW` | Event(source) → ReadModel → Screen(Display) |
| Edit Button → Modal → Save | `STATE_CHANGE` | Screen(Modal) → Command → Event |
| Status Badge Updates | `AUTOMATION` | Processor → Command → Event → ReadModel |
| Search/Filter | `STATE_VIEW` | Screen(Search) → ReadModel(Filtered) |
| Delete with Confirm | `STATE_CHANGE` | Screen(Confirm) → Command → Event |

---

## Field Extraction Strategy

### From Form Inputs
```html
<input name="email" type="email" required>
→ { "name": "email", "type": "String", "optional": false }

<input name="quantity" type="number" min="1">
→ { "name": "quantity", "type": "Int" }

<select name="status">
  <option>Draft</option>
  <option>Published</option>
</select>
→ { "name": "status", "type": "String", "example": "Draft|Published" }
```

### From API Responses (Network Tab)
```json
// POST /api/orders response
{ "orderId": "uuid-123", "status": "pending", "total": 99.50 }
→ Event fields:
  { "name": "orderId", "type": "UUID" },
  { "name": "status", "type": "String", "example": "pending" },
  { "name": "total", "type": "Decimal" }
```

### From Displayed Data
```html
<td data-field="createdAt">2024-01-15</td>
→ { "name": "createdAt", "type": "Date", "example": "2024-01-15" }
```

---

## Specification Extraction from UI Behavior

### Example: Add to Cart Flow
```
OBSERVED:
1. User views product page (product displayed)
2. User clicks "Add to Cart" button
3. Cart icon shows updated count
4. Toast: "Item added to cart"

EXTRACTED SPECIFICATION:
{
  "title": "Add item to cart",
  "given": [
    { "title": "Product ABC exists", "type": "SPEC_EVENT", "id": "evt-product-created" }
  ],
  "when": [
    { "title": "Add to Cart clicked", "type": "SPEC_COMMAND", "id": "cmd-add-to-cart" }
  ],
  "then": [
    { "title": "Item Added to Cart", "type": "SPEC_EVENT", "id": "evt-item-added" },
    { "title": "Cart displays updated count", "type": "SPEC_READMODEL", "id": "rm-cart-summary" }
  ]
}
```

---

## File Structure
```
analysis/
├── ui-high-level-analysis.json   # State 1
├── ui-flows.json                  # State 2
├── ui-config.json                 # State 4: detailed model
└── flow-1/
    ├── ui-high-level-analysis.json
    ├── ui-flows.json
    ├── ui-config.json
    └── sub-flow-1/...
```

---

## Example Output (High-Level)

```json
{
  "slices": [
    {
      "id": "view-products", "title": "View Products", "sliceType": "STATE_VIEW",
      "status": "Created", "index": 1, "context": "Display product catalog",
      "readmodels": [{ "id": "rm-products", "title": "Product List", "type": "READMODEL", "fields": [], "dependencies": [] }],
      "commands": [], "events": [], "screens": [], "processors": [], "specifications": [], "aggregates": ["Product"]
    },
    {
      "id": "add-to-cart", "title": "Add to Cart", "sliceType": "STATE_CHANGE",
      "status": "Created", "index": 2, "context": "User adds item to shopping cart",
      "commands": [{ "id": "cmd-add-to-cart", "title": "Add to Cart", "type": "COMMAND", "fields": [], "dependencies": [] }],
      "events": [{ "id": "evt-item-added", "title": "Item Added to Cart", "type": "EVENT", "fields": [], "dependencies": [] }],
      "readmodels": [], "screens": [], "processors": [], "specifications": [], "aggregates": ["Cart"]
    },
    {
      "id": "process-payment", "title": "Process Payment", "sliceType": "AUTOMATION",
      "status": "Created", "index": 3, "context": "Background payment processing",
      "processors": [{ "id": "proc-payment", "title": "Payment Processor", "type": "AUTOMATION", "fields": [], "dependencies": [] }],
      "commands": [{ "id": "cmd-charge", "title": "Charge Payment", "type": "COMMAND", "fields": [], "dependencies": [] }],
      "events": [{ "id": "evt-charged", "title": "Payment Charged", "type": "EVENT", "fields": [], "dependencies": [] }],
      "readmodels": [], "screens": [], "specifications": [], "aggregates": ["Payment"]
    }
  ]
}
```

**❌ WRONG:** Single slice, `sliceType: "PROCESS"`, `status: "open"`, empty elements
**✅ CORRECT:** Multiple slices, valid enums, elements with empty fields

---

## Tips for UI Exploration

1. **Start logged out** - capture auth flows first
2. **Use realistic test data** - extract actual field examples
3. **Watch for loading states** - may indicate async/automation
4. **Check error states** - try invalid inputs for validation rules
5. **Note URL changes** - paths often match aggregate/entity patterns
6. **Inspect network for hidden APIs** - background data fetches reveal automations
7. **Look for websocket connections** - real-time updates = automation slices

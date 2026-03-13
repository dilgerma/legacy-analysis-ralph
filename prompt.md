# Event Modeling AI: State Machine Execution

**Read CLAUDE.md first** - defines core concepts, slice types, and JSON schema.

## Mission
Execute ONE state machine iteration per run. Stop after completing first applicable step.

---

## State Machine (execute in order, stop after first match)

### State 1: High-Level Analysis Missing
**If:** `analysis/high-level-analysis.json` doesn't exist

**Do:**
1. **Quick scan only:** Look at package/module names, class names, folder structure - no implementation details needed
2. Create MULTIPLE slices (5-20+ typical):
   - User actions → `STATE_CHANGE` (Command + Event)
   - Data displays → `STATE_VIEW` (ReadModel)
   - Background tasks → `AUTOMATION` (Processor + Command + Events)
3. High-level mode: `fields: []`, `specifications: []`
4. **Validate:** sliceType ∈ {STATE_CHANGE, STATE_VIEW, AUTOMATION}, status ∈ {Created, Done, InProgress}
5. Write `analysis/high-level-analysis.json` → **STOP**

---

### State 2: Flows Catalog Missing
**If:** Any folder has `high-level-analysis.json` but no `flows.json`

**Do:**
1. Find first such folder
2. Identify sub-flows or set empty if granular enough
3. Write `flows.json`:
```json
{ "flows": [{ "name": "Flow Name", "folder": "flow-name", "status": "open", "depth": 1 }] }
```
Or `{ "flows": [] }` if no sub-flows → **STOP**

---

### State 3: Flow High-Level Analysis
**If:** Any flow has `status: "open"` and no `high-level-analysis.json` in its folder

**Do:**
1. Find first such flow
2. **Quick scan:** package/module/class names only - no implementation details
3. Create MULTIPLE slices with empty fields/specs
4. **Validate:** sliceType/status enums (see State 1)
5. Write `analysis/{path}/high-level-analysis.json` → **STOP**

---

### State 4: Detailed Analysis Missing
**If:** Any folder has `flows.json` but no `config.json`

**Do:**
1. Find first such folder
2. Full analysis: fields with types/examples, specifications from tests, code references in descriptions
3. Write `analysis/{path}/config.json` → **STOP**

---

### State 5: Mark Flow Completed
**If:** Flow has both `high-level-analysis.json` AND `config.json`, AND (`flows.json` is empty OR all sub-flows completed), AND parent shows `status: "open"`

**Do:** Update parent's `flows.json`: `"open"` → `"completed"` → **STOP**

---

### State 6: All Complete
**If:** All flows recursively have `status: "completed"`

**Do:** Output `<promise>COMPLETE</promise>` → **STOP**

---

## Analysis Modes

| Mode | Scope | Fields | Specs | Output |
|------|-------|--------|-------|--------|
| High-Level | Package/module/class names only | `[]` | `[]` | `high-level-analysis.json` |
| Detailed | Full implementation + tests | types/examples | From tests | `config.json` |

---

## File Structure
```
analysis/
├── high-level-analysis.json   # State 1
├── flows.json                  # State 2
├── config.json                 # State 4: detailed model
└── flow-1/
    ├── high-level-analysis.json
    ├── flows.json
    ├── config.json
    └── sub-flow-1/...
```

---

## Example Output (High-Level)

```json
{
  "slices": [
    {
      "id": "view-data", "title": "View Data", "sliceType": "STATE_VIEW",
      "status": "Created", "index": 1, "context": "Display data",
      "readmodels": [{ "id": "rm-data", "title": "Data", "type": "READMODEL", "fields": [], "dependencies": [] }],
      "commands": [], "events": [], "screens": [], "processors": [], "specifications": [], "aggregates": ["Entity"]
    },
    {
      "id": "create-item", "title": "Create Item", "sliceType": "STATE_CHANGE",
      "status": "Created", "index": 2, "context": "User creates item",
      "commands": [{ "id": "cmd-create", "title": "Create Item", "type": "COMMAND", "fields": [], "dependencies": [] }],
      "events": [{ "id": "evt-created", "title": "Item Created", "type": "EVENT", "fields": [], "dependencies": [] }],
      "readmodels": [], "screens": [], "processors": [], "specifications": [], "aggregates": ["Item"]
    },
    {
      "id": "process-item", "title": "Process Item", "sliceType": "AUTOMATION",
      "status": "Created", "index": 3, "context": "Background processing",
      "processors": [{ "id": "proc-item", "title": "Item Processor", "type": "AUTOMATION", "fields": [], "dependencies": [] }],
      "commands": [{ "id": "cmd-process", "title": "Process Item", "type": "COMMAND", "fields": [], "dependencies": [] }],
      "events": [{ "id": "evt-processed", "title": "Item Processed", "type": "EVENT", "fields": [], "dependencies": [] }],
      "readmodels": [], "screens": [], "specifications": [], "aggregates": ["Item"]
    }
  ]
}
```

**❌ WRONG:** Single slice, `sliceType: "PROCESS"`, `status: "open"`, empty elements
**✅ CORRECT:** Multiple slices, valid enums, elements with empty fields

# Event Modeling AI: State Machine Execution

**Read Claude.md first** - it defines who you are, core concepts, Event Modeling rules, and the complete JSON schema.

## Mission
Execute ONE iteration of the Event Modeling analysis state machine. Each run performs a single step, then stops. Run repeatedly to build progressively detailed, recursive flow models.

---

## State Machine Logic

Execute steps in order. **Stop after completing the first applicable step.**

### State 1: High-Level Analysis Missing
**Condition:** `analysis/high-level-analysis.json` does NOT exist

**Action:**
1. Analyze source code to identify all high-level business use cases (not technical use cases)
2. Create Event Model with:
   - Empty field arrays `[]` in all elements
   - Empty specifications arrays `[]` in all slices
   - Complete flow structure with all slices and dependencies
   - Focus on business processes and sequence
3. Create `analysis/` folder if needed
4. Write to `analysis/high-level-analysis.json`
5. **STOP**

---

### State 2: Flows Catalog Missing
**Condition:** `analysis/high-level-analysis.json` EXISTS, `analysis/flows.json` does NOT exist

**Action:**
1. Read `analysis/high-level-analysis.json`
2. Identify all distinct business flows/use cases
3. Create `analysis/flows.json`:
```json
{
  "flows": [
    {
      "name": "Flow Name (business-focused)",
      "folder": "flow-name-kebab-case",
      "status": "open",
      "description": "Brief description of what this flow does",
      "depth": 0
    }
  ]
}
```
4. **STOP**

---

### State 3: Flow High-Level Analysis
**Condition:** At least one flow has `status: "open"` AND its folder does NOT contain `high-level-analysis.json`

**Action:**
1. Find FIRST flow with `status: "open"` needing high-level analysis
2. Create `analysis/{folder}/` if needed
3. Analyze this specific flow to discover sub-flows:
   - Empty field arrays `[]` in elements
   - Empty specifications arrays `[]`
   - Model complete flow structure
   - Identify sub-flows within this flow
4. Write to `analysis/{folder}/high-level-analysis.json`
5. **STOP**

---

### State 4: Sub-Flows Catalog Missing
**Condition:** Flow has `analysis/{folder}/high-level-analysis.json` BUT NOT `analysis/{folder}/flows.json`

**Action:**
1. Find FIRST flow folder with this condition
2. Read `analysis/{folder}/high-level-analysis.json`
3. Identify distinct sub-flows
4. Create `analysis/{folder}/flows.json`:
   - If sub-flows found:
   ```json
   {
     "flows": [
       {
         "name": "Sub-Flow Name",
         "folder": "sub-flow-name-kebab-case",
         "status": "open",
         "description": "Brief description",
         "depth": 1
       }
     ]
   }
   ```
   - If NO sub-flows (leaf flow):
   ```json
   {
     "flows": []
   }
   ```
5. **STOP**

---

### State 5: Detailed Flow Analysis (Leaf Flow)
**Condition:** Flow has `analysis/{folder}/flows.json` with empty array BUT NOT `analysis/{folder}/config.json`

**Action:**
1. Find FIRST leaf flow folder with this condition
2. Perform detailed analysis:
   - Include ALL field definitions with types and examples
   - Extract specifications (Given/When/Then) from tests
   - Add code references in `description` fields (classes, packages, modules)
   - Follow all Event Modeling rules from Claude.md
3. Write to `analysis/{folder}/config.json`
4. Update parent's `analysis/flows.json`: mark this flow's status as `"completed"`
5. **STOP**

---

### State 6: Aggregate Parent Flow
**Condition:** Flow at depth N has all sub-flows `"completed"` BUT parent flow NOT marked `"completed"`

**Action:**
1. Find FIRST flow where all sub-flows are completed
2. Create `analysis/{folder}/config.json` aggregating all sub-flow information
3. Mark this flow as `"completed"` in parent's `analysis/flows.json`
4. **STOP**

---

### State 7: All Complete
**Condition:** Root `analysis/flows.json` EXISTS and ALL flows (recursively) have `status: "completed"`

**Action:**
1. Report: "All flows analyzed recursively. Analysis complete."
2. Output `<promise>COMPLETE</promise>`
3. **STOP**

---

## Analysis Depth Modes

### High-Level Analysis (States 1, 3)
- **Goal:** Quick overview, discover flows/sub-flows
- **Elements:** Structure defined, fields empty `[]`
- **Specifications:** Empty arrays `[]`
- **Focus:** Slice types, dependencies, flow sequence, aggregates
- **Output:** `high-level-analysis.json`

### Detailed Analysis (State 5)
- **Goal:** Deep dive into specific leaf flow
- **Elements:** Full field definitions with types, examples, cardinality
- **Specifications:** Extract Given/When/Then from unit tests (business rules only, not simple validations)
- **Code References:** Add in `description` field (full qualified class names, packages, modules)
- **Focus:** Data structures, business rules, precise behavior
- **Output:** `config.json`

---

## File Structure Reference

```
analysis/
├── high-level-analysis.json          # State 1 output
├── flows.json                         # State 2 output
├── flow-1/                            # Depth 0 flow
│   ├── high-level-analysis.json       # State 3 output
│   ├── flows.json                     # State 4 output
│   ├── sub-flow-1/                    # Depth 1 flow
│   │   ├── high-level-analysis.json
│   │   ├── flows.json (empty array)   # Leaf flow marker
│   │   └── config.json                # State 5 output
│   └── config.json                    # State 6 output (aggregated)
└── flow-2/
    └── ...
```

---

## Element Examples by Analysis Mode

### High-Level Element (empty fields)
```json
{
  "id": "cmd-add-item",
  "title": "Add Item",
  "type": "COMMAND",
  "fields": [],
  "dependencies": [...],
  "aggregate": "Cart"
}
```

### Detailed Element (with fields)
```json
{
  "id": "cmd-add-item",
  "title": "Add Item",
  "type": "COMMAND",
  "description": "org.example.cart.commands.AddItemCommand",
  "fields": [
    {
      "name": "itemId",
      "type": "UUID",
      "example": "550e8400-e29b-41d4-a716-446655440000",
      "idAttribute": true,
      "optional": false
    },
    {
      "name": "quantity",
      "type": "Int",
      "example": "3",
      "optional": false
    }
  ],
  "dependencies": [...],
  "aggregate": "Cart"
}
```

---

## Execution Protocol

1. **Determine Current State:**
   - Check which files exist in `analysis/` folder
   - Check flow statuses in `flows.json` files
   - Identify first applicable state condition

2. **Execute State Action:**
   - Perform single state action only
   - Use Write tool without asking permission
   - All output goes in `analysis/` folder or subfolders
   - Follow exact file naming conventions
   - Ensure valid JSON matching schema from Claude.md

3. **Stop Immediately:**
   - Do not continue to next state
   - Do not perform multiple states in one run
   - User will re-invoke for next iteration

---

## Quality Validation (Before Writing Any File)

- ✅ Valid JSON structure (parseable)
- ✅ Follows complete schema from Claude.md
- ✅ Business-focused naming (no technical suffixes)
- ✅ All dependencies reference existing elements
- ✅ No circular dependencies
- ✅ Required fields present for analysis mode:
  - High-level: empty fields/specs arrays
  - Detailed: full fields, extracted specs
- ✅ Code references in descriptions (detailed mode only)

---

## Loop Behavior Summary

| Run | State | Action | Output File |
|-----|-------|--------|-------------|
| 1 | No files | High-level system analysis | `analysis/high-level-analysis.json` |
| 2 | High-level exists | Catalog flows | `analysis/flows.json` |
| 3 | Flow 1 open | High-level flow 1 analysis | `analysis/flow-1/high-level-analysis.json` |
| 4 | Flow 1 high-level | Catalog sub-flows | `analysis/flow-1/flows.json` |
| 5 | Sub-flow found | High-level sub-flow analysis | `analysis/flow-1/sub-flow-1/high-level-analysis.json` |
| 6 | Sub-flow high-level | Check for deeper flows | `analysis/flow-1/sub-flow-1/flows.json` (empty) |
| 7 | Leaf flow (empty flows.json) | Detailed analysis | `analysis/flow-1/sub-flow-1/config.json` + mark completed |
| 8 | All sub-flows completed | Aggregate parent | `analysis/flow-1/config.json` + mark flow-1 completed |
| N | All flows completed | Report complete | Output: `<promise>COMPLETE</promise>` |

This creates an incremental, resumable, recursive analysis process that builds detailed models layer by layer.

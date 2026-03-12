# Event Modeling AI: State Machine Execution

**Read Claude.md first** - it defines who you are, core concepts, Event Modeling rules, and the complete JSON schema.

## Mission
Execute ONE iteration of the Event Modeling analysis state machine. Each run performs a single step, then stops. Run repeatedly to build progressively detailed, recursive flow models.

---

## State Machine Logic

Execute steps in order. **Stop after completing the first applicable step.**

The state machine searches **recursively** through all flow folders at all depths.

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
**Condition:** ANY folder with `high-level-analysis.json` but NO `flows.json` (search recursively)

**Action:**
1. Find FIRST folder (at any depth) with `high-level-analysis.json` but no `flows.json`
2. Read the `high-level-analysis.json` from that folder
3. Identify distinct sub-flows/operations within this scope
4. Create `flows.json` in that folder:
   - If sub-flows identified:
   ```json
   {
     "flows": [
       {
         "name": "Sub-Flow Name (business-focused)",
         "folder": "sub-flow-name-kebab-case",
         "status": "open",
         "description": "Brief description",
         "depth": 1
       }
     ]
   }
   ```
   Note: Set `depth` to parent's depth + 1
   - If this is granular enough (no further breakdown needed):
   ```json
   {
     "flows": []
   }
   ```
5. **STOP**

---

### State 3: Flow High-Level Analysis
**Condition:** ANY flow has `status: "open"` AND its folder does NOT contain `high-level-analysis.json` (search recursively through all flows.json files)

**Action:**
1. Recursively search all `flows.json` files at all depths
2. Find FIRST flow with `status: "open"` that doesn't have `high-level-analysis.json` in its folder
3. Create the folder path if needed (e.g., `analysis/flow-1/sub-flow-2/`)
4. Analyze this specific flow to discover its internal structure:
   - Empty field arrays `[]` in elements
   - Empty specifications arrays `[]`
   - Model complete flow structure with slices and dependencies
   - Identify potential sub-flows within this flow
5. Write to `analysis/{path}/high-level-analysis.json`
6. **STOP**

---

### State 4: Detailed Analysis Missing
**Condition:** ANY folder with `flows.json` but NO `config.json` (search recursively)

**Action:**
1. Find FIRST folder (at any depth) with `flows.json` but no `config.json`
2. Perform detailed analysis at this level:
   - Include ALL field definitions with types and examples
   - Extract specifications (Given/When/Then) from tests (business rules only)
   - Add code references in `description` fields (classes, packages, modules)
   - Follow all Event Modeling rules from Claude.md
   - Create a complete, visualizable event model
3. Write to `analysis/{path}/config.json`
4. **STOP**

---

### State 5: Mark Flow Completed
**Condition:** A flow has both `high-level-analysis.json` AND `config.json` in its folder, AND either:
  - Has `flows.json` with empty array `[]`, OR
  - Has `flows.json` with sub-flows where ALL sub-flows are marked `"completed"`

AND the flow's status in parent's `flows.json` is still `"open"`

**Action:**
1. Find FIRST flow meeting this condition
2. Update parent's `flows.json`: change this flow's status from `"open"` to `"completed"`
3. **STOP**

---

### State 6: All Complete
**Condition:** Root `analysis/flows.json` EXISTS and ALL flows (recursively) have `status: "completed"`

**Action:**
1. Report: "All flows analyzed recursively at all depths. Analysis complete."
2. Output `<promise>COMPLETE</promise>`
3. **STOP**

---

## Analysis Depth Modes

### High-Level Analysis (States 1, 3)
- **Goal:** Quick structural overview, discover sub-flows
- **Elements:** Structure defined, fields empty `[]`
- **Specifications:** Empty arrays `[]`
- **Focus:** Slice types, dependencies, flow sequence, aggregates, identifying sub-flows
- **Output:** `high-level-analysis.json`

### Detailed Analysis (State 4)
- **Goal:** Complete visualizable event model at this scope level
- **Elements:** Full field definitions with types, examples, cardinality
- **Specifications:** Extract Given/When/Then from unit tests (business rules only, not simple validations)
- **Code References:** Add in `description` field (full qualified class names, packages, modules)
- **Focus:** Data structures, business rules, precise behavior
- **Output:** `config.json`
- **Note:** Created at EVERY level (root, flow-1, sub-flow-1, etc.) for progressive refinement

---

## File Structure Reference

```
analysis/
├── high-level-analysis.json          # State 1: Structural skeleton
├── flows.json                         # State 2: Top-level flow catalog
├── config.json                        # State 4: DETAILED system-wide model (visualizable)
├── flow-1/                            # Depth 1
│   ├── high-level-analysis.json       # State 3: Structural skeleton of flow-1
│   ├── flows.json                     # State 2: Sub-flow catalog
│   ├── config.json                    # State 4: DETAILED flow-1 model (visualizable)
│   ├── sub-flow-1/                    # Depth 2
│   │   ├── high-level-analysis.json   # State 3: Structural skeleton
│   │   ├── flows.json                 # State 2: Sub-sub-flows or []
│   │   └── config.json                # State 4: DETAILED sub-flow-1 model (visualizable)
│   └── sub-flow-2/
│       ├── high-level-analysis.json
│       ├── flows.json
│       └── config.json                # State 4: DETAILED sub-flow-2 model (visualizable)
└── flow-2/
    └── ... (same pattern repeats)
```

**Key Insight:** Every folder gets a `config.json` with a complete, detailed event model that can be visualized independently. Deeper levels provide more granular views of the same domain.

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
| 1 | State 1 | System-wide structural skeleton | `analysis/high-level-analysis.json` |
| 2 | State 2 | Catalog top-level flows | `analysis/flows.json` |
| 3 | State 4 | **Detailed system-wide model** | `analysis/config.json` ⭐ |
| 4 | State 3 | Flow-1 structural skeleton | `analysis/flow-1/high-level-analysis.json` |
| 5 | State 2 | Catalog flow-1 sub-flows | `analysis/flow-1/flows.json` |
| 6 | State 4 | **Detailed flow-1 model** | `analysis/flow-1/config.json` ⭐ |
| 7 | State 3 | Sub-flow-1 structural skeleton | `analysis/flow-1/sub-flow-1/high-level-analysis.json` |
| 8 | State 2 | Catalog sub-sub-flows (or empty) | `analysis/flow-1/sub-flow-1/flows.json` |
| 9 | State 4 | **Detailed sub-flow-1 model** | `analysis/flow-1/sub-flow-1/config.json` ⭐ |
| 10 | State 5 | Mark sub-flow-1 completed | Update `analysis/flow-1/flows.json` |
| ... | ... | Continue for all flows recursively | ... |
| N | State 6 | All flows completed | `<promise>COMPLETE</promise>` |

**Progressive Refinement:** Each level gets a complete visualizable event model (config.json). Drill down for increasing detail:
- `analysis/config.json` - System-wide view (high-level operations)
- `analysis/flow-1/config.json` - Detailed view of flow-1
- `analysis/flow-1/sub-flow-1/config.json` - Even more detailed view of sub-flow-1

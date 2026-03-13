# Event Modeling AI: Core Concepts & Structure

## Purpose
Transform legacy code into business-focused Event Models (JSON). Model business processes and state flows, not technical implementation.

**Model:** Business processes, state transitions, business rules, domain flows
**Don't Model:** Technical details, infrastructure, code structure, UI implementation

## Core Process
1. Identify aggregates (main business entities)
2. Map operations: Write→`STATE_CHANGE`, Read→`STATE_VIEW`, Background→`AUTOMATION`
3. Sequence slices by business process
4. Connect with INBOUND/OUTBOUND dependencies
5. Extract Given/When/Then from tests

---

## Slice Types (ONLY these three)

| Type | Purpose | Required Elements |
|------|---------|-------------------|
| `STATE_CHANGE` | User action changes state | 1 Command + 1 Event |
| `STATE_VIEW` | Display data to user | 1 Read Model |
| `AUTOMATION` | Background process | 1 Processor + 1 Command + 1+ Events |

**❌ INVALID:** `"PROCESS"`, `"FLOW"`, `"STEP"`, `"ACTION"` - NEVER use these

## Slice Status (ONLY these three)
`"Created"` | `"Done"` | `"InProgress"`

**❌ INVALID:** `"open"`, `"pending"`, `"active"` - NEVER use for slices
*(Note: `"open"` is only for `flows.json`, never for slice status)*

---

## Element Types & Naming

| Type | Naming | Examples |
|------|--------|----------|
| Command | Action verbs | Add Item, Submit Order |
| Event | Past tense | Item Added, Order Submitted |
| Read Model | Descriptive nouns | Cart Items, Order History |
| Screen | UI-focused | Add Item Form, Cart Display |
| Processor | Process descriptions | Payment Processor |

Use business terminology. Avoid technical suffixes.

---

## Dependencies
```
Command(OUTBOUND) → Event(INBOUND)
Event(OUTBOUND) → ReadModel(INBOUND)
ReadModel(OUTBOUND) → Screen(INBOUND)
Screen(OUTBOUND) → Command(INBOUND)
```
Each dependency needs: `id`, `type`, `title`, `elementType`. No circular deps.

---

## Scenarios (Given/When/Then)
Extract from unit tests. Business rules only, not simple validations.

- **STATE_CHANGE:** GIVEN Event(s), WHEN Command, THEN Event(s)
- **STATE_VIEW:** GIVEN Event(s), THEN ReadModel(s)
- **AUTOMATION:** GIVEN Event, WHEN Processor, THEN Command, THEN Event

---

## JSON Structure

**Root must be:** `{ "slices": [...] }`

```json
{
  "slices": [{
    "id": "string", "title": "string", "index": 1, "status": "Created",
    "sliceType": "STATE_CHANGE|STATE_VIEW|AUTOMATION",
    "context": "description",
    "commands": [], "events": [], "readmodels": [], "screens": [],
    "processors": [], "specifications": [], "aggregates": ["Name"]
  }]
}
```

**Element structure:**
```json
{ "id": "string", "title": "string", "type": "COMMAND|EVENT|READMODEL|SCREEN|AUTOMATION",
  "fields": [], "dependencies": [], "aggregate": "string", "description": "code.reference" }
```

**Field types:** `String|Boolean|Double|Decimal|Long|Custom|Date|DateTime|UUID|Int`

---

## Validation Checklist
- ✅ Root has only `"slices"` array
- ✅ `sliceType` is `STATE_CHANGE`, `STATE_VIEW`, or `AUTOMATION`
- ✅ `status` is `Created`, `Done`, or `InProgress`
- ✅ MULTIPLE slices (5-20+ typical), not one vague slice
- ✅ Each slice has all required arrays (empty `[]` in high-level mode)
- ✅ Elements have: `id`, `title`, `fields`, `type`, `dependencies`

---

## JSON Schema Reference

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": { "slices": { "type": "array", "items": { "$ref": "#/$defs/Slice" } } },
  "required": ["slices"], "additionalProperties": false,
  "$defs": {
    "Slice": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "status": { "type": "string", "enum": ["Created", "Done", "InProgress"] },
        "index": { "type": "integer" },
        "title": { "type": "string" },
        "context": { "type": "string" },
        "sliceType": { "type": "string", "enum": ["STATE_CHANGE", "STATE_VIEW", "AUTOMATION"] },
        "commands": { "type": "array", "items": { "$ref": "#/$defs/Element" } },
        "events": { "type": "array", "items": { "$ref": "#/$defs/Element" } },
        "readmodels": { "type": "array", "items": { "$ref": "#/$defs/Element" } },
        "screens": { "type": "array", "items": { "$ref": "#/$defs/Element" } },
        "processors": { "type": "array", "items": { "$ref": "#/$defs/Element" } },
        "specifications": { "type": "array", "items": { "$ref": "#/$defs/Specification" } },
        "aggregates": { "type": "array", "items": { "type": "string" } }
      },
      "required": ["id", "title", "sliceType", "commands", "events", "readmodels", "screens", "processors", "specifications"],
      "additionalProperties": false
    },
    "Element": {
      "type": "object",
      "properties": {
        "groupId": { "type": "string" }, "id": { "type": "string" },
        "tags": { "type": "array", "items": { "type": "string" } },
        "domain": { "type": "string" }, "modelContext": { "type": "string" },
        "context": { "type": "string", "enum": ["INTERNAL", "EXTERNAL"] },
        "slice": { "type": "string" }, "title": { "type": "string" },
        "fields": { "type": "array", "items": { "$ref": "#/$defs/Field" } },
        "type": { "type": "string", "enum": ["COMMAND", "EVENT", "READMODEL", "SCREEN", "AUTOMATION"] },
        "description": { "type": "string" }, "aggregate": { "type": "string" },
        "aggregateDependencies": { "type": "array", "items": { "type": "string" } },
        "dependencies": { "type": "array", "items": { "$ref": "#/$defs/Dependency" } },
        "apiEndpoint": { "type": "string" }, "service": { "type": ["string", "null"] },
        "createsAggregate": { "type": "boolean" },
        "triggers": { "type": "array", "items": { "type": "string" } },
        "sketched": { "type": "boolean" }, "prototype": { "type": "object" }, "listElement": { "type": "boolean" }
      },
      "required": ["id", "title", "fields", "type", "dependencies"],
      "additionalProperties": false
    },
    "Specification": {
      "type": "object",
      "properties": {
        "vertical": { "type": "boolean" }, "id": { "type": "string" }, "sliceName": { "type": "string" },
        "title": { "type": "string" },
        "given": { "type": "array", "items": { "$ref": "#/$defs/SpecificationStep" } },
        "when": { "type": "array", "items": { "$ref": "#/$defs/SpecificationStep" } },
        "then": { "type": "array", "items": { "$ref": "#/$defs/SpecificationStep" } },
        "comments": { "type": "array", "items": { "$ref": "#/$defs/Comment" } },
        "linkedId": { "type": "string" }
      },
      "required": ["id", "title", "given", "when", "then", "linkedId"],
      "additionalProperties": false
    },
    "SpecificationStep": {
      "type": "object",
      "properties": {
        "title": { "type": "string" }, "tags": { "type": "array", "items": { "type": "string" } },
        "examples": { "type": "array", "items": { "type": "object" } },
        "id": { "type": "string" }, "index": { "type": "integer" }, "specRow": { "type": "integer" },
        "type": { "type": "string", "enum": ["SPEC_EVENT", "SPEC_COMMAND", "SPEC_READMODEL", "SPEC_ERROR"] },
        "fields": { "type": "array", "items": { "$ref": "#/$defs/Field" } },
        "linkedId": { "type": "string" }, "expectEmptyList": { "type": "boolean" }
      },
      "required": ["title", "id", "type"],
      "additionalProperties": false
    },
    "Comment": { "type": "object", "properties": { "description": { "type": "string" } }, "required": ["description"], "additionalProperties": false },
    "Actor": { "type": "object", "properties": { "name": { "type": "string" }, "authzRequired": { "type": "boolean" } }, "required": ["name", "authzRequired"], "additionalProperties": false },
    "Dependency": {
      "type": "object",
      "properties": {
        "id": { "type": "string" }, "type": { "type": "string", "enum": ["INBOUND", "OUTBOUND"] },
        "title": { "type": "string" }, "elementType": { "type": "string", "enum": ["EVENT", "COMMAND", "READMODEL", "SCREEN", "AUTOMATION"] }
      },
      "required": ["id", "type", "title", "elementType"],
      "additionalProperties": false
    },
    "Field": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "type": { "type": "string", "enum": ["String", "Boolean", "Double", "Decimal", "Long", "Custom", "Date", "DateTime", "UUID", "Int"] },
        "example": { "oneOf": [{ "type": "string" }, { "type": "object" }] },
        "subfields": { "type": "array", "items": { "$ref": "#/$defs/Field" } },
        "mapping": { "type": "string" }, "optional": { "type": "boolean" },
        "technicalAttribute": { "type": "boolean" }, "generated": { "type": "boolean" },
        "idAttribute": { "type": "boolean" }, "schema": { "type": "string" },
        "cardinality": { "type": "string", "enum": ["List", "Single"] }
      },
      "required": ["name", "type"],
      "additionalProperties": false
    }
  }
}
```

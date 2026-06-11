# Mermaid Syntax Validation Checklist

Use this checklist to validate any Mermaid diagram before output. Walk through each section relevant to the diagram type.

## Universal Rules (All Diagram Types)

### Structure

- [ ] Diagram starts with a valid directive on the first line (no leading whitespace)
- [ ] No content before the diagram directive
- [ ] No trailing content after diagram ends (except comments `%%`)
- [ ] All `subgraph` blocks have matching `end` keywords
- [ ] No nested subgraphs deeper than 3 levels (rendering degrades)

### Node IDs

- [ ] IDs are unique within the diagram
- [ ] IDs do not start with a number
- [ ] IDs do not use reserved keywords: `end`, `graph`, `subgraph`, `style`, `class`, `click`, `callback`, `classDef`, `linkStyle`, `direction`
- [ ] IDs use only alphanumeric characters, underscores, or hyphens
- [ ] IDs are descriptive (not single letters like `A`, `B`, `C`)

### Labels

- [ ] Labels with special characters are wrapped in double quotes: `node["My Label (v2)"]`
- [ ] Characters requiring escaping: `(`, `)`, `[`, `]`, `{`, `}`, `<`, `>`, `"`, `#`, `&`
- [ ] No unmatched brackets inside labels
- [ ] Labels do not contain raw HTML unless using `%%{init:}%%` config

### Edges / Arrows

- [ ] Arrow syntax is valid for the diagram type
- [ ] Edge labels use correct syntax: `-->|label|` (both pipes present)
- [ ] No spaces inside arrow operators (`-- >` is invalid, use `-->`)
- [ ] Directional consistency (avoid circular references unless intentional)

### Styling

- [ ] `classDef` declarations appear before their usage
- [ ] `style` targets existing node IDs
- [ ] Color values are valid hex (`#ff0000`) or named CSS colors
- [ ] `linkStyle` index matches actual link order (0-based)

---

## Flowchart Specific

```
flowchart TD|LR|BT|RL
```

- [ ] Uses `flowchart` (not deprecated `graph`)
- [ ] Direction is one of: `TD`, `TB`, `LR`, `RL`, `BT`
- [ ] Node shapes are consistent:
  - `[text]` — rectangle (process)
  - `(text)` — rounded rectangle
  - `{text}` — rhombus / diamond (decision)
  - `[(text)]` — cylinder (database/storage)
  - `([text])` — stadium (start/end)
  - `[[text]]` — subroutine
  - `((text))` — circle
  - `>text]` — asymmetric / flag
  - `{{text}}` — hexagon
- [ ] Decision nodes (`{}`) have labeled outgoing edges (`-->|Yes|`, `-->|No|`)
- [ ] Subgraph titles are meaningful: `subgraph authFlow[Authentication Flow]`

---

## Sequence Diagram Specific

```
sequenceDiagram
```

- [ ] `participant` or `actor` declarations appear before first usage
- [ ] Arrow types used correctly:
  - `->>` solid with arrowhead (synchronous)
  - `-->>` dashed with arrowhead (response/async)
  - `-x` solid with cross (lost/failed)
  - `--x` dashed with cross
  - `-)` solid with open arrow (async fire-and-forget)
- [ ] `activate` / `deactivate` pairs match (or use `+`/`-` shorthand)
- [ ] `alt`, `opt`, `loop`, `par`, `critical`, `break` blocks have matching `end`
- [ ] `Note` syntax: `Note right of Participant: text` (not `Notes`)
- [ ] No duplicate participant aliases
- [ ] Messages are concise action descriptions, not implementation details

---

## Class Diagram Specific

```
classDiagram
```

- [ ] Visibility modifiers used correctly: `+` public, `-` private, `#` protected, `~` package
- [ ] Methods include return type: `+getName() String`
- [ ] Relationships use correct notation:
  - `<|--` inheritance
  - `*--` composition
  - `o--` aggregation
  - `-->` association
  - `..>` dependency
  - `..|>` implementation/realization
- [ ] Cardinality is specified: `"1" --> "*"`
- [ ] `<<interface>>`, `<<abstract>>`, `<<enumeration>>` annotations used where appropriate
- [ ] Class names are PascalCase

---

## ER Diagram Specific

```
erDiagram
```

- [ ] Relationship notation (NOT arrows):
  - `||` exactly one
  - `o|` zero or one
  - `}|` one or more
  - `}o` zero or more
- [ ] Relationship format: `ENTITY1 ||--o{ ENTITY2 : "relationship label"`
- [ ] Relationship labels are in quotes if multi-word
- [ ] Entity names are UPPER_CASE or PascalCase (consistent)
- [ ] Attributes include type and constraint:
  ```
  ENTITY {
      type name constraint
      string id PK
      string foreignId FK
      string uniqueField UK
  }
  ```
- [ ] `PK`, `FK`, `UK` constraints are marked
- [ ] No arrow syntax (`-->`) used — ER diagrams use relationship notation only

---

## C4 Diagram Specific

```
C4Context | C4Container | C4Component | C4Deployment
```

- [ ] Correct diagram level selected:
  - `C4Context` — system context (people + systems)
  - `C4Container` — containers within a system
  - `C4Component` — components within a container
  - `C4Deployment` — deployment nodes
- [ ] Element functions used correctly:
  - `Person(alias, "Label", "Description")`
  - `System(alias, "Label", "Description")`
  - `System_Ext(alias, "Label", "Description")`
  - `Container(alias, "Label", "Technology", "Description")`
  - `Component(alias, "Label", "Technology", "Description")`
  - `Rel(from, to, "Label", "Technology")`
- [ ] `_Ext` suffix used for external systems/containers
- [ ] `_Boundary` used for grouping: `System_Boundary(alias, "Label")`
- [ ] `title` is set
- [ ] Descriptions are business-oriented, not technical jargon

---

## State Diagram Specific

```
stateDiagram-v2
```

- [ ] Uses `stateDiagram-v2` (not v1)
- [ ] Start state: `[*] --> FirstState`
- [ ] End state: `LastState --> [*]`
- [ ] Transition labels: `State1 --> State2 : event`
- [ ] Composite states use proper nesting:
  ```
  state "Composite" as comp {
      [*] --> Inner1
      Inner1 --> Inner2
  }
  ```
- [ ] `<<fork>>` and `<<join>>` used for parallel states
- [ ] `<<choice>>` used for conditional branching

---

## Gantt Chart Specific

```
gantt
```

- [ ] `title` is set
- [ ] `dateFormat` is declared (e.g., `YYYY-MM-DD`)
- [ ] `section` used to group related tasks
- [ ] Task status keywords valid: `done`, `active`, `crit` (critical)
- [ ] Task dependencies use `after taskId` syntax
- [ ] Task durations specified as date range or duration (`30d`, `2w`)
- [ ] Task IDs are unique within the chart
- [ ] `excludes weekends` if applicable

---

## Mindmap Specific

```
mindmap
```

- [ ] Uses **spaces** for indentation (NOT tabs)
- [ ] Consistent indent increment (recommended: 4 spaces per level)
- [ ] Root node is on the first indented line
- [ ] Node shapes:
  - `Root` — default
  - `(Rounded)` — rounded rectangle
  - `[Square]` — square
  - `((Circle))` — circle
  - `)Cloud(` — cloud
  - `{{Hexagon}}` — hexagon
- [ ] Maximum depth: 5-6 levels (deeper is unreadable)

---

## Pie Chart Specific

```
pie
```

- [ ] `title` is set
- [ ] Values are positive numbers
- [ ] Labels are in quotes: `"Category" : 45`
- [ ] Values represent meaningful proportions
- [ ] No more than 7-8 slices (readability limit)

---

## Common Error Messages & Fixes

| Error                   | Cause                            | Fix                                                  |
| ----------------------- | -------------------------------- | ---------------------------------------------------- |
| "Parse error on line X" | Syntax error at that line        | Check brackets, quotes, arrows on that line          |
| "Duplicate node"        | Same ID used twice               | Rename one instance                                  |
| "Unknown diagram type"  | Misspelled directive             | Check spelling: `flowchart`, `sequenceDiagram`, etc. |
| "Expecting 'end'"       | Missing `end` for subgraph/block | Count opening and closing blocks                     |
| "Invalid arrow"         | Wrong arrow for diagram type     | Consult arrow syntax for the specific diagram type   |
| Diagram renders blank   | Empty or whitespace-only content | Ensure nodes and edges are defined                   |
| Labels show raw text    | Special chars not escaped        | Wrap labels in double quotes                         |

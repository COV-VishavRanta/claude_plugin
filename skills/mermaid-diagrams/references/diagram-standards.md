# Mermaid Diagram Industry Standards

Standards and conventions for each diagram type. Apply these when creating or reviewing Mermaid diagrams.

---

## General Standards (All Diagrams)

### Layout & Direction

- **Hierarchical / dependency flows** → `TD` (top-down)
- **Process / timeline / data flows** → `LR` (left-right)
- **Bottom-up call chains** → `BT` (bottom-top)
- Keep diagrams **max 25 nodes** — split into multiple diagrams if larger

### Naming

- Node IDs: `camelCase` or `snake_case`, descriptive (`apiGateway`, `userService` — never `A`, `B`, `n1`)
- Subgraph IDs: `camelCase` with descriptive title: `subgraph authLayer["Authentication Layer"]`
- Labels: Title Case for nodes, sentence case for edge labels

### Edges

- **Always label edges** when the relationship type isn't immediately obvious
- Use consistent arrow styles within a diagram (don't mix `-->` and `==>` without semantic reason)
- Thick arrows (`==>`) for primary/critical paths; dashed (`-.->`) for optional/async

### Subgraphs

- Group logically related nodes (by layer, domain, or deployment boundary)
- Limit nesting to **2 levels max** (subgraph within subgraph)
- Always provide a human-readable title

### Styling

- Use `classDef` sparingly — only when color adds semantic value
- Suggested semantic classes:
  - `external` — gray fill for external/third-party systems
  - `database` — blue fill for data stores
  - `critical` — red border for critical path components
  - `async` — dashed border for async/event-driven components

---

## Flowchart Standards

```mermaid
flowchart TD
```

### Node Shape Conventions

| Shape         | Syntax     | Use For                            |
| ------------- | ---------- | ---------------------------------- |
| Rectangle     | `[text]`   | Process, service, component        |
| Rounded       | `(text)`   | Start/end, intermediate state      |
| Stadium       | `([text])` | Terminal / trigger / event         |
| Diamond       | `{text}`   | Decision / condition               |
| Cylinder      | `[(text)]` | Database, data store, cache        |
| Hexagon       | `{{text}}` | Preparation, setup step            |
| Parallelogram | `[/text/]` | Input / output                     |
| Subroutine    | `[[text]]` | Predefined process / external call |
| Circle        | `((text))` | Connector / junction point         |

### Best Practices

- Decision nodes must have **all outgoing edges labeled** (`-->|Yes|`, `-->|No|`)
- Terminal nodes (start/end) use stadium shape `([text])`
- Keep max **one decision per vertical level** for readability
- Cross-subgraph edges should be minimized

---

## Sequence Diagram Standards

```mermaid
sequenceDiagram
```

### Participant Conventions

- Declare all participants at the top before interactions
- Use `participant` for systems/services, `actor` for humans
- Use aliases for long names: `participant API as API Gateway`
- Order participants left-to-right matching the logical flow (caller → callee)

### Message Conventions

- Synchronous request: `->>` (solid arrow)
- Asynchronous response: `-->>` (dashed arrow)
- Self-call: `Participant->>Participant: action`
- Messages should be **verb phrases**: "Validate token", "Query user", "Return 200 OK"

### Activation

- Use `activate`/`deactivate` for long-running operations
- Shorthand `+`/`-` on arrows for inline activation

### Grouping

- `alt` / `else` for conditional branches
- `opt` for optional flows
- `loop` for repeated operations
- `par` for parallel execution
- `critical` for non-interruptible sequences
- Every block must end with `end`

### Best Practices

- Max **15 interactions** per diagram — split into sub-sequences for complex flows
- Include error/failure paths using `alt` blocks
- Show response codes in return messages: `-->>Client: 201 Created`
- Note important side effects: `Note right of DB: Triggers audit log`

---

## Class Diagram Standards

```mermaid
classDiagram
```

### Class Conventions

- Class names: **PascalCase**
- Attributes: `visibility type name` format (`+String email`)
- Methods: `visibility name(params) returnType` format (`+login(credentials) Boolean`)
- Visibility: `+` public, `-` private, `#` protected, `~` package

### Relationship Conventions

| Relationship | Syntax  | Meaning                       |
| ------------ | ------- | ----------------------------- |
| Inheritance  | `<\|--` | "is a"                        |
| Composition  | `*--`   | "owns" (lifecycle dependent)  |
| Aggregation  | `o--`   | "has" (independent lifecycle) |
| Association  | `-->`   | "uses" / "references"         |
| Dependency   | `..>`   | "depends on" (transient)      |
| Realization  | `..\|>` | "implements"                  |

### Best Practices

- Always specify **cardinality**: `"1" --> "*"` Order
- Use `<<interface>>`, `<<abstract>>`, `<<enum>>` annotations
- Group related classes with `namespace` blocks
- Max **10 classes** per diagram — split by domain/module for larger models

---

## ER Diagram Standards

```mermaid
erDiagram
```

### Entity Conventions

- Entity names: **UPPER_CASE** or **PascalCase** (pick one, be consistent)
- Always include attribute blocks with type, name, and constraints

### Attribute Format

```
ENTITY {
    type name constraint
    string id PK
    string email UK
    string orderId FK
    datetime createdAt
}
```

### Relationship Notation

| Symbol | Meaning      |
| ------ | ------------ |
| `\|\|` | Exactly one  |
| `o\|`  | Zero or one  |
| `}\|`  | One or more  |
| `}o`   | Zero or more |

### Relationship Format

```
ENTITY_A ||--o{ ENTITY_B : "relationship label"
```

### Best Practices

- Always label relationships with a verb phrase: `"places"`, `"contains"`, `"belongs to"`
- Mark all PKs, FKs, and unique constraints
- Include audit columns (`createdAt`, `updatedAt`) on every entity
- Max **12 entities** per diagram — split into domain-bounded diagrams for larger schemas

---

## C4 Diagram Standards

### Level Selection

| Level      | Directive      | Shows                         |
| ---------- | -------------- | ----------------------------- |
| Context    | `C4Context`    | People + systems              |
| Container  | `C4Container`  | Containers within a system    |
| Component  | `C4Component`  | Components within a container |
| Deployment | `C4Deployment` | Infrastructure nodes          |

### Element Conventions

- Always include `title`
- Use **business language** in descriptions, not technical jargon
- External systems use `_Ext` suffix: `System_Ext`, `Container_Ext`
- Group with boundaries: `System_Boundary`, `Container_Boundary`

### Relationship Format

```
Rel(source, target, "Description", "Protocol/Technology")
```

### Best Practices

- Include **protocol labels** on all relationships: `"HTTPS"`, `"gRPC"`, `"AMQP"`
- External systems should be visually distinct (use `_Ext` variants)
- Max **10 elements** per diagram at Context level, **15** at Container/Component
- Always provide the 4th argument (technology) in `Rel()` calls

---

## State Diagram Standards

```mermaid
stateDiagram-v2
```

### Conventions

- Always use `stateDiagram-v2` (v1 is deprecated)
- Start: `[*] --> FirstState`
- End: `FinalState --> [*]`
- Transitions: `State1 --> State2 : event_name`
- Composite states for nested lifecycles

### Best Practices

- Label **every transition** with the triggering event
- Use `<<choice>>` for conditional branching
- Use `<<fork>>` / `<<join>>` for parallel states
- Max **12 states** per diagram

---

## Gantt Chart Standards

```mermaid
gantt
```

### Required Elements

- `title` — descriptive project/phase name
- `dateFormat` — always declare (typically `YYYY-MM-DD`)
- `section` — group related tasks

### Task Format

```
Task Name    :status, id, start, duration_or_end
```

### Status Keywords

- `done` — completed
- `active` — in progress
- `crit` — critical path

### Best Practices

- Use `after taskId` for dependencies
- Include `excludes weekends` when applicable
- Keep to **max 20 tasks** per chart
- Use sections to separate phases/sprints

---

## Mindmap Standards

```mermaid
mindmap
```

### Conventions

- **Spaces only** — no tabs (whitespace-sensitive)
- 4 spaces per indent level
- Root node at first indented line

### Best Practices

- Max **5 levels** deep
- Balance branches (no single branch with 10+ items while others have 2)
- Use node shapes to distinguish categories

---

## Pie Chart Standards

```mermaid
pie
```

### Conventions

- Always include `title`
- Labels in quotes: `"Category" : value`
- Values are positive numbers

### Best Practices

- Max **7 slices** (combine small categories into "Other")
- Order slices largest to smallest
- Use `showData` directive when exact values matter

---
name: mermaid-diagrams
description: "Create, update, validate, and fix Mermaid diagrams to industry standards. Use when: generating architecture diagrams, creating flowcharts, producing sequence diagrams, building ER diagrams, designing class diagrams, making C4 diagrams, creating state diagrams, generating Gantt charts, drawing pie charts, building mindmaps, fixing Mermaid syntax errors, reviewing diagram quality, standardizing diagram output."
argument-hint: "Describe the diagram to create or paste existing Mermaid code to validate/fix"
---

# Mermaid Diagram Standards & Validation

## Purpose

Ensure all Mermaid diagrams produced by agents follow industry-standard conventions, are syntactically correct, visually clear, and consistently styled. This skill is the single source of truth for Mermaid output quality.

## When to Use

- An agent generates or updates a Mermaid diagram (auto-invoked)
- A user asks to create, review, or fix a Mermaid diagram (slash command)
- A diagram has syntax errors or rendering issues
- Diagram output needs to conform to team/project standards

## Procedure

### Step 1 — Identify Diagram Type

Determine the correct Mermaid diagram type for the use case:

| Use Case                               | Diagram Type | Mermaid Directive                           |
| -------------------------------------- | ------------ | ------------------------------------------- |
| Process flow, decision logic           | Flowchart    | `flowchart TD` / `flowchart LR`             |
| API calls, component interaction       | Sequence     | `sequenceDiagram`                           |
| Object model, domain model             | Class        | `classDiagram`                              |
| Lifecycle, FSM                         | State        | `stateDiagram-v2`                           |
| Database schema, data model            | ER           | `erDiagram`                                 |
| System context, containers, components | C4           | `C4Context` / `C4Container` / `C4Component` |
| Project timeline, sprints              | Gantt        | `gantt`                                     |
| Distribution, proportions              | Pie          | `pie`                                       |
| Brainstorming, topic breakdown         | Mindmap      | `mindmap`                                   |
| Git branching strategy                 | Gitgraph     | `gitGraph`                                  |
| User flow, customer experience         | User Journey | `journey`                                   |

### Step 2 — Apply Industry Standards

Follow the conventions in [./references/diagram-standards.md](./references/diagram-standards.md) for the identified diagram type. Key principles:

1. **Direction**: Use `TD` (top-down) for hierarchical/flow diagrams, `LR` (left-right) for process/timeline diagrams
2. **Node IDs**: Use meaningful camelCase or snake_case IDs — never single letters like `A`, `B`, `C`
3. **Labels**: Human-readable, concise, title-case labels on every node
4. **Edge Labels**: Label edges when the relationship isn't obvious
5. **Grouping**: Use `subgraph` to cluster related nodes logically
6. **Consistent Shapes**: Use appropriate shapes for node types (rectangles for processes, diamonds for decisions, cylinders for databases, etc.)
7. **Color/Styling**: Only apply styles when they add semantic meaning (e.g., red for errors, green for success)

### Step 3 — Validate Syntax

Run through the syntax checklist in [./references/syntax-validation.md](./references/syntax-validation.md). Common issues to catch and fix:

- **Missing diagram declaration** — every block must start with a valid directive (`flowchart TD`, `sequenceDiagram`, etc.)
- **Unescaped special characters** — quotes, parentheses, brackets in labels must be escaped or wrapped in `"double quotes"`
- **Mismatched brackets** — `[`, `(`, `{`, `[[`, `((` must have matching closers
- **Invalid arrow syntax** — use `-->`, `==>`, `-.->`, `--o`, `--x` correctly per diagram type
- **Duplicate node IDs** — each ID must be unique within a diagram
- **Subgraph nesting errors** — ensure all `subgraph` blocks have matching `end` keywords
- **Reserved keywords used as IDs** — avoid `end`, `graph`, `subgraph`, `style`, `class`, `click` as node IDs
- **Missing semicolons in ER diagrams** — ER relationship lines don't use arrows
- **Incorrect participant declarations** — in sequence diagrams, declare participants before use
- **Indentation in mindmaps** — mindmap hierarchy is whitespace-sensitive

### Step 4 — Review & Improve

After validation, review for quality:

1. **Readability**: Can a new team member understand this diagram without external context?
2. **Completeness**: Are all relevant components, relationships, and flows represented?
3. **Simplicity**: Remove noise — collapse trivial nodes, remove redundant edges
4. **Consistency**: Does it match the style of other diagrams in the project?
5. **Accessibility**: Avoid relying solely on color to convey meaning

### Step 5 — Output Format

Always output Mermaid diagrams in a fenced code block:

````
```mermaid
<diagram content>
```
````

If fixing an existing diagram, show **before** and **after** with a brief explanation of what changed and why.

## Quick Syntax Reference

### Flowchart

```
flowchart TD
    startNode[Start Process] --> decisionNode{Is Valid?}
    decisionNode -->|Yes| processNode[Process Data]
    decisionNode -->|No| errorNode[Handle Error]
    processNode --> endNode([End])
```

### Sequence Diagram

```
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant DB as Database

    Client->>API: POST /resource
    API->>DB: INSERT record
    DB-->>API: Success
    API-->>Client: 201 Created
```

### Class Diagram

```
classDiagram
    class User {
        +String id
        +String email
        +login() Boolean
    }
    class Order {
        +String id
        +Date createdAt
        +calculate() Decimal
    }
    User "1" --> "*" Order : places
```

### ER Diagram

```
erDiagram
    USER ||--o{ ORDER : places
    ORDER ||--|{ LINE_ITEM : contains
    PRODUCT ||--o{ LINE_ITEM : "included in"

    USER {
        string id PK
        string email
        string name
    }
    ORDER {
        string id PK
        string userId FK
        date createdAt
    }
```

### C4 Context

```
C4Context
    title System Context Diagram

    Person(user, "User", "End user of the system")
    System(app, "Application", "Main application")
    System_Ext(email, "Email Service", "Sends notifications")

    Rel(user, app, "Uses", "HTTPS")
    Rel(app, email, "Sends emails", "SMTP")
```

### State Diagram

```
stateDiagram-v2
    [*] --> Draft
    Draft --> Review : submit
    Review --> Approved : approve
    Review --> Draft : reject
    Approved --> Published : publish
    Published --> [*]
```

### Gantt Chart

```
gantt
    title Project Timeline
    dateFormat YYYY-MM-DD
    section Planning
        Requirements    :done, req, 2024-01-01, 2024-01-15
        Design          :active, des, 2024-01-16, 2024-02-01
    section Development
        Implementation  :dev, after des, 30d
        Testing         :test, after dev, 14d
```

## Error Patterns — Auto-Fix Table

| Error Pattern                       | Fix                                         |
| ----------------------------------- | ------------------------------------------- | --- | ---------------------- | ----- | --- |
| `graph TD` (deprecated)             | Replace with `flowchart TD`                 |
| Node ID starts with number          | Prefix with letter: `1server` → `server1`   |
| Unquoted label with special chars   | Wrap in double quotes: `node["Label (v2)"]` |
| `-->                                | `without closing`                           | `   | Add closing pipe: `--> | label | `   |
| `subgraph` without `end`            | Add matching `end` keyword                  |
| Tabs in mindmap indentation         | Convert to spaces (4-space indent)          |
| `erDiagram` with arrow syntax       | Use ER notation: `\|\|--o{`                 |
| Missing `dateFormat` in gantt       | Add `dateFormat YYYY-MM-DD`                 |
| Participant used before declaration | Move `participant` lines to top             |
| Style applied to non-existent node  | Verify node ID exists in diagram            |

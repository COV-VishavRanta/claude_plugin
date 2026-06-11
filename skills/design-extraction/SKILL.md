---
name: design-extraction
description: "Extract UI design details from Figma links or design images for an ADO work item (User Story or Feature). Use when: acquiring design references for frontend planning, extracting layout/color/typography/spacing from Figma, analyzing design images for component structure, feeding design data into implementation plans or feature workflows."
argument-hint: "Provide an ADO work item ID (Story or Feature)"
---

# Design Extraction

## Purpose

Acquire and extract comprehensive UI design details for a given ADO work item. Searches the work item for Figma links or image attachments, then extracts structured design data (layout, components, colors, typography, spacing, states) that calling agents can use for frontend planning.

## When to Use

- An agent needs visual design data to generate a frontend implementation plan
- An agent needs design context for a feature workflow
- The user asks to extract or analyze a design for a story or feature

## Required Tools

| Tool Namespace    | Purpose                                      |
| ----------------- | -------------------------------------------- |
| `ado/*`           | Fetch ADO work item details, attachments     |
| `figma/*`         | Fetch Figma design files, frames, components |
| `AskUserQuestion` | Ask user for missing design references       |

> If `figma/*` tools are unavailable, the skill can still operate via image-based analysis (Step 3 path). If `ado/*` tools are unavailable, the skill cannot proceed — halt immediately.

---

## Input

The calling agent must provide:

| Parameter          | Required | Description                                                                                         |
| ------------------ | -------- | --------------------------------------------------------------------------------------------------- |
| **Work Item ID**   | Yes      | ADO work item ID (User Story or Feature)                                                            |
| **Work Item Data** | No       | Pre-fetched ADO work item fields (if already available from calling agent — avoids duplicate fetch) |

If the calling agent has already fetched the ADO work item, it should pass the work item data to avoid a redundant API call. Otherwise, this skill fetches it in Step 1.

---

## Output

This skill produces a **Design Extraction Report** — a structured summary returned to the calling agent (not written to a file). The calling agent decides how to use the data.

### Output Structure

```
## Design Extraction Report

### Source
- **Type**: Figma / Image / User-provided
- **Reference**: {Figma URL or "Image attachment from ADO" or "Image provided by user"}
- **Confidence**: ✅ High (Figma) / ⚠️ Medium (Image analysis)

### Layout Structure
- Page layout type (single column, sidebar, grid, etc.)
- Section breakdown with hierarchy
- Grid/flex arrangements and spacing values
- Responsive behavior (breakpoints, stacking, hiding)

### UI Elements Inventory
| Element | Type | Location | Details |
|---|---|---|---|
| {e.g., "Create Campaign"} | Button (Primary) | Top-right header | Blue fill, white text, rounded-md |
| {e.g., Campaign list} | DataTable | Main content | 5 columns, sortable headers, pagination |

### Base Components Identified

{List of all base/primitive UI components detected in the design. This enables calling agents to run a component audit against the project's UI library without re-analyzing the design.}

| Base Component | Category | Usage in Design |
|---|---|---|
| Button | Action | Primary CTA, secondary actions, icon buttons |
| Input | Form | Search field, text inputs in forms |
| Select | Form | Dropdown filters, form selects |
| Table | Data Display | Main data listing |
| Card | Data Display | Content cards in grid |
| Toast | Feedback | Success/error notifications |
| Dialog | Feedback | Confirmation modals |
| Calendar | Specialized | Date picker in filter/form |

### Typography
| Usage | Level | Font Size | Font Weight | Line Height | Color |
|---|---|---|---|---|---|
| Page title | H1 | 24px | 700 (Bold) | 32px | #1A1A1A |
| Section heading | H2 | 18px | 600 (Semi) | 24px | #333333 |
| Body text | Body | 14px | 400 (Regular) | 20px | #666666 |
| Labels | Label | 12px | 500 (Medium) | 16px | #999999 |

### Colors & Theming
| Usage | Color | Hex | Opacity | Notes |
|---|---|---|---|---|
| Primary action | Blue | #2563EB | 100% | Buttons, links |
| Background | White | #FFFFFF | 100% | Page background |
| Card background | Gray | #F9FAFB | 100% | Card surfaces |
| Border | Light gray | #E5E7EB | 100% | Dividers, card borders |
| Error | Red | #DC2626 | 100% | Validation errors |

### Spacing & Sizing
| Element | Padding | Margin | Gap | Border Radius | Notes |
|---|---|---|---|---|---|
| Page container | 24px | — | — | — | Main content wrapper |
| Card | 16px | 0 0 16px 0 | — | 8px | Content cards |
| Button (primary) | 8px 16px | — | 8px (icon gap) | 6px | Standard action button |
| Table cell | 12px 16px | — | — | — | Data table cells |

### Interactive States
| Element | State | Visual Change |
|---|---|---|
| Button | Hover | Background darken 10%, cursor pointer |
| Button | Disabled | Opacity 50%, cursor not-allowed |
| Table row | Hover | Background #F3F4F6 |
| Input | Focus | Border #2563EB, ring 2px |
| Input | Error | Border #DC2626, error text below |

### UI States (Page-level)
| State | Description | Visual Treatment |
|---|---|---|
| Loading | Data being fetched | Skeleton placeholders matching layout |
| Empty | No data available | Centered illustration + message + CTA |
| Error | API/network failure | Error banner or inline error message |
| Success | Action completed | Toast notification or inline success |

### Data Display Patterns
| Pattern | Details |
|---|---|
| List/Table | {columns, sorting, pagination style} |
| Cards | {grid layout, card content structure} |
| Forms | {field layout, label position, validation display} |
| Navigation | {tab style, breadcrumb format, menu structure} |

### Icons & Assets
| Icon/Asset | Location | Type | Notes |
|---|---|---|---|
| {e.g., Plus icon} | Create button | Lucide/Heroicons | 16px, white fill |
| {e.g., Search icon} | Filter input | Lucide/Heroicons | 16px, gray-400 |

### Design Notes
- {Any additional observations about the design — animations, transitions, special behaviors}
- {Accessibility considerations visible in the design}
- {Mobile/responsive variants if present}
```

---

## Procedure

### Step 1 — Obtain Work Item Data

**If work item data was passed by the calling agent** → skip to Step 2.

**If only the Work Item ID was provided:**

1. Use `ado/*` tools to fetch the work item by ID.
2. Retrieve: Title, Description (HTML/Markdown body), Acceptance Criteria, Links/Attachments.
3. Confirm the work item type is `User Story`, `Feature`, `Product Backlog Item`, or `Story`. If not, return an error to the calling agent:

> ⚠️ Work item `{ID}` is of type `{actual-type}`. Design extraction is intended for User Stories or Features.

Proceed anyway — the calling agent can decide whether to halt.

---

### Step 2 — Search for Figma Links

Search the work item's **Description**, **Acceptance Criteria**, and **Links/Attachments** for Figma URLs.

**Pattern matching — look for:**

- `https://www.figma.com/design/...`
- `https://www.figma.com/file/...`
- `https://www.figma.com/proto/...`
- `https://figma.com/design/...`
- `https://figma.com/file/...`
- Any URL containing `figma.com`

**If one or more Figma links are found** → proceed to **Step 3a** with the link(s).

**If no Figma links found** → proceed to **Step 2b**.

### Step 2b — Search for Image Attachments

Check the work item's attachments and linked resources for design images:

- File extensions: `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.svg`
- Attachment names containing keywords: `design`, `mockup`, `wireframe`, `ui`, `screen`, `layout`, `figma`

**If image attachments are found** → download/access them and proceed to **Step 3b**.

**If no design references found at all** → proceed to **Step 2c**.

### Step 2c — Ask User for Design Reference

Use `AskUserQuestion` to request a design reference:

- **Question**: "No design reference was found in ADO work item `{ID}` ({Title}). How would you like to provide a design?"
- **Options**:
  - "I have a Figma link" → Ask for the URL, then proceed to **Step 3a**
  - "I will attach/provide a design image" → Ask user to provide the image, then proceed to **Step 3b**
  - "Skip design extraction" → Return a skip result to the calling agent

**If user selects "Skip"** → return this to the calling agent:

```
## Design Extraction Report

### Source
- **Type**: Skipped
- **Reference**: User chose to skip design extraction for work item {ID}
- **Confidence**: N/A

> No design data available. The calling agent should decide how to proceed without design input.
```

---

### Step 3a — Extract Design from Figma

Use `figma/*` tools to fetch design details from the Figma URL.

#### 3a.1 — Fetch the Design File/Frame

1. Parse the Figma URL to extract the file key and any node/frame IDs.
2. Use the appropriate `figma/*` tool to retrieve the design file or specific frame.
3. If the URL points to a specific frame/component, focus extraction on that node and its children.
4. If the URL points to the full file, identify the most relevant frame(s) — look for frames named after the story/feature or the most recently modified frames.

#### 3a.2 — Extract Comprehensive Design Data

From the Figma data, extract **all** of the following:

**Layout & Structure:**

- Auto-layout direction (horizontal/vertical), padding, spacing, alignment
- Frame dimensions (width, height, constraints)
- Layout grids (columns, rows, gutters)
- Nesting hierarchy — parent-child component relationships
- Responsive constraints (fill container, hug contents, fixed)

**Typography:**

- Font family, size, weight, line height, letter spacing
- Text alignment, decoration (underline, strikethrough)
- Text color (fill)
- For each distinct text style used in the design

**Colors & Fills:**

- Background fills (solid, gradient, image)
- Exact hex/rgba values with opacity
- Fill types for each element (solid color, linear gradient, etc.)
- Stroke colors, widths, and dash patterns

**Spacing & Sizing:**

- Padding (top, right, bottom, left) for each container
- Gap between child elements (auto-layout spacing)
- Margin (inferred from frame positioning)
- Border radius (corner radius per corner if different)
- Element dimensions (width, height)

**Effects:**

- Drop shadows (color, offset, blur, spread)
- Inner shadows
- Background blur
- Layer blur

**Components & Variants:**

- Component names and their variant properties
- Component instances and overrides
- Boolean properties (show/hide elements)
- Text overrides

**Interactive States (if prototyping data is available):**

- Hover states, pressed states, disabled states
- Transitions and animations
- Prototype flows and connections

**Icons & Images:**

- Icon names/references used
- Image fills and their aspect ratios
- SVG vector data references

#### 3a.3 — Compile into Output Structure

Organize all extracted data into the **Output Structure** defined above. Set confidence to ✅ High for values directly from Figma.

---

### Step 3b — Extract Design from Image

When the design source is an image (not Figma), analyze it visually.

#### 3b.1 — Analyze the Image

Use the image viewing capability to examine the design image. Extract the same categories as Step 3a.2, but note:

- **Colors**: Sample dominant colors from the image. Provide best-estimate hex values.
- **Typography**: Estimate font sizes, weights, and styles from visual appearance.
- **Spacing**: Estimate pixel values from visual proportions.
- **Components**: Identify UI components by visual pattern matching.

#### 3b.2 — Confidence Marking

All values extracted from images carry ⚠️ INFERRED confidence unless clearly obvious (e.g., a large heading is clearly H1-level).

Mark specific uncertainties:

- Colors: "⚠️ Sampled from image — verify exact hex values"
- Spacing: "⚠️ Estimated from visual — verify with design file"
- Typography: "⚠️ Visual estimate — confirm font family and size"

#### 3b.3 — Compile into Output Structure

Organize into the same **Output Structure**. Set confidence to ⚠️ Medium overall.

---

### Step 4 — Return Results to Calling Agent

Return the complete **Design Extraction Report** to the calling agent. The report is structured text, not a file — the calling agent decides whether and where to persist it.

Include a brief summary at the top:

```
### Summary
- **Design source**: {Figma / Image / Skipped}
- **Components identified**: {count}
- **Unique colors**: {count}
- **Typography styles**: {count}
- **Interactive states documented**: {count}
- **Overall confidence**: {✅ High / ⚠️ Medium}
- **Gaps/Ambiguities**: {count — see Design Notes}
```

---

## Multiple Designs

If multiple Figma links or images are found in the work item:

1. List all found references via `AskUserQuestion`:
   - "Multiple design references found in work item `{ID}`. Which should I extract?"
   - Options: List each found reference + "Extract all"
2. If user selects "Extract all" → run extraction for each and combine results, noting which section came from which design.
3. If user selects one → extract only that reference.

---

## Error Handling

| Scenario                    | Action                                                     |
| --------------------------- | ---------------------------------------------------------- |
| ADO work item not found     | Return error: "Work item `{ID}` not found in ADO"          |
| Figma URL is inaccessible   | Report the error, fall back to asking user for alternative |
| Figma MCP tools unavailable | Fall back to asking user for a design image instead        |
| Image cannot be analyzed    | Report the error, ask user for a Figma link instead        |
| User provides neither       | Return skip result to calling agent                        |

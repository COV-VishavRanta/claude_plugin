---
paths:
  - "**/*.tsx"
  - "**/*.jsx"
  - "**/*.vue"
  - "**/*.svelte"
  - "**/*.css"
  - "**/*.scss"
  - "**/*.module.css"
description: "Industry-standard UI/design-system coding standards for component and layout implementation. Use when: building UI components from Figma designs, implementing design tokens, creating layouts, styling components, applying responsive patterns."
---

# Design System & UI Implementation Standards

## Design Token Architecture

### Token Hierarchy

Follow a three-tier token structure:

1. **Global tokens** — Raw values (colors, sizes, fonts) defined once
2. **Semantic tokens** — Contextual aliases (e.g., `color-primary`, `spacing-md`)
3. **Component tokens** — Scoped overrides per component (e.g., `button-padding`)

### Token Naming Convention

```
{category}-{property}-{variant}-{state}
```

- `color-text-primary`
- `color-bg-surface-hover`
- `spacing-gap-sm`
- `font-size-heading-lg`

### Token Source of Truth

- All design tokens MUST be derived from the Figma design file
- Never hardcode raw color hex values, font sizes, or spacing values inline
- Map every Figma variable/style to a CSS custom property or theme token
- If the design system uses a token config (e.g., `tailwind.config`, theme file), all tokens go there

## Layout Patterns

### Layout vs. Component Distinction

| Type          | Purpose                                      | Examples                                         |
| ------------- | -------------------------------------------- | ------------------------------------------------ |
| **Layout**    | Page-level structure, slot-based composition | `DashboardLayout`, `AuthLayout`, `SidebarLayout` |
| **Component** | Reusable UI element with props               | `Button`, `Card`, `DataTable`                    |

### Responsive Design

- **Mobile-first** — Base styles for smallest viewport, enhance upward
- Use the framework's breakpoint system (Tailwind breakpoints, CSS media queries)
- Test at standard breakpoints: 320px, 576px, 768px, 1024px, 1280px, 1536px, 1920px
- Use fluid typography and spacing where appropriate
- Avoid fixed pixel widths for containers — use `max-width` with percentage fallbacks

### Spacing System

- Use consistent spacing scale (4px/8px base or framework-provided scale)
- Derive all spacing from design tokens — never use arbitrary pixel values
- Use gap-based layouts (Flexbox gap, Grid gap) over margin-based spacing
- Maintain consistent vertical rhythm with a line-height based spacing unit

## Styling Best Practices

### CSS Architecture

- Scope styles to components — avoid global style leaks
- Use the project's chosen styling approach consistently (CSS Modules, Tailwind, styled-components, etc.)
- Follow utility-first patterns when using utility CSS frameworks
- Keep specificity low — avoid `!important` and deep nesting

### Color Usage

- Use semantic color tokens: `var(--color-primary)` not `#3B82F6`
- Ensure sufficient color contrast (WCAG AA minimum: 4.5:1 for text, 3:1 for large text)
- Support dark/light mode through token switching, not conditional classes
- Never assume color alone conveys meaning — pair with icons, text, or patterns

### Typography

- Define a type scale derived from the design system (6-8 sizes max)
- Use relative units (`rem`, `em`) for font sizes
- Set appropriate line heights: 1.25-1.5 for body, 1.1-1.3 for headings
- Limit font weights used per project (typically 2-3: regular, medium, bold)
- Pick the font family from the Figma design link:
  - If typography variables/styles are defined, use those values directly
  - If variables are not defined, refer to the Figma Properties panel to identify the font family, size, weight, and line height

## Accessibility (a11y)

### Baseline Requirements

- All interactive elements must be keyboard navigable
- Use semantic HTML elements (`button`, `nav`, `main`, `article`) over generic `div`/`span`
- Provide accessible labels for all form inputs and interactive elements
- Images must have meaningful `alt` text (or `alt=""` for decorative images)
- Ensure focus indicators are visible and meet contrast requirements
- Use ARIA attributes only when native HTML semantics are insufficient

### Component-Level a11y

- Buttons: Use `<button>` for actions, `<a>` for navigation
- Forms: Associate `<label>` with inputs, provide error messages linked via `aria-describedby`
- Modals: Trap focus, restore focus on close, use `role="dialog"` and `aria-modal`
- Lists/Tables: Use proper semantic elements with headers and labels

## Figma-to-Code Mapping Rules

### Figma Auto-Layout → CSS

| Figma Property           | CSS Equivalent                            |
| ------------------------ | ----------------------------------------- |
| Auto Layout (Horizontal) | `display: flex; flex-direction: row;`     |
| Auto Layout (Vertical)   | `display: flex; flex-direction: column;`  |
| Spacing between items    | `gap: {value}em;`                         |
| Padding                  | `padding: {top} {right} {bottom} {left};` |
| Fill container           | `flex: 1;` or `width: 100%;`              |
| Hug contents             | `width: fit-content;`                     |
| Fixed width/height       | `width: {value}em; height: {value}em;`    |

### Figma Constraints → CSS

| Figma Constraint | CSS Equivalent                                             |
| ---------------- | ---------------------------------------------------------- |
| Left & Right     | `position: absolute; left: 0; right: 0;` or `width: 100%;` |
| Center           | `margin: auto;` or Flex/Grid centering                     |
| Scale            | Percentage-based widths                                    |

### Figma Effects → CSS

| Figma Effect             | CSS Equivalent                                               |
| ------------------------ | ------------------------------------------------------------ |
| Drop Shadow              | `box-shadow: {x}em {y}em {blur}em {spread}em {color};`       |
| Inner Shadow             | `box-shadow: inset {x}em {y}em {blur}em {spread}em {color};` |
| Layer Blur               | `filter: blur({value}em);`                                   |
| Background Blur          | `backdrop-filter: blur({value}em);`                          |
| Corner Radius            | `border-radius: {value}em;`                                  |
| Individual Corner Radius | `border-radius: {tl} {tr} {br} {bl};`                        |

## Quality Checklist

Before marking a component/layout as complete:

- [ ] All design tokens derived from Figma — no hardcoded values
- [ ] Component matches Figma design at all specified breakpoints
- [ ] Keyboard navigation works for all interactive elements
- [ ] Color contrast passes WCAG AA
- [ ] Component is properly typed (TypeScript/prop types)
- [ ] States (hover, focus, disabled, loading, error) match Figma variants
- [ ] Component exports follow project barrel-file conventions

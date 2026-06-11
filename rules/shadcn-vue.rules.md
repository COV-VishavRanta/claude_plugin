---
paths:
  - "**/*.tsx"
  - "**/*.jsx"
  - "**/*.ts"
  - "**/*.css"
description: "Use when: installing, using, customizing, or updating shadcn-vue components in the Vue frontend. Covers CLI installation, component composition, theming, and update workflow."
---

# shadcn-vue Usage Standards

## Installation - CLI First

- Always use the shadcn-vue CLI from the `frontend/` directory to install or add components. Never manually copy-paste generated component source files.

```bash
npx shadcn-vue@latest add <component-name>
```

- To add multiple components at once:

```bash
npx shadcn-vue@latest add button card input dialog
```

- If shadcn-vue has not been initialized yet, run this first from `frontend/`:

```bash
npx shadcn-vue@latest init
```

- Start with the CLI-generated primitive, then update or wrap it to satisfy the requirement. Do not build a duplicate base primitive manually when the registry already provides it.

## Component Location & Imports

- Generated shadcn-vue components belong in `frontend/src/components/ui/`.
- Project-specific wrappers belong in `frontend/src/components/`.

```ts
import { Button } from "./components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "./components/ui/card";
```

- Do not import from `reka-ui` directly in feature components when a shadcn-vue wrapper exists.

## Customization Rules

- Extend instead of forking. Wrap generated components for feature-specific behavior.
- Only edit files inside `frontend/src/components/ui/` for global defaults, reusable variants, or generator-output fixes required by the project.
- When directly editing a generated `ui/` component, add this marker at the top of the file:

```ts
// CUSTOMIZED: <reason>
```

## Variants & Theming

- Follow the existing `class-variance-authority` pattern when adding variants.
- Put design tokens in shared CSS custom properties rather than hardcoded values inside components.
- Prefer semantic, token-backed classes over ad-hoc CSS overrides.

## Updating Components

- Refresh generated components with the CLI rather than manually replacing files:

```bash
npx shadcn-vue@latest add <component-name>
```

- Before updating, check for a `// CUSTOMIZED:` marker and re-apply those changes carefully after regeneration.
- Run `npm run type-check` from `frontend/` after adding or updating components.

## Accessibility

- Preserve accessibility behavior provided by generated components.
- Provide accessible labels for icon-only controls.
- Do not remove keyboard or focus behavior from wrappers.

## What Not To Do

- Do not manually create a base component in `frontend/src/components/ui/` when shadcn-vue can generate it.
- Do not bypass the CLI install step when the requirement maps to an existing shadcn-vue component.
- Do not scatter overrides for generated components across feature files.
- Do not use `!important` to fight generated styles.

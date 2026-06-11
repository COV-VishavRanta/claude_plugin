---
paths:
  - "**/*.tsx"
  - "**/*.jsx"
  - "**/*.ts"
  - "**/*.css"
description: "Use when: installing, using, customizing, or updating shadcn/ui components in the frontend. Covers CLI installation, component composition, theming, and update workflow."
---

# shadcn/ui Usage Standards

## Installation — CLI First

- **Always use the shadcn CLI on the frontend directory** to add components. Never manually copy-paste component source files.

```bash
npx shadcn@latest add <component-name>
```

- To add multiple components at once:

```bash
npx shadcn@latest add button card input dialog
```

- If the project has not been initialized with shadcn yet, run `init` first from the `frontend/` directory:

```bash
npx shadcn@latest init
```

- When shadcn prompts for configuration (style, base color, CSS variables), accept the project defaults or follow the team's design-token decisions in `design.instructions.md`.

## Component Location & Imports

- shadcn components are installed to `src/components/ui/` by default — do not relocate them.
- Import shadcn components using the `@/components/ui/` path alias:

```tsx
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
```

- Never import from `@radix-ui/*` directly in application code. Use the shadcn wrapper.

## Customization Rules

- **Extend, don't fork.** When a shadcn component needs project-specific behavior, create a wrapper in `src/components/` that composes the base `ui/` component:

```tsx
// src/components/submit-button.tsx
import { Button, type ButtonProps } from "@/components/ui/button";
import { Loader2 } from "lucide-react";

export function SubmitButton({
  children,
  isLoading,
  ...props
}: ButtonProps & { isLoading?: boolean }) {
  return (
    <Button disabled={isLoading} {...props}>
      {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
      {children}
    </Button>
  );
}
```

- Only edit files inside `src/components/ui/` when you need to change the base variant, default styles, or add a new variant that applies globally.
- Document any direct edits to `ui/` files with a comment: `// CUSTOMIZED: <reason>` at the top of the file so future updates can be reconciled.

## Adding Variants

- Use the `cva` (class-variance-authority) pattern already established by shadcn for adding new variants:

```tsx
const buttonVariants = cva("...", {
  variants: {
    variant: {
      // existing variants...
      destructive: "bg-destructive text-destructive-foreground ...",
      // new project variant
      warning: "bg-amber-500 text-white hover:bg-amber-600",
    },
  },
});
```

## Theming & Design Tokens

- All color and spacing tokens go in `globals.css` via CSS custom properties — never inline raw hex/rgb values.
- When adding a new shadcn component, verify its CSS variables are present in `globals.css`. If missing, add them.
- Use Tailwind utility classes that reference CSS variables (e.g., `bg-primary`, `text-muted-foreground`), not hardcoded colors.

## Updating Components

- To update a specific component to the latest shadcn version:

```bash
npx shadcn@latest add <component-name> --overwrite
```

- **Before overwriting**, check if the component has a `// CUSTOMIZED:` comment. If it does:
  1. Back up the current file or review the git diff after overwriting.
  2. Re-apply customizations on top of the updated base.
  3. Run tests to confirm nothing broke.

- To check which components are installed and their status:

```bash
npx shadcn@latest diff
```

## Composition Patterns

- Prefer composing multiple shadcn primitives over building custom UI from scratch. Example — a confirmation dialog:

```tsx
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from "@/components/ui/alert-dialog";
```

- Use shadcn `Form` components with React Hook Form and Zod for all form implementations — do not build form controls from scratch.

## Accessibility

- shadcn components ship with Radix UI accessibility built in. Do not override `role`, `aria-*`, or keyboard-handler props unless you have a specific accessibility requirement.
- Always provide visible labels or `aria-label` when using icon-only buttons or inputs.

## What Not to Do

- Do not install Radix UI primitives separately — shadcn bundles them.
- Do not create a new `ui/` component manually when one already exists in the shadcn registry — run `npx shadcn@latest add` instead.
- Do not use `!important` overrides on shadcn component styles. Use variant props or the `cn()` utility for conditional classes.
- Do not scatter component customizations across the codebase. Keep base variants in `ui/`, project wrappers in `src/components/`.

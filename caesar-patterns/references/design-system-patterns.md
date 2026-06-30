# Design System Patterns — Caesar → Daiton

The distilled, self-contained reference for Caesar's design-system primitives and
the stack compatibility. This is the shortest reference because the stacks are
byte-compatible — the lift is almost entirely mechanical.

## Stack compatibility (byte-compatible)

| Layer | Caesar (`ch-front-end`) | Daiton (`frontend/`) | Lift? |
|---|---|---|---|
| **UI primitives** | shadcn/ui + `@radix-ui` | shadcn/ui + `@radix-ui` | ✓ identical |
| **Styling** | Tailwind **v4** + `cn()` (clsx + tailwind-merge) | Tailwind v4 + `cn()` | ✓ identical |
| **Icons** | mostly `lucide-react` (some `@radix-ui/react-icons`) | `lucide-react` only | ✓ swap the handful of Radix icons on port |
| **Table** | TanStack Table | TanStack Table | ✓ |
| **Server state** | TanStack Query | TanStack Query | ✓ |
| **Routing** | TanStack Router | TanStack Router | ✓ |
| **Local state** | Zustand (scoped stores) | Zustand | ✓ |
| **Graph** | n/a | Sigma + graphology | Daiton-only |
| **AI chat** | n/a | Vercel `ai` | Daiton-only |

**Verdict:** stack-compatible. A component lift is mechanical — the base card,
for example, uses only `Card`/`Badge`/`Button`/`Separator`/`cn` + `lucide-react`,
all present in Daiton.

## The two adaptation rules (all you need to remember)

When porting a Caesar component, exactly two things change:

### 1. Import alias

```ts
// Caesar
import { Card } from '@/components/ui/card';
// Daiton — map the alias (verify Daiton's tsconfig paths)
import { Card } from '@/components/ui/card';   // often the same alias, but CHECK
```

### 2. Icons

```ts
// Caesar (sometimes)
import { Cross1Icon, TrashIcon } from '@radix-ui/react-icons';
// Daiton — always lucide-react
import { X, Trash2 } from 'lucide-react';
```

Swap any `@radix-ui/react-icons` import for the `lucide-react` equivalent. The
shadcn/Radix *primitives* (`@radix-ui/react-dialog`, `@radix-ui/react-dropdown-menu`,
etc.) stay as-is — only the *icon* package changes.

## The shadcn/ui primitive set (already in Daiton)

Every component below exists identically in Daiton under `components/ui/`. When a
Caesar component depends on one, the dependency is already satisfied:

`Card` · `Badge` · `Button` · `Separator` · `ScrollArea` · `Sheet` · `Dialog` ·
`Command` · `DropdownMenu` · `Popover` · `Tooltip` · `Tabs` · `Input` ·
`Textarea` · `Select` · `Checkbox` · `Switch` · `Avatar` · `Skeleton` ·
`Toast`/`Sonner` · `AlertDialog` · `Accordion` · `Collapsible` · `Progress` ·
`Label` · `Table`

## `cn()` — the class-merge utility

```ts
// The standard shadcn helper (identical in Caesar and Daiton)
import { clsx, type ClassVarMapping } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassVarMapping[]) {
  return twMerge(clsx(inputs));
}
```

Use `cn()` for every conditional className so Tailwind classes merge (later wins)
rather than concatenate. This is how Caesar composes variant styles on cards,
buttons, and table rows. Daiton has it.

## Design tokens

Caesar's tokens live in the Tailwind v4 CSS config (e.g. `@theme` block /
`app.css`). Caesar's are clinical/EMR-themed (e.g. `--emr-…` for clinical status
colors). **When you lift a Caesar component, swap any `var(--emr-…)` reference for
the equivalent Daiton token.** Daiton's tokens are brain-themed (entity-type
colors for Person/Organization/etc.).

The entity-type color/icon mapping Daiton should seed from:
`frontend/src/features/graph/entity-icons.ts` — the icon lookup already exists
there; the goal is to make it *config-owned* (per `EntityConfig`) rather than
hardcoded in the graph feature.

## Class-variance-authority (`cva`) for variants

Caesar uses `cva` (class-variance-authority) for component variants — the
`size`/`statusVariant` on `EntityCardBase`, the Badge variants, button sizes.
Daiton has `cva`. When lifting a component with a `variants` object, bring the
`cva` definition verbatim.

```ts
// The EntityCardBase size variant model (from entity-ui-patterns.md)
const sizeVariants = cva('', {
  variants: {
    size: {
      compact: 'p-3 text-sm',     // icon + title + status only
      standard: 'p-4',             // + reason + metadata + AI footer
      detailed: 'p-6 text-base',   // same, larger scale
    },
  },
});
```

## Responsive hooks

`ch-front-end/src/hooks/use-breakpoints.tsx` — the breakpoint hook Caesar uses for
the master-detail mobile/desktop switch. Daiton has its own; align on the same
breakpoints (`sm`/`md`/`lg`) as the Tailwind config so `MasterDetailLayout`
behaves identically.

`ch-front-end/src/hooks/use-dialog-state.tsx` — the dialog/sheet open-state hook.
Port if you need the same controlled-open pattern.

## Don't forget

- **Render-verify** after any design-system change — a component that typechecks
  but renders wrong (stale token, swapped icon) is not done.
- **Swap tokens, not just components.** A lifted card that still references
  `var(--emr-status-active)` will render with a missing color.
- **Keep `lucide-react` as the single icon source.** Don't reintroduce
  `@radix-ui/react-icons` — it's the one dependency Caesar carries that Daiton
  deliberately dropped.

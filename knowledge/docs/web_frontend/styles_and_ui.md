---
title: Styling, UI Libraries, and Visual Tools
---

## Tailwind CSS Best Practices

- Establish design tokens in `tailwind.config.js` (colors, spacing, radius)
- Use small, composable components and variants
- Extract repeated utility groups into component classes via `@apply` if needed
- Keep markup readable: prefer `clsx` over long conditional strings

```ts
// src/lib/cx.ts
import clsx, { type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'
export function cx(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

## Accessible Primitives with Radix UI

- Install: `pnpm add @radix-ui/react-*`
- Use for dialogs, popovers, menus, tabs; bring your own styles (Tailwind)
- Ensures keyboard navigation and ARIA correctness

## Variant Patterns

Consider class-variance-authority for ergonomic variants:

```bash
pnpm add class-variance-authority tailwind-merge clsx
```

```ts
// src/components/ui/button.tsx
import { cva, type VariantProps } from 'class-variance-authority'
import { cx } from '@/lib/cx'

const buttonStyles = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand focus-visible:ring-offset-2 disabled:opacity-50 disabled:pointer-events-none h-9 px-4 py-2',
  {
    variants: {
      variant: {
        primary: 'bg-brand text-white hover:bg-brand-700',
        secondary: 'bg-white text-gray-900 border border-gray-200 hover:bg-gray-50',
        ghost: 'bg-transparent hover:bg-gray-100',
      },
      size: {
        sm: 'h-8 px-3',
        md: 'h-9 px-4',
        lg: 'h-10 px-6',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  },
)

export type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> &
  VariantProps<typeof buttonStyles>

export function Button({ className, variant, size, ...props }: ButtonProps) {
  return <button className={cx(buttonStyles({ variant, size }), className)} {...props} />
}
```

## Visual Tools (Open Source)

- Storybook: visual component explorer with controls, docs, and add-ons
- VisBug: Chrome extension to inspect and tweak layout/spacing live
- GrapesJS: drag-and-drop editor useful for landing pages or prototyping

```{tip}
Use Storybook for day-to-day visual arrangement and state exploration; reach for GrapesJS only if you truly need WYSIWYG page composition.
```

## CSS Quality

- Optional: Stylelint for CSS and Tailwind class ordering

```bash
pnpm add -D stylelint stylelint-config-standard stylelint-config-prettier \
  stylelint-config-recess-order
```

`.stylelintrc.json`:

```json
{
  "extends": ["stylelint-config-standard", "stylelint-config-recess-order", "stylelint-config-prettier"]
}
```

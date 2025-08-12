---
title: Daily Workflow and Component-First Development
---

## Dev Commands

```bash
# Start dev server with HMR
pnpm dev

# Lint and format
pnpm lint
pnpm format

# Build and preview production
pnpm build
pnpm preview
```

## Component-First Workflow

1. Create a component (e.g., `src/components/ui/Button.tsx`)
2. Render and iterate in Storybook
3. Use the component in app pages
4. Add tests

## Storybook Setup (visual arrangement)

```bash
pnpm dlx storybook@latest init --type react-vite
pnpm storybook
```

Core uses:

- Browse components in isolation
- Edit props and styles live
- Add stories that document states, variants, and edge cases
- Use a11y and interactions addons to validate UX early

## Example Button Component with Tailwind

`src/components/ui/Button.tsx`:

```tsx
import { type ButtonHTMLAttributes } from 'react'
import clsx from 'clsx'

type ButtonProps = ButtonHTMLAttributes<HTMLButtonElement> & {
  variant?: 'primary' | 'secondary' | 'ghost'
}

export function Button({ variant = 'primary', className, ...props }: ButtonProps) {
  const styles = {
    base: 'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand focus-visible:ring-offset-2 disabled:opacity-50 disabled:pointer-events-none h-9 px-4 py-2',
    primary: 'bg-brand text-white hover:bg-brand-700',
    secondary: 'bg-white text-gray-900 border border-gray-200 hover:bg-gray-50',
    ghost: 'bg-transparent hover:bg-gray-100',
  }
  const finalClass = clsx(styles.base, styles[variant], className)
  return <button className={finalClass} {...props} />
}
```

`src/components/ui/Button.stories.tsx`:

```tsx
import type { Meta, StoryObj } from '@storybook/react'
import { Button } from './Button'

const meta = {
  title: 'UI/Button',
  component: Button,
  parameters: { layout: 'centered' },
  args: { children: 'Click me' },
} satisfies Meta<typeof Button>
export default meta

type Story = StoryObj<typeof meta>

export const Primary: Story = { args: { variant: 'primary' } }
export const Secondary: Story = { args: { variant: 'secondary' } }
export const Ghost: Story = { args: { variant: 'ghost' } }
```

## Git Hooks (optional but recommended)

```bash
pnpm add -D husky lint-staged
pnpx husky init
# edit .husky/pre-commit
```

`.husky/pre-commit`:

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm lint -f compact
pnpm format
```

Alternatively, configure `lint-staged` for staged-only formatting.

---
title: Testing, Linting, and Type Safety
---

## Vitest + React Testing Library

```bash
pnpm add -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom
```

`vite.config.ts` test block:

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    coverage: { reporter: ['text', 'html'] },
  },
})
```

`src/test/setup.ts`:

```ts
import '@testing-library/jest-dom'
```

Example test `src/components/ui/Button.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { Button } from './Button'

test('renders and clicks', async () => {
  const user = userEvent.setup()
  const onClick = vi.fn()
  render(<Button onClick={onClick}>Click</Button>)
  await user.click(screen.getByRole('button', { name: /click/i }))
  expect(onClick).toHaveBeenCalled()
})
```

Run tests:

```bash
pnpm vitest
```

## Playwright (E2E)

```bash
pnpm dlx playwright@latest install --with-deps
```

Add a basic test in `tests/example.spec.ts` and run:

```bash
pnpm playwright test
```

## ESLint + Prettier

- Run `pnpm lint` and `pnpm format` in CI
- Ensure `eslint-config-prettier` is included to avoid rule conflicts

## TypeScript Strict Mode

- Enforce strictness in `tsconfig.json` and keep zero `any` in code reviews

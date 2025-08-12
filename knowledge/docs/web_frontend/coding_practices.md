---
title: Coding Practices and Guidelines
---

## Principles

- Prefer simplicity and readability over cleverness
- Keep functions short; extract helpers
- Name things descriptively (no 1–2 letter names)
- Use TypeScript strictly to avoid runtime surprises
- Add tests for critical logic and components

## Folder Structure

```
src/
  components/
    ui/
    layout/
  pages/
  hooks/
  lib/
  styles/
```

Keep UI components dumb and composable; move data fetching or heavy logic into hooks or lib.

## React Patterns

- Use function components and hooks
- Lift state up only when needed; prefer local state first
- Derive state; avoid duplicating source of truth
- Prefer controlled components for forms
- Extract pure utilities into `lib/`

## TypeScript Tips

- Enable `strict`, `noUncheckedIndexedAccess`, and meaningful `tsconfig`
- Avoid `any`; prefer precise unions and discriminated unions
- Define component prop types; keep them minimal and focused

## Styling and Theming

- Define tokens (colors, spacing, radius) in Tailwind config
- Use class utilities consistently; extract variants with CVA if complex
- Ensure accessible color contrast (use Radix or Tailwind palette wisely)

## Accessibility (a11y)

- Use semantic elements and ARIA where appropriate
- Ensure focus styles and keyboard navigation work
- Test with `@storybook/addon-a11y` and Playwright

## Git and Reviews

- Small, focused commits with meaningful messages
- Run `pnpm lint` and `pnpm test` before pushing
- Prefer PRs that add tests and docs (Storybook stories)

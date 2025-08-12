---
title: Choosing the Stack
---

## Decision Framework

When selecting tools, prefer:

- Simplicity: fewer moving parts to start, add complexity only when needed
- Popularity and docs: easier to learn, better community support
- Open source: transparent and flexible

## Frameworks

- React (recommended): ubiquitous, works great with Vite, large ecosystem
- Svelte/SvelteKit: minimal boilerplate, great DX; excellent alternative
- Solid: very fast and ergonomic; smaller ecosystem
- Astro: great for content sites; can mix frameworks; supports partial hydration

For a first professional project: pick React + Vite + TypeScript for ecosystem depth and job-market alignment. Svelte is a close second for DX.

## Bundler/Dev Server

- Vite (recommended): HMR, lightning-fast dev and builds, TS support out of the box

## Languages

- TypeScript: strict mode for safety and maintainability

## Styling and UI

- Tailwind CSS: utility-first, fast iteration, configurable design tokens
- Radix UI: accessible, unstyled primitives; style with Tailwind
- Optional: shadcn/ui (on top of Radix + Tailwind; mostly Next.js-focused, but concepts carry to Vite)
- Alternatives: CSS Modules, CSS-in-JS (vanilla-extract, Emotion), UnoCSS

## Component Workbench / Visual Arrangement

- Storybook (recommended): open-source, explore components in isolation, documentation, a11y, visual tests
- VisBug: open-source Chrome extension to visually tweak pages in the browser
- GrapesJS: open-source visual editor/builder; useful for drag-and-drop prototypes

## Testing and Quality

- Vitest (unit) + React Testing Library
- Playwright (E2E)
- ESLint (flat config) + Prettier + Stylelint (optional)

## Deployment

- GitHub Pages (simple static hosting)
- Netlify / Vercel (CI-connected, preview deployments)

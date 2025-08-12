---
title: Getting Started (Beginner Friendly)
---

# Getting Started

Welcome! This short guide gets you productive fast with a modern, professional frontend setup using TypeScript and open-source tools.

## What You’ll Build

- A pure frontend website (no backend needed) using React + Vite + TypeScript
- Live reload on save, with fast production builds
- Components for layout and UI (so you don’t write raw HTML all the time)
- Tailwind CSS for styling, Storybook to visually design components

## Why This Stack

- Vite: instant dev server and hot reload
- React: popular, component-based UI
- TypeScript: catches many bugs early
- Tailwind: speeds up styling; consistent design system
- Storybook: visual component explorer and documentation

## Install the Basics

```bash
# Node LTS and pnpm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install --lts && nvm use --lts
corepack enable && corepack prepare pnpm@latest --activate
```

## Create Your App

```bash
pnpm create vite@latest my-frontend -- --template react-ts
cd my-frontend && pnpm install
pnpm dev
```

Open the shown local URL in your browser. You now have hot reload on save.

## Add Styling and Visual Tools

```bash
pnpm add -D tailwindcss postcss autoprefixer
pnpx tailwindcss init -p
```

Follow the Tailwind section in the Setup page to wire it into `src/index.css` and `tailwind.config.js`.

Add Storybook for visual component development:

```bash
pnpm dlx storybook@latest init --type react-vite
pnpm storybook
```

## First Component

Create `src/components/ui/Button.tsx` and a matching Storybook story. See the Workflow page for copy-paste templates you can adapt.

## Next Steps

1. Read the Setup page for strict TypeScript, ESLint, and Tailwind configuration
2. Use the Workflow page to build components first in Storybook
3. Explore Styles & UI for Tailwind and Radix UI patterns
4. Review Coding Practices and Architecture for scalable structure
5. Add tests via Testing & Quality, then deploy using Deployment

```{tip}
Don’t try to learn everything at once. Start with Vite + React + TypeScript + Tailwind, then add Storybook and tests once basic pages render.
```

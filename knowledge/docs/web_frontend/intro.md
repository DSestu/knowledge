---
title: Web Frontend — Professional Starter Guide
---

# Web Frontend — Professional Starter Guide

```{note}
Audience: Beginners to intermediate developers who want to build a modern, professional, purely frontend website using TypeScript, component-based UI, and open‑source tooling with instant rebuild on save.
```

## Goals

- Build a pure frontend site with embedded JavaScript written in TypeScript
- Hot reload on save and fast builds
- Avoid hand-writing raw HTML for everything by using components
- Use open-source tools to visually explore/arrange components and edit styles
- Professional quality: type safety, linting, testing, CI, and accessible UI

## Recommended Stack (TL;DR)

- Framework: React + Vite + TypeScript
- Styling: Tailwind CSS (with design tokens) + Radix UI primitives
- Component Dev/Docs: Storybook (visual arrangement/testing of components)
- Quality: ESLint (flat config), Prettier, Stylelint (optional), TypeScript strict
- Testing: Vitest + React Testing Library, Playwright for E2E
- Package Manager: pnpm
- Deployment: GitHub Pages, Netlify, or Vercel

Why this stack:

- Vite offers instant dev server with HMR and fast production builds
- React gives a mature ecosystem, component model, and hiring-friendly skills
- Tailwind speeds up consistent styling; Radix provides accessible primitives
- Storybook is open-source and ideal for visually iterating on component design

## What You’ll Build

- A Vite React + TypeScript project with strict type checking
- Tailwind configured with design tokens
- Storybook to visually explore your components
- Linting, formatting, unit and e2e tests, and CI-ready scripts

```{contents}
:local:
:depth: 2
```

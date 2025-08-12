---
title: Deployment (GitHub Pages, Netlify, Vercel)
---

## Production Build

```bash
pnpm build
```

Outputs to `dist/`.

## GitHub Pages

If deploying to `user.github.io/repo`, set Vite base path when building on CI:

```ts
// vite.config.ts
export default defineConfig(({ mode }) => ({
  base: mode === 'production' ? '/REPO_NAME/' : '/',
  // ...
}))
```

GitHub Actions workflow `.github/workflows/gh-pages.yml`:

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [ main ]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
          cache: 'pnpm'
      - run: corepack enable && corepack prepare pnpm@latest --activate
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

## Netlify

- Build command: `pnpm build`
- Publish directory: `dist`

## Vercel

- Framework preset: Vite
- Build command: `pnpm build`
- Output directory: `dist`

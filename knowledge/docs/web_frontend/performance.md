---
title: Performance and Optimization
---

## Build-Time

- Keep dependencies lean; prefer native browser features where possible
- Code-split heavy routes with dynamic import
- Use Vite’s analyzer plugins if needed to inspect bundle

## Runtime

- Avoid unnecessary re-renders (memoize expensive computation)
- Use React lazy + Suspense for non-critical components
- Optimize images (modern formats, responsive sizes)

## Lighthouse and Web Vitals

- Run Lighthouse in Chrome DevTools for performance and a11y regressions
- Track Core Web Vitals (CLS, LCP, FID/INP) locally during development

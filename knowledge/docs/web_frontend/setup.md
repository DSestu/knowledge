---
title: Project Setup (Vite + React + TypeScript)
---

## Prerequisites

- Linux or macOS terminal (Windows WSL ok)
- Node.js LTS managed via nvm, and pnpm as the package manager

```bash
# Install nvm (if you don't have it)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
# restart shell, then
nvm install --lts
nvm use --lts

# Install pnpm
corepack enable
corepack prepare pnpm@latest --activate
pnpm -v
```

## Create Project

```bash
pnpm create vite@latest my-frontend -- --template react-ts
cd my-frontend
pnpm install
git init && git add -A && git commit -m "chore: scaffold vite react ts"
```

## Strict TypeScript

Set strictness in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": false,
    "moduleResolution": "Bundler",
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src"]
}
```

## Vite Config (aliases)

`vite.config.ts`:

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'node:path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
})
```

## ESLint (flat config) + Prettier

```bash
pnpm add -D eslint @eslint/js typescript-eslint prettier eslint-config-prettier \
  eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-jsx-a11y
```

Create `eslint.config.js`:

```js
// eslint.config.js
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import reactHooks from 'eslint-plugin-react-hooks'
import reactPlugin from 'eslint-plugin-react'
import jsxA11y from 'eslint-plugin-jsx-a11y'

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  reactPlugin.configs.flat.recommended,
  {
    files: ['**/*.{ts,tsx}'],
    plugins: { 'react-hooks': reactHooks, 'jsx-a11y': jsxA11y },
    rules: {
      'react/react-in-jsx-scope': 'off',
      'react/prop-types': 'off',
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
      'jsx-a11y/alt-text': 'warn',
    },
  },
  {
    ignores: ['dist', 'coverage', '.storybook'],
  },
]
```

Create `prettier.config.cjs`:

```js
module.exports = {
  semi: false,
  singleQuote: true,
  trailingComma: 'all',
}
```

Add scripts in `package.json`:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier -w ."
  }
}
```

## Tailwind CSS

```bash
pnpm add -D tailwindcss postcss autoprefixer
pnpx tailwindcss init -p
```

`tailwind.config.js` content:

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{ts,tsx,css}'],
  theme: {
    extend: {
      colors: {
        brand: {
          DEFAULT: '#2563EB',
          50: '#EFF6FF',
          100: '#DBEAFE',
          600: '#2563EB',
          700: '#1D4ED8',
        },
      },
    },
  },
  plugins: [],
}
```

`src/index.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --radius: 0.5rem;
}
```

Use it in `src/main.tsx`:

```ts
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.tsx'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

## Suggested Project Structure

```
my-frontend/
├── src/
│   ├── components/
│   │   └── ui/
│   ├── pages/
│   ├── hooks/
│   ├── lib/
│   ├── assets/
│   ├── index.css
│   └── main.tsx
├── public/
├── vite.config.ts
├── tsconfig.json
├── eslint.config.js
├── package.json
└── README.md
```

Run the dev server (auto rebuild on save):

```bash
pnpm dev
```

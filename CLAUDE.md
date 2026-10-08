# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Layout

Two independent projects live side by side; there is no root-level build or package manifest.

- `backend/` — .NET solution (`CvBuilder.slnx`, the XML slnx format). The solution is currently **empty** (no projects added yet). New .NET projects should be created here and added to the solution.
- `frontend/` — React 19 + TypeScript + Vite 8 app. All frontend commands must be run from inside `frontend/`.

Commands run against each half separately — treat them as two workspaces.

## Frontend (`frontend/`)

Standard Vite scripts (from `frontend/package.json`):

```bash
npm run dev       # start Vite dev server with HMR
npm run build     # tsc -b (project references) then vite build
npm run lint      # eslint .
npm run preview   # preview the production build
```

Notes:
- `npm run build` runs `tsc -b` first, so TypeScript errors fail the build. There is no separate `typecheck` script — use `npx tsc -b --noEmit` (or just `tsc -b`) if you want to check types without bundling.
- TypeScript is configured as a project-references setup: root `tsconfig.json` references `tsconfig.app.json` (src, DOM lib, `jsx: react-jsx`, `verbatimModuleSyntax`, `noUnusedLocals`, `noUnusedParameters`, `erasableSyntaxOnly`) and `tsconfig.node.json` (Vite config). Add new source paths to `tsconfig.app.json`'s `include`.
- `verbatimModuleSyntax` is enabled, so type-only imports must use `import type { ... }`.
- No test framework is configured yet. If adding one, prefer Vitest (natural fit with Vite) and wire it into `package.json` scripts.
- ESLint flat config (`eslint.config.js`) extends `js.recommended`, `typescript-eslint recommended`, `react-hooks` flat recommended, and `react-refresh/vite`. `dist` is globally ignored.

## Backend (`backend/`)

The solution file exists but contains no projects. Before adding backend code:
1. `cd backend`
2. Create a project (e.g. `dotnet new webapi -n CvBuilder.Api`)
3. Add it to the solution: `dotnet sln CvBuilder.slnx add CvBuilder.Api/CvBuilder.Api.csproj`

Once projects exist, the usual flow is `dotnet build`, `dotnet run --project <proj>`, `dotnet test`. There are no backend conventions established yet — decisions about framework version, project layout, and test strategy are open.

## State of the project

Everything under `frontend/src/` (App.tsx, App.css, assets) is still the untouched Vite + React starter — a counter button and links to Vite/React docs. There is no application logic yet, no routing, no state management, no API client, and no backend wired up. When implementing features, treat the current contents of `App.tsx` and `index.css` as scaffolding to replace, not as examples of the project's conventions.

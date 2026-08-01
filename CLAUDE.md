# values.trade-bo

Back-office frontend. React + TanStack Router (file-based) + TanStack Query + TanStack Form + TypeScript.

## Stack
- **Package manager**: npm — always use `npm`, never `pnpm`/`yarn`/`bun` commands or lockfiles.
- **Router**: `@tanstack/react-router`, file-based routing (`src/routes/`).
- **Server state**: `@tanstack/react-query` v5. No other data-fetching library.
- **Forms**: `@tanstack/react-form`. No `react-hook-form`, `formik`, or uncontrolled-form rolling by hand.
- **Language**: TypeScript, strict mode.
- **Lint/format**: Biome (replaces ESLint + Prettier). Run `npx biome check --write .` before considering work done.
- **Styling**: Tailwind CSS. No CSS Modules, no styled-components, no inline `style=` unless dynamic values require it.

Detailed API usage (loaders, search params, query keys, mutations, optimistic updates, form state/validation, etc.) is covered by the `tanstack-router`, `tanstack-query`, `tanstack-form`, and `typescript-rules` skills — they load automatically when relevant. This file only holds project-specific decisions and rules that apply everywhere, so it doesn't repeat what the skills already say.

## Project structure (layer-based)

```
src/
  routes/        # TanStack Router file-based routes (route.tsx + route.lazy.tsx split)
  components/    # Shared/reusable UI components (Tailwind-styled)
  hooks/         # Custom hooks (state logic, side effects, DI seams)
  api/           # Fetch functions + queryOptions() factories, one file per resource
  lib/           # Framework-agnostic utilities (no React imports)
  types/         # Shared types/interfaces not co-located with a single feature
```

- Co-locate a component's tests/styles with the component itself; only put things in `types/`, `lib/`, `api/` when they're genuinely shared across routes/components.
- `api/*.ts` exports `queryOptions()` objects (see tanstack-query skill) — routes call `queryClient.ensureQueryData(...)` in loaders, components call `useQuery`/`useSuspenseQuery` with the same options object.
- Query keys are hierarchical arrays (`['resource', id]`) — see tanstack-query skill for invalidation patterns.

## Non-negotiables

- **`any` is banned.** Unknown external data (API responses, localStorage, URL/search params) comes in as `unknown` and is narrowed with a type guard. Type assertions are a last resort; if used 3+ times for the same shape, fix the type instead.
- **No error suppression.** Every `catch` logs and either re-throws or produces a typed error/Result — never a silent fallback that swallows the failure.
- **Function components only.** Class components are not used (except an Error Boundary if one is ever needed).
- **Imports are absolute** from `src/` — no relative-path spaghetti (`../../../`) crossing more than one directory level.
- **Auth/redirect guards live in `beforeLoad`**, never as conditional rendering inside a route component (avoids flash-of-protected-content).
- **Data fetching only through TanStack Query** (route loaders + hooks) — no `useEffect` fetch calls.
- **Secrets never touch the frontend.** No API keys/tokens in `import.meta.env.*` that get bundled client-side; the backend proxies anything sensitive.
- Delete unused code and debug `console.log()` immediately — don't leave commented-out code, git history is the record.

## Commands

Not yet scaffolded (no `package.json`). Once the project is bootstrapped, add here: dev server, build, test, and `biome check` commands, plus how to run a single test.

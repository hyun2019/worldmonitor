# Repository Guidelines

## Project Structure & Module Organization
`src/` contains the Vite + TypeScript app. Keep UI code in `src/components/`, data fetching and analysis logic in `src/services/`, static datasets and feature flags in `src/config/`, reusable helpers in `src/utils/`, and shared types in `src/types/`. End-to-end support code lives in `src/e2e/`, while Playwright specs and golden snapshots live in `e2e/`. Desktop packaging lives in `src-tauri/`, helper scripts in `scripts/`, and raw data assets in `data/`.

## Build, Test, and Development Commands
Use `npm run dev` for the default geopolitics build and `npm run dev:tech` for the tech variant. Run `npm run build` for a standard production bundle, or `npm run build:full` / `npm run build:tech` when validating a specific variant. Use `npm run typecheck` before opening a PR.

Run browser regression tests with `npm run test:e2e:full`, `npm run test:e2e:tech`, or `npm run test:e2e` for the full suite. Use `npm run test:e2e:visual` for screenshot regression coverage and `npm run test:e2e:visual:update` only when intentional UI changes require new baselines. Run desktop-side Node tests with `npm run test:sidecar`.

## Coding Style & Naming Conventions
This repo uses strict TypeScript (`strict`, `noUnusedLocals`, `noUncheckedIndexedAccess`). Follow the existing style: 2-space indentation, semicolons, single quotes, and `@/` imports for `src/*`. Use `PascalCase` for components and classes, `camelCase` for functions and variables, and kebab-case for utility filenames such as `signal-aggregator.ts`. Keep modules focused; put cross-cutting logic in `src/services/` instead of UI components.

## Testing Guidelines
Add or update Playwright coverage for map behavior, runtime fetches, and variant-specific UI changes. Place specs in `e2e/*.spec.ts`; snapshots belong beside the spec in `e2e/*-snapshots/`. Prefer targeted test names that describe the behavior under test, for example `updates protest marker click payload after data refresh`.

## Commit & Pull Request Guidelines
Recent history favors short, imperative commit subjects such as `Fix Tauri desktop runtime reliability` or `Remove @tauri-apps/cli from devDependencies`. Keep commits scoped to one change. PRs should state user-visible impact, list commands run (`npm run typecheck`, relevant test scripts), and include screenshots for dashboard, map, or visual snapshot changes.

## Security & Configuration Tips
When adding RSS feeds in `src/config/feeds.ts`, also add the feed domain to the allowlist in `api/rss-proxy.js` or the feed will fail with HTTP 403. Be explicit about variant-sensitive changes: `VITE_VARIANT=full` is the default, `VITE_VARIANT=tech` powers the tech deployment, and desktop flows also depend on `VITE_DESKTOP_RUNTIME=1`.

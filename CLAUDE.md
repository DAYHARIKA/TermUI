# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

TermUI is a **full-stack terminal application framework** — the Next.js equivalent for the terminal. It is not just a UI component library; it provides everything needed to build production terminal apps: layout engine, JSX/component model, routing, global state, animations, theming, real-time data, hot-reload dev server, and a project scaffolding CLI. It is a **Bun monorepo** managed with **Turborepo**, containing 13 packages under `packages/` and a documentation website under `website/`.

Runtime: **Bun ≥ 1.3.0** required for development. Published packages still target Node 18+ ESM/CJS for npm consumers.

## Commands

All commands run from the repo root unless noted.

```bash
# Install dependencies
bun install

# Build all packages (respects Turbo dependency order)
bun run build

# Run all tests
bun run test

# Run tests in watch mode
bun run test:watch

# Run tests for a single package (e.g. core)
bun --filter @termuijs/core run test

# Run a single test file
bun vitest run packages/core/src/layout/LayoutEngine.test.ts

# Type-check all packages
bun run typecheck

# Lint all packages
bun run lint

# Clean all build artifacts and node_modules
bun run clean
```

Tests live in `packages/*/src/**/*.test.ts` and run with **Vitest** (globals enabled, Node environment). The root `vitest.config.ts` configures this globally. Vitest runs via `bun vitest run` (Bun as the script runner; the `--bun` transformer flag is intentionally NOT used because the worker pool hangs on this suite size).

The `website/` is a separate Vite/React app (`cd website && bun run dev`). It is not part of the monorepo's Turbo pipeline.

## Package Architecture

The dependency graph flows from primitives upward:

```
@termuijs/core          ← screen buffer, flexbox layout, input parser, event system
    ↓
@termuijs/widgets        ← widget class hierarchy (Box, Text, Table, List, etc.)
    ↓
@termuijs/jsx            ← JSX runtime + React-like hooks (useState, useEffect, useContext, memo)
@termuijs/tss            ← Terminal Style Sheets (CSS-like theming)
@termuijs/ui             ← composite interactive widgets (Select, Modal, Tabs, etc.)
    ↓
@termuijs/store          ← Zustand-like global state (depends on jsx)
@termuijs/motion         ← spring/easing animations (depends on core)
@termuijs/router         ← file-based screen routing (depends on core + widgets)
@termuijs/data           ← real-time system stats (CPU, memory, disk)
@termuijs/quick          ← fluent builder API for dashboards (depends on core + widgets)
@termuijs/testing        ← in-memory test renderer: render/query/fireKey/assert
@termuijs/dev-server     ← hot-reload dev server using Bun.spawn (<200ms restart)
create-termui-app        ← project scaffolding CLI
```

**Turbo task dependencies**: `build` requires upstream packages to build first (`^build`). `typecheck` and `test` also depend on `build`. Run `bun run build` before `bun run typecheck` or `bun run test` if the `dist/` folders are missing.

## Core Concepts

### `@termuijs/core`
- `App` — application lifecycle manager: creates `Terminal`, `Screen`, `Renderer`, `LayerManager`, `InputParser`, `FocusManager`. Runs a render loop at configurable FPS.
- `Terminal` — raw TTY abstraction (alternate screen, mouse, resize).
- `Screen` — double-buffered cell grid; `Renderer` diffs and flushes ANSI escape sequences.
- `LayoutEngine` — Yoga-inspired flexbox layout operating on `LayoutNode` trees.
- `EventEmitter` — typed event bus used throughout the framework.

### `@termuijs/widgets`
- All widgets extend the base `Widget` class, which implements the `RootWidget` interface expected by `App`.
- Widgets form a tree; each implements `getLayoutNode()`, `render(screen)`, and optional `mount()`/`unmount()`.
- Widget categories: `display/` (Box, Text, LogView), `input/` (List, TextInput, VirtualList), `data/` (Table, Gauge, Sparkline, BarChart), `feedback/` (ProgressBar, Spinner, Scrollbar).

### `@termuijs/jsx`
- Custom JSX runtime (not React). Components are functions; reconciliation is handled by `reconciler.ts`.
- Hooks (`hooks.ts`) use a **Fiber** model — each component instance has a fiber tracking hook state, effects, intervals, and context values.
- Hook rules apply identically to React: call only at the top level of a component.
- `context.ts` implements `createContext`/`useContext` via fiber parent-chain traversal.
- `memo.ts` implements `memo()` for skipping re-renders.

### `@termuijs/dev-server`
- Spawns the user's entry file via `Bun.spawn()` (not Node `fork`). Bun runs `.ts`/`.tsx` natively, so no `--loader tsx` is needed.
- IPC wires through Bun's `ipc:` callback at spawn time; reload signals use `child.send({ type: 'reload' })`.
- File watcher uses `fs.watch` with debounce; rapid saves coalesce into one restart.

### Testing
- Use `@termuijs/testing` for unit tests. It provides `render()` returning a `TestInstance` with `query`, `fireKey`, and assertion helpers — no real TTY needed.
- Test helpers for raw scenarios: `tests/helpers/mock-screen.ts`, `tests/helpers/mock-stdin.ts`.

## Build System

Each package uses **tsup** for bundling, producing both ESM (`dist/index.js`) and CJS (`dist/index.cjs`) with TypeScript declarations. The shared `tsconfig.base.json` sets `target: ES2022`, `module: NodeNext`, strict mode.

Each package has its own `tsconfig.json` that extends the base. Package exports use the `"exports"` field with `types`/`import`/`require` conditions.

Published `dist/` runs on Node 18+ as well as Bun. Only the dev workflow (hot-reload, tests, scripts) requires Bun.

## Website

`website/` is a TanStack Start + React + Vite app with MDX docs. It is not part of the monorepo's Turbo pipeline. The `prebuild` script runs `bun scripts/generate-llm-docs.mjs` to generate LLM-friendly docs before the Vite build.

## CI

GitHub Actions (`.github/workflows/ci.yml`) runs on push/PR to `main`: installs with `bun install --frozen-lockfile`, runs `bun run build`, then `bun vitest run` and `bun run typecheck` on Ubuntu.

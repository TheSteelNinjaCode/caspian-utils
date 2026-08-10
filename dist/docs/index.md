---
title: Caspian Docs
description: Packaged Caspian documentation manifest for AI task routing, feature discovery, and file placement. Use this page when deciding which Caspian doc to read first or which feature guide applies.
related:
  title: Next Steps
  description: Start with installation, review the CLI commands, then use the structure and component guides to place Caspian files correctly.
  links:
    - /docs/ai-validation-checklist
    - /docs/installation
    - /docs/commands
    - /docs/core-runtime-map
    - /docs/pulsepoint-runtime-map
    - /docs/mcp
    - /docs/file-uploads
    - /docs/websockets
    - /docs/file-conventions
    - /docs/project-structure
    - /docs/components
    - /docs/testing
---

This directory contains the packaged Caspian documentation set for AI-aware feature discovery, task routing, and file-placement guidance.

Treat these docs as reusable Caspian feature guidance. Treat `./caspian.config.json` as the single source of truth for which optional features are enabled in the project being analyzed.

The docs can mention optional features even when those features are disabled in a project. Their job is to explain how a feature works, when a doc applies, and which files to inspect next once the feature is confirmed as relevant.

Before making feature, tooling, or file-placement decisions in a Caspian project, read `./caspian.config.json` almost immediately. That file tells you which optional capabilities are enabled, such as Prisma, MCP, WebSockets, TypeScript, Tailwind, backend-only mode, and component scan directories.

## Default Stack

When generating or editing a Caspian app, treat these as the default choices unless the task explicitly requires something else:

- Use PulsePoint for reactive frontend behavior.
- For first-party HTML events and reactivity, use PulsePoint `on*` attributes, state, refs, effects, directives, and `pp.rpc()` instead of ordinary DOM wiring with ids, `data-*` state, `querySelector`, `addEventListener`, or manual `innerHTML`.
- For PulsePoint performance, treat state as "render required," keep non-rendering timers/request generations/cursors in refs, keep high-frequency state in the smallest useful component, and classify broad authored renders separately from runtime reconciliation overhead. Read [pulsepoint.md](./pulsepoint.md#high-performance-authoring) before changing the runtime for a slow input, search, filter, sidebar, or provider.
- Treat every authored route, layout, and component HTML file like a React component return value **in shape only**: normally one top-level parent HTML element or one imported `x-*` root, with any owned plain `<script>` kept inside that same root. A component must satisfy this; a `.py` page or layout may have sibling top-level nodes and gets a compiler-supplied `display: contents` boundary host — see [pulsepoint.md](./pulsepoint.md) "Multi-root pages and layouts".
- When a page or layout owns a `<script>`, its single root must be a **native** element. A script authored inside an `x-*` root becomes the child's slot content and silently never runs, which then makes every handler in that template throw `ReferenceError`. A handler is likewise evaluated in the scope of the template that *authored* the markup, not the component the element ends up inside. See [pulsepoint.md](./pulsepoint.md) "A template whose root is an `x-*` tag can lose its `<script>`" and "A handler in slot content runs in the authoring template's scope".
- That single-root analogy is the *only* place React applies to markup. Template files are plain HTML — no JSX, no `{cond && <div/>}`, no `{list.map(...)}`, no unquoted `class={...}`. Conditionals are `hidden="{...}"` and lists are `<template pp-for="…">`. See [pulsepoint.md](./pulsepoint.md) "PulsePoint Is Not JSX" before writing any template.
- When `caspian.config.json` has `tailwindcss: true`, use Python `merge_classes(...)` plus browser `twMerge(...)` as the only supported Tailwind class-merging path.
- Use `@rpc()` plus `pp.rpc()` for browser-triggered reads, writes, streaming, and uploads.
- When `caspian.config.json` has `websocket: true`, use **named sockets** for long-lived bidirectional live channels: a Python `@socket()` function consumed by `pp.socket(name, args?, handlers?)` in the owning component script. They are the socket counterpart of `@rpc()`/`pp.rpc()` and share one endpoint. Hand-written `@app.websocket(...)` routes with a native browser `WebSocket` are an escape hatch for wires the JSON-frame contract cannot carry (binary frames, non-JSON protocols), and then the app owns every origin/auth/limit check itself. Keep normal browser-triggered data work on RPC.
- When `caspian.config.json` has `prisma: true`, use the generated Prisma Python ORM from `src/lib/prisma/**` for Python-side database reads and writes. Do not invent a second fetch layer, raw driver wrapper, JSON active store, or browser-side data path that bypasses Prisma.
- Use `Validate` and `Rule` from `casp.validate` for server-side input validation and sanitization.
- Treat `public/` as a direct URL-root mapping: `public/icons/app.png` is `/icons/app.png` without a per-directory route. Use the restricted-inline mapping in `PublicFilesMiddleware` for every top-level directory that receives untrusted runtime uploads.

## AI Doc Shape

Each packaged feature doc should help AI answer the same five questions quickly:

- When does this doc apply?
- Which `caspian.config.json` flag, if any, gates the feature?
- Which app-owned files usually change?
- Which `main.py` or installed `casp` runtime files own the behavior?
- Which behavior should be verified before editing or explaining the feature?

Keep project-specific facts, temporary file inventory, and current app feature flags out of packaged docs. Put those in `AGENTS.md`, `.github/copilot-instructions.md`, or the project code instead.

Use [core-runtime-map.md](./core-runtime-map.md) for Python-side Caspian runtime ownership and [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) for browser-side PulsePoint runtime ownership.

## Redundancy Rule

Prefer one canonical explanation plus links from related docs.

- Put the full authored-template and PulsePoint runtime contract in [pulsepoint.md](./pulsepoint.md).
- Put the full route and layout placement contract in [routing.md](./routing.md).
- Put the full reusable component contract in [components.md](./components.md).
- Put the full Python runtime ownership map in [core-runtime-map.md](./core-runtime-map.md).
- Put the full PulsePoint feature lookup map in [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md).

Other docs should summarize these rules only when the reminder prevents a common mistake, then link to the canonical page instead of restating the whole rule block.

## Docs Location

The packaged Caspian docs referenced by this index live here:

- `node_modules/caspian-utils/dist/docs/`

## Available Documents

- `index.md` - packaged docs manifest and the default first entry point for AI retrieval in this directory
- `ai-validation-checklist.md` - workflow and representative prompts for checking whether AI can find the correct Caspian docs, core files, and verification checkpoints
- `installation.md` - First-time setup flow for creating a new Caspian application
- `commands.md` - Main Caspian CLI workflows for project creation, generation, updates, and config-aware maintenance
- `core-runtime-map.md` - map of `main.py` plus installed `casp` modules to the packaged docs that explain them and the behaviors AI should trace there
- `pulsepoint-runtime-map.md` - fast feature-to-runtime lookup for PulsePoint state, effects, refs, render-performance ownership, context, portals, lists, events, RPC, uploads, streaming, SPA navigation, scroll restoration, and Tailwind merge behavior
- `mcp.md` - MCP-specific layout, launch flow, and AI routing rules for projects where `caspian.config.json` enables MCP
- `database.md` - Prisma schema, migration, seed, and client-generation workflow for projects where `caspian.config.json` enables Prisma, plus Python-side helper caveats
- `auth.md` - Session-backed authentication with `casp.auth`, centralized `auth_config.py`, public-vs-private route mode guidance, RPC-first signout guidance, RBAC, and OAuth provider helpers
- `file-conventions.md` - quick decision guide for `index.py`, `layout.py`, `loading.html`, `not-found.html`, and `error.html`, plus the owning runtime files to verify
- `components.md` - Create reusable Python components, choose component granularity and render ownership, use template-backed or single-file `html(...)` UI, resolve HTML-first `x-*` tags and slots, keep one authored root, and apply the Python-side `merge_classes(...)` contract when Tailwind CSS is enabled
- `pulsepoint.md` - Default reactive frontend runtime contract for component scripts, first-party `on*` events, high-performance state/ref ownership, effects, directives, SPA navigation scroll restoration, `pp-reset-scroll`, and direct browser `twMerge(...)` usage when Tailwind CSS is enabled
- `fetch-data.md` - Initial server-side data loading and browser-triggered RPC flows with `pp.rpc()`, including responsive debounced search, stale-response control, streaming, uploads, and auth-aware actions
- `websockets.md` - WebSocket feature-gate guidance for projects where `caspian.config.json` has `websocket: true`: the named-socket layer (`@socket()` + `pp.socket(...)` over one shared endpoint), the first-frame argument contract, `Socket`/`SocketSender`/`SocketPool` broadcast patterns, per-socket auth and RBAC, origin checks and connection/size/rate/idle limits, the raw-endpoint escape hatch, and when to choose sockets over RPC or SSE
- `file-uploads.md` - Route-local file uploads and file-manager flows with `@rpc()`, `pp.rpc()`, Prisma metadata, public asset storage, and BrowserSync ignore rules
- `state.md` - Request-scoped server state with `StateManager`, session-backed JSON persistence, and listener callbacks for transient flows
- `cache.md` - Route-level HTML caching with `Cache`, `CacheHandler`, TTL behavior, file-system storage, and invalidation patterns
- `validation.md` - Input validation and sanitization with `Validate`, `Rule`, direct field checks, and multi-rule workflows for routes and RPC actions
- `metadata.md` - Static and dynamic metadata, SEO inheritance, and Open Graph or Twitter card tags
- `routing.md` - Next.js App Router-style file-based routing with `src/app`, dynamic segments, dashboard and section layouts, route groups, nested layouts, shared-shell scroll-reset ownership, single-parent authored templates, and backend Python companions that do not own visible markup
- `project-structure.md` - Default Caspian layout and where route files, reusable UI in `src/components/`, reusable non-UI code in `src/lib/`, and database files belong
- `static-export.md` - App-owned static HTML export (SSG, like Next.js `output: export`) via `npm run static`/`settings/build-static.py`, dynamic-route pre-rendering with `static_paths`, and the robust port-aware preview server `npm run static:serve`/`settings/serve-static.py`; not a shipped feature and not gated by a `caspian.config.json` flag, so verify the scripts exist in the project
- `testing.md` - Recommended app-owned testing, type-checking, and linting gate over `main.py` and `src/**` (pyright + ruff + pytest behind one command, so the gate and Pylance in the editor report the same thing), plus the auto-fix command and why ruff must not auto-delete component imports used as `<x-*>` tags (F401 unfixable); not a shipped feature and not gated by a `caspian.config.json` flag, so verify the actual command, tools, and config in the project

## AI Retrieval Notes

If an AI tool needs Caspian documentation, start with this directory and use this file as the manifest.

Preferred lookup order:

1. Read `node_modules/caspian-utils/dist/docs/index.md` to discover available local docs.
2. Read `./caspian.config.json` before making any feature assumption. A doc existing in the package does not mean that feature is enabled in the current project.
3. Before authoring or editing any `src/app/**` or component template, apply the root-shape rule: one authored root by default, any owned plain `<script>` inside that root, and no handwritten `pp-component`. It is a hard invariant for components; a `.py` page or layout may be multi-root.
4. Before inventing browser JavaScript, check whether the interaction is first-party UI behavior. If it is, use PulsePoint `on*` attributes, `pp.state`, refs, effects, directives, and `pp.rpc()` rather than id-driven DOM scripting.
5. When Prisma is enabled, route all Python database work through the generated Prisma Python ORM. Use `database.md` before schema, migration, seed, or ORM decisions, and use `fetch-data.md` before wiring route data or RPC flows.
6. If the task asks for WebSockets, live bidirectional channels, chat/presence/live feeds, socket auth or origin checks, or a native browser `WebSocket` client, confirm `caspian.config.json` has `websocket: true`, read `websockets.md` starting at "Named Sockets", then inspect `src/lib/websocket/sockets.py` (the whole socket surface: registry, `Socket`/`SocketPool`, auth delegation, limits), `main.py` (the single `@app.websocket(SOCKET_PATH)` wiring), the owning `src/app/**` route, and `settings/bs-config.json`. Add a channel as a `@socket()` function consumed by `pp.socket(...)` — do not hand-write an endpoint or a native `WebSocket` unless the wire cannot be JSON frames. If `websocket` is false and the user wants it, ask first, then enable the config flag and follow the update workflow.
7. Treat `caspian.config.json` as the single source of truth for optional features. Use feature-specific docs only when the matching flag is enabled. If a feature is disabled and the user wants it, ask first, then update `caspian.config.json` and follow the update workflow in `commands.md`.
8. If the task touches `main.py` or `.venv/Lib/site-packages/casp/**`, read `core-runtime-map.md` to jump to the controlling runtime file and the matching feature doc.
9. If the task names a PulsePoint feature, directive, slow interaction, unnecessary rerender, or reconciliation concern, read `pulsepoint-runtime-map.md` for the fastest feature-to-owner lookup, then read `pulsepoint.md` for authoring and performance rules.
10. After the feature is confirmed, inspect the actual project files that decide behavior, such as `package.json`, `main.py`, `src/app/**`, `src/lib/**`, `settings/**`, `prisma/**`, and the installed `casp` runtime.
11. Use `file-conventions.md` for quick decisions about `index.py`, `layout.py`, `loading.html`, `not-found.html`, and `error.html`; use `commands.md` for scaffold and update workflows, `project-structure.md` for placement decisions, and the feature docs such as `mcp.md`, `database.md`, `auth.md`, `fetch-data.md`, `websockets.md`, and `file-uploads.md` for task-specific guidance.
12. Prefer packaged Caspian docs before upstream documentation when generating code, commands, or migration guidance.
13. Use `ai-validation-checklist.md` when you want to verify that the docs lead AI to the correct files and behavior checkpoints.
14. Keep `index.md` and cross-links aligned so AI can quickly discover the right doc.
15. If the task is about tests, type checking, linting, a quality gate, CI checks, or making an app production-ready, read `testing.md`. It is an app-owned convention (not a shipped feature and not gated by a `caspian.config.json` flag), so confirm the actual gate command in `package.json`, the orchestrator in `settings/`, and the tools and config in `pyproject.toml` before assuming they exist.
16. If the task is about static export, static-site generation, `output: export`-style output, previewing a static build over HTTP, pre-rendering dynamic routes, or a preview server whose port is already in use, read `static-export.md`. It is also an app-owned convention (not a shipped feature, not gated by a `caspian.config.json` flag), so confirm `settings/build-static.py`, `settings/serve-static.py`, and the `static`/`static:serve` scripts exist in the project before assuming static export is available.

## Maintenance

When adding more pages, keep this index updated with:

- The filename
- A short description of what the page covers

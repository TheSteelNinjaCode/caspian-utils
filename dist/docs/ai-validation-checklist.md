---
title: AI Validation Checklist
description: Use this page when testing whether AI can find the right Caspian docs, core files, performance owner, and verification checkpoints for a task after documentation or runtime changes.
related:
  title: Related docs
  description: Start with the manifest, use the runtime map when needed, then validate against the matching feature guide and owning runtime behavior.
  links:
    - /docs/index
    - /docs/core-runtime-map
    - /docs/routing
    - /docs/auth
    - /docs/fetch-data
    - /docs/pulsepoint
---

This page gives AI a small validation workflow for checking whether the packaged Caspian docs lead to the correct source files and runtime behaviors.

Use it after changing packaged docs, after changing `main.py` or installed `casp` internals, or when you want to verify that AI can resolve a task without drifting across unrelated files.

## Goal

The goal is not to restate feature docs. The goal is to verify four things quickly:

- AI starts from the correct manifest and feature gate.
- AI reaches the correct packaged feature doc.
- AI reaches the controlling app or runtime file.
- AI reaches a concrete behavior checkpoint before making assumptions about behavior.

## Fast Workflow

1. Read [index.md](./index.md) to discover the local Caspian docs.
2. Read `./caspian.config.json` before assuming any optional feature is enabled.
3. If the task touches `main.py` or `.venv/Lib/site-packages/casp/**`, read [core-runtime-map.md](./core-runtime-map.md).
4. Read the matching feature doc such as [routing.md](./routing.md), [auth.md](./auth.md), [fetch-data.md](./fetch-data.md), [pulsepoint.md](./pulsepoint.md), or [file-uploads.md](./file-uploads.md).
5. Verify the claim against the actual project code and the installed runtime.
6. Confirm the relevant behavior in the owning files surfaced by [core-runtime-map.md](./core-runtime-map.md) before treating the behavior as settled.

## Pass Criteria

A documentation path is in good shape when AI can do all of these without broad repo wandering:

- identify whether the feature is enabled from `caspian.config.json`
- choose the correct packaged doc from [index.md](./index.md)
- jump to the controlling runtime file through [core-runtime-map.md](./core-runtime-map.md) when needed
- distinguish app-owned files from installed framework internals
- identify the concrete behavior that must be verified in the owning file
- avoid suggesting changes to `main.py` or `.venv/Lib/site-packages/casp/**` unless the task truly requires runtime edits

## Representative Prompt Suite

Use prompts like these to check whether AI lands on the correct docs and files.

| Prompt | First docs AI should read | Controlling files AI should reach | Verification focus |
| --- | --- | --- | --- |
| Create a protected dashboard section with child routes and a shared shell. | [index.md](./index.md), [routing.md](./routing.md), [auth.md](./auth.md), [project-structure.md](./project-structure.md) | `src/app/**`, `src/lib/auth/auth_config.py`, `main.py`, `.venv/Lib/site-packages/casp/layout.py` | section layout ownership, route privacy mode, and child-route wrapping |
| Make a grouped shell keep sidebar scroll while resetting page content on child-route navigation. | [index.md](./index.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md), [core-runtime-map.md](./core-runtime-map.md) | `src/app/**/layout.py`, `public/js/pp-reactive-v2.min.js`, `main.py` | `pp-reset-scroll` placement, push-vs-history scroll behavior, and shared-shell ownership |
| Show a loading state while a section's child routes navigate. | [index.md](./index.md), [file-conventions.md](./file-conventions.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md) | `src/app/**/loading.py`, the owning `layout.py`, `.venv/Lib/site-packages/casp/loading.py`, `public/js/pp-reactive-v2.min.js` | AI must add a `loading.py` exporting a synchronous parameterless `loading()` and mark the pane with `pp-loading-content="true"` — **not** build a spinner component, an `isLoading` state, or a `pp:navigation:start` listener. Also: closest-ancestor scope resolution, static-markup-only loader, in-page waits kept in component state, and no unrequested loaders added to other subtrees |
| Create a new contact page with interactive form behavior. | [index.md](./index.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md), [components.md](./components.md) | `src/app/**/index.py`, `main.py`, `.venv/Lib/site-packages/casp/components_compiler.py`, `public/js/pp-reactive-v2.min.js` | single-root template shape, plain-script-inside-root authoring, and safe template materialization |
| Add a button, filter, or form interaction to a route template. | [index.md](./index.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) | `src/app/**/index.py`, `public/js/pp-reactive-v2.min.js`, `main.py` | PulsePoint `on*` event attributes, `pp.state`, directives, `Object.fromEntries(new FormData(...).entries())` for simple form submits, and avoiding id-driven `querySelector` or `addEventListener` wiring |
| A debounced server search pauses the first key typed after the timer fires. Determine whether the component or runtime owns the problem. | [index.md](./index.md), [pulsepoint.md](./pulsepoint.md), [components.md](./components.md), [fetch-data.md](./fetch-data.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) | Owning `src/app/**` or `src/components/**`; `public/js/pp-reactive-v2.min.js` only when profiling points to runtime reconciliation | State versus ref ownership, debounce frequency versus render cost, stale-response generations, the smallest useful render owner, and `pp.enablePerf()` evidence before a runtime edit |
| Add a file manager page with upload progress and persisted metadata. | [index.md](./index.md), [file-uploads.md](./file-uploads.md), [fetch-data.md](./fetch-data.md), [database.md](./database.md) | `src/app/**/index.py`, `src/lib/**`, `prisma/schema.prisma` when Prisma applies | route-owned upload actions, persisted metadata flow, and mandatory Prisma Python ORM use for database reads and writes when Prisma is enabled |
| Add `public/icons/app.png` and make it browser-accessible without editing routes. | [index.md](./index.md), [project-structure.md](./project-structure.md), [core-runtime-map.md](./core-runtime-map.md), [auth.md](./auth.md) | `public/**`, `main.py`, `.venv/Lib/site-packages/casp/runtime_security.py` | root-relative URL mapping, no per-directory route or prefix registry, traversal and symlink containment, `GET`/`HEAD` only, and middleware order |
| Store untrusted uploads in a custom public directory. | [index.md](./index.md), [file-uploads.md](./file-uploads.md), [project-structure.md](./project-structure.md), [core-runtime-map.md](./core-runtime-map.md) | `src/app/**/index.py`, `src/lib/**`, `main.py`, `.venv/Lib/site-packages/casp/runtime_security.py` | top-level `inline_safe_subdirectories` registration, attachment mode for HTML/SVG/unknown types, `nosniff`, path containment, and BrowserSync ignore rules |
| Explain how a plain component `<script>` stays inert during template materialization. | [index.md](./index.md), [pulsepoint.md](./pulsepoint.md), [components.md](./components.md), [routing.md](./routing.md) | `main.py`, `.venv/Lib/site-packages/casp/components_compiler.py`, `public/js/pp-reactive-v2.min.js` | component-root deferral, script-source capture, native-execution prevention, and component-scope evaluation |
| Debug why `StateManager` does not persist across a full redirect. | [index.md](./index.md), [state.md](./state.md), [auth.md](./auth.md), [core-runtime-map.md](./core-runtime-map.md) | `main.py`, `.venv/Lib/site-packages/casp/state_manager.py` | wire vs non-wire reset behavior and `request.state.session` dependency |
| Trace the auth cookie and CSRF cookie names used in development. | [index.md](./index.md), [auth.md](./auth.md), [core-runtime-map.md](./core-runtime-map.md) | `main.py`, `.venv/Lib/site-packages/casp/auth.py` | development cookie scoping and CSRF or session naming ownership |
| Find how route params and query params reach `page()`. | [index.md](./index.md), [routing.md](./routing.md), [core-runtime-map.md](./core-runtime-map.md) | `main.py`, `.venv/Lib/site-packages/casp/layout.py` | path-dict delivery, query coercion, and `request` injection |
| Decide whether a dashboard child route should use page caching. | [index.md](./index.md), [cache.md](./cache.md), [routing.md](./routing.md), [fetch-data.md](./fetch-data.md) | `main.py`, `.venv/Lib/site-packages/casp/cache_handler.py`, the route's `index.py` | route-level cache ownership, cacheable HTML scope, and invalidation points |
| Add a PulsePoint context provider to a reusable component. | [index.md](./index.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md), [pulsepoint.md](./pulsepoint.md), [components.md](./components.md) | `public/js/pp-reactive-v2.min.js`, the component's `.py`, `.venv/Lib/site-packages/casp/components_compiler.py` | lowercase HTML-first `*.provider` usage, logical ancestry, single-root template shape |
| Render a conditional panel, a modal, or a table of rows in a route template. | [index.md](./index.md), [pulsepoint.md](./pulsepoint.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md), [routing.md](./routing.md) | `src/app/**/index.py`, `public/js/pp-reactive-v2.min.js` | `hidden="{...}"` for conditionals, `<template pp-for="…">` with `key` for lists, quoted brace attributes, and **no** JSX (`{cond && <div/>}`, `{list.map(...)}`, `class={...}`) |
| Add upload progress to an existing file manager. | [index.md](./index.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md), [file-uploads.md](./file-uploads.md), [fetch-data.md](./fetch-data.md) | `src/app/**/index.py`, `src/lib/**`, `public/js/pp-reactive-v2.min.js` | `pp.rpc` upload options, state replacement, persisted metadata, BrowserSync ignore rules |
| Add or debug a live WebSocket channel. | [index.md](./index.md), [websockets.md](./websockets.md), [core-runtime-map.md](./core-runtime-map.md), [pulsepoint.md](./pulsepoint.md), [auth.md](./auth.md) | `caspian.config.json`, `main.py`, `src/lib/websocket/**`, `src/app/**`, `settings/bs-config.json` | `websocket` feature gate, endpoint registration, origin checks, auth/session handling, native `WebSocket` client lifecycle, and BrowserSync URL selection |
| Decide whether MCP files should be created. | [index.md](./index.md), [mcp.md](./mcp.md), [commands.md](./commands.md), [project-structure.md](./project-structure.md) | `caspian.config.json`, `settings/restart-mcp.ts`, `package.json`, `src/lib/mcp/**` when enabled | `mcp` feature gate, update workflow, nested FastMCP config ownership |

Treat the table as a prompt pack for spot checks, not as a full validation matrix.

## Common Failure Modes

- AI skips `caspian.config.json` and assumes an optional feature is enabled because a packaged doc exists.
- AI reads only the packaged feature doc and never checks `main.py` or the installed runtime.
- AI edits framework internals when the task only requires app-owned route or helper changes.
- AI treats runtime HTML examples as authored template examples.
- AI reads the React comparison in [pulsepoint.md](./pulsepoint.md), [components.md](./components.md), [index.md](./index.md), or [routing.md](./routing.md) as permission to write JSX, and emits `{cond && (<div/>)}`, `{list.map(item => (<tr/>))}`, `className`, `onClick`, or unquoted `class={...}`. The React comparison covers the hook API and component decomposition only. An unquoted brace attribute is invalid HTML: the parser shreds it, the component root never compiles, and the page renders blank with **no console error**. Confirm every brace expression in an attribute is quoted, and that the file would still be valid HTML with all `{}` deleted.
- AI generates a route or component template with a valid-looking root element but leaves a sibling top-level `<script>` or second top-level element after it. This includes single-file `html("""...""")` components where the `<script>` is placed after the root's closing tag instead of nested inside it; the string handed to `html(...)` must resolve to one top-level element, script included.
- AI starts first-party interactivity with ids, `data-*` attributes, `querySelector`, `addEventListener`, manual `fetch`, manual `innerHTML`, or per-input `pp-ref` payload collection instead of PulsePoint `on*` attributes, form submit events, state, directives, and `pp.rpc()`.
- AI assumes every visible pause is a PulsePoint runtime defect and edits reconciliation before proving which setter requested the render. A debounced state setter can still synchronously render a large owner when its timer fires. Keep non-rendering query text, timers, request generations, and cursors in refs; keep rendered results in state; and profile before changing the runtime. The inverse is also wrong: a value the template must display belongs in state, not a ref.
- AI bypasses the generated Prisma Python ORM in a Prisma-enabled project with direct database drivers, custom fetch helpers, JSON active stores, browser-side database fetches, or a second app-owned database abstraction.
- AI extracts one-route backend logic into `src/lib/**` even though the logic is not shared; route-specific first-render queries, RPC actions, validation, and response shaping belong in the owning `src/app/**/index.py`.
- AI hand-builds a *requested* navigation loading state — a spinner component, a global `isLoading` store, a `pp:navigation:start` listener, a manual overlay — when `loading.py` already provides it. (The opposite over-correction also counts: scattering unrequested `loading.py` files through subtrees that never asked for one. The file is optional; no loader means a plain fade.) The same reflex produces hand-rolled subtree shells instead of `layout.py`, and in-route 404/500 branches instead of `not_found.py` / `error.py`. Before building any of these, check [file-conventions.md](./file-conventions.md) for the file that already owns the behavior. The inverse error also counts: routing an in-page wait (an `@rpc()` call, a submit, a filter) through `loading.py`, which only ever fires on route-to-route navigation.
- AI puts `<x-*>` component tags, `{ }` bindings, or a `<script>` inside `loading.py`. The collected loader is injected as raw HTML and never mounted, so those render as literal text or nothing. Loaders are plain elements and CSS only.
- AI puts `pp-reset-scroll="true"` on the whole shell or `body` when only the page-content pane should reset, causing persistent sidebars or rails to lose their scroll position.
- AI decides behavior from memory without checking the owning implementation details.

## Decision Rule

- If the task is feature-oriented, start from [index.md](./index.md) and the matching feature doc.
- If the task is behavior-oriented and the owner is unclear, jump to [core-runtime-map.md](./core-runtime-map.md).
- If the task is still ambiguous after that, verify the controlling file and the specific behavior in scope before widening the search.

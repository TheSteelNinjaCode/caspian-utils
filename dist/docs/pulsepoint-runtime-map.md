---
title: PulsePoint Runtime Map
description: Use this page when AI needs a fast feature-to-runtime lookup for PulsePoint behavior before editing `src/app/**`, component templates, or `public/js/pp-reactive-v2.js`.
related:
  title: Related docs
  description: Use the PulsePoint guide for authoring rules, the routing and component guides for template placement, and the core runtime map when Python-side transforms are involved.
  links:
    - /docs/pulsepoint
    - /docs/routing
    - /docs/components
    - /docs/fetch-data
    - /docs/core-runtime-map
    - /docs/index
---

This page is the quick AI routing layer for PulsePoint core features.

Use [pulsepoint.md](./pulsepoint.md) for the full authoring contract. Use this file when you need to jump from a feature name, directive, or runtime symptom to the files and behavior checkpoints that matter.

## Source Of Truth

- `public/js/pp-reactive-v2.js` is the shipped browser runtime.
- `main.py` owns the final render pipeline that calls `transform_components(...)` and `transform_scripts(...)`.
- `.venv/Lib/site-packages/casp/components_compiler.py` injects `pp-component` after component expansion and validates the single-root contract.
- `.venv/Lib/site-packages/casp/scripts_type.py` rewrites authored body `<script>` tags to `type="text/pp"`.
- `.venv/Lib/site-packages/casp/html_attrs.py` owns Python-side class and attribute helpers such as `merge_classes(...)`.

If an inspected browser DOM disagrees with authored template source, remember that the runtime DOM includes framework-managed output. Do not copy runtime-only attributes back into authored templates.

## Feature Map

| PulsePoint feature | Authoring surface | Runtime owner | Verify before changing |
| --- | --- | --- | --- |
| Component roots | `src/app/**/index.html`, `layout.html`, component `.html` files | `components_compiler.py`, `pp-reactive-v2.js` | one authored root, final expanded root receives one `pp-component`, no sibling scripts |
| Component scripts | plain `<script>` inside the authored root | `scripts_type.py`, `pp-reactive-v2.js` | authored scripts are plain, runtime scripts become `type="text/pp"`, one owned script per root |
| Template expressions | text and attributes with `{...}` | `pp-reactive-v2.js` | top-level script bindings are exported, nested bindings are not assumed |
| State | `pp.state(initial)` | `pp-reactive-v2.js` | setters accept values or updater functions, state belongs to the component instance |
| Effects | `pp.effect(...)`, `pp.layoutEffect(...)` | `pp-reactive-v2.js` | callbacks may return cleanup functions, promises are not awaited |
| Refs | `pp.ref(...)`, `pp-ref` | `pp-reactive-v2.js` | generated ref internals are runtime-managed; do not author `data-pp-ref` |
| Context | `pp.createContext(...)`, lowercase `<themecontext.provider>`, `pp.context(token)` | `TemplateCompiler.ts`, `NestedBoundaryManager.ts`, `pp-reactive-v2.js` | authored provider tags are HTML-first and lowercase; `TemplateCompiler.transformContextProviderTags(...)` rewrites `*.provider` to runtime-owned `pp-context-provider`; ancestry is logical component ancestry; do not invent `pp-context` or `pp.provideContext` |
| Portals | `pp.portal(ref, target?)` | `pp-reactive-v2.js` | context should preserve logical ancestry through the registry |
| Lists | `<template pp-for="item in items">` | `pp-reactive-v2.js` | `pp-for` belongs on `<template>`, use plain `key`, not `pp-key` |
| Events | native `onclick`, `oninput`, `onchange`, `onsubmit` | `pp-reactive-v2.js` | first-party events belong in `on*` attributes; event scope exposes `event`, `e`, `$event`, `target`, `currentTarget`, and `el`; normal form submits should use `Object.fromEntries(new FormData(event.currentTarget).entries())` instead of per-input refs; avoid id-driven `querySelector`/`addEventListener` for normal UI |
| RPC | `pp.rpc(...)` in scripts, `@rpc()` in Python | `pp-reactive-v2.js`, `casp/rpc.py` | use `pp.rpc`, not legacy `pp.fetchFunction`; protected actions use `@rpc(require_auth=True)` |
| Upload progress | `pp.rpc(..., { onUploadProgress })` | `pp-reactive-v2.js`, `casp/rpc.py` | XHR path is used for progress callbacks; replace state from returned payload |
| Streaming | `pp.rpc(..., { onStream })` | `pp-reactive-v2.js`, `casp/rpc.py`, `casp/streaming.py` | server generators become SSE responses |
| SPA navigation | `body[pp-spa="true"]`, links | `pp-reactive-v2.js`, `main.py` | same-origin eligible links intercept; root-layout mismatches hard reload |
| Scroll restoration | `pp-reset-scroll="true"` | `pp-reactive-v2.js` | push navigation resets window; mark only content panes that should reset |
| Tailwind merge | `{twMerge(...)}`, Python `merge_classes(...)` | `html_attrs.py`, `pp-reactive-v2.js` | Python emits frontend-ready expressions when Tailwind is enabled |

## AI Decision Rules

- Read `caspian.config.json` first when the task depends on Tailwind, generated files, or optional project features.
- Read [routing.md](./routing.md) before adding route or layout templates.
- Read [components.md](./components.md) before adding reusable Python components or `x-*` imports.
- Read [fetch-data.md](./fetch-data.md) before adding browser-triggered backend work.
- Use this map when the task names a PulsePoint feature and you need the owning runtime file quickly.
- Verify implemented behavior in `public/js/pp-reactive-v2.js` before adding new PulsePoint API claims.
- If an interaction is normal first-party HTML behavior, route it through PulsePoint before considering standard DOM scripting.

## Copy-Safe Authoring Rules

- Author one root element or one imported `x-*` root per route, layout, or component template.
- Keep any owned plain `<script>` inside that same root.
- Do not handwrite `pp-component`, `type="text/pp"`, `data-pp-ref`, `pp-owner`, `pp-event-owner`, or other runtime-managed attributes.
- Use `pp.rpc(...)` for current browser-to-server calls.
- Use native `on*` attributes for button clicks, form submits, input changes, filters, toggles, and menus instead of adding ids and manual listeners.
- For ordinary form submits, bind `onsubmit` on the form and read named fields with `Object.fromEntries(new FormData(event.currentTarget).entries())`; reserve `pp-ref` for imperative access rather than routine payload collection.
- Use lowercase provider tags such as `<themecontext.provider>` and `pp.context(...)` for context.
- Use `pp-for` only on `<template>` and plain `key` for keyed lists.
- Prefer PulsePoint state and directives over manual `innerHTML` repainting.
- Keep direct DOM APIs inside `pp.ref(...)` plus `pp.effect(...)` only when a third-party or browser API integration actually requires them.

## Compact Examples

Context provider:

```html
<section>
  <script>
    const ThemeContext = pp.createContext("light");
    const [theme, setTheme] = pp.state("dark");
  </script>

  <themecontext.provider value="{theme}">
    <button onclick="setTheme(theme === 'dark' ? 'light' : 'dark')">
      Theme: {theme}
    </button>
  </themecontext.provider>
</section>
```

Upload progress:

```html
<section>
  <input type="file" onchange="{uploadFile(event.target.files?.[0])}" />
  <p>{progress}%</p>

  <script>
    const [progress, setProgress] = pp.state(0);

    async function uploadFile(file) {
      if (!file) return;

      await pp.rpc("upload_asset", { file }, {
        onUploadProgress: (event) => setProgress(event.percentage ?? 0),
      });
    }
  </script>
</section>
```

Grouped shell scroll reset:

```html
<section class="dashboard-shell">
  <aside class="dashboard-sidebar">...</aside>
  <main class="dashboard-content" pp-reset-scroll="true">
    <slot />
  </main>
</section>
```

## Verification Prompts

Use these prompts after docs or runtime changes to confirm AI can route correctly:

- "Create an interactive filter in a Caspian route template."
- "Explain why authored scripts are plain `<script>` but browser output shows `type=\"text/pp\"`."
- "Debug a dashboard sidebar losing scroll during child-route navigation."
- "Add an upload widget with progress and a reactive file list."
- "Use context to share theme state between parent and child components."

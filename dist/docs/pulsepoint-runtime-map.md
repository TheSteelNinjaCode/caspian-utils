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
| Component roots | `src/app/**/index.html`, `layout.html`, component `.html` files | `components_compiler.py`, `public/js/pp-reactive-v2.js` | one authored root, final expanded root receives one `pp-component`, no sibling scripts |
| Component scripts | plain `<script>` inside the authored root | `scripts_type.py`, `public/js/pp-reactive-v2.js` | authored scripts are plain, runtime scripts become `type="text/pp"`, one owned script per root |
| Template expressions | text and attributes with `{...}` | `public/js/pp-reactive-v2.js` | top-level script bindings are exported, nested bindings are not; every destructuring shape is supported at the top level (array, object, nested, defaults, rest, holes), so any tuple-returning hook reaches template scope |
| State | `pp.state(initial)` | `public/js/pp-reactive-v2.js` | setters accept values or updater functions, state belongs to the component instance |
| Effects | `pp.effect(...)`, `pp.layoutEffect(...)` | `public/js/pp-reactive-v2.js` | callbacks may return cleanup functions, promises are not awaited |
| Refs | `pp.ref(...)`, `pp-ref` on native or `x-*` tags | `components_compiler.py`, `TemplateCompiler.ts`, `Component.ts`, `RefBindingManager.ts`, shipped in `public/js/pp-reactive-v2.js` | component-tag refs are reserved by the Python compiler, stamped with runtime-owned `pp-ref-owner`, captured in the parent's scope, and bound to the child's concrete root; do not author `data-pp-ref`, `pp-ref-owner`, or `pp-ref-forward`; callback refs may return synchronous cleanup functions that run on replacement or detach |
| Context | `pp.createContext(...)`, lowercase `<themecontext.provider>`, `pp.context(token)` | `TemplateCompiler.ts`, `NestedBoundaryManager.ts`, `public/js/pp-reactive-v2.js` | authored provider tags are HTML-first and lowercase; `TemplateCompiler.transformContextProviderTags(...)` rewrites `*.provider` to runtime-owned `pp-context-provider`; ancestry is logical component ancestry; do not invent `pp-context` or `pp.provideContext` |
| Portals | `pp.portal(ref, target?)` | `public/js/pp-reactive-v2.js` | context should preserve logical ancestry through the registry |
| Props into a component | `x-*` attributes, Python `get_attributes({...}, props)` + `{{ attributes }}` | `html_attrs.py`, `components_compiler.py`, `public/js/pp-reactive-v2.js` | `pp.props` is read from the **rendered root's attributes**, so a prop accepted in Python but not re-emitted by `get_attributes` is silently `undefined` — no server error, no console warning. Brace expressions (`volume="{vol}"`) keep their real type; literals arrive as strings (`volume="0"` makes `volume === 0` false); valueless attributes arrive as `true`; JS reserved words such as `class` are dropped. See [components.md](./components.md#every-prop-a-template-reads-must-be-forwarded-to-the-root) |
| Stable ids | `pp.id()` | `public/js/pp-reactive-v2.js` | derived from component id plus hook slot; stable across rerenders, unique per instance; use for `id`/`for` pairing and `aria-*` instead of counters |
| External stores | `pp.syncExternalStore(subscribe, getSnapshot)` | `public/js/pp-reactive-v2.js` | `subscribe` must be stable — wrap in `pp.callback(..., [])`; runtime re-reads the snapshot right after subscribing to close the render-to-subscribe gap; cleanup runs on dispose |
| Imperative handles | `pp.imperativeHandle(ref, createHandle, deps?)` | `public/js/pp-reactive-v2.js` | parent passes its own `pp.ref(...)` down as a normal prop (object props pass by reference); do not author `pp-ref-forward`; handle publishes in the layout-effect phase and clears on unmount |
| Transitions | `pp.transition()` | `public/js/pp-reactive-v2.js` | returns `[isPending, startTransition]`; not concurrent — rendering is still synchronous; handles sync and promise scopes; overlapping scopes stay pending until the last settles |
| Deferred values | `pp.deferredValue(value, initialValue?)` | `public/js/pp-reactive-v2.js` | lags one commit behind the source |
| Optimistic updates | `pp.optimistic(passthrough, reducer?)` | `public/js/pp-reactive-v2.js` | pending actions are dropped when `passthrough` changes, so a confirmed server value never double-counts the guess; without a reducer each action replaces the value |
| Error boundaries | `pp.errorBoundary()` | `public/js/pp-reactive-v2.js` | returns `[error, reset]`; render **and** effect/cleanup throws walk logical component ancestry to the nearest boundary; a boundary also catches its own throws; latches until `reset()`; caps at 5 captures per arming; event-handler errors are not routed — use `try`/`catch` |
| Lists | `<template pp-for="item in items">` | `public/js/pp-reactive-v2.js` | `pp-for` belongs on `<template>`, accepts arrays and synchronous iterables, captures rendered row values for events/props, and uses plain `key`, not `pp-key` |
| Events | native `onclick`, `oninput`, `onchange`, `onsubmit` | `public/js/pp-reactive-v2.js` | first-party events belong in `on*` attributes; event scope exposes `event`, `e`, `$event`, `target`, `currentTarget`, and `el`; normal form submits should use `Object.fromEntries(new FormData(event.currentTarget).entries())` instead of per-input refs; avoid id-driven `querySelector`/`addEventListener` for normal UI |
| RPC | `pp.rpc(...)` in scripts, `@rpc()` in Python | `public/js/pp-reactive-v2.js`, `casp/rpc.py` | use `pp.rpc`, not legacy `pp.fetchFunction`; protected actions use `@rpc(require_auth=True)` |
| Upload progress | `pp.rpc(..., { onUploadProgress })` | `public/js/pp-reactive-v2.js`, `casp/rpc.py` | XHR path is used for progress callbacks; the callback receives `{ loaded, total, percent }` and `percent` can be `null` when length is not computable; replace state from returned payload |
| Streaming | `pp.rpc(..., { onStream })` | `public/js/pp-reactive-v2.js`, `casp/rpc.py`, `casp/streaming.py` | server generators become SSE responses |
| SPA navigation | `body[pp-spa="true"]`, links | `public/js/pp-reactive-v2.js`, `main.py` | same-origin eligible links intercept; root-layout mismatches hard reload |
| Scroll restoration | `pp-reset-scroll="true"` | `public/js/pp-reactive-v2.js` | push navigation resets window; mark only content panes that should reset |
| Tailwind merge | `{twMerge(...)}`, Python `merge_classes(...)` | `html_attrs.py`, `public/js/pp-reactive-v2.js` | Python emits frontend-ready expressions when Tailwind is enabled |
| Markup deferral | `<template pp-component>` / `<template pp-owner>` wrappers | render pipeline (`main.py`), `public/js/pp-reactive-v2.js` (`materializeTemplateComponentBoundaries`) | top-level roots ship inside an inert `<template>`; runtime materializes them on mount and SPA navigation before scanning. Browser never validates `<template>` contents, so `{...}` is safe in any attribute/position (SVG geometry, `src`/`href`, form `value`/date, table/select text) — no per-tag workarounds. See [pulsepoint.md](./pulsepoint.md) "Component markup is deferred inside an inert `<template>`" |

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
- Use `pp.id()` for any generated `id`, `for`, `aria-labelledby`, or `aria-describedby` value instead of a loop index or module-level counter.
- Use `pp.syncExternalStore(...)` for browser or global sources the component does not own (`matchMedia`, `localStorage`, `navigator.onLine`, a shared store) instead of `pp.state` plus `pp.effect`.
- Wrap a component subtree that can fail in a parent with `pp.errorBoundary()` rather than leaving a render throw to the console.
- In a Python component, list every prop the template's `{...}` expressions reference in `get_attributes({...}, props)`. If a binding "does nothing" — an icon toggle that never switches, a `hidden` that never applies — check that prop is on the rendered root before debugging the expression.
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
        onUploadProgress: (event) => setProgress(Math.round(event.percent ?? 0)),
      });
    }
  </script>
</section>
```

Error boundary with fallback and retry:

```html
<section>
  <script>
    const [error, reset] = pp.errorBoundary();
  </script>

  <div hidden="{!error}">
    <p>Could not load this panel: {error?.message}</p>
    <button onclick="{reset()}">Retry</button>
  </div>

  <div hidden="{!!error}">
    <x-revenue-chart />
  </div>
</section>
```

Accessible field pairing with stable ids:

```html
<div>
  <label for="{emailId}">Email</label>
  <input id="{emailId}" type="email" aria-describedby="{hintId}" />
  <p id="{hintId}">We never share this.</p>

  <script>
    const emailId = pp.id();
    const hintId = pp.id();
  </script>
</div>
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

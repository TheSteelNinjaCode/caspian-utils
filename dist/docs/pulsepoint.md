---
title: PulsePoint Runtime Guide
description: Learn how AI agents should use PulsePoint as the default reactive frontend contract in Caspian, including the bundled runtime loaded from `public/js/main.js` and implemented in `public/js/pp-reactive-v2.js`, component script rules, and supported directives.
related:
  title: Related docs
  description: Read the components, routing, data-fetching, and project-structure docs alongside the PulsePoint runtime contract.
  links:
    - /docs/components
    - /docs/routing
    - /docs/fetch-data
    - /docs/project-structure
    - /docs/index
---

## Purpose

This file documents the PulsePoint runtime that is currently shipped in `public/js/pp-reactive-v2.js` and loaded by `public/js/main.js`. Treat it as the working contract for AI-generated code.

If a task involves `pp.state`, `pp.effect`, `pp.layoutEffect`, `pp-ref`, `pp-spread`, `pp-for`, context, portals, SPA navigation, or component boundary behavior, read this page first and keep generated code aligned with the runtime implemented in this repo.

Use `components.md` for authoring Python `@component` files and same-name HTML templates. Use this page for the browser-side `pp-component` contract and `script[type="text/pp"]` behavior after those templates are rendered.

PulsePoint is the default reactive frontend layer for Caspian.

Do not assume React, Vue, Svelte, JSX, Alpine, HTMX, or older PulsePoint docs unless the task explicitly asks for a different frontend contract.

## Default Frontend Rule

When a Caspian page needs reactive browser behavior, use PulsePoint.

- Use PulsePoint component roots, scripts, directives, and runtime helpers for interactive UI.
- Keep server-rendered HTML plus PulsePoint enhancement as the baseline architecture.
- Only introduce another frontend runtime when the user explicitly asks for it or the project already depends on one.

## Source layer vs raw runtime

- Source authoring under `src/` may use convenience syntax that is transformed before it reaches the browser.
- This page describes the bundled browser runtime shipped in `public/js/pp-reactive-v2.js`.
- When source-layer examples conflict with the bundled runtime, follow the bundled runtime.
- In authored route, layout, and component templates, do not add `pp-component` manually. The Python side injects it onto the template root during render.
- In authored templates, write PulsePoint logic in a plain `<script>` inside that root. `main.py` runs `transform_scripts(...)`, so the runtime receives `script[type="text/pp"]`.
- When reading or debugging runtime HTML, look for `pp-component` roots and `script[type="text/pp"]`.

## Runtime shape

PulsePoint is a browser-side component runtime. It executes one component script per root, renders HTML from the component scope, morphs the DOM in place, binds native event handlers, restores focus, updates refs, applies portals, and then runs layout/effect hooks.

A component root is any element with `pp-component`.

Important:

- `pp-component` is the instance id used for the component registry, saved state, cached template, parent tracking, and instance lookup.
- In normal Caspian authoring, `pp-component` arrives from the Python render pipeline on route, layout, and component roots rather than being handwritten in source templates.
- Reusing the same `pp-component` value for multiple live roots will destroy the previous registered instance and replace it with the new one.
- If a root is manually constructed with an empty `pp-component`, the runtime will assign an anonymous id. AI-generated markup should still use stable unique ids.
- `pp.mount()` bootstraps every `[pp-component]` in the document, so do not manually instantiate `Component` in authored app code.

## Component roots and scripts

- Each runtime component root should have at most one `script[type="text/pp"]` block.
- The runtime only treats `script[type="text/pp"]` as component logic.
- The script lookup walks the current root and skips nested `pp-component` boundaries, so a parent does not consume a child component's script.
- If multiple matching scripts exist in the same root, the first matching owned script wins. Generate one script per root.
- Authored component and route HTML templates must have exactly one top-level lowercase HTML root because the compiler injects `pp-component` onto that root during render. Think React-style single parent wrapper, not sibling top-level tags.
- In authored Caspian templates, write plain `<script>` inside the single root and let the render pipeline rewrite it before mount.
- A scriptless component root still mounts and can receive props, refs, events, and nested children, but it has no local runtime scope beyond its props.
- Component scripts are plain JavaScript executed with `new Function(...)`. Do not use `import`, `export`, or top-level `await` inside them.
- The runtime auto-returns supported top-level bindings from the script. Do not rely on manual `return { ... }` objects.
- `pp.props` contains the current prop bag for the component.
- `pp.props.children` contains the root's initial inner HTML before the owning `script` block is removed from the render template.

Bindings exported to the template:

- Top-level function declarations.
- Top-level identifier declarations such as `const title = "Pulse"`.
- Top-level object destructuring identifiers such as `const { title, subtitle } = pp.props`.
- Top-level array destructuring identifiers when the initializer is `pp.state(...)`, such as `const [count, setCount] = pp.state(0)`.

Bindings that should not be assumed:

- Generic top-level array destructuring is not auto-exported unless it comes from `pp.state(...)`.
- Nested declarations inside functions, conditions, loops, and callbacks are not auto-exported.
- There is no standalone `props` variable injected by the runtime. Use `pp.props`.

Example:

```html
<div pp-component="counter-card-1">
  <h2>{title}</h2>
  <p>Count: {count}</p>
  <button onclick="setCount(count + 1)">Increment</button>

  <script type="text/pp">
    const { title } = pp.props;
    const [count, setCount] = pp.state(0);

    function reset() {
      setCount(0);
    }
  </script>
</div>
```

That example shows runtime HTML after mountable roots already exist. In authored Caspian templates, you normally write the root without `pp-component` and keep the logic in a plain `<script>` so the Python render path can inject both runtime attributes before the browser runtime sees the HTML.

## Hooks and runtime API

Hooks exposed inside component scripts through `pp`:

- `pp.state(initial)` returns `[value, setValue]`.
- `pp.effect(callback, deps?)` runs after render and may return a cleanup function.
- `pp.layoutEffect(callback, deps?)` runs synchronously after DOM mutation/ref binding/portal application and before `pp.effect`.
- `pp.ref(initialValue?)` returns `{ current }`.
- `pp.memo(factory, deps)` memoizes a computed value.
- `pp.callback(callback, deps)` memoizes a function.
- `pp.reducer(reducer, initialState)` returns `[state, dispatch]`.
- `pp.context(token)` resolves a provided context value from ancestor components.
- `pp.provideContext(token, value)` provides a context value to descendant components.
- `pp.portal(ref, target?)` registers a ref-managed element for portal rendering and returns an object that includes `sourceParent`.
- `pp.props` exposes the current props.

Global helpers exposed on the `pp` singleton:

- `pp.createContext(defaultValue)` creates a context token.
- `pp.mount()` bootstraps the runtime. It is idempotent.
- `pp.redirect(url)` performs SPA-aware navigation when enabled.
- `pp.rpc(name, data?, optionsOrAbort?)` performs the current route RPC bridge.
- `pp.enablePerf()` enables render timing collection.
- `pp.disablePerf()` disables render timing collection.
- `pp.getPerfStats()` returns collected render timings.
- `pp.resetPerfStats()` clears collected render timings.

Notes:

- The global `pp` singleton auto-mounts once `public/js/main.js` loads the bundled runtime and the DOM is ready. Manual `pp.mount()` is still safe because it short-circuits after the first mount.
- `pp.state` setters accept either a value or an updater function.
- `pp.effect` and `pp.layoutEffect` are cleanup-style hooks. Returned promises are not awaited.
- `pp.portal(ref)` defaults to `document.body` when no target is provided.
- Older docs may call the RPC helper `pp.fetchFunction()`. In the current bundled runtime the implemented global API is `pp.rpc()`.
- Keep template-facing bindings at the top level so the AST-based exporter can see them.

## Context

Context is implemented in the current runtime.

How it works:

- Create a token with `pp.createContext(defaultValue)`.
- Provide a value from a component with `pp.provideContext(token, value)`.
- Read it in a descendant component with `pp.context(token)`.
- Resolution walks the logical component parent chain stored in the registry, not the live DOM.
- Portaled descendants still resolve providers through component ancestry.
- Descendant consumers rerender when a provided value changes.
- Unchanged provided values do not trigger consumer refreshes.
- If a provider stops providing a context or is destroyed, consumers fall back to the next ancestor provider or the token default.

Important:

- The same token object must be shared between provider and consumer.
- Context is component-level, not directive-based. There is no `pp-context` attribute.
- `pp.context(token)` resolves from ancestors. A component does not consume the value it provides in the same render.
- If provider and consumer live in different component script scopes, pass the token through props or store it in shared global/outer state.

Example:

```html
<section pp-component="theme-provider-1">
  <div pp-component="theme-label-1" theme-token="{ThemeContext}">
    <p>{theme}</p>

    <script>
      const { themeToken } = pp.props;
      const theme = pp.context(themeToken);
    </script>
  </div>

  <script>
    const ThemeContext = pp.createContext("light");
    const [theme] = pp.state("dark");

    pp.provideContext(ThemeContext, theme);
  </script>
</section>
```

## Props and nested components

- Child component props are derived from DOM attributes.
- Attribute names are converted from kebab-case to camelCase for the prop bag.
- Native `on*` attributes and `pp-component` are not included in props.
- Empty attributes become boolean `true` props.
- Pure prop expressions such as `title="{pageTitle}"` are evaluated in the parent scope.
- Mixed strings such as `class="card {isActive ? 'active' : ''}"` are interpolated in the parent scope.
- Non-expression attribute values are passed through as strings.
- `children` is injected into props using the root's initial inner HTML.

Nested components:

- Nested `pp-component` roots are treated as component boundaries during DOM reconciliation.
- Parent prop interpolation still runs on nested child root attributes before the child refreshes.
- Nested roots that contain their own `script` block are masked during parent template compilation.
- Scriptless nested component roots are not fully masked during parent template compilation in the current source. Avoid generating child-local interpolations inside a nested root unless that child has its own `script` block.

## Template expressions and attributes

- Use `{expression}` in text nodes and attribute values.
- Pure bindings like `value="{count}"` are evaluated as expressions.
- Mixed text like `class="card {isActive ? 'active' : ''}"` is supported.
- Arrays in template expressions are joined without commas.
- `null`, `undefined`, and boolean expression results render as an empty string in text output.
- Supported boolean attributes are normalized so truthy values emit the bare attribute and falsy values remove it.
- `<textarea value="{draft}"></textarea>` is normalized into textarea content.
- Use `pp-spread="{...attrs}"` to spread an object expression into attributes.
- `pp-spread` omits nullish values and escapes `&`, `"`, and `<` in emitted attribute values.
- Use plain `key` for keyed diffing. `pp-key` is not implemented.

Example:

```html
<div pp-component="profile-card-1">
  <button pp-spread="{...buttonAttrs}" hidden="{isLoading}">Save</button>

  <script>
    const buttonAttrs = {
      class: "btn btn-primary",
      "aria-label": "save",
    };

    const isLoading = false;
  </script>
</div>
```

## Refs

- Use `pp-ref="nameInput"` when the ref object or callback is already available in scope.
- Use `pp-ref="{registerRef(id)}"` when you want the compiler to capture a dynamic ref expression.
- `pp.ref(null)` is the normal way to create a ref object.
- Callback refs and `{ current }` refs are both supported.
- Captured brace-form refs are compiled into an internal `data-pp-ref` token and rebound after render.
- Plain `pp-ref` bindings are preserved across rerenders, including no-op rerenders that skip DOM diffing.
- Ref callbacks may be called with `null` during cleanup.
- The runtime generates `data-pp-ref` internally. Do not author it.
- Do not author `pp-event-owner`, `pp-owner`, or `pp-dynamic-*` attributes by hand.

Example:

```html
<div pp-component="focus-box-1">
  <input pp-ref="nameInput" />
  <button onclick="nameInput.current?.focus()">Focus</button>

  <script>
    const nameInput = pp.ref(null);
  </script>
</div>
```

## Lists and keyed diffing

- Use `pp-for` only on `<template>`.
- Supported forms are `item in items` and `(item, index) in items`.
- The collection must be an array. Non-arrays are treated as empty lists.
- Loop content can contain interpolations, events, refs, and nested components.
- Event handlers inside loops are rewritten so loop variables still resolve after rerender.
- Use stable unique `key` values on repeated sibling elements.
- Duplicate keys trigger warnings and reduce diff quality.
- Keyed reconciliation preserves DOM identity across reorders and insertions.

Example:

```html
<div pp-component="todo-list-1">
  <ul>
    <template pp-for="(todo, index) in todos">
      <li key="{todo.id}">
        {index + 1}. {todo.title}
        <button onclick="removeTodo(todo.id)">Remove</button>
      </li>
    </template>
  </ul>

  <script>
    const [todos, setTodos] = pp.state([
      { id: 1, title: "First task" },
      { id: 2, title: "Second task" },
    ]);

    function removeTodo(id) {
      setTodos(todos.filter((todo) => todo.id !== id));
    }
  </script>
</div>
```

## Events

- Use native `on*` attributes such as `onclick`, `oninput`, and `onsubmit`.
- Event values may be raw code or wrapped in `{...}`.
- The runtime injects `event`, `e`, `$event`, `target`, `currentTarget`, and `el`.
- Do not use hyphenated event attrs like `on-click`.
- Event attributes are removed from the live DOM after binding and rebound after DOM morphing.
- Owned template/event-owner internals are runtime-managed. Do not author them directly.

## SPA, loading, and navigation helpers

- `body[pp-spa="true"]` enables client-side navigation interception.
- `a[pp-spa="false"]` disables interception for that link.
- External links, downloads, `_blank`, and modifier-key clicks bypass SPA interception.
- `pp-loading-content="true"` marks the page region that gets swapped or faded during navigation.
- Route-specific loading states are looked up with `pp-loading-url`.
- `pp-loading-transition` accepts JSON with `fadeIn` and `fadeOut` timing values.
- `pp-reset-scroll="true"` resets scroll positions during navigation.
- Navigation dispatches `pp:navigation:start`, `pp:navigation:complete`, and `pp:navigation:error` events on `document`.

RPC notes:

- `pp.rpc(name, data?, optionsOrAbort?)` posts to the current route.
- Passing `true` as the third argument means `abortPrevious: true`.
- The options object supports `abortPrevious`, `onStream`, `onStreamError`, `onStreamComplete`, `onUploadProgress`, and `onUploadComplete`.
- File uploads switch to the XHR path when upload progress callbacks are needed.
- Streamed `text/event-stream` responses are supported when a stream handler is provided.
- Redirect headers are honored through `pp.redirect()`.

Notes:

- `pp.redirect()` uses SPA navigation for same-origin URLs when SPA mode is enabled. Otherwise it falls back to normal navigation.
- Root-layout mismatches during SPA navigation trigger a hard reload.
- `pp.mount()` bootstraps every `[pp-component]` it finds, so generated code should call it only through the global runtime if you are manually mounting at all.

## AI rules

Use these rules when generating or editing PulsePoint runtime code:

- Treat PulsePoint as the default reactive frontend for Caspian app code.
- Use the bundled runtime contract in `public/js/pp-reactive-v2.js`.
- In authored Caspian templates, do not handwrite `pp-component` or `type="text/pp"`; let the render pipeline inject them.
- If you are explicitly editing raw runtime HTML or internals, keep `pp-component` unique per live instance.
- In authored templates, use a plain `<script>` inside the root. In runtime HTML, the owned script appears as `script[type="text/pp"]`.
- Keep template-facing variables at top level.
- Use `pp.createContext`, `pp.context`, and `pp.provideContext` for context. Do not invent `pp-context`.
- Keep `pp-for` on `<template>` and use plain `key`.
- Use native `on*` attributes, not framework-specific event syntax.
- Use refs and portals only through the implemented `pp` APIs.
- Use `pp.rpc()` for the bundled runtime API instead of older `pp.fetchFunction()` wording.
- Avoid generating internal runtime attributes.
- Avoid scriptless nested components when the child template contains its own bindings.

## What to avoid

Do not generate these unless the current source explicitly adds support:

- React, Vue, Svelte, Alpine, HTMX, or JSX-first patterns as the default Caspian frontend approach
- `pp-context`
- `pp-key`
- `data-pp-ref`
- `pp-owner`
- `pp-event-owner`
- `pp-dynamic-script`
- `pp-dynamic-meta`
- `pp-dynamic-link`
- handwritten `pp-component="..."` or `type="text/pp"` in authored route, layout, or component templates
- `pp.fetchFunction()` as the current raw runtime helper name
- made-up hooks, directives, or globals not present in the current bundled runtime

## Review notes

These are current runtime caveats that matter for authors and AI tools:

- `pp-component` is the registry key for instances, state, parent tracking, and templates. Treat it as unique per mounted root.
- Nested roots without their own `script[type="text/pp"]` block are not fully isolated during parent template compilation.
- The global `pp` singleton auto-mounts on DOM ready, and `pp.mount()` is idempotent.
- `pp.effect` and `pp.layoutEffect` are cleanup-style hooks. Their callbacks are not promise-aware.
- `pp.context()` resolves through ancestor components, not the current component's own pending providers.
- `pp.portal()` preserves logical ancestry through the registry, so context and prop refresh behavior continue to work through portaled descendants.

## Final reminder

If a feature is not implemented in the current bundled runtime, do not invent it.

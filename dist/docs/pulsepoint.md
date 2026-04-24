---
title: PulsePoint Runtime Guide
description: Use this page when the task mentions PulsePoint, `pp.state`, `pp.effect`, `pp-ref`, `pp-style`, `pp-for`, portals, SPA navigation, or `public/js/pp-reactive-v2.js`.
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

This file documents the PulsePoint contract for the shipped Caspian browser runtime. Treat it as the AI-facing source of truth when generating or reviewing interactive Caspian UI.

If a task involves `pp.state`, `pp.effect`, `pp.layoutEffect`, `pp-ref`, `pp-style`, `pp-spread`, `pp-for`, context, portals, SPA navigation, or component boundary behavior, read this page first and keep generated code aligned with the runtime implemented in this repo.

Use `components.md` for authoring Python `@component` files and same-name HTML templates. Use this page for the browser-side PulsePoint contract, the authoring rules that feed it, and the React-style mental model used by the shipped runtime.

PulsePoint is the default reactive frontend layer for Caspian. In the current runtime it follows a React-like component pattern, but it is HTML-first rather than JSX-first.

Do not assume React, Vue, Svelte, JSX, Alpine, HTMX, or older PulsePoint docs unless the task explicitly asks for a different frontend contract.

## Source Of Truth

When documenting or generating PulsePoint code, follow this order:

- `public/js/pp-reactive-v2.js` is the shipped browser runtime contract AI should follow.
- `main.py` is the render-pipeline source of truth for how Caspian injects runtime attributes and rewrites scripts before the browser sees the HTML.
- If you are working inside PulsePoint or Caspian runtime development code and there is an authoring source tree behind the shipped files, use it only as an implementation detail. Do not assume that source tree exists in generated apps or shipped framework output.

Important current facts:

- `public/js/pp-reactive-v2.js` exposes the global `pp` runtime and auto-mounts on DOM ready.
- `main.py` renders the final HTML, runs `transform_components(...)`, then runs `transform_scripts(...)` before returning the response.

If docs, generated examples, or older notes disagree with `public/js/pp-reactive-v2.js` plus `main.py`, follow the code that actually runs.

## Default Frontend Rule

When a Caspian page needs reactive browser behavior, use PulsePoint.

- Use PulsePoint component roots, scripts, directives, and runtime helpers for interactive UI.
- Use PulsePoint state, effects, refs, and template directives as the default reactivity model in authored Caspian templates.
- When the browser needs CRUD operations or follow-up reads from the backend, call `pp.rpc()` from PulsePoint code and back it with route or backend `@rpc()` actions.
- Keep server-rendered HTML plus PulsePoint enhancement as the baseline architecture.
- Only introduce another frontend runtime when the user explicitly asks for it or the project already depends on one.

## Authoring Model

PulsePoint authoring in this repo is split into two layers:

- The authored layer: route, layout, and component templates under `src/` with plain HTML plus a plain `<script>`.
- The runtime layer: the browser sees `pp-component` roots and `script[type="text/pp"]` after Caspian transforms the HTML.

For authored Caspian templates:

- Keep exactly one top-level lowercase HTML root element.
- Put the component logic inside a plain `<script>` inside that same root.
- Do not handwrite `pp-component="..."`.
- Do not handwrite `type="text/pp"`.

Caspian already handles those details for you during render.

That means AI-generated examples should default to authored template source, not raw runtime HTML.

Authored example:

```html
<section>
  <h2>{title}</h2>
  <p>Count: {count}</p>

  <button onclick="setCount(count + 1)">Increment</button>
  <button onclick="reset()">Reset</button>

  <script>
    const { title = "Counter" } = pp.props;
    const [count, setCount] = pp.state(0);

    function reset() {
      setCount(0);
    }
  </script>
</section>
```

When that template reaches the browser, Caspian will already have injected the component id and rewritten the owned script to `type="text/pp"`. Those runtime attributes are for the rendered output, not for authored source examples.

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

- Each runtime component root should have at most one owned script.
- At runtime, the owned script is `script[type="text/pp"]`.
- The script lookup walks the current root and skips nested `pp-component` boundaries, so a parent does not consume a child component's script.
- If multiple matching runtime scripts exist in the same root, the first matching owned script wins. Generate one script per root.
- Authored route, layout, and component templates still need one top-level lowercase HTML root so Caspian can inject the component boundary correctly.
- A scriptless component root still mounts and can receive props, refs, events, and nested children, but it has no local runtime scope beyond its props.
- Component scripts are plain JavaScript executed with `new Function(...)`. Do not use `import`, `export`, or top-level `await` inside them.
- The runtime auto-returns supported top-level bindings from the script. Do not rely on manual `return { ... }` objects.
- `pp.props` contains the current prop bag for the component.
- `pp.props.children` contains the root's initial inner HTML before the owned script is removed from the render template.

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
<div>
  <h2>{title}</h2>
  <p>Count: {count}</p>
  <button onclick="setCount(count + 1)">Increment</button>

  <script>
    const { title } = pp.props;
    const [count, setCount] = pp.state(0);

    function reset() {
      setCount(0);
    }
  </script>
</div>
```

That is the authored form. In browser-inspected runtime HTML, Caspian will already have added the component id and the owned script type.

## Hooks and runtime API

PulsePoint uses a React-style mental model inside each component script: stateful render scope, dependency-based effects, refs, reducer-style updates, context consumption, and portals.

Hooks exposed inside component scripts through `pp`:

- `pp.state(initial)` returns `[value, setValue]`.
- `pp.effect(callback, deps?)` runs after render and may return a cleanup function.
- `pp.layoutEffect(callback, deps?)` runs synchronously after DOM mutation/ref binding/portal application and before `pp.effect`.
- `pp.ref(initialValue?)` returns `{ current }`.
- `pp.memo(factory, deps)` memoizes a computed value.
- `pp.callback(callback, deps)` memoizes a function.
- `pp.reducer(reducer, initialState)` returns `[state, dispatch]`.
- `pp.context(token)` resolves a provided context value from ancestor components.
- `pp.portal(ref, target?)` registers a ref-managed element for portal rendering and returns an object that includes `sourceParent`.
- `pp.props` exposes the current props.

Global helpers exposed through the `pp` singleton and also merged into the component runtime:

- `pp.createContext(defaultValue)` creates a context token.
- `pp.mount()` bootstraps the runtime. It is idempotent.
- `pp.redirect(url)` performs SPA-aware navigation when enabled.
- `pp.rpc(name, data?, optionsOrAbort?)` performs the current route RPC bridge.
- `pp.enablePerf()` enables render timing collection.
- `pp.disablePerf()` disables render timing collection.
- `pp.getPerfStats()` returns collected render timings.
- `pp.resetPerfStats()` clears collected render timings.

Notes:

- The global `pp` singleton auto-mounts once the runtime is loaded and the DOM is ready. Manual `pp.mount()` is still safe because it short-circuits after the first mount.
- `pp.state` setters accept either a value or an updater function.
- `pp.effect` and `pp.layoutEffect` are cleanup-style hooks. Returned promises are not awaited.
- `pp.portal(ref)` defaults to `document.body` when no target is provided.
- Older docs may call the RPC helper `pp.fetchFunction()`. In the current bundled runtime the implemented global API is `pp.rpc()`.
- Keep template-facing bindings at the top level so the AST-based exporter can see them.
- For predictable code generation, prefer passing an explicit dependency array to `pp.effect`, `pp.layoutEffect`, `pp.memo`, and `pp.callback`.

## Context

Context is implemented in the current runtime, but the current API follows a React-style `Context.Provider` pattern rather than a legacy `pp.provideContext(...)` helper.

How it works:

- Create a token with `pp.createContext(defaultValue)`.
- Provide a value with `Context.Provider`.
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
- If provider and consumer live in different component script scopes, pass the token through props or store it in shared outer or global state.
- The preferred authoring style is a provider tag in the template. The runtime also supports imperative `ThemeContext.Provider({ value })` calls during render, but the tag form is clearer for AI-generated authored templates.
- Do not invent or document `pp.provideContext`. The current runtime explicitly does not expose it.

Provider example:

```html
<section>
  <script>
    const ThemeContext = pp.createContext("light");
    const [theme] = pp.state("dark");
  </script>

  <ThemeContext.Provider value="{theme}">
    <p>This subtree receives the provided theme.</p>
  </ThemeContext.Provider>
</section>
```

Consumer example for a child component that receives the token through props:

```html
<div>
  <p>{theme}</p>

  <script>
    const theme = pp.context(pp.props.themeToken);
  </script>
</div>
```

When a child component needs the same token object, pass it from the provider scope as a prop such as `theme-token="{ThemeContext}"`.

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
- For dynamic inline styles in authored templates, prefer `pp-style="{styleText}"` over `style="{styleText}"` so HTML/CSS tooling does not parse the brace expression as raw CSS.
- `pp-style` is an authoring alias. The compiler rewrites it to a native `style` attribute in rendered output.
- If the same element already has a static `style` attribute, `pp-style` is merged into that `style` value during compilation.
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
<div>
  <p pp-style="{noticeStyle}">Notice</p>
  <button pp-spread="{...buttonAttrs}" hidden="{isLoading}">Save</button>

  <script>
    const noticeStyle = "color: red;";

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
<div>
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
<div>
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
- For file managers, use upload callbacks for progress UI but replace the asset list from returned RPC state with `pp.state(...)` and `pp-for` instead of manual DOM repainting. See [file-uploads.md](./file-uploads.md).
- Streamed `text/event-stream` responses are supported when a stream handler is provided.
- Redirect headers are honored through `pp.redirect()`.

Notes:

- `pp.redirect()` uses SPA navigation for same-origin URLs when SPA mode is enabled. Otherwise it falls back to normal navigation.
- Root-layout mismatches during SPA navigation trigger a hard reload.
- `pp.mount()` bootstraps every `[pp-component]` it finds, so generated code should call it only through the global runtime if you are manually mounting at all.

## Runtime Output And Debugging

When you inspect rendered HTML in the browser, you should expect to see runtime-managed attributes and elements that are not part of authored source.

Normal runtime output includes:

- `pp-component="..."` on mounted roots.
- `script[type="text/pp"]` for owned component scripts.
- internal attributes such as captured ref tokens and event-owner bookkeeping.

These are runtime details.

- Mention them in docs when explaining how the browser runtime works.
- Do not use them as the default form in authored examples.
- Do not tell AI to handwrite them in route, layout, or component templates unless the task is explicitly about raw runtime HTML or runtime internals.

## AI rules

Use these rules when generating or editing PulsePoint runtime code:

- Treat PulsePoint as the default reactive frontend for Caspian app code.
- Treat `pp.rpc()` as the default browser-to-server path for CRUD operations and interactive backend reads.
- Use `public/js/pp-reactive-v2.js` as the shipped runtime contract AI should follow.
- Keep `main.py` in view because it injects the runtime-facing attributes and rewrites authored scripts before the browser sees them.
- If a development-only source tree exists behind the shipped runtime, treat it as optional implementation detail rather than something generated apps are guaranteed to contain.
- In authored Caspian templates, do not handwrite `pp-component` or `type="text/pp"`; let the render pipeline inject them.
- Prefer PulsePoint state and template directives over manual DOM mutation for reactive updates.
- If you are explicitly editing raw runtime HTML or internals, keep `pp-component` unique per live instance.
- In authored templates, use a plain `<script>` inside the root. In runtime HTML, the owned script appears as `script[type="text/pp"]`.
- Keep template-facing variables at top level.
- Follow the React-style context pattern: `pp.createContext(...)`, `<Context.Provider value="{...}">`, and `pp.context(token)`.
- Do not invent `pp.provideContext`, `pp-context`, or other legacy context helpers.
- Keep `pp-for` on `<template>` and use plain `key`.
- Use native `on*` attributes, not framework-specific event syntax.
- Use refs and portals only through the implemented `pp` APIs.
- Use `pp.rpc()` for the bundled runtime API instead of older `pp.fetchFunction()` wording.
- For upload managers, keep the authoritative asset list in `pp.state(...)` and route uploads through `pp.rpc()` with `onUploadProgress` as needed.
- Avoid generating internal runtime attributes.
- Avoid scriptless nested components when the child template contains its own bindings.
- Prefer authored-template examples over runtime-inspected HTML examples unless the doc is specifically explaining runtime internals.

## What to avoid

Do not generate these unless the current source explicitly adds support:

- React, Vue, Svelte, Alpine, HTMX, or JSX-first patterns as the default Caspian frontend approach
- `pp-context`
- `pp-key`
- `data-pp-ref`
- `pp-context-provider`
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
- `public/js/pp-reactive-v2.js` is the runtime surface that ships and should be assumed to exist.
- Caspian already injects `pp-component` and rewrites owned scripts to `type="text/pp"` during render.
- Nested roots without their own `script[type="text/pp"]` block are not fully isolated during parent template compilation.
- The global `pp` singleton auto-mounts on DOM ready, and `pp.mount()` is idempotent.
- `pp.effect` and `pp.layoutEffect` are cleanup-style hooks. Their callbacks are not promise-aware.
- `pp.context()` resolves through ancestor components, not the current component's own pending providers.
- `pp.provideContext` is not part of the current runtime API. Use `Context.Provider`.
- `pp.portal()` preserves logical ancestry through the registry, so context and prop refresh behavior continue to work through portaled descendants.

## Final reminder

If a feature is not implemented in the current bundled runtime, do not invent it.

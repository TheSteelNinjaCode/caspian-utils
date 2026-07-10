---
title: PulsePoint Runtime Guide
description: Use this page when the task mentions PulsePoint, `pp.state`, `pp.effect`, `pp-ref`, `pp-style`, `pp-for`, portals, `pp-reset-scroll`, SPA navigation, or `public/js/pp-reactive-v2.js`.
related:
  title: Related docs
  description: Read the components, routing, data-fetching, and project-structure docs alongside the PulsePoint runtime contract.
  links:
    - /docs/components
    - /docs/routing
    - /docs/fetch-data
    - /docs/pulsepoint-runtime-map
    - /docs/core-runtime-map
    - /docs/project-structure
    - /docs/index
---

## Purpose

This file documents the PulsePoint contract for the shipped Caspian browser runtime. Treat it as the AI-facing source of truth when generating or reviewing interactive Caspian UI.

If a task involves `pp.state`, `pp.effect`, `pp.layoutEffect`, `pp-ref`, `pp-style`, `pp-spread`, `pp-for`, context, portals, `pp-reset-scroll`, SPA navigation, or component boundary behavior, read this page first and keep generated code aligned with the current Caspian runtime.

Use `components.md` for authoring Python `@component` files, same-name HTML templates, and HTML-first `x-*` component tags. Use this page for the browser-side PulsePoint contract, the authoring rules that feed it, and the React-style mental model used by the shipped runtime.

PulsePoint is the default reactive frontend layer for Caspian. In the current runtime it follows a React-like component pattern, but it is HTML-first rather than JSX-first.

Do not assume React, Vue, Svelte, JSX, Alpine, HTMX, or older PulsePoint docs unless the task explicitly asks for a different frontend contract.

## Hard Invariants

Apply these before generating any template, even without reading the rest of this page:

1. One authored top-level root per route, layout, or component template; the owned plain `<script>` lives inside that root, never as a sibling.
2. Never handwrite `pp-component`, `type="text/pp"`, `data-pp-ref`, or other runtime-managed attributes; the render pipeline injects them.
3. Bind first-party events with native `on*` attributes in the HTML; never wire normal UI with ids, `querySelector`, `addEventListener`, or manual `innerHTML`.
4. For ordinary form submits, use `onsubmit="{handler(event)}"` plus `Object.fromEntries(new FormData(event.currentTarget).entries())`; refs are for imperative access only.
5. Keep reactive values in `pp.state(...)`; keep template-facing bindings at the top level of the script.
6. Call the backend with `pp.rpc(...)` backed by Python `@rpc()` actions; do not invent fetch wrappers or `pp.fetchFunction()`.
7. `pp-for` goes only on `<template>` with plain `key`; context uses `pp.createContext(...)`, a lowercase `<token.provider>` tag, and `pp.context(token)`.
8. If an API is not in `public/js/pp-reactive-v2.js`, it does not exist; do not invent hooks, directives, or globals.

## Source Of Truth

When documenting or generating PulsePoint code, follow this order:

- `public/js/pp-reactive-v2.js` is the shipped browser runtime contract AI should follow.
- `main.py` is the render-pipeline source of truth for how Caspian injects runtime attributes and rewrites scripts before the browser sees the HTML.
- If you are working inside PulsePoint or Caspian runtime development code and there is an authoring source tree behind the shipped files, use it only as an implementation detail. Do not assume that source tree exists in generated apps or shipped framework output.

Important current facts:

- `public/js/pp-reactive-v2.js` exposes the global `pp` runtime and auto-mounts on DOM ready.
- `main.py` renders the final HTML, runs `transform_components(...)`, then runs `transform_scripts(...)` before returning the response.
- `.venv/Lib/site-packages/casp/components_compiler.py` injects `pp-component` on the final resolved root after component expansion.
- `.venv/Lib/site-packages/casp/scripts_type.py` rewrites authored body `<script>` tags to `type="text/pp"`.
- Authored route and component templates compose reusable server components as HTML-first `x-*` tags before the browser runtime mounts.

If docs, generated examples, or older notes disagree with `public/js/pp-reactive-v2.js` plus `main.py`, follow the code that actually runs.

Use [core-runtime-map.md](./core-runtime-map.md) when the controlling runtime file is not obvious yet.

Use [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) when the task names a specific PulsePoint feature or directive and you need a quick feature-to-runtime lookup before reading the full guide.

## Default Frontend Rule

When a Caspian page needs reactive browser behavior, use PulsePoint.

- Use PulsePoint component roots, scripts, directives, and runtime helpers for interactive UI.
- Use PulsePoint state, effects, refs, and template directives as the default reactivity model in authored Caspian templates.
- Bind first-party events in the HTML with PulsePoint-handled native `on*` attributes such as `onclick`, `oninput`, `onchange`, and `onsubmit`.
- For ordinary forms, bind the submit event in the `<form>` and collect named fields with `Object.fromEntries(new FormData(event.currentTarget).entries())`. Let input `name` attributes define the payload keys, then validate and normalize those values in Python. Do not use `pp.ref(...)` on every input or an effect-managed listener just to read submitted values.
- When the browser needs CRUD operations or follow-up reads from the backend, call `pp.rpc()` from PulsePoint code and back it with route or backend `@rpc()` actions.
- Keep server-rendered HTML plus PulsePoint enhancement as the baseline architecture.
- For dashboards, admin areas, account sections, docs sections, and other grouped subtrees, keep shared shell markup and shared PulsePoint behavior in the parent folder's `layout.html`, then keep child-route PulsePoint state local to each `index.html`. Follow the same mental model as the Next.js App Router.
- Only introduce another frontend runtime when the user explicitly asks for it or the project already depends on one.

## PulsePoint-First Events And Reactivity

For first-party Caspian HTML, PulsePoint is not a later enhancement after ordinary JavaScript. It is the event, state, and reactivity model AI should use from the start.

Default to this workflow:

- Put the button, form, input, toggle, menu, filter, upload control, or list markup directly in the route, layout, or component HTML template.
- Bind events with native `on*` attributes handled by PulsePoint, for example `onclick="save()"`, `oninput="setQuery(event.target.value)"`, or `onsubmit="{submitForm(event)}"`.
- This is the PulsePoint `onClick`/native event-attribute model. Authored examples use lowercase HTML spellings such as `onclick` because browser HTML normalizes attribute names, but the important rule is to bind the event in the template instead of wiring it later with DOM selectors.
- For simple form submissions, let HTML own the form shape and let the submit event carry the values: use `const data = Object.fromEntries(new FormData(event.currentTarget).entries())` inside the handler, then pass that object directly to `pp.rpc(...)`. In a form-level submit handler, `event.target` is usually the form too, but `event.currentTarget` is the copy-safe default because it always means the element that owns the handler.
- Keep reactive values in `pp.state(...)`.
- Render conditional text, classes, attributes, lists, and styles with template expressions, `pp-for`, `pp-style`, `pp-spread`, and other PulsePoint-supported template features.
- Use `pp.ref(...)` and `pp-ref` when a real imperative element reference is needed, such as focus, measurement, media, canvas, third-party widgets, or resetting a specific file/password input after a successful action.
- Use `pp.effect(...)` or `pp.layoutEffect(...)` for lifecycle work that must happen after render.
- Use `pp.rpc(...)` for browser-triggered backend reads and writes.

Avoid building a parallel JavaScript layer for normal UI behavior:

- Do not add ids only so a script can find elements with `document.querySelector(...)` or `document.getElementById(...)`.
- Do not use `data-*` attributes as a private client state system when PulsePoint state or props should own the data.
- Do not create click-in buttons by placing intent in `data-*` attributes and then scanning for those attributes. Put the action directly on the element with `onclick`, `oninput`, `onchange`, `onsubmit`, or another native `on*` event attribute.
- Do not bind normal first-party clicks, input changes, submits, filters, menus, or toggles with `addEventListener(...)`.
- Do not use form refs plus input refs plus `pp.effect(...)` solely to construct an RPC payload from normal submitted fields.
- Do not repaint first-party lists or panels with manual `innerHTML` writes when `pp.state(...)` plus `pp-for` can express the same UI.
- Do not create a custom client-side store, event bus, or hydration routine for behavior that belongs in a PulsePoint component script.

Use direct DOM APIs only as a narrow escape hatch: third-party widgets, browser APIs that require imperative access, measurements, focus, media, canvas, or behavior the current PulsePoint runtime cannot express declaratively. Even then, keep the imperative code inside the owning PulsePoint component script, usually through `pp.ref(...)` plus `pp.effect(...)`, so PulsePoint still owns the component's state, cleanup, and event flow.

Preferred authored pattern:

```html
<section>
  <input value="{query}" oninput="setQuery(event.target.value)" />
  <button onclick="clearSearch()" disabled="{query.length === 0}">Clear</button>

  <ul>
    <template pp-for="item in filteredItems">
      <li key="{item.id}">{item.label}</li>
    </template>
  </ul>

  <script>
    const [query, setQuery] = pp.state("");
    const items = pp.props.items ?? [];
    const filteredItems = items.filter((item) =>
      item.label.toLowerCase().includes(query.toLowerCase())
    );

    function clearSearch() {
      setQuery("");
    }
  </script>
</section>
```

Avoid this first-party pattern:

```html
<section>
  <input id="search" />
  <button id="clear-search">Clear</button>
  <ul id="results"></ul>

  <script>
    const input = document.querySelector("#search");
    const button = document.querySelector("#clear-search");
    const results = document.querySelector("#results");

    button.addEventListener("click", () => {
      input.value = "";
      results.innerHTML = "";
    });
  </script>
</section>
```

That second shape recreates a separate event and rendering system inside a Caspian component. It is harder to maintain because it bypasses PulsePoint's rerender, event rebinding, refs, cleanup, and backend RPC conventions.

Preferred form submit pattern:

```html
<section>
  <form onsubmit="{saveProfile(event)}" class="grid gap-4" novalidate>
    <input name="name" value="{{ user_name }}" autocomplete="name" required />
    <input name="email" type="email" value="{{ user_email }}" autocomplete="email" required />
    <input name="password" type="password" autocomplete="new-password" />

    <button type="submit" disabled="{isSubmitting}">
      {isSubmitting ? "Saving..." : "Save changes"}
    </button>
  </form>

  <script>
    const [isSubmitting, setIsSubmitting] = pp.state(false);

    async function saveProfile(event) {
      event.preventDefault();
      if (isSubmitting) return;

      const data = Object.fromEntries(new FormData(event.currentTarget).entries());

      setIsSubmitting(true);
      try {
        await pp.rpc("save_profile", data, { abortPrevious: true });
      } finally {
        setIsSubmitting(false);
      }
    }
  </script>
</section>
```

Avoid this for a normal form:

```html
<section>
  <form pp-ref="{formRef}">
    <input pp-ref="{nameRef}" name="name" />
    <input pp-ref="{emailRef}" name="email" />
  </form>

  <script>
    const formRef = pp.ref(null);
    const nameRef = pp.ref(null);
    const emailRef = pp.ref(null);

    pp.effect(() => {
      const form = formRef.current;
      if (!form) return;

      const submit = (event) => {
        event.preventDefault();
        return pp.rpc("save_profile", {
          name: nameRef.current?.value ?? "",
          email: emailRef.current?.value ?? "",
        });
      };

      form.addEventListener("submit", submit);
      return () => form.removeEventListener("submit", submit);
    }, []);
  </script>
</section>
```

Refs are still useful when the feature actually requires imperative access. They are not the first choice for reading standard form fields that the browser already exposes through the submit event. Keep client code minimal and put reviewable data normalization in the route's Python `@rpc()` action unless the client must transform values for UX before submitting.

Back the form with a route-owned `@rpc()` action in the sibling `index.py`. If Prisma is enabled, the action should use the generated Prisma Python ORM for persistence.

```python
from casp.rpc import rpc
from casp.validate import Validate
from src.lib.prisma import prisma


@rpc()
async def save_profile(name: str, email: str, password: str = ""):
    name = name.strip()
    email = email.strip().lower()

    if not name:
        raise ValueError("Name is required.")
    if Validate.with_rules(email, "required|email") is not True:
        raise ValueError("A valid email address is required.")

    profile = await prisma.user.update(
        where={"email": email},
        data={"name": name},
    )

    return profile.to_dict()
```

Basic two-way state pattern:

```html
<section>
  <label>
    Name
    <input name="name" value="{name}" oninput="setName(event.target.value)" />
  </label>

  <p>Hello, {name || "friend"}.</p>
  <button onclick="setName('')">Clear</button>

  <script>
    const [name, setName] = pp.state("");
  </script>
</section>
```

## Authoring Model

Treat this section as the canonical authored-vs-runtime contract for Caspian templates. When another packaged doc needs this rule set, link here instead of restating the full explanation.

PulsePoint authoring is split into two layers:

- The authored layer: route, layout, and component templates under `src/` with plain HTML plus a plain `<script>`.
- The runtime layer: the browser sees `pp-component` roots and `script[type="text/pp"]` after Caspian transforms the HTML.

For authored Caspian templates:

- Keep exactly one authored top-level parent node.
- In source, that parent may be a native HTML element or a single imported `x-*` component tag, but after component expansion it must resolve to one final HTML root.
- Put the component logic inside a plain `<script>` inside that same root.
- Do not handwrite `pp-component="..."`.
- Do not handwrite `type="text/pp"`.

Treat that single-root rule as a hard invariant for AI-generated templates. A sibling `<script>` after the root or any second top-level element will break Caspian's `pp-component` injection and fail the render.

Keep visible route, layout, and component markup in the HTML templates. Treat `index.py` and `layout.py` as backend companions for data, metadata, props, RPC actions, auth, caching, redirects, and other server-side preparation, not as template bodies.

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

Invalid authored shape:

```html
<section>
  <h2>{title}</h2>
</section>

<script>
  const { title = "Counter" } = pp.props;
</script>
```

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
- Authored route, layout, and component templates still need one top-level parent node so Caspian can inject the component boundary correctly after component expansion.
- A scriptless component root still mounts and can receive props, refs, events, and nested children, but it does not create local bindings from those props on its own.
- Component scripts are plain JavaScript executed with `new Function(...)`. Do not use `import`, `export`, or top-level `await` inside them.
- The runtime auto-returns supported top-level bindings from the script. Do not rely on manual `return { ... }` objects.
- `pp.props` contains the current prop bag for the component.
- `pp.props.children` contains the root's initial inner HTML before the owned script is removed from the render template.
- Props are not auto-injected as standalone top-level template or handler variables. Read them through `pp.props` or explicitly destructure them in the component script.

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
- `pp.effect` and `pp.layoutEffect` are cleanup-style hooks. They may return only a synchronous cleanup function; returning a promise warns and is ignored, so start async work inside the effect instead.
- `pp.portal(ref)` defaults to `document.body` when no target is provided.
- Older docs may call the RPC helper `pp.fetchFunction()`. In the current bundled runtime the implemented global API is `pp.rpc()`.
- Keep template-facing bindings at the top level so the AST-based exporter can see them.
- For predictable code generation, prefer passing an explicit dependency array to `pp.effect`, `pp.layoutEffect`, `pp.memo`, and `pp.callback`.

Effect with cleanup and dependencies:

```html
<section>
  <p>Elapsed: {seconds}s</p>
  <button onclick="setRunning(!running)">{running ? "Pause" : "Resume"}</button>

  <script>
    const [seconds, setSeconds] = pp.state(0);
    const [running, setRunning] = pp.state(true);

    pp.effect(() => {
      if (!running) return;
      const id = setInterval(() => setSeconds((s) => s + 1), 1000);
      return () => clearInterval(id);
    }, [running]);
  </script>
</section>
```

Use the same shape for any subscription: attach in the effect body, detach in the returned cleanup. The cleanup also runs on component disposal, which is why route-owned `WebSocket` clients close their socket from a cleanup effect.

Memoized derived value:

```html
<section>
  <input value="{query}" oninput="setQuery(event.target.value)" />
  <p>{visible.length} of {items.length} items</p>

  <script>
    const [query, setQuery] = pp.state("");
    const items = pp.props.items ?? [];

    const visible = pp.memo(
      () => items.filter((item) => item.label.toLowerCase().includes(query.toLowerCase())),
      [query]
    );
  </script>
</section>
```

For cheap derivations, a plain top-level `const` recomputed each render is fine; reach for `pp.memo` when the computation is heavy or the result identity matters for dependencies.

Reducer for multi-field update logic:

```html
<section>
  <p>{cart.count} items, total {cart.total}</p>
  <button onclick="dispatch({ type: 'add', price: 10 })">Add</button>
  <button onclick="dispatch({ type: 'clear' })">Clear</button>

  <script>
    const [cart, dispatch] = pp.reducer((state, action) => {
      if (action.type === "add") {
        return { count: state.count + 1, total: state.total + action.price };
      }
      if (action.type === "clear") {
        return { count: 0, total: 0 };
      }
      return state;
    }, { count: 0, total: 0 });
  </script>
</section>
```

Portal pattern for overlays that must escape clipping ancestors:

```html
<div>
  <button onclick="setOpen(true)">Open dialog</button>

  <div pp-ref="dialogRef" hidden="{!open}" class="modal">
    <p>Rendered under document.body.</p>
    <button onclick="setOpen(false)">Close</button>
  </div>

  <script>
    const [open, setOpen] = pp.state(false);
    const dialogRef = pp.ref(null);

    pp.portal(dialogRef);
  </script>
</div>
```

`pp.portal(ref)` moves the ref-managed element to `document.body` by default, or to the element passed as the second argument. Portaled content keeps its logical component ancestry, so context, props, and events keep working.

## Context

Context is implemented in the current runtime with a React-style provider pattern rather than a legacy `pp.provideContext(...)` helper. Because Caspian templates are HTML-first, authored provider tags should be written in lowercase HTML form, for example `<themecontext.provider>`, even when the JavaScript token is named `ThemeContext`.

How it works:

- Create a token with `pp.createContext(defaultValue)`.
- Provide a value with a lowercase provider tag such as `<themecontext.provider value="{theme}">`.
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
- The preferred authoring style is a lowercase provider tag in the template. The TypeScript runtime's `TemplateCompiler.transformContextProviderTags(...)` recognizes `*.provider` tags case-insensitively and rewrites them to runtime-owned `<pp-context-provider>` boundaries. The runtime context lookup is also case-insensitive, so `<themecontext.provider>` can resolve a script binding named `ThemeContext`.
- The runtime also supports imperative `ThemeContext.Provider({ value })` calls during render, but that form provides from the component boundary rather than documenting the HTML subtree shape. Use it only when component-wide provision is intended.
- Do not invent or document `pp.provideContext`. The current runtime explicitly does not expose it.

Provider example:

```html
<section>
  <script>
    const ThemeContext = pp.createContext("light");
    const [theme] = pp.state("dark");
  </script>

  <themecontext.provider value="{theme}">
    <p>This subtree receives the provided theme.</p>
  </themecontext.provider>
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
- Inside the child component, those values should be accessed through `pp.props` or a top-level destructure such as `const { title } = pp.props`; they are not implicitly available as bare identifiers.

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
- Plain objects, functions, and symbols are invalid template children; the runtime warns and omits them instead of rendering accidental strings such as `[object Object]`.
- Supported boolean attributes are normalized so truthy values emit the bare attribute and falsy values remove it.
- `<textarea value="{draft}"></textarea>` is normalized into textarea content.
- Use `pp-spread="{...attrs}"` to spread an object expression into attributes.
- `pp-spread` omits nullish values, omits known HTML boolean attributes when their value is `false`, emits them bare when `true`, preserves string-valued `aria-*`/`data-*` booleans, and escapes `&`, `"`, and `<` in emitted attribute values.
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

### Component markup is deferred inside an inert `<template>`

The render pipeline wraps each top-level `pp-component` root in an inert `<template pp-component="…">`, and the runtime materializes it back into live DOM on `mount()` (and on every SPA navigation) before it scans for roots. The browser never parses, validates, or fetches the contents of a `<template>`, so raw `{...}` placeholders never reach live DOM at first paint.

- Because of this, `{...}` is safe in **any** attribute or position — including slots the browser would otherwise validate eagerly: SVG geometry (`d`, `viewBox`, `points`, `transform`), URL attributes (`src`, `srcset`, `href`, `poster`), form `value` on `date`/`number`/`color` inputs, and text placed directly inside `<table>` or `<select>`.
- Do not add per-tag workarounds to dodge first-paint validation (static-path `hidden` toggles instead of binding `d`, `data-*` URL holders, `hidden`-gated `<img src>`, or an SSR-resolved initial value). The deferral removes the whole class at once.
- `pp-style` and the `<input>`/`<select>`/`checked`/`defaultvalue`/`<textarea>` value rewrites still apply — but for different reasons that deferral does not replace: authoring-source tooling (`style="{...}"` breaks HTML/CSS linters) and attribute-vs-property correctness for controlled form fields.

## Refs

- Refs are for imperative element access. Do not use `pp-ref` as the default way to read normal form input values on submit; prefer the form's `onsubmit` event plus `Object.fromEntries(new FormData(event.currentTarget).entries())`.
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
- Collections may be arrays or synchronous iterables such as `Set`, `Map`, typed arrays, strings, and generators. Non-null non-iterable values warn and render as empty lists.
- Loop content can contain interpolations, events, refs, and nested components.
- Event handlers inside loops capture each rendered item, so later collection changes do not retarget an existing row handler.
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

- Use native `on*` attributes such as `onclick`, `oninput`, `onchange`, and `onsubmit` for first-party events.
- Treat this as the HTML form of `onClick`-style PulsePoint event binding. Prefer lowercase examples in authored HTML because the browser normalizes attribute names.
- Event values may be raw code or wrapped in `{...}`.
- The runtime injects `event`, `e`, `$event`, `target`, `currentTarget`, and `el`.
- For form submit handlers, prefer `onsubmit="{submitForm(event)}"` and `Object.fromEntries(new FormData(event.currentTarget).entries())` when collecting named fields for `pp.rpc(...)`.
- `event.target` can also work for form-level submit handlers, but examples use `event.currentTarget` so nested event origins do not change which element becomes the `FormData` source.
- Do not use hyphenated event attrs like `on-click`.
- Event attributes are removed from the live DOM after binding and rebound after DOM morphing.
- Owned template/event-owner internals are runtime-managed. Do not author them directly.
- Do not replace normal PulsePoint event attributes with id-driven `querySelector(...)` plus `addEventListener(...)` wiring. If an imperative listener is unavoidable for an integration, attach and clean it up from `pp.effect(...)`.
- Do not replace a normal `onclick` with a `data-action`, `data-target`, or similar attribute plus a script that scans the DOM. PulsePoint event attributes are the event contract for first-party Caspian UI.

## SPA, loading, and navigation helpers

- `body[pp-spa="true"]` enables client-side navigation interception.
- `a[pp-spa="false"]` disables interception for that link.
- External links, downloads, `_blank`, and modifier-key clicks bypass SPA interception.
- `pp-loading-content="true"` marks the page region that gets swapped or faded during navigation.
- Route-specific loading states are looked up with `pp-loading-url`.
- `pp-loading-transition` accepts JSON with `fadeIn` and `fadeOut` timing values.
- The runtime saves scroll positions per history entry while SPA navigation is enabled.
- Push navigation resets window scroll to the top.
- Saved history-entry scroll state is used during history traversal instead of relying only on the live DOM scroll at the time the user clicks Back or Forward.
- Unmarked scrollable containers may keep their outgoing scroll on push navigation so shared shells such as sidebars, rails, and docs nav panes stay stable across child-route changes.
- `pp-reset-scroll="true"` on a scroll container opts that container into reset-on-navigation behavior. Use it on the main content pane of a grouped shell when page content should start at the top on each child-route navigation.
- `pp-scroll-key="stable-name"` gives a scroll container a stable restoration identity. Prefer it when an element has no stable `id`, or when its classes or position can change between routes.
- `body[pp-reset-scroll="true"]` is the global override for routes that should reset window scroll and every scrollable element.
- Navigation dispatches `pp:navigation:start`, `pp:navigation:complete`, and `pp:navigation:error` events on `document`.

RPC notes:

- `pp.rpc(name, data?, optionsOrAbort?)` posts to the current route.
- Passing `true` as the third argument means `abortPrevious: true`.
- The options object supports `abortPrevious`, `onStream`, `onStreamError`, `onStreamComplete`, `onUploadProgress`, and `onUploadComplete`.
- File uploads switch to the XHR path when upload progress callbacks are needed.
- The `onUploadProgress` callback receives `{ loaded, total, percent }`. `percent` is a 0-100 number, and both `total` and `percent` are `null` when the browser cannot compute the upload length. Do not document or generate `event.percentage`.
- For file managers, use upload callbacks for progress UI but replace the asset list from returned RPC state with `pp.state(...)` and `pp-for` instead of manual DOM repainting. See [file-uploads.md](./file-uploads.md).
- Streamed `text/event-stream` responses are supported when a stream handler is provided. Each `onStream` chunk is JSON-parsed when the event data looks like JSON; otherwise the raw string is passed through. If the response streams but no `onStream` handler was given, the runtime warns and discards the stream.
- Redirect headers are honored through `pp.redirect()`.

Notes:

- `pp.redirect()` uses SPA navigation for same-origin URLs when SPA mode is enabled. Otherwise it falls back to normal navigation.
- Root-layout mismatches during SPA navigation trigger a hard reload.
- `pp.mount()` bootstraps every `[pp-component]` it finds, so generated code should call it only through the global runtime if you are manually mounting at all.

Upload with progress pattern:

```html
<section>
  <input type="file" onchange="{uploadFile(event.target.files?.[0])}" />
  <progress max="100" value="{percent ?? 0}"></progress>

  <script>
    const [percent, setPercent] = pp.state(null);

    async function uploadFile(file) {
      if (!file) return;

      await pp.rpc("upload_asset", { file }, {
        onUploadProgress: ({ percent }) => setPercent(percent),
        onUploadComplete: () => setPercent(100),
      });
    }
  </script>
</section>
```

Streaming pattern for server generators that return `text/event-stream`. This is the default path for AI/LLM/chat token streaming: a generator `@rpc()` that `yield`s tokens (often by `async for`-ing over a Python SDK stream) consumed by `pp.rpc(..., { onStream })` that appends each chunk to reactive state. Do not reinvent one-way streaming with raw `fetch`/`ReadableStream`, `EventSource`, or a WebSocket; see [fetch-data.md](./fetch-data.md) for the Python side.

```html
<section>
  <button onclick="ask()" disabled="{isStreaming}">Ask</button>
  <pre>{answer}</pre>

  <script>
    const [answer, setAnswer] = pp.state("");
    const [isStreaming, setIsStreaming] = pp.state(false);

    function ask() {
      setAnswer("");
      setIsStreaming(true);

      pp.rpc("ask_question", { topic: "caspian" }, {
        onStream: (chunk) => setAnswer((current) => current + chunk),
        onStreamError: () => setIsStreaming(false),
        onStreamComplete: () => setIsStreaming(false),
      });
    }
  </script>
</section>
```

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
- For first-party HTML interactions, use PulsePoint `on*` event attributes, state, refs, effects, directives, and `pp.rpc()` before reaching for DOM APIs.
- For simple forms, bind `onsubmit` in the authored HTML and read named fields with `Object.fromEntries(new FormData(event.currentTarget).entries())`; do not generate per-input refs or effect-managed submit listeners just to gather values.
- Treat `pp.rpc()` as the default browser-to-server path for CRUD operations and interactive backend reads.
- Use `public/js/pp-reactive-v2.js` as the shipped runtime contract AI should follow.
- Keep `main.py` in view because it injects the runtime-facing attributes and rewrites authored scripts before the browser sees them.
- If a development-only source tree exists behind the shipped runtime, treat it as optional implementation detail rather than something generated apps are guaranteed to contain.
- In authored Caspian templates, do not handwrite `pp-component` or `type="text/pp"`; let the render pipeline inject them.
- For grouped subtrees, follow the section layout pattern in [routing.md](./routing.md), keep the shared interactive shell in the parent folder's `layout.html`, and keep route-specific PulsePoint code in each child `index.html`.
- For grouped shells with independent shell and content scrolling, put `pp-reset-scroll="true"` on the content pane rather than the whole shell when only the page content should reset between child-route navigations.
- Prefer PulsePoint state and template directives over manual DOM mutation for reactive updates.
- Avoid generating ids, `data-*` state, `querySelector`, `getElementById`, `addEventListener`, manual `innerHTML`, or custom event buses for normal Caspian UI behavior.
- Avoid data-attribute click wiring such as `data-action="save"` plus a delegated listener. Use `onclick="save()"` or `onsubmit="{save(event)}"` in authored HTML and keep reactive state in `pp.state(...)`.
- If you are explicitly editing raw runtime HTML or internals, keep `pp-component` unique per live instance.
- In authored templates, use a plain `<script>` inside the root. In runtime HTML, the owned script appears as `script[type="text/pp"]`.
- Keep template-facing variables at top level.
- Follow the HTML-first context pattern: `pp.createContext(...)`, a lowercase provider tag such as `<themecontext.provider value="{...}">`, and `pp.context(token)`.
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
- standard DOM scripting as the default first-party interaction model, including id/data-attribute driven `querySelector(...)`, `addEventListener(...)`, or manual `innerHTML` rendering for normal buttons, forms, filters, toggles, uploads, and reactive lists
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
- `pp.provideContext` is not part of the current runtime API. Use an HTML-first lowercase provider tag such as `<themecontext.provider>`.
- `pp.portal()` preserves logical ancestry through the registry, so context and prop refresh behavior continue to work through portaled descendants.

## Final reminder

If a feature is not implemented in the current bundled runtime, do not invent it.

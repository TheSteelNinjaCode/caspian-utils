---
title: PulsePoint Runtime Guide
description: Use this page when the task mentions PulsePoint, performance, rerenders, slow inputs or searches, `pp.state`, `pp.ref`, `pp.effect`, `pp-ref`, `pp-style`, `pp-for`, portals, `pp-reset-scroll`, SPA navigation, or `public/js/pp-reactive-v2.min.js`. Read "PulsePoint Is Not JSX" before writing templates and "High-performance authoring" before changing runtime reconciliation.
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

Use `components.md` for authoring Python `@component` files and HTML-first `x-*` component tags. Use this page for the browser-side PulsePoint contract and the authoring rules that feed it.

PulsePoint is the default reactive frontend layer for Caspian. Its **script API** deliberately mirrors React hooks. Its **markup is plain HTML**, compiled by a template compiler that has no JSX support of any kind. Those two facts are independent, and conflating them is the single most common way generated PulsePoint code fails. Read the next section before writing a template.

Do not assume React, Vue, Svelte, JSX, Alpine, HTMX, or older PulsePoint docs unless the task explicitly asks for a different frontend contract.

## Hard Invariants

Apply these before generating any template, even without reading the rest of this page:

1. One authored top-level root per route, layout, or component template; the owned plain `<script>` lives inside that root, never as a sibling. A **component** with sibling top-level nodes becomes a props-less fragment; a page or layout with sibling top-level nodes gets a `display: contents` boundary host. Either way there is no fragment tag to write.
2. Never handwrite `pp-component`, `data-pp-ref`, or other runtime-managed attributes; the render pipeline injects them. Keep owned component logic in a plain, untyped `<script>` inside the root.
3. Bind first-party events with native `on*` attributes in the HTML; never wire normal UI with ids, `querySelector`, `addEventListener`, or manual `innerHTML`.
4. For ordinary form submits, use `onsubmit="{handler(event)}"` plus `Object.fromEntries(new FormData(event.currentTarget).entries())`; refs are for imperative access only.
5. Keep reactive values in `pp.state(...)`; keep template-facing bindings at the top level of the script.
6. Call the backend with `pp.rpc(...)` backed by Python `@rpc()` actions; do not invent fetch wrappers or `pp.fetchFunction()`.
7. `pp-for` goes only on `<template>` with plain `key`; context uses `pp.createContext(...)`, a lowercase `<token.provider>` tag, and `pp.context(token)`.
8. If an API is not in `public/js/pp-reactive-v2.min.js`, it does not exist; do not invent hooks, directives, or globals.
9. **Every brace expression in an attribute must be inside quotes**: `class="{expr}"`, never `class={expr}`. Unquoted braces are shredded by the HTML parser.
10. **A `{...}` expression produces text, never elements.** There is no JSX. Conditionals use `hidden="{...}"`, lists use `<template pp-for="…">`.

## PulsePoint Is Not JSX

This section exists because the React comparison in this doc, in `components.md`, and in `index.md` has repeatedly been over-read as permission to write JSX. It is not.

**The React comparison is scoped to exactly two things:**

1. The **hook API inside `<script>`** — `pp.state`, `pp.effect`, `pp.memo`, `pp.ref`, dependency arrays, cleanup functions. These behave like their React counterparts.
2. **Component decomposition** — split a page into small single-responsibility components with props, the way you would split a React tree. This is about *file and responsibility shape*, not syntax.

**The React comparison does not extend to markup — at all.** A Caspian template is an HTML file. It is parsed by an HTML parser first, then compiled. It is never parsed as JavaScript, so JSX expressions in element position are not "unsupported" — they are not even seen as code.

### The JSX constructs that break PulsePoint

Each row is a real failure, not a style preference.

| JSX construct | What actually happens | PulsePoint form |
|---|---|---|
| `{cond && (<div>…</div>)}` | The `{cond && (` becomes literal text; the `<div>` becomes a real always-visible element; the `)}` becomes literal text. | `<div hidden="{!cond}">…</div>` |
| `{cond ? <A/> : <B/>}` | Same — both branches render, plus visible stray text. | Two elements with complementary `hidden` bindings. |
| `{items.map(i => (<li>{i.name}</li>))}` | The `<li>` renders exactly once with the literal text `{items.map(i => (` before it. | `<template pp-for="item in items"><li key="{item.id}">{item.name}</li></template>` |
| An unquoted template-literal class, `class={` … `}` | **Silent HTML corruption.** The parser splits the unquoted value on spaces, producing junk attributes like `${b}="" 'a'=""`. The component root fails to compile and the page renders blank. | Quote the whole thing: ``class="{`a ${b}`}"`` |
| `selected={x === 'y'}` | Same unquoted-attribute corruption, because of the spaces around `===`. | `selected="{x === 'y'}"`, or bind `value` on the `<select>`. |
| `className=` | Not an HTML attribute. Ignored. | `class=` |
| `onClick={fn}` | Not a DOM event attribute in HTML (attributes are case-insensitive, so this lands as `onclick` with a JSX value). | `onclick="{fn()}"` |
| `htmlFor=`, `key={x}` in JSX braces | `htmlFor` is not HTML; `key` is a plain attribute here. | `for=`, `key="{x}"` |
| `<>…</>` fragments | Parsed as an unknown tag. | One real root element — or, in a page/layout only, plain sibling nodes with no wrapper at all. |
| `style={{color:'red'}}` | Double-brace object literal is not CSS text. | `pp-style="{'color:red'}"` — a **string**, not an object. |
| `dangerouslySetInnerHTML` | Does not exist. | Server-render trusted HTML, or use `pp-for` over real data. |

### Why the corruption is silent

`class={...}` and `selected={...}` fail in the parser, not in PulsePoint. The template compiler never gets valid markup, the component root never mounts, and the runtime's reveal step never clears `<body style="opacity: 0">`. The result is a **blank white page with no console error** — the hardest possible failure to diagnose. Quoting the attribute is not cosmetic.

### The one-line test

Before writing a template, ask: *"Would this file be valid HTML if I deleted every `{}` from it?"* If not, it is JSX and it will not work.

## Complete Directive And API Surface

**These lists are closed.** PulsePoint has no other directives, template syntax, or globals. If something is not listed here, it does not exist — do not infer it from React, Vue, Alpine, or another PulsePoint version. Verify against `public/js/pp-reactive-v2.min.js` when in doubt.

### Author-facing template syntax (the whole list)

| Syntax | Where it goes | Purpose |
|---|---|---|
| `{expression}` | Text nodes and **quoted** attribute values | Interpolate a value as text/attribute content |
| `on*` (`onclick`, `oninput`, `onchange`, `onsubmit`, any native DOM event attribute) | Any element | Event binding |
| `pp-for="item in items"` / `"(item, index) in items"` | **`<template>` only** | Keyed list rendering |
| `key="{expr}"` | Repeated sibling inside a `pp-for` template | Keyed diffing identity |
| `pp-ref="name"` / `pp-ref="{expr}"` | Native elements and `x-*` component tags | Imperative element access |
| `defaultvalue="{expr}"` (lowercase) | `<input>`, `<textarea>`, `<select>` | Seed an **uncontrolled** field once, without binding it to state |
| `defaultchecked="{expr}"` (lowercase) | `<input type="checkbox">`, `<input type="radio">` | Seed an **uncontrolled** checkbox/radio once |
| `pp-style="{cssText}"` | Any element | Dynamic inline style, as a **CSS string** |
| `pp-spread="{...obj}"` | Any element | Spread an object into attributes |
| `<token.provider value="{v}">` (lowercase) | Anywhere in markup | Context provider |
| `pp-spa="false"` | An `<a>` that must use normal browser navigation | Opt one link out of the SPA interception enabled automatically by `pp.mount()` |
| `pp-reset-scroll="true"` | A scroll container, or `<body>` | Reset that container's scroll on navigation |
| `pp-scroll-key="stable-name"` | A scroll container | Stable scroll-restoration identity |
| `pp-loading-content="true"` | The region swapped during navigation, usually a layout's content pane | Marks the navigation content region. Without it the runtime falls back to `document.body` |
| `pp-loading-url="/route"` | Runtime-emitted, one per `loading.py` | Route-scope loading lookup. Not authored by hand — add a `loading.py` instead |
| `pp-loading-transition='{"fadeIn":…,"fadeOut":…}'` | An element inside the `loading.py` markup | Navigation fade timings; defaults to 250 ms each way |

There is **no** `pp-if`, `pp-show`, `pp-else`, `pp-model`, `pp-bind`, `pp-class`, `pp-text`, `pp-html`, `pp-on`, or `pp-key`. Conditionals are `hidden="{...}"`; two-way binding is `value="{state}"` plus an `oninput` handler.

**A form control is controlled or uncontrolled for its lifetime, not both.** `value="{state}"` / `checked="{state}"` is the controlled form; `defaultvalue` / `defaultchecked` is the uncontrolled form, applied once and then left alone so the user's typing is never overwritten. Switching an element between the two — usually by binding `value` to state that starts `undefined` — makes the runtime log `[PP-WARN] <input#email> changed from uncontrolled to controlled.` once per element. Fix it by giving the state a defined initial value, not by adding both attributes. These are lowercase HTML attributes; the React spellings `defaultValue` and `defaultChecked` are not props here and do nothing.

### Runtime-managed — never author these

The render pipeline or the browser runtime writes these. Handwriting them corrupts instance tracking or reconciliation:

`pp-component`, `pp-owner`, `pp-event-owner`, `pp-ref-owner`, `pp-ref-forward`, `<pp-context-provider>` (the generated tag), `data-pp-context-token`, `data-pp-context-value`, `data-pp-fragment-root`, `data-pp-script-source`, `data-pp-ref`, `data-pp-input-value`, `data-pp-default-value`, `data-pp-checked-value`, `data-pp-default-checked`, `data-pp-select-value`, `pp-keep`, `pp-keep-run`, and `pp-keep-content`.

The server also emits `meta[name="pp-root-layout"]` for SPA compatibility checks. These names may appear in rendered or transient runtime HTML, but none belongs in authored route or component templates.

Two boundary *shapes* are also runtime-owned and appear only in rendered DOM:

- `<div pp-component="…" style="display: contents">` — the **boundary host**. The render pipeline emits it when a template has no single element of its own to carry `pp-component`: a multi-root page or layout, and a component whose authored root is another `x-*` tag. It is layout-neutral, so it changes identity and scope, never geometry. See "Multi-root pages and layouts" below.
- `<pp-fragment style="display: contents">`, materialized at mount from a `<!--pp:id-->…<!--/pp-->` comment pair — the **range boundary**, a boundary around a run of siblings rather than around one element. Comment markers are legal inside `<tbody>`, `<ul>` and `<select>` where a wrapper element would be foster-parented out by the HTML parser, so this is the wire used where an element host cannot go; the runtime skips materialization under those parents and leaves the markers in place. The compiler emits this pair for a **component** whose template has sibling top-level nodes — the fragment shape — so never hand-write `<!--pp:…-->` markers or a `<pp-fragment>` tag; write the siblings and let the compiler frame them. A multi-root *page or layout* still uses the boundary host above. See "Fragment components" below.

### Component-script hooks (the whole list)

`pp.state`, `pp.effect`, `pp.layoutEffect`, `pp.ref`, `pp.memo`, `pp.callback`, `pp.reducer`, `pp.context`, `pp.portal`, `pp.id`, `pp.errorBoundary`, `pp.syncExternalStore`, `pp.imperativeHandle`, `pp.transition`, `pp.deferredValue`, `pp.optimistic`, plus the `pp.props` bag.

### Runtime utilities (the whole list)

`pp.createContext`, `pp.mount`, `pp.redirect`, `pp.rpc`, `pp.socket`, `pp.enablePerf`, `pp.disablePerf`, `pp.getPerfStats`, `pp.resetPerfStats`.

React hooks with **no** PulsePoint equivalent: `useContext`-as-a-provider-call, `useInsertionEffect`, `useDebugValue`, `useActionState`, `useFormStatus`, `forwardRef`, `memo()` as a component wrapper, `lazy`, `Suspense`, `startTransition` as a free function. Do not generate them.

### Identifiers injected into event handlers

Inside an `on*` attribute the runtime provides `event`, plus the aliases `e`, `$event`, `target` (→ `event.target`), `currentTarget` and `el` (both → `event.currentTarget`). An alias is skipped when the component scope already declares that name.

## Source Of Truth

When documenting or generating PulsePoint code, follow this order:

- `public/js/pp-reactive-v2.min.js` is the shipped browser runtime contract AI should follow.
- `main.py` is the render-pipeline source of truth for component transformation, runtime-attribute injection, and final inert-template deferral.
- If you are working inside PulsePoint or Caspian runtime development code and there is an authoring source tree behind the shipped files, use it only as an implementation detail. Do not assume that source tree exists in generated apps or shipped framework output.

Important current facts:

- `public/js/pp-reactive-v2.min.js` exposes the global `pp` runtime and auto-mounts on DOM ready.
- `main.py` renders the final HTML, transforms components, and defers outermost component roots inside inert templates before returning the response.
- `.venv/Lib/site-packages/casp/components_compiler.py` injects `pp-component` on the final resolved root after component expansion.
- `public/js/pp-reactive-v2.min.js` captures and empties plain component scripts before materialization or morph insertion, then evaluates their source in component scope.
- Authored route and component templates compose reusable server components as HTML-first `x-*` tags before the browser runtime mounts.

If docs, generated examples, or older notes disagree with `public/js/pp-reactive-v2.min.js` plus `main.py`, follow the code that actually runs.

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
- For dashboards, admin areas, account sections, docs sections, and other grouped subtrees, keep shared shell markup and shared PulsePoint behavior in the parent folder's `layout.py`, then keep child-route PulsePoint state local to each route's page template. Follow the same mental model as the Next.js App Router.
- Only introduce another frontend runtime when the user explicitly asks for it or the project already depends on one.

## PulsePoint-First Events And Reactivity

For first-party Caspian HTML, PulsePoint is not a later enhancement after ordinary JavaScript. It is the event, state, and reactivity model AI should use from the start.

Default to this workflow:

- Put the button, form, input, toggle, menu, filter, upload control, or list markup directly in the `html(...)` template of the route, layout, or component that owns it.
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
- The runtime layer: the browser sees `pp-component` roots with plain owned scripts whose source PulsePoint captures before the roots become live.

For authored Caspian templates:

- Keep exactly one authored top-level parent node.
- In source, that parent may be a native HTML element or a single imported `x-*` component tag, but after component expansion it must resolve to one final HTML root.
- Put the component logic inside a plain `<script>` inside that same root.
- Do not handwrite `pp-component="..."`.

Single-root is the shape to write by default. A **component** may have sibling roots, but the alternative is rarely what you meant: a sibling `<script>` after the root, a second top-level element, or stray top-level text makes the component a *fragment* (see "Fragment components" below), and a fragment has no root element to carry props — `pp.props` is silently empty until a call site passes an attribute and gets `FragmentPropsError`. `TemplateRootError` covers a component template with no root at all, or whose only root is an unresolvable `x-*` tag.

### Multi-root pages and layouts

A **page** (`page()` → `html(...)` in `index.py`) and a **layout** (`layout()` in `layout.py`) may return more than one top-level node. When the compiler finds no single root there, it wraps the whole template in a layout-neutral boundary host instead of raising:

```python
# src/app/demo/index.py — legal
return html(r"""
  <h1>Title</h1>
  <p>Body</p>
  <script>
    const [count, setCount] = pp.state(0);
  </script>
""")
```

renders as:

```html
<div pp-component="page_…" style="display: contents"><h1>Title</h1><p>Body</p>…</div>
```

What this does and does not change:

- **It applies to pages and layouts only** — a page's `index.py` and a layout's `layout.py`. Components take the fragment shape instead (next bullet).
- **A component takes a different shape.** A multi-root component is a *fragment* and gets the comment-pair range boundary instead of this host — see "Fragment components" below.
- **`display: contents` means the host is not in layout**, so it does not break a flex/grid parent, and margins, gaps and selectors behave as if the children were direct siblings. It is still a real element in the DOM and in `querySelector` results.
- **The owned `<script>` still belongs inside the template**, and is still one script for the whole boundary — the host is the boundary, so the script's `pp.state`, refs and handlers cover every root.
- **`pp.props` still reads from the host.** A layout or page host carries no `x-*` attributes, so this matters mainly for the component case below.

Prefer a real wrapper element when you have a natural one; the host exists so that a page whose sections are genuinely siblings does not need a meaningless `<div>`.

### Fragment components

A **component** whose `html(...)` has sibling top-level nodes is a fragment — the equivalent of React's `<>…</>`. There is nothing to type: write the siblings.

```python
# src/app/demo/Tally.py — legal
@component
def Tally(**props):
    return html(r"""
      <button onclick="{setN(n + 1)}">{n}</button>
      <p>{n}</p>
      <script>const [n, setN] = pp.state(0);</script>
    """)
```

renders as:

```html
<!--pp:tally_7efd1e81--><button onclick="{setN(n + 1)}">{n}</button><p>{n}</p>…<!--/pp-->
```

The comment pair is the boundary. At mount the runtime replaces it with a live `<pp-fragment style="display: contents" pp-component="tally_7efd1e81">`, which behaves as an ordinary boundary from then on: state, events, refs, context and re-renders all work, and sibling instances stay independent.

Why comments rather than the `display: contents` host a page gets:

- **A fragment adds no element to the served markup.** The host is a real `<div>` in the response; the fragment is not.
- **Comments are legal in every content context.** A wrapper element written inside `<tbody>`, `<tr>`, `<select>` or `<optgroup>` is foster-parented out by the HTML parser, taking its children with it. This makes a component that returns two `<tr>` rows expressible for the first time.

Two rules:

- **A fragment cannot receive props.** Props arrive as attributes on a component's rendered root, and a fragment has no root — `pp.props` would be silently empty. Passing any attribute (including `pp-ref`) on the `<x-*>` tag of a fragment component raises `FragmentPropsError` at render time, naming the attributes. Give the component a single native root when it takes props.
- **Inside `<tbody>`/`<tr>`/`<select>`/`<optgroup>` a fragment is grouping only.** The runtime deliberately leaves markers under those parents as comments rather than inserting a wrapper the next re-render would hoist, so the content renders but the fragment owns no identity there. A fragment that needs its own `<script>`, state or events must sit somewhere an element could also live.

Never hand-write `<!--pp:…-->` or `<pp-fragment>`. They are compiler and runtime output; an authored marker is refused by the render cache and will not behave like a boundary.

### The composition host

The same layout-neutral host appears in a second case: a component whose authored root is *itself* another component tag.

```python
return html(r"""<x-card title="{label}">…</x-card>""", label=label)
```

The inner `<x-card>` root already owns a `pp-component`, so this component is given its own host to own — and the props the parent passed on *its* `x-*` tag are forwarded onto that host, which is what makes them resolve as `pp.props` in this component's script, evaluated in the parent's scope. The runtime also stamps `pp-ref-forward` on the host so a parent's `pp-ref` on this component still lands on the concrete DOM root inside, matching ref forwarding through a wrapper.

Practical consequence when reading rendered DOM: an extra `display: contents` div between two component roots is expected output, not a bug and not something to author around.

A page or layout whose authored root is an `x-*` tag is given a host too, and it claims its `<script>` the same way a component does. Slot content authored by a page or layout is owned by the alias `app`, and the runtime resolves that alias — to the nearest page boundary, or to the root boundary for a layout — before deciding which component a slot-authored script belongs to, so the script runs in the page or layout scope exactly as authored. See [A slot-authored `<script>` belongs to the template that authored it](#a-slot-authored-script-belongs-to-the-template-that-authored-it).

Authoring is Python-only: `index.py` **is** the route's template body (`page()` returns markup through `html(r"""...""")`), `layout.py` is the shell's, and a component's markup lives in its own `.py`. There is no separate HTML file to keep markup in. The same module also owns that route's data, metadata, props, `@rpc()` actions, auth, caching, and redirects — markup and server logic are deliberately co-located, not split across files.

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

When that template reaches the browser, Caspian has injected the component id and deferred the root inside an inert template. PulsePoint captures the plain script source before materialization so native execution cannot occur, then evaluates that source in component scope.

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
- The owned script remains a plain, untyped `<script>`.
- Before a deferred root or later morph becomes live, PulsePoint captures and empties that script; the captured source is evaluated only through the component runtime.
- The script lookup walks the current root and skips nested `pp-component` boundaries, so a parent does not consume a child component's script.
- If multiple matching runtime scripts exist in the same root, the first matching owned script wins. Generate one script per root.
- Authored route, layout, and component templates still need one top-level parent node so Caspian can inject the component boundary correctly after component expansion.
- A scriptless component root still mounts and can receive props, refs, events, and nested children, but it does not create local bindings from those props on its own.
- Component scripts are plain JavaScript executed with `new Function(...)`. Do not use `import`, `export`, or top-level `await` inside them.
- The runtime auto-returns supported top-level bindings from the script. Do not rely on manual `return { ... }` objects.
- `pp.props` contains the current prop bag for the component.
- `pp.props` is computed from the **rendered root element's attributes**, not from any server-side signature. A prop that never lands on the root is `undefined` in the browser, with no server error and no console warning. In a Python component this means every prop the template references must be forwarded through `get_attributes({...}, props)`; see [components.md](./components.md#every-prop-a-template-reads-must-be-forwarded-to-the-root).
- Prop value types depend on how the attribute reaches the root. An unevaluated brace expression such as `volume="{vol}"` is evaluated in the parent component's scope and keeps its real type (number, boolean, object, function). A literal attribute such as `volume="0"` arrives as the **string** `"0"`, so `volume === 0` is false. A valueless attribute arrives as boolean `true`. Coerce or compare loosely when the value may be a server-rendered literal.
- Attribute names round-trip through kebab-case: `isFullscreen` renders as `is-fullscreen` and returns as `pp.props.isFullscreen`.
- Attribute names that are JavaScript reserved words are dropped from the prop bag, so `pp.props.class` and `pp.props.for` never exist. Use a non-reserved prop name when the script must read the value.
- `pp.props.children` contains the root's initial inner HTML before the owned script is removed from the render template.
- Props are not auto-injected as standalone top-level template or handler variables. Read them through `pp.props` or explicitly destructure them in the component script.

Bindings exported to the template:

- Top-level function declarations.
- Top-level identifier declarations such as `const title = "Pulse"`.
- Every identifier in a top-level destructuring pattern, regardless of initializer: arrays, objects, nested patterns, defaults, rest elements, and holes. This includes `const { title, subtitle } = pp.props`, `const [count, setCount] = pp.state(0)`, and tuples returned by any other hook or helper.

Bindings that should not be assumed:

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

That is the authored form. In browser-inspected runtime HTML, Caspian will already have added the component id; the owned script remains plain and is removed after PulsePoint captures its source.

### A slot-authored `<script>` belongs to the template that authored it

A template whose authored root is an `x-*` tag writes its `<script>` inside that root, which makes the script travel as **slot content**: it is compiled into a `<template pp-owner="…">` wrapper naming the scope that authored it, physically inside the child's boundary in the served HTML. The runtime resolves that owner and hands the script back to the template that wrote it:

- A **component** whose authored root is another component tag gets a composition host carrying the same id the slot content is owned by, so it claims its script directly.
- A **page or layout** authors slot content owned by the alias `app`. The runtime resolves `app` to the nearest page boundary (or to the root boundary for a layout) and that boundary claims the script, so `pp.state`, effects, and functions declared there exist in the page or layout scope exactly as authored.

The recovered script is executed once, by its owner — it is **not** rendered into the child's DOM as slot content, so no inert script element appears in the child's subtree and the script's source text is never compiled as brace expressions.

This shape is fully supported — a page whose root is a composition component does not need a wrapper element:

```python
@component
def page(**props):
    return html(r"""
      <x-shell>
        <button onclick="{doSignout()}">Sign out</button>

        <script>
          function doSignout() { pp.rpc("signout"); }
        </script>
      </x-shell>
    """)
```

The slot bindings resolve in the page's scope, the child keeps its own independent scope, and a state name shared by both (the page's `myVar` and the child's `myVar`) stays two separate states — the same behavior as two `useState` calls in different React components.

To confirm ownership from the browser, read the **served HTML** — view source or the document's network response, not the post-mount DOM, because the runtime empties and removes the script element once it has captured the source. The `<script>` sits inside a `<template pp-owner="X">`; `X` is either the owning boundary's `pp-component` id or the alias `app`, which resolves to the page or layout that authored it.

### A handler in slot content runs in the authoring template's scope

**Symptom:**

```
[PP-ERROR] Handler failed: ReferenceError: doSignout is not defined
Code: doSignout()
```

with a stack frame inside the function the event manager compiles (`eval at getCompiledHandler`). The click fires — binding worked — and the function is plainly declared *somewhere*, so the natural next guesses are that the element moved, that it needs its own component, or that a portal broke the connection. None of those is the cause.

**Rule: the function an `on*` handler calls must be defined by the owned script of the template that literally authored that markup — not by the component the element ends up inside.**

Markup passed as children is compiled in the authoring template's scope and carries that owner through to event binding, so the handler is evaluated against the author's component scope wherever the element is finally rendered. Two consequences worth stating outright:

- **Wrapping the element in a component that defines the function does not fix it.** The element then carries its own `pp-component` *and* the event-owner marker from the authoring template, and the owner wins. An agent that reaches for a wrapper component will conclude the runtime is broken.
- **Portaling is not the cause.** `pp.portal` preserves logical component ancestry, and context, props, and events keep working through a portaled subtree. A handler inside a portaled dropdown, popover, or dialog fails for the reason above, not because it was portaled.

To diagnose, find the template whose source literally contains the `on*` attribute; that template's script must declare the function. When you need to confirm from the browser, the element carries an internal `__pp_event_owner_<event>` property (`__pp_event_owner_click`) naming the boundary whose scope the handler resolves in — compare it against the boundary you expected. That is runtime bookkeeping for debugging only, not a stable API: never read or write it from app code.

Wrong — the handler is authored by the page, but the function lives in the child that renders it:

```html
<!-- authored by the page -->
<x-user-menu>
  <button onclick="{doSignout()}">Sign out</button>
</x-user-menu>

<!-- authored by UserMenu — its script cannot serve the page's markup -->
<div class="menu">
  {children}
  <script>
    function doSignout() { pp.rpc("signout"); }
  </script>
</div>
```

Right — the function moves to the template that authored the handler. The script may stay inside the `x-*` root; as slot content it is still owned by the authoring page (see the previous section):

```html
<!-- authored by the page -->
<x-user-menu>
  <button onclick="{doSignout()}">Sign out</button>

  <script>
    function doSignout() { pp.rpc("signout"); }
  </script>
</x-user-menu>
```

The alternative fix is to move the *markup* instead of the function: let the child render its own button and pass data in as props. Either way, the handler and its function belong to one template.

## Hooks and runtime API

PulsePoint uses a React-style mental model **inside each component script**: stateful render scope, dependency-based effects, refs, reducer-style updates, context consumption, and portals. That similarity stops at the `<script>` boundary — the surrounding markup is plain HTML, never JSX. See [PulsePoint Is Not JSX](#pulsepoint-is-not-jsx).

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
- `pp.id()` returns a stable, DOM-safe unique id for the life of that hook slot.
- `pp.syncExternalStore(subscribe, getSnapshot)` subscribes to a mutable source outside PulsePoint and rerenders when its snapshot changes.
- `pp.imperativeHandle(ref, createHandle, deps?)` publishes an imperative API on a parent-owned ref instead of the raw DOM node.
- `pp.transition()` returns `[isPending, startTransition]`.
- `pp.deferredValue(value, initialValue?)` returns a copy of `value` that catches up one commit later.
- `pp.optimistic(passthrough, reducer?)` returns `[optimisticState, addOptimistic]`.
- `pp.errorBoundary()` returns `[error, reset]` and marks the component as an error boundary.
- `pp.props` exposes the current props.

Global helpers exposed through the `pp` singleton and also merged into the component runtime:

- `pp.createContext(defaultValue)` creates a context token.
- `pp.mount()` bootstraps the runtime. It is idempotent.
- `pp.redirect(url)` performs SPA-aware navigation when enabled.
- `pp.rpc(name, data?, optionsOrAbort?)` performs the current route RPC bridge.
- `pp.socket(name, args?, handlers?)` opens a named server socket: a bidirectional, long-lived channel to a server-side `@socket()` function. It connects to the framework's single socket endpoint with the function's name in the `name` query parameter, sends `args` as the connection's first frame (one JSON object — the same payload shape `pp.rpc` would post), and every later frame is one JSON value in either direction. `handlers` accepts `onOpen`, `onMessage(value)`, `onError(error)`, and `onClose({ code, reason, wasClean })`; a server frame shaped `{"error": "..."}` (that key alone) is reserved for failures and routed to `onError`. The returned handle exposes `send(value)` (returns `false` once closed; frames sent before the connection opens are buffered and flushed), `close(code?, reason?)`, and `readyState`. Open the socket inside `pp.effect(..., [])`, keep the handle in `pp.ref(...)`, and close it in the effect's cleanup. Use it only for genuinely bidirectional channels; ordinary reads/writes stay on `pp.rpc`, and one-way server push stays on RPC streaming.
- `pp.enablePerf()` enables render timing collection.
- `pp.disablePerf()` disables render timing collection.
- `pp.getPerfStats()` returns collected render timings.
- `pp.resetPerfStats()` clears collected render timings.

Notes:

- The global `pp` singleton auto-mounts once the runtime is loaded and the DOM is ready. Manual `pp.mount()` is still safe because it short-circuits after the first mount.
- `pp.state` setters accept either a value or an updater function.
- `pp.effect` and `pp.layoutEffect` are cleanup-style hooks. They may return only a synchronous cleanup function; returning a promise warns and is ignored, so start async work inside the effect instead.
- A function used through `pp-ref` may return a synchronous cleanup function. PulsePoint runs it when that callback ref is replaced or detached; a callback that returns no cleanup is instead called with `null` on detach.
- `pp.portal(ref)` defaults to `document.body` when no target is provided.
- Older docs may call the RPC helper `pp.fetchFunction()`. In the current bundled runtime the implemented global API is `pp.rpc()`.
- Keep template-facing bindings at the top level so the AST-based exporter can see them.
- For predictable code generation, prefer passing an explicit dependency array to `pp.effect`, `pp.layoutEffect`, `pp.memo`, and `pp.callback`.
- Top-level destructuring is exported to template scope for every form: array patterns, object patterns, nested patterns, defaults, rest elements, and holes. `const [isPending, startTransition] = pp.transition()` and `const [error, reset] = pp.errorBoundary()` reach the template the same way `pp.state` does.
- `pp.id()` is derived from the component id plus the hook slot, so the same slot keeps its id across rerenders and two instances of the same component never collide. Use it for `id`/`for` pairing and `aria-*` references instead of hand-rolled counters.
- `pp.syncExternalStore` requires a referentially stable `subscribe`. Wrap it in `pp.callback(..., [])`, otherwise the store is unsubscribed and resubscribed on every render. The runtime re-reads the snapshot immediately after subscribing so a change that lands between render and subscription is not lost.
- `pp.transition` does not deprioritize rendering the way React's concurrent scheduler does, because PulsePoint renders synchronously. It does give a correct `isPending` flag for both synchronous and promise-returning scopes, which is the common use (disabling a submit button while `pp.rpc()` resolves). Overlapping scopes stay pending until the last one settles.
- `pp.optimistic` drops its pending actions as soon as `passthrough` changes, which is the point at which the server has confirmed or rejected the guess. Without a reducer, each action replaces the value.
- `pp.deferredValue` lags by one commit. Use it to keep an input responsive while an expensive derived subtree catches up.

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

Stable ids for accessible field pairing:

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

Never derive ids from an index or a module-level counter. `pp.id()` is the only form that stays stable across rerenders and unique across instances of the same component.

External store subscription:

```html
<section>
  <p>{isNarrow ? "Compact layout" : "Wide layout"}</p>

  <script>
    const subscribe = pp.callback((onStoreChange) => {
      const query = window.matchMedia("(max-width: 640px)");
      query.addEventListener("change", onStoreChange);
      return () => query.removeEventListener("change", onStoreChange);
    }, []);

    const isNarrow = pp.syncExternalStore(
      subscribe,
      () => window.matchMedia("(max-width: 640px)").matches
    );
  </script>
</section>
```

Prefer this over `pp.state` plus `pp.effect` for anything the component does not own. It reads the snapshot during render and closes the gap between render and subscription, which a hand-rolled effect silently drops.

Imperative handle for a child-owned API. The parent passes its own ref down as an ordinary prop:

```html
<!-- parent -->
<div>
  <button onclick="{dialogControl.current.open()}">Open</button>
  <x-confirm-dialog control-ref="{dialogControl}" />

  <script>
    const dialogControl = pp.ref(null);
  </script>
</div>
```

```html
<!-- ConfirmDialog component -->
<section>
  <script>
    const [open, setOpen] = pp.state(false);

    pp.imperativeHandle(
      pp.props.controlRef,
      () => ({
        open: () => setOpen(true),
        close: () => setOpen(false),
      }),
      []
    );
  </script>

  <div hidden="{!open}">
    <button onclick="{setOpen(false)}">Close</button>
  </div>
</section>
```

Notes:

- A prop whose bound expression evaluates to an object is passed through by reference, so `control-ref="{dialogControl}"` arrives as the real ref object on `pp.props.controlRef`.
- Do **not** author `pp-ref-forward` to make this work. That attribute is runtime-managed, and `pp-ref` on an `x-*` tag binds the child's root DOM node, which is a different feature. Use a normal prop when the parent needs a child-defined API rather than an element.
- The handle is published in the layout-effect phase and cleared on unmount, so parents should call into it from event handlers or effects, not during their own render.

Optimistic update around an RPC call:

```html
<section>
  <button onclick="{addLike()}">Like ({likes})</button>

  <script>
    const [confirmedLikes, setConfirmedLikes] = pp.state(pp.props.likes ?? 0);
    const [likes, addOptimisticLike] = pp.optimistic(
      confirmedLikes,
      (state, delta) => state + delta
    );

    async function addLike() {
      addOptimisticLike(1);
      const result = await pp.rpc("like_post", { postId: pp.props.postId });
      setConfirmedLikes(result.likes);
    }
  </script>
</section>
```

The optimistic `+1` is discarded the moment `confirmedLikes` changes, so the server value never double-counts the guess.

Transition for pending async work:

```html
<section>
  <button onclick="{save()}" disabled="{isSaving}">
    {isSaving ? "Saving..." : "Save"}
  </button>

  <script>
    const [isSaving, startSaving] = pp.transition();

    function save() {
      startSaving(() => pp.rpc("save_profile", { name: pp.props.name }));
    }
  </script>
</section>
```

### Error boundaries

`pp.errorBoundary()` marks a component as the recovery point for render and effect failures. Without one, a throwing render is logged to the console and leaves the subtree in whatever state it reached.

```html
<section>
  <script>
    const [error, reset] = pp.errorBoundary();
  </script>

  <div hidden="{!error}">
    <p>Something went wrong: {error?.message}</p>
    <button onclick="{reset()}">Try again</button>
  </div>

  <div hidden="{!!error}">
    <x-report-chart />
  </div>
</section>
```

Behavior to rely on:

- Errors thrown during a child's render or effects bubble up the logical component ancestry to the nearest boundary. If nothing catches, the runtime logs `[PP-ERROR] Render Cycle Failed` as before.
- Unlike React, a boundary also catches throws from its **own** render and effects, because in Caspian the boundary and its fallback markup live in the same single-root component.
- Effect callbacks and effect cleanups take the same path as render failures, so a throwing subscription teardown is not silently swallowed.
- The boundary **latches**: it keeps holding the error until `reset()` is called.
- A boundary absorbs at most five errors without an intervening `reset()`, then logs and stops catching. This stops a fallback that itself throws from spinning the render loop. `reset()` re-arms it.
- Event handler errors are not routed to boundaries. Handle those with `try`/`catch` inside the handler, which matches React.

## High-performance authoring

PulsePoint renders synchronously when a component requests an update. That makes state ownership and component boundaries part of the performance contract: the runtime can optimize reconciliation, but it cannot decide that an authored state change was unnecessary.

### State means "render required"

Use `pp.state(...)` when changing the value must affect at least one render-facing behavior:

- template text, attributes, lists, or conditional visibility
- a bound child prop or provided context value
- a memo, effect, or other render dependency that must follow the value
- authoritative data returned from an RPC call

Use `pp.ref(...)` when a mutable value must persist but changing it should not render:

- debounce and timeout handles
- request generations or stale-response tokens
- pagination cursors that are not displayed
- previous values and imperative integration handles
- transient search text used only to build a later RPC payload

A ref mutation is intentionally invisible to the template. Do not replace state with a ref when the UI is expected to update.

### Debounce frequency and render cost are different

A debounce answers "how often should this work start?" It does not answer "how much work happens when it starts?"

This shape can still stall a large owner:

```html
<script>
  const [query, setQuery] = pp.state("");

  function onSearch(value) {
    clearTimeout(onSearch.timer);
    onSearch.timer = setTimeout(() => setQuery(value), 300);
  }

  pp.effect(() => {
    loadRows(query);
  }, [query]);
</script>
```

When the timer fires, `setQuery` asks the whole owning component to render even if `query` never appears in its markup. If that owner also contains a large list, dialogs, forms, providers, or many nested components, typing that resumes near the timer can compete with the render.

When query text exists only for an RPC request, keep the query and timer in refs, debounce the request itself, and render only the accepted result:

```html
<section>
  <input
    type="search"
    placeholder="Search products"
    oninput="scheduleSearch(event.target.value)"
  />

  <template pp-for="row in rows">
    <article key="{row.id}">{row.label}</article>
  </template>

  <script>
    const [rows, setRows] = pp.state([]);
    const query = pp.ref("");
    const timer = pp.ref(null);
    const generation = pp.ref(0);

    function scheduleSearch(value) {
      query.current = value;
      const requestGeneration = ++generation.current;

      clearTimeout(timer.current);
      timer.current = setTimeout(
        () => search(requestGeneration),
        300
      );
    }

    async function search(requestGeneration) {
      const response = await pp.rpc(
        "search_products",
        { query: query.current },
        { abortPrevious: true }
      );

      if (requestGeneration !== generation.current) return;
      setRows(response.items ?? []);
    }

    pp.effect(
      () => () => clearTimeout(timer.current),
      []
    );
  </script>
</section>
```

The native input remains immediate, timer/query bookkeeping does not render, and only the authoritative rows update the list. The generation check prevents an older response from overwriting a newer search. See [fetch-data.md](./fetch-data.md#search-filters-and-request-races) for the RPC-side rules.

### Keep high-frequency state close to the control

Component boundaries define who owns a render. A keystroke-frequency value should live in the smallest component that needs to render from it.

- Split a search/toolbar from a large result grid when their update frequencies differ.
- Split frequently edited form state from unrelated dialogs, page shells, and providers.
- Keep shared state in a parent only when multiple children genuinely need the same render-driving value.
- Do not create wrapper components that still own all high-frequency state and therefore rerender every subtree.

For a controlled input that genuinely must write state on every keystroke, keep its owner small. If an expensive derived consumer may lag by one commit, use `pp.deferredValue(...)` for that consumer:

```html
<section>
  <input value="{query}" oninput="setQuery(event.target.value)" />
  <x-large-results query="{deferredQuery}" />

  <script>
    const [query, setQuery] = pp.state("");
    const deferredQuery = pp.deferredValue(query, "");
  </script>
</section>
```

`pp.deferredValue` is not a substitute for correct ownership, and `pp.transition()` does not time-slice or deprioritize work. PulsePoint remains synchronous.

### Avoid duplicate and invisible commits

Each setter that changes a value requests a render, even when the eventual UI change is small.

- Do not set `loading=true` for a background refresh when existing rows remain visible and no loading indicator changes.
- Batch related state changes in the same synchronous turn when possible; PulsePoint coalesces scheduled updates.
- Do not mirror the same value in multiple state hooks.
- Keep pagination/request cursors in refs when moving the cursor should only start work, then put returned rows and visible metadata in state.
- Memoize only genuinely expensive derivations or values whose identity matters. `pp.memo` does not make an unnecessary owner render free.
- Use keyed `pp-for` rows so reconciliation can preserve row identity.

### Keyed rows are reconciled per row, not per list

A render rebuilds the whole component as one HTML string and reconciles it against the live DOM. For a list, that used to mean changing one row cost the same as rebuilding every row: the entire list was re-serialized, re-parsed and re-walked.

The runtime now remembers the markup each keyed row produced. A row that renders byte-identically again is not re-parsed and not re-diffed — its live node is kept, along with its attributes, text and bound handlers. Runs of consecutive unchanged rows collapse together, so a list where a handful of rows changed costs about what those rows cost, largely independent of how long the list is.

This is automatic, but it only engages for rows the runtime can safely stand in for. A row participates when:

- the loop body renders **exactly one root element** per row,
- that element carries a non-empty, unique `key`,
- the loop is not nested inside another `pp-for`, and
- the row's markup contains no nested component boundary, owned slot content, or context provider.

The same machinery extends to a keyed list that toggles between empty and populated (an accordion pane, a filtered-to-nothing list, a collapsed section): when a list of simple component rows closes, the runtime sets the destroyed rows aside and a reopen with the same keys revives them — same DOM nodes, same identities — instead of rebuilding and re-mounting every row. Revival is reuse, not remount: a revived row's script re-runs only when its props actually changed (the same rule as any prop refresh), never merely because the list closed and reopened. Do not put side effects in a row's script that depend on re-running per reopen; derive row output from props.

Practical consequences when authoring a large list:

- **Always key rows.** An unkeyed row can never be reused, so an unkeyed list still reconciles in full.
- **Keep the row body single-rooted.** A loop body with two sibling elements, or with bare text next to the element, opts the whole loop out. Wrap the row in one element.
- **A row built entirely from a child component** (`<template pp-for="row in rows"><x-row key="{row.id}" … /></template>`) is reconciled the normal way, because the parent is responsible for refreshing that boundary's bound props. Prefer plain markup in the row when the list is large and its rows rarely change.
- Keep values that feed a row stable. A row whose markup is regenerated identically is free; a row whose markup differs by even one character is full work.

### A mounted child boundary is reconciled by its attributes, not by its markup

The same idea applies to composition, which is the shape most pages actually have: a shell that assembles many `x-*` components rather than one component owning a wall of markup.

A child component's internals belong to the child, so the parent never reconciles into them — it only reconciles the child's root, whose attributes carry the props. Once the child is mounted, the parent therefore emits that root **with an empty body** instead of re-emitting the child's markup, which means a parent render does not re-serialize or re-parse subtrees it was going to ignore. In a real browser, a shell of 30 small children went from 0.82 ms to about 0.30 ms per parent update, and the cost stopped scaling with how much markup the children contain.

This is automatic and needs no authoring change, but the shape still matters:

- **The first render of a boundary always emits its full markup**, because the child bootstraps from what the parent emitted. A boundary that a conditional removes and later restores starts that cycle again.
- **A boundary is excluded when the parent is responsible for its content**: owned/slot content (`pp-owner`), a context provider inside it, or a parent-owned `pp-ref` capture inside it. Those keep re-emitting in full.
- **Props stay live.** Attributes on the boundary root are emitted and reconciled on every render exactly as before, so a child whose props changed still re-renders.
- **Prefer passing data as props over projecting large slot content** into a frequently re-rendering shell — projected content is re-rendered by its owner, so it does not get this treatment.

If a reused boundary is ever found not to be mounted, the runtime logs `[PP-WARN] Nested boundary content reuse was abandoned…`, permanently disables the reuse for that component, and re-renders it from full markup. Seeing that warning means the cache and the DOM disagreed — report it rather than working around it.

Two related properties follow from the same idea, and they are worth knowing when you reason about what a parent render costs:

- **A bound prop is committed to the child's root only when its value changes.** `active="{selected === 3}"` is evaluated every render, but the attribute is written only on the render where the answer actually flips. The live attribute holds the evaluated value the whole time — it never flickers back to the authored `{expr}` text mid-render, so a `MutationObserver` or a CSS transition watching a component root sees only real changes.
- **A child whose props did not change is not re-walked.** A leaf child costs its parent one prop comparison, not a traversal of its subtree.

Together with the elided body, a mounted child that nothing changed about costs its parent: one prop evaluation per bound attribute, one shallow comparison, and no DOM writes.

### Prop identity decides whether a child re-renders

A child re-renders when its props changed, and props are compared **shallowly, by identity**. Every brace expression on a boundary root is re-evaluated on each parent render, so an expression that *builds* a value returns a new identity every time and the child re-renders even when the data is identical.

This is the single most common cause of "my whole page re-renders when I type one character":

```html
<!-- Every parent render creates a new array and a new function, so this child
     re-renders on every parent render even when `products` never changed. -->
<x-product-table
  rows="{products.filter((p) => p.active)}"
  on-select="{(row) => setSelected(row.id)}"
/>
```

Give each of them a stable identity, and the child is skipped entirely when nothing it depends on changed:

```html
<x-product-table
  rows="{activeProducts}"
  options="{tableOptions}"
  on-select="{selectRow}"
/>

<script>
  const [products, setProducts] = pp.state([]);
  const [selected, setSelected] = pp.state(null);

  const activeProducts = pp.memo(
    () => products.filter((p) => p.active),
    [products]
  );

  // Constant for the life of the component.
  const tableOptions = pp.memo(() => ({ dense: true }), []);

  const selectRow = pp.callback((row) => setSelected(row.id), []);
</script>
```

Rules of thumb:

- **Primitives are free.** `label="Total"`, `count="{items.length}"` and `active="{selected === row.id}"` compare by value, so they only "change" when the value really changes. Prefer passing primitives over passing an object the child immediately destructures.
- **Build object props in the script, never inline in the attribute.** An attribute value that starts with `{{` is read as a server template block, not a PulsePoint binding, so an inline object literal is not valid there — assign it to a name in `<script>` (memoized) and pass that name.
- **`pp.memo` for derived arrays/objects, `pp.callback` for handlers** passed as props. Both need a dependency array; without one they rebuild every render and nothing is gained.
- **Do not memoize primitives.** `pp.memo(() => a + b, [a, b])` costs more than the addition.
- This only matters for values crossing a component boundary. Inside one component's own markup, an inline object or arrow function is harmless.

### Give a provider a stable context value

Context values are compared by identity too, and a provider's consumers are refreshed whenever that identity changes. Providing a freshly built object makes **every consumer in the subtree** re-render on **every** provider render — the widest-blast-radius version of the problem above:

```html
<!-- Wrong: a new object every render, so all consumers refresh every render. -->
<script>
  const [theme, setTheme] = pp.state("light");
  ThemeContext.Provider({ value: { theme, setTheme } });
</script>
```

```html
<!-- Right: identity changes only when `theme` does. -->
<script>
  const [theme, setTheme] = pp.state("light");

  const themeValue = pp.memo(() => ({ theme, setTheme }), [theme]);
  ThemeContext.Provider({ value: themeValue });
</script>
```

Split unrelated values into separate contexts when they change at different rates. A context carrying both a rarely-changing `user` and a per-keystroke `query` forces every `user` consumer to re-render on each keystroke; two contexts keep each consumer tied to what it actually reads.

### Keep effects from doing work nobody asked for

An effect with no dependency array runs after **every** render of its component:

```html
<script>
  // Runs on every render, refetching on unrelated state changes.
  pp.effect(() => { loadRows(page); });

  // Runs only when `page` changes.
  pp.effect(() => { loadRows(page); }, [page]);

  // Runs once on mount.
  pp.effect(() => { subscribe(); return () => unsubscribe(); }, []);
</script>
```

Dependencies are compared by identity as well, so an object or function rebuilt each render belongs in `pp.memo`/`pp.callback` before it is used as a dependency — otherwise the array never matches and the effect runs every render anyway.

### Diagnose the owner before blaming the runtime

Nearly every "PulsePoint is slow" report is an authoring or ownership problem, and the fix is in the template or the component split — not in the runtime. Classify the symptom first:

| Evidence | Likely classification | First action |
| --- | --- | --- |
| A state setter fires, but the changed value is not rendered | Component authoring issue | Move non-rendering bookkeeping to a ref or call the action directly |
| A debounce fires just before typing stalls | Component/request ownership issue | Debounce the RPC or expensive action, not an unnecessary broad-owner setter |
| Loading and result setters produce two large commits | Component authoring issue | Remove invisible loading commits or isolate the visible indicator |
| A high-frequency control owns a large list/provider/dialog subtree | Component-boundary issue | Move the state to a smaller focused component |
| A child re-renders on every parent render although its visible props look unchanged | Component authoring issue | A prop is a new object/array/function identity each render — see "Prop identity decides whether a child re-renders" |
| Every context consumer re-renders whenever the provider renders | Component authoring issue | The provided `value` is a fresh object each render — memoize it |
| Changing one row of a large list costs about as much as changing every row | Component authoring issue | Check the row is keyed, single-rooted, and not a component boundary, so per-row reuse can engage |
| Rendered HTML is byte-identical but a large stable subtree is still reconciled | Possible runtime issue | Capture `pp.getPerfStats()` for the interaction and report it with the component shape |
| One small necessary binding change spends disproportionate time in `domDiff` or `bootstrapNested` | Possible runtime issue | Reduce it to the smallest reproducing template, then report it with the phase numbers |

Enable measurements only while profiling:

```html
<script>
  pp.enablePerf();
  pp.resetPerfStats();

  // Reproduce one interaction, then inspect:
  console.log(pp.getPerfStats());

  pp.disablePerf();
</script>
```

Correlate the expensive component and phase with the setter or RPC completion that requested it. Compare the same interaction, DOM size, data, and number of repetitions before and after a change. Input paint timing and RPC latency are separate from render timing, so measure them separately.

Do not "fix" a PulsePoint performance problem with `querySelector`, manual listeners, `innerHTML`, or parallel DOM state. Those bypass ownership instead of correcting it.

## Context

Context uses a React-style provider pattern: a token, a provider tag in markup, and `pp.context(token)` in descendants. Because Caspian templates are HTML-first, authored provider tags should be written in lowercase HTML form, for example `<themecontext.provider>`, even when the JavaScript token is named `ThemeContext`.

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
- The preferred authoring style is a lowercase provider tag in the template. The runtime recognizes `*.provider` tags case-insensitively and rewrites them to runtime-owned `<pp-context-provider>` boundaries. The context lookup is also case-insensitive, so `<themecontext.provider>` can resolve a script binding named `ThemeContext`.
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

In Caspian single-file Python components, the root element's attributes must be authored via `get_attributes(...)` and `{{ attributes }}`; see [components.md](./components.md#receiving-props-in-a-python-component). Attributes accepted from an `x-*` tag but not forwarded to the rendered root are dropped and never reach `pp.props`.

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
- **Attribute brace expressions must be quoted.** Write `class="{expr}"`, `disabled="{isSaving}"`, `selected="{role === 'admin'}"`. The unquoted JSX form `class={expr}` is not a PulsePoint syntax variant — it is invalid HTML. An unquoted value ends at the first space, so everything after it is re-parsed as further attribute names, and the element (and usually the whole component root) is destroyed before the compiler ever runs. This is silent: no console error, just a blank page.
- **An interpolation evaluates to a value, never to markup.** The compiler serializes the result with the JSX-child rules in "Value serialization is the JSX child contract" below — it is *not* a plain `String(...)` coercion — and HTML-escapes it. Elements cannot come out of an expression, so there is no JSX-style element-returning branch or `.map()`. Use `hidden="{...}"` for conditionals and `<template pp-for="…">` for lists.
- Objects, functions, and symbols in text position log `[PP-WARN] Invalid template child` and render nothing — that warning usually means JSX-shaped code was attempted.
- Follow HTML-first attribute naming. Native event attributes are lowercase DOM event attributes (`onclick`, `oninput`, `onsubmit`). Every other attribute on a component boundary is a prop: kebab-case names become camel-cased only inside `pp.props` (`selected-value` becomes `selectedValue`, `open-change` becomes `openChange`, and `on-open-change` becomes `onOpenChange`). A function prop does not need an `on-` prefix; that prefix is only an API naming convention. Use `on-click` when a component intentionally exposes an `onClick` prop, because lowercase `onclick` is reserved for the native DOM event on the boundary element.
- Pure bindings like `value="{count}"` are evaluated as expressions.
- For dynamic inline styles in authored templates, prefer `pp-style="{styleText}"` over `style="{styleText}"` so HTML/CSS tooling does not parse the brace expression as raw CSS.
- `pp-style` is an authoring alias. The compiler rewrites it to a native `style` attribute in rendered output.
- If the same element already has a static `style` attribute, `pp-style` is merged into that `style` value during compilation.
- Mixed text like `class="card {isActive ? 'active' : ''}"` is supported.
- Arrays in template expressions are joined without commas.
- `null`, `undefined`, and boolean expression results render as an empty string in text output.
- Plain objects, functions, and symbols are invalid template children; the runtime warns and omits them instead of rendering accidental strings such as `[object Object]`.
- Supported boolean attributes are normalized so truthy values emit the bare attribute and falsy values remove it.
- Boolean expressions bound to string-valued attributes such as `aria-pressed`, `aria-expanded`, and `data-*` serialize as the literal strings `"true"` and `"false"`, including on nested component boundaries.
- `<textarea value="{draft}"></textarea>` is normalized into textarea content.
- Use `pp-spread="{...attrs}"` to spread an object expression into attributes.
- `pp-spread` omits nullish values, omits known HTML boolean attributes when their value is `false`, emits them bare when `true`, preserves string-valued `aria-*`/`data-*` booleans, and escapes `&`, `"`, and `<` in emitted attribute values.
- Use plain `key` for keyed diffing. `pp-key` is not implemented.
- Only top-level declarations in the component script reach template scope, but every destructuring shape is supported there: `const [a, b] = anyTuple()`, `const [, only] = pair`, `const [first = 1, ...rest] = list`, `const [{ id }] = rows`, and the object-pattern forms. A binding declared inside a block or function is not exported.

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

### Value serialization is the JSX child contract

"PulsePoint Is Not JSX" is a rule about **structure**: an expression cannot produce elements. It is not a rule about **values**. An interpolation serializes its result exactly the way React serializes a JSX child, deliberately and with runtime tests behind it. Read this before binding a value whose type is not a string or a number — nearly every "my binding is blank" report is one of these rows working as designed.

**Text position.** `{expression}` between tags renders as:

| Result | Rendered text | Note |
| --- | --- | --- |
| `"text"`, `42`, `12n` | the value, HTML-escaped | |
| `true` / `false` | nothing | **both** booleans, not only `false` |
| `null` / `undefined` | nothing | |
| `""` | nothing | it is the empty string, not a blank |
| `0` | `0` | falsy, but printed |
| `NaN` | `NaN` | |
| an array | each item by these same rules, concatenated with no separator | `[1, null, 2]` renders `12` |
| an object, function, or symbol | nothing, plus `[PP-WARN] Invalid template child` | React throws here; PulsePoint omits and warns |

**The blank-binding trap.** The boolean and nullish rows cost the most time, because the value is correct and the output is empty. A flag that is `false`, or an absent optional serialized as `null`, reaches the browser intact and then renders as an empty element:

```html
<p>Admin: <span>{admin}</span></p>
<!-- admin === false -> renders nothing at all -->
```

Bind a *display expression*, never the raw value, whenever the value can be boolean or nullish:

```html
<p>Admin: <span>{admin ? 'yes' : 'no'}</span></p>
<p>Role: <span>{role ? role : 'none'}</span></p>
```

The same rule is why the React `&&` idiom leaks a zero. `{items.length && 'has items'}` renders `0` on an empty list, because `0` prints and only booleans and nullish values vanish. Write `{items.length > 0 ? 'has items' : ''}`.

There is no escape hatch from an interpolation to raw HTML. Text is always escaped, and an attribute value is escaped so it cannot close its quote or introduce an event handler. Trusted markup is the server's job.

**Attribute position.** `name="{expression}"` follows the same family of rules, split by the kind of attribute:

| Attribute kind | Result | Effect |
| --- | --- | --- |
| HTML boolean (`disabled`, `hidden`, `checked`, `readonly`, …) | `true` | attribute present, bare |
| | `false`, `null`, `undefined` | attribute absent |
| | falsy primitive (`""`, `0`, `NaN`) | attribute absent |
| | truthy primitive | present, carrying that value |
| anything else (`class`, `title`, `aria-*`, `data-*`) | `true` / `false` | the literal strings `"true"` / `"false"` |
| | `null` / `undefined` | **attribute present with an empty value** |
| | `0`, `NaN` | `"0"`, `"NaN"` |

Two of those rows differ from React.

A boolean on a non-boolean attribute is serialized rather than dropped, because `aria-*` and `data-*` flags are string-valued in the DOM: an author writing `aria-expanded="{open}"` means `"true"`/`"false"`, not presence. This is intended.

A nullish value on a non-boolean attribute leaves the attribute **present and empty** — `title="{maybeTitle}"` emits `title=""` where React would omit the attribute entirely. On `class` or `title` this is harmless. On a URL attribute it is not: an empty `src` re-requests the current document and an empty `href` links to it. Guard the expression whenever the attribute is a URL:

```html
<img src="{avatar ? avatar : placeholderUrl}" alt="" />
```

The same binding on a **component boundary** does remove the attribute when the value is nullish, so a nullish `src` behaves differently on a plain `<img>` than on a component that forwards it. Do not depend on either shape; write the guard.

**`pp-for` collections.** A non-iterable collection renders no rows and logs a `[PP-WARN]`. `null` and `undefined` render no rows silently. Binding an object where an array was intended is therefore a blank list rather than an error.

### Component markup is deferred inside an inert `<template>`

The render pipeline wraps each top-level `pp-component` root in an inert `<template pp-component="…">`, and the runtime materializes it back into live DOM on `mount()` (and on every SPA navigation) before it scans for roots. The browser never parses, validates, or fetches the contents of a `<template>`, so raw `{...}` placeholders never reach live DOM at first paint.

- Because of this, `{...}` is safe in **any** attribute or position — including slots the browser would otherwise validate eagerly: SVG geometry (`d`, `viewBox`, `points`, `transform`), URL attributes (`src`, `srcset`, `href`, `poster`), form `value` on `date`/`number`/`color` inputs, and text placed directly inside `<table>` or `<select>`.
- Before materialization, PulsePoint captures and empties each plain component script so inserting the fragment cannot trigger native browser execution; scripts introduced by later morphs receive the same treatment.
- Do not add per-tag workarounds to dodge first-paint validation (static-path `hidden` toggles instead of binding `d`, `data-*` URL holders, `hidden`-gated `<img src>`, or an SSR-resolved initial value). The deferral removes the whole class at once.
- `pp-style` and the `<input>`/`<select>`/`checked`/`defaultvalue`/`<textarea>` value rewrites still apply — but for different reasons that deferral does not replace: authoring-source tooling (`style="{...}"` breaks HTML/CSS linters) and attribute-vs-property correctness for controlled form fields.

## Refs

- Refs are for imperative element access. Do not use `pp-ref` as the default way to read normal form input values on submit; prefer the form's `onsubmit` event plus `Object.fromEntries(new FormData(event.currentTarget).entries())`.
- `pp-ref` works on native elements and on `x-*` component tags. On a component tag it resolves in the parent's scope and binds to the component's concrete root DOM element, so `<x-input pp-ref="{nameInput}" />` exposes the rendered `<input>` without a wrapper or `querySelector(...)`.
- Use the bare-name form, such as `pp-ref="nameInput"`, when the ref object or callback is already available under that name in scope. The runtime resolves this form by scope lookup.
- Use the brace-expression form, such as `pp-ref="{nameInput}"`, `pp-ref="{registerRef(id)}"`, or `pp-ref="{el => setNode(el)}"`, when the compiler should capture an expression. This is the preferred general form for component tags and dynamic or callback expressions.
- `pp.ref(null)` is the normal way to create a ref object.
- Callback refs and `{ current }` refs are both supported.
- Captured brace-form refs are compiled into an internal `data-pp-ref` token and rebound after render.
- Component refs are attached before layout effects and passive effects run, and detach with `null` (or a callback's returned synchronous cleanup) when the owning element unmounts.
- A composition component whose root is another `x-*` component forwards the ref through Caspian's layout-neutral component hosts to the eventual concrete DOM root.
- Plain `pp-ref` bindings are preserved across rerenders, including no-op rerenders that skip DOM diffing.
- Ref callbacks may be called with `null` during cleanup.
- The runtime generates `data-pp-ref` internally. Do not author it.
- Do not author `pp-event-owner`, `pp-owner`, or the other runtime-managed names listed above.

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

Component example:

```html
<div>
  <x-input pp-ref="{nameInput}" name="name" />
  <button onclick="nameInput.current?.focus()">Focus</button>

  <script>
    const nameInput = pp.ref(null);
  </script>
</div>
```

## Conditional rendering

PulsePoint has **no conditional directive**. There is no `pp-if`, no `pp-show`, no `pp-else`, and no JSX-style `{cond && (<div/>)}`. Conditional UI is expressed three ways, in this order of preference.

**1. `hidden="{...}"` — the default.** Bind the native boolean attribute. The element stays in the DOM and keeps its state; only its visibility changes.

```html
<div hidden="{!message}" class="notice">{message}</div>

<div hidden="{!showForm}">
  <form onsubmit="{submitForm(event)}"> … </form>
</div>
```

Notes:

- `hidden` is in the runtime's boolean-attribute set, so a truthy value emits the bare attribute and a falsy value removes it.
- Under Tailwind's preflight, `[hidden]:where(:not([hidden="until-found"]))` is `display: none !important`, so `hidden` reliably beats a `flex`/`grid`/`block` utility on the same element. **Without Tailwind preflight**, `hidden` is only a UA-stylesheet `display: none` and any `display` rule overrides it — in that case hide with a bound class instead.
- Because the subtree still exists, guard expressions inside it: use `{deleteTarget?.name}`, not `{deleteTarget.name}`. A hidden element is still rendered, so a throw there still breaks the component.

**2. A ternary inside an interpolation — for text, classes, and attributes.**

```html
<h3>{editingUser ? 'Edit User' : 'Create User'}</h3>
<button disabled="{isSaving}">{isSaving ? 'Saving…' : 'Save'}</button>
<span class="{'badge ' + (user.role === 'admin' ? 'badge-admin' : 'badge-user')}">{user.role}</span>
```

Template literals work too, as long as the attribute is quoted — the compiler tracks nested braces and quotes: ``class="{`badge ${isAdmin ? 'badge-admin' : ''}`}"``. String concatenation is easier to read and avoids nesting mistakes.

**3. `pp-for` over a 0-or-1 length collection — when the node must truly leave the DOM.** Use this when a hidden subtree would be wrong (an expensive child component, a duplicate form field name, a video that must stop).

```html
<template pp-for="item in selected ? [selected] : []">
  <x-detail-panel key="{item.id}" item="{item}" />
</template>
```

For an empty-state row, bind `hidden` on the placeholder rather than branching:

```html
<tbody>
  <tr hidden="{users.length !== 0}">
    <td colspan="5">No users found.</td>
  </tr>
  <template pp-for="user in users">
    <tr key="{user.id}">…</tr>
  </template>
</tbody>
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
- **A handler in slot content is evaluated in the scope of the template that authored the markup**, not the component the element ends up inside — so the function it calls must be declared by that template's owned script. This is the cause of `ReferenceError: <fn> is not defined` from a handler that clearly fired, including inside a portaled dropdown or dialog. See [A handler in slot content runs in the authoring template's scope](#a-handler-in-slot-content-runs-in-the-authoring-templates-scope).
- Do not replace normal PulsePoint event attributes with id-driven `querySelector(...)` plus `addEventListener(...)` wiring. If an imperative listener is unavoidable for an integration, attach and clean it up from `pp.effect(...)`.
- Do not replace a normal `onclick` with a `data-action`, `data-target`, or similar attribute plus a script that scans the DOM. PulsePoint event attributes are the event contract for first-party Caspian UI.

## SPA, loading, and navigation helpers

- `pp.mount()` enables client-side navigation interception automatically; no `<body pp-spa="true">` opt-in is required.
- `a[pp-spa="false"]` disables interception for that link.
- External links, downloads, `_blank`, and modifier-key clicks bypass SPA interception.
- **Navigation loading UI is optional; when it is wanted, author it as `src/app/**/loading.py`, not as a hand-rolled spinner.** Routes with no loader navigate with a plain fade, which is the common case. See [file-conventions.md](./file-conventions.md#loadingpy) for the full contract. The three directives below are how that file is wired up; only `pp-loading-content` is authored by hand.
- `pp-loading-content="true"` marks the page region that gets swapped or faded during navigation. Put it on the content pane of the layout that owns the subtree; with no such element the runtime falls back to `document.body` and the whole shell flashes. It applies with or without a `loading.py`.
- `pp-loading-url` is emitted by the server for each `loading.py`, carrying that file's URL scope. The runtime matches the destination pathname against it and walks up toward `/` for the closest ancestor loader. Never author it.
- `pp-loading-transition` accepts JSON with `fadeIn` and `fadeOut` timing values (number of ms, or a string with `ms`/`s`/`m`); it goes on an element inside the `loading.py` markup. Invalid JSON warns and falls back to 250 ms each way.
- The collected loader markup is injected as raw HTML and never mounted, so `<x-*>` tags and `{ }` bindings inside a `loading.py` do not resolve. Keep loaders to plain elements and CSS.
- An in-page wait — an `@rpc()` call, a submit, a filter, an upload — is component state, not navigation loading. Do not route it through these directives.
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
- The complete options object supports `abortPrevious`, `url`, `csrfUrl`, `credentials`, `onStream`, `onStreamError`, `onStreamComplete`, `onUploadProgress`, and `onUploadComplete`.
- `url` overrides the current-route RPC URL. `csrfUrl` overrides the URL used for the preliminary CSRF-cookie bootstrap GET. `credentials` is a standard Fetch `RequestCredentials` value; same-origin calls default to `"same-origin"` and cross-origin overrides default to `"include"`.
- When `abortPrevious` is enabled, a later aborting call cancels the active request and the cancelled promise resolves to `{ cancelled: true }`. A streamed response remains the active request until its final chunk, so it can still be cancelled after the response headers and early chunks have arrived.
- File uploads switch to the XHR path when upload progress callbacks are needed.
- A payload containing a `File` or non-empty `FileList` becomes multipart. The runtime writes every non-file field before the first file so a streaming server upload handler can read companion arguments even when the caller's object listed a file first. Objects are JSON-stringified, nullish fields are omitted, and a `FileList` becomes repeated parts under its original field name.
- The `onUploadProgress` callback receives `{ loaded, total, percent }`. `percent` is a 0-100 number, and both `total` and `percent` are `null` when the browser cannot compute the upload length. Do not document or generate `event.percentage`.
- `onUploadComplete` belongs to the same XHR progress path and runs only after a successful upload response (and any accepted redirect).
- For file managers, use upload callbacks for progress UI but replace the asset list from returned RPC state with `pp.state(...)` and `pp-for` instead of manual DOM repainting. See [file-uploads.md](./file-uploads.md).
- Streamed `text/event-stream` responses are supported when a stream handler is provided. Each `onStream` chunk is JSON-parsed when the event data looks like JSON; otherwise the raw string is passed through. If the response streams but no `onStream` handler was given, the runtime warns and discards the stream.
- Same-origin `X-PP-Redirect` headers (and redirect-status `Location` headers) are honored through `pp.redirect()`. Invalid or cross-origin server redirect targets are ignored.

Notes:

- After the runtime mounts, `pp.redirect()` uses SPA navigation for same-origin URLs. Cross-origin targets use normal browser navigation.
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
- plain owned component `<script>` elements during deferred bootstrap; the runtime removes them from the rendered component template after capturing their source.
- internal attributes such as captured ref tokens and event-owner bookkeeping.

These are runtime details.

- Mention them in docs when explaining how the browser runtime works.
- Do not use them as the default form in authored examples.
- Do not tell AI to handwrite them in route, layout, or component templates unless the task is explicitly about raw runtime HTML or runtime internals.

## AI rules

Use these rules when generating or editing PulsePoint runtime code:

- **Write plain HTML, never JSX.** Before finishing any template, verify it would still be valid HTML with every `{}` deleted, that every attribute brace expression is quoted, that conditionals use `hidden="{...}"`, and that lists use `<template pp-for="…">` with `key`. This check catches the highest-frequency generation failure and costs one pass.
- Treat PulsePoint as the default reactive frontend for Caspian app code.
- For first-party HTML interactions, use PulsePoint `on*` event attributes, state, refs, effects, directives, and `pp.rpc()` before reaching for DOM APIs.
- For simple forms, bind `onsubmit` in the authored HTML and read named fields with `Object.fromEntries(new FormData(event.currentTarget).entries())`; do not generate per-input refs or effect-managed submit listeners just to gather values.
- Treat `pp.rpc()` as the default browser-to-server path for CRUD operations and interactive backend reads.
- Use `public/js/pp-reactive-v2.min.js` as the shipped runtime contract AI should follow.
- Keep `main.py` in view because it injects runtime-facing component attributes and defers roots before the browser sees live component DOM.
- If a development-only source tree exists behind the shipped runtime, treat it as optional implementation detail rather than something generated apps are guaranteed to contain.
- In authored Caspian templates, do not handwrite `pp-component`; let the render pipeline inject it, and keep owned logic in a plain `<script>`.
- For grouped subtrees, follow the section layout pattern in [routing.md](./routing.md), keep the shared interactive shell in the parent folder's `layout.py`, and keep route-specific PulsePoint code in each child route's page template.
- For grouped shells with independent shell and content scrolling, put `pp-reset-scroll="true"` on the content pane rather than the whole shell when only the page content should reset between child-route navigations.
- Prefer PulsePoint state and template directives over manual DOM mutation for reactive updates.
- Treat `pp.state(...)` as a request to render. Keep timers, request generations, undisplayed pagination cursors, and transient RPC-only query text in `pp.ref(...)`; do not move a value to a ref when markup or render dependencies must react to it.
- A debounce limits frequency but does not reduce the cost of the update it eventually runs. For server search, debounce the RPC, discard stale responses, and keep only accepted results in state instead of waking a page-sized owner through an undisplayed query state.
- Keep high-frequency input state in the smallest useful component boundary. Use `pp.deferredValue(...)` only when an expensive consumer may safely lag one commit, and remember that `pp.transition()` is not concurrent scheduling.
- Before changing runtime reconciliation, use `pp.enablePerf()` to decide whether the app requested a redundant broad render or stable byte-identical output still incurred disproportionate runtime work.
- Avoid generating ids, `data-*` state, `querySelector`, `getElementById`, `addEventListener`, manual `innerHTML`, or custom event buses for normal Caspian UI behavior.
- Avoid data-attribute click wiring such as `data-action="save"` plus a delegated listener. Use `onclick="save()"` or `onsubmit="{save(event)}"` in authored HTML and keep reactive state in `pp.state(...)`.
- If you are explicitly editing raw runtime HTML or internals, keep `pp-component` unique per live instance.
- In authored templates, use a plain, untyped `<script>` inside the root. PulsePoint safely captures it during deferred bootstrap and morph insertion.
- Keep template-facing variables at top level.
- Follow the HTML-first context pattern: `pp.createContext(...)`, a lowercase provider tag such as `<themecontext.provider value="{...}">`, and `pp.context(token)`.
- `pp.provideContext` and `pp-context` do not exist. Do not invent them or other context helpers.
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

JSX constructs in markup — these are the highest-frequency generation failure, so check for them explicitly before finishing a template:

- `{cond && (<element/>)}` or `{cond ? <A/> : <B/>}` element-returning branches — use `hidden="{...}"`
- `{list.map(item => (<element/>))}` — use `<template pp-for="item in list">`
- unquoted brace attributes: `class={…}`, `selected={…}`, `value={…}`, `disabled={…}` — always quote: `class="{…}"`
- `className`, `htmlFor`, `onClick`/`onChange`/`onSubmit` camelCase event props, camelCase `defaultValue`/`defaultChecked` (the lowercase HTML attributes are real — see the directive table), `dangerouslySetInnerHTML`
- `style={{ color: "red" }}` object literals — `pp-style` takes a CSS **string**
- `<>…</>` fragments — there is no fragment tag; a page or layout may simply have sibling top-level nodes, and a component may not
- `key` written as a JSX brace prop outside a quoted attribute
- imported/returned components as JavaScript values — components are server-side `x-*` tags

Also avoid:

- React, Vue, Svelte, Alpine, HTMX, or JSX-first patterns as the default Caspian frontend approach
- standard DOM scripting as the default first-party interaction model, including id/data-attribute driven `querySelector(...)`, `addEventListener(...)`, or manual `innerHTML` rendering for normal buttons, forms, filters, toggles, uploads, and reactive lists
- `pp-context`
- `pp-key`
- any runtime-managed name listed in [Runtime-managed — never author these](#runtime-managed--never-author-these)
- handwritten `pp-component="..."` in authored route, layout, or component templates
- `pp.fetchFunction()` as the current raw runtime helper name
- made-up hooks, directives, or globals not present in the current bundled runtime

## Review notes

These are current runtime caveats that matter for authors and AI tools:

- `pp-component` is the registry key for instances, state, parent tracking, and templates. Treat it as unique per mounted root.
- `public/js/pp-reactive-v2.min.js` is the runtime surface that ships and should be assumed to exist.
- Caspian injects `pp-component` and preserves owned scripts as plain elements inside deferred roots.
- PulsePoint captures and empties plain scripts before a deferred root or later morph becomes live, preventing native browser execution while preserving component-scope evaluation.
- Nested roots without their own plain owned `<script>` block are not fully isolated during parent template compilation.
- The global `pp` singleton auto-mounts on DOM ready, and `pp.mount()` is idempotent.
- `pp.effect` and `pp.layoutEffect` are cleanup-style hooks. Their callbacks are not promise-aware.
- Callback refs may return synchronous cleanup functions, which run instead of a later `callback(null)` detach call.
- `pp.context()` resolves through ancestor components, not the current component's own pending providers.
- `pp.provideContext` is not part of the current runtime API. Use an HTML-first lowercase provider tag such as `<themecontext.provider>`.
- `pp.portal()` preserves logical ancestry through the registry, so context and prop refresh behavior continue to work through portaled descendants.
- Top-level array destructuring is exported to template scope for any initializer, not only `pp.state` and `pp.reducer`. A binding declared inside a block or function is still not exported.
- `pp.transition()` does not deprioritize work. PulsePoint renders synchronously; the hook provides an accurate `isPending` flag, not a concurrent scheduler.
- `pp.state(...)` changes schedule a component render; `pp.ref(...)` mutations do not. This is an ownership contract, not merely a syntax choice.
- Debouncing a state setter does not make the resulting render cheaper. A large owner should not hold RPC-only query text, timer handles, request generations, or undisplayed pagination cursors in state.
- `pp.syncExternalStore(...)` resubscribes whenever `subscribe` changes identity, so pass a `pp.callback(..., [])`-wrapped function.
- `pp.optimistic(...)` clears its pending actions on the render where `passthrough` changes, using a render-phase update rather than an effect, so there is no frame showing stale optimistic output.
- `pp.errorBoundary()` catches render, effect, and effect-cleanup throws, including the boundary component's own. It does not catch event-handler errors. It latches until `reset()` and stops catching after five captures without a reset.
- `pp.imperativeHandle(...)` needs the parent's ref passed down as an ordinary prop. `pp-ref` on an `x-*` tag is a different feature that binds the child's root DOM node, and `pp-ref-forward` is runtime-managed and must not be authored.

## Final reminder

If a feature is not implemented in the current bundled runtime, do not invent it.

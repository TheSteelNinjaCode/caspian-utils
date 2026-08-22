---
title: File Conventions
description: Use this page when deciding what belongs in `index.py`, `layout.py`, `loading.py`, `not_found.py`, or `error.py` in a Caspian app.
related:
  title: Related docs
  description: Use the routing guide for URL and subtree behavior, the structure guide for placement, the metadata guide for SEO fields, the cache guide for route-level caching, and the runtime maps when the task crosses into framework internals.
  links:
    - /docs/routing
    - /docs/project-structure
    - /docs/metadata
    - /docs/cache
    - /docs/pulsepoint
    - /docs/core-runtime-map
    - /docs/pulsepoint-runtime-map
    - /docs/index
---

This page is the quick decision guide for the special file conventions under `src/app`.

Use it when a task names `index.py`, `layout.py`, `loading.py`, `not_found.py`, or `error.py`, or when the question is where a page, route logic, shared shell, navigation loading UI, 404 page, or 500 page should live.

**Each of these is optional, and each is a shipped convention with a runtime behind it.** Add one only when the app actually wants that behavior — an app with no `loading.py` anywhere is perfectly normal. What the convention settles is *how*: when a task does call for a subtree shell, a navigation loading state, a 404, or a 500 page, add the named file rather than hand-rolling an equivalent inside a route.

**Authoring is Python-only, special files included.** A route is one `index.py` whose `page()` returns markup through `html(...)`; a subtree shell is one `layout.py` whose `layout()` returns its template. The special files follow the same rule: `loading.py` exports `loading()`, and `not_found.py` / `error.py` export `page()`. The runtime indexes these `.py` names under `src/app`.

Treat `caspian.config.json` and the actual project tree as the source of truth for which features exist in the current workspace. For runtime ownership, verify these rules against:

- `main.py` for route registration and the global `not_found.py` / `error.py` handling
- `.venv/Lib/site-packages/casp/component_decorator.py` for `html(...)`
- `.venv/Lib/site-packages/casp/layout.py` for `layout()` resolution and nested layout behavior
- `.venv/Lib/site-packages/casp/loading.py` plus `public/js/pp-reactive-v2.min.js` for `loading.py` collection and SPA loading behavior
- `.venv/Lib/site-packages/casp/caspian_config.py` for how `index.py`, `layout.py`, and `loading.py` are indexed under `src/app`

## Quick Map

| File | Purpose | Add it when | Verify against |
| --- | --- | --- | --- |
| `index.py` | The route: `page()` returns the page markup via `html(...)`, plus metadata, auth checks, redirects, caching, and route-owned `@rpc()` actions | The route exists | `main.py`, `.venv/Lib/site-packages/casp/layout.py` |
| `layout.py` | Shared shell for a route subtree: `layout()` returns the wrapper template (and optionally props/metadata) | Multiple child routes share wrapper markup, props, or metadata defaults | `.venv/Lib/site-packages/casp/layout.py`, `routing.md`, `metadata.md` |
| `loading.py` | Route-scope loading UI used during SPA navigation, via `loading()` | Optional. Only when a subtree is meant to show a loading state while the next route renders — and then **always this over a hand-rolled navigation spinner** | `.venv/Lib/site-packages/casp/loading.py`, `public/js/pp-reactive-v2.min.js` |
| `not_found.py` | Global 404 page, via `page()` | The app needs a branded fallback for unmatched URLs | `main.py` |
| `error.py` | Global 500 page, via `page(error_message, error_trace)` | The app needs a safe fallback for unhandled exceptions | `main.py` |

## Authored Markup Rule

For route markup, layout templates, and the special files, keep one top-level parent HTML element or one imported `x-*` component root.

- Keep any owned plain `<script>` inside that root, not after it.
- Do not handwrite `pp-component`; keep owned component logic in a plain `<script>` inside the single root.
- Components used as `<x-*>` tags resolve from the Python imports at the top of the owning module.

It is the recommended shape everywhere, and the only shape that can receive props. A **component** with sibling top-level nodes becomes a fragment, which has no root for props to land on. A **route or layout** with sibling top-level nodes is wrapped in a layout-neutral boundary host, so a page whose sections are genuinely siblings needs no wrapper `<div>`. See [pulsepoint.md](./pulsepoint.md) "Multi-root pages and layouts".

The root may be a native element or an imported `x-*` tag, in both cases with the owned `<script>` kept inside it. When the root is an `x-*` tag, the script travels as slot content wrapped in `<template pp-owner="…">`, and the runtime resolves that owner back to the authoring page or layout, which claims and executes the script in its own scope. No wrapper element is needed around a composition root. See [pulsepoint.md](./pulsepoint.md) "A slot-authored `<script>` belongs to the template that authored it".

## `index.py`

`index.py` is the route. Its `page()` function returns the page markup, and the same module owns everything else the route needs:

- `page()` returning `html(r"""...""", **context)`
- metadata
- auth checks or redirects
- route-level `Cache(...)`
- route-owned `@rpc()` actions
- server-side context computed before rendering

Keep logic here when it belongs only to this route. That includes route-specific first-render queries, `@rpc()` actions, auth checks, redirects, filters, validation, upload orchestration, and response shaping. Move logic to `src/lib/**` only when it is shared across routes, components, integrations, or features. Do not extract one-route logic into a library just because it is written in Python.

When `caspian.config.json` has `prisma: true`, route-specific database access in `index.py` should use the generated Prisma Python ORM from `src/lib/prisma/**`. Shared database helpers may live in `src/lib/**`, but they should still call the generated Prisma Python ORM rather than a separate fetch or driver layer.

Example:

```python
from casp.component_decorator import html
from casp.layout import Metadata
from src.components.Button import Button

metadata = Metadata(
    title="Dashboard | Caspian",
    description="Overview page for the dashboard.",
)


async def page(params: dict, request=None):
    return html(r"""
<section class="space-y-4 p-6">
  <h1 class="text-2xl font-semibold">Dashboard</h1>

  <x-button onclick="setFilter('open')">
    Show Open
  </x-button>

  <script>
    const [filter, setFilter] = pp.state("all");
  </script>
</section>
""")
```

Inside `html(...)`, `{{ ... }}` is server-side Jinja and `{ ... }` stays live for PulsePoint. Prefer a raw string (`r"""..."""`) whenever the markup contains a `<script>` with backslashes.

Current route-parameter behavior:

- path params arrive as a single `params` dict
- query params can be injected by name
- `request` is injected by keyword when declared

When one page needs to influence a wrapping layout, `page()` can return `(page_html, layout_props_dict)`.

## `layout.py`

`layout.py` is the shared wrapper for a route subtree. Its `layout()` function returns the wrapper template as a **raw string** — the runtime compiles it later with `children`, `{{ layout.* }}`, and `{{ metadata.* }}` in scope, so do not render it through `html(...)` yourself.

`layout()` may return:

- a template string — the shared shell markup with a `<slot />` insertion point
- `(template_string, props_dict)` — shell plus `{{ layout.* }}` props
- a props dict alone — the subtree gets a passthrough `<slot />` shell
- `None` — passthrough shell only

Any `<x-*>` tags in the returned template resolve from the Components imported at the top of `layout.py`, exactly like components resolve for a page.

Example:

```python
from casp.component_decorator import html
from casp.layout import Metadata
from src.components.DocsNav import DocsNav

metadata = Metadata(
    title="Docs Section | Caspian",
    description="Shared metadata for the docs subtree.",
)


def layout():
    return html(r"""
<div class="docs-shell">
  <aside class="docs-nav"><x-docs-nav /></aside>

  <main class="docs-content" pp-reset-scroll="true">
    <slot />
  </main>
</div>
"""), {
        "shell_class": "docs-shell",
    }
```

Use nested `layout.py` files for sections like `dashboard/`, `docs/`, `account/`, or route groups such as `(marketing)/`.

The child outlet must be a real authored `<slot />` element in the returned template. The runtime parses the layout markup and replaces real slot elements during nested rendering; it does not invent an implicit slot or replace escaped documentation text such as `&lt;slot /&gt;`.

In grouped shells with separate shell and content scrolling, put `pp-reset-scroll="true"` on the content pane that should reset on child-route navigation. Leave persistent shell scrollers such as sidebars unmarked when they should keep their own scroll position.

Important runtime detail: `layout()` may be synchronous or async in the installed runtime. Keep async layout work focused on shared subtree props or metadata; put route-specific first-render data in `page()` and browser-triggered work in route-owned `@rpc()` actions.

## `loading.py`

`loading.py` is the shipped route-scope loading UI for SPA navigation.

**It is optional.** Most apps have few of these or none, and a route with no loader above it simply navigates with a plain fade — that is normal, not a gap to fill. Do not add loaders that were not asked for.

**When one is wanted, it already exists — do not build a parallel one.** Instead of a spinner component, a skeleton overlay, an `isLoading` state, or a `pp:navigation:start` listener *for navigation between routes*, add or edit the closest `loading.py`. A hand-rolled navigation loader re-implements a runtime that already picks the right loader by URL scope, swaps the right region, and times the fade.

Scope rule: this covers **route-to-route SPA navigation only**. An in-page wait — an `@rpc()` call, a form submit, a filter refetch, an upload — is ordinary component state (`pp.state` plus a bound `hidden` or class) and never belongs in `loading.py`.

### The file

One file per subtree, exporting a function named `loading`:

```python
from casp.component_decorator import html


def loading():
    return html(r"""
<div class="space-y-4 p-6">
  <div class="h-4 animate-pulse rounded bg-muted"></div>
  <div class="h-24 animate-pulse rounded bg-muted/70"></div>
</div>
""")
```

Requirements the runtime enforces:

- the function must be named `loading`. A `loading.py` without it raises `AttributeError` naming the file.
- it must be **synchronous** and take **no parameters**. `async def loading()` — or any awaitable return — raises `TypeError`. There is no `request`, no route params, no session, and no `page()`-style data loading here.
- it must return markup through `html(r"""...""")`, exactly like a page or component.

### Scope resolution

The runtime indexes `src/app/**/loading.py` separately from routes and layouts, deriving each file's URL scope from its folder. During navigation the browser runtime looks up the loader for the destination pathname and walks **up** the path toward `/` until one matches, so the closest ancestor loader wins:

| File | URL scope | Applies to |
| --- | --- | --- |
| `src/app/loading.py` | `/` | Every route with no closer loader |
| `src/app/dashboard/loading.py` | `/dashboard` | `/dashboard` and its descendants |
| `src/app/(marketing)/loading.py` | `/` | **Every route in the app, not just the group.** Group folders are stripped from the scope exactly as they are from the URL, so a loader directly inside `(marketing)/` collapses to the root scope and becomes the app-wide fallback. A loader in `(marketing)/pricing/` correctly scopes to `/pricing` |

A route with no loader at any level above it navigates with a plain fade and no loading markup. That is a valid choice, not a bug.

Two files can therefore claim the same scope — `src/app/loading.py` and `src/app/(group)/loading.py` both register `/`. The lookup takes the first match in document order, so give only one file a given scope rather than relying on that ordering.

**Dynamic segments do not match.** The lookup compares the scope string against the real pathname, so a loader in `src/app/users/[id]/` registers the literal scope `/users/[id]`, which no visited URL ever equals. Put the loader at the static parent (`src/app/users/loading.py`) instead.

### The markup is static HTML

The loader is collected once and injected as raw markup, so it is **not** a component and is never mounted:

- Jinja **does** run — `{{ ... }}` and the `html(...)` keyword context are interpolated when `loading()` is called.
- `<x-*>` component tags are **not** expanded. They survive into the document as literal unknown tags and render as nothing.
- PulsePoint `{ ... }` bindings are **not** compiled. They render as literal text.
- `<script>`, `pp-for`, `pp-ref`, and event attributes do nothing.

So keep a loader to plain elements and CSS classes — skeletons, pulses, bars. If a loading state needs an icon component or a binding, it is in-page state in a real component, not `loading.py`.

Two more consequences worth knowing: `loading()` is called once per file and cached until the file's mtime changes, so its output is shared by every request and must never contain per-user or per-request data; and **every** loader in the app is embedded in the hidden registry of **every** rendered page, so keep them small.

### Where the loader is painted

Mark the live container the loader replaces with `pp-loading-content="true"`, usually the content pane of the layout that owns the subtree:

```html
<main class="docs-content" pp-loading-content="true" pp-reset-scroll="true">
  <slot />
</main>
```

With no such container, the runtime falls back to `document.body`, which flashes the entire shell. Marking the pane is what keeps a sidebar, topbar, or rail on screen during navigation.

The attribute is also used without a loader: when no `loading.py` scope matches, the runtime still fades that same region out and back in.

### Fade timing

Put `pp-loading-transition` on an element **inside** the loader markup to override the 250 ms default in each direction:

```python
return html(r"""
<div pp-loading-transition='{"fadeIn": 120, "fadeOut": "1s"}'>
  <div class="h-1 animate-pulse bg-primary"></div>
</div>
""")
```

Values are milliseconds as a number, or a string with a `ms`, `s`, or `m` suffix. Invalid JSON logs a console warning and falls back to 250/250.

### Not a first-paint loader

This is navigation-only UI. It never appears on a full page load or a hard refresh — nothing has navigated yet — and it is inert in a static export, where there is no SPA navigation to trigger it. Links opted out with `pp-spa="false"`, external links, downloads, `_blank`, and modifier-key clicks all bypass it too.

## `not_found.py`

`src/app/not_found.py` is the global 404 page. It exports `page()` and returns markup through `html(...)`, exactly like a route, and may declare module-level `metadata`.

When `main.py` catches a `404`, it looks for this exact root-level file, renders it through the nested layout pipeline, and returns it with a 404 status.

Important scope rule:

- this is a global root-level file, not a per-folder `not_found.py` convention in the current runtime

The runtime passes `request` into the template context and sets default page metadata for the response.

Use this file for branded invalid-URL fallbacks instead of leaving the app on the generic FastAPI or Starlette message.

## `error.py`

`src/app/error.py` is the global 500 page for unhandled exceptions. Its `page()` receives the error values as parameters — `def page(error_message: str, error_trace: Optional[str] = None)` — rather than reading them from a template context.

When `main.py` catches a general exception, it looks for this exact root-level file, renders it through the nested layout pipeline, and returns it with a 500 status.

The runtime supplies `error_message` and `error_trace`, and makes `request` available to the render.

In production, `error_trace` is suppressed. Keep this page safe for users and avoid exposing sensitive internals outside development environments.

If rendering `error.py` fails, the runtime falls back to a minimal plain HTML 500 response.

## Decision Rule

Use this order when deciding where a concern belongs:

1. Every route is one `index.py` with `page()` returning `html(...)` markup.
2. Keep route-specific logic in that same `index.py`; move logic into `src/lib/**` only when it is actually shared.
3. Put shared subtree wrapper markup, props, and metadata defaults in `layout.py`.
4. Add `loading.py` only when a subtree is asked for a scoped SPA-navigation loading state — and then never a hand-rolled navigation spinner. No loader is the default. In-page waits stay in component state.
5. Use root `not_found.py` for unmatched URLs.
6. Use root `error.py` for unhandled exceptions.

Use [routing.md](./routing.md) for the broader file-based routing model, [metadata.md](./metadata.md) for page and layout metadata, [cache.md](./cache.md) for route caching in `index.py`, and [pulsepoint.md](./pulsepoint.md) for the authored template contract and browser runtime behavior.

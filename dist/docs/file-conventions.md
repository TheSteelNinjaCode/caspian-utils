---
title: File Conventions
description: Use this page when deciding what belongs in `index.py`, `layout.py`, `loading.html`, `not-found.html`, or `error.html` in a Caspian app.
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

Use it when a task names `index.py`, `layout.py`, `loading.html`, `not-found.html`, or `error.html`, or when the question is where a page, route logic, shared shell, loading UI, 404 page, or 500 page should live.

**Authoring is Python-only.** A route is one `index.py` whose `page()` returns markup through `html(...)`; a subtree shell is one `layout.py` whose `layout()` returns its template string. The only authored `.html` files are the special app-level templates: `loading.html`, `not-found.html`, and `error.html`.

Treat `caspian.config.json` and the actual project tree as the source of truth for which features exist in the current workspace. For runtime ownership, verify these rules against:

- `main.py` for route registration and the global `not-found.html` / `error.html` handling
- `.venv/Lib/site-packages/casp/component_decorator.py` for `html(...)`
- `.venv/Lib/site-packages/casp/layout.py` for `layout()` resolution and nested layout behavior
- `.venv/Lib/site-packages/casp/loading.py` plus `public/js/pp-reactive-v2.min.js` for `loading.html` collection and SPA loading behavior
- `.venv/Lib/site-packages/casp/caspian_config.py` for how `index.py`, `layout.py`, and `loading.html` are indexed under `src/app`

## Quick Map

| File | Purpose | Add it when | Verify against |
| --- | --- | --- | --- |
| `index.py` | The route: `page()` returns the page markup via `html(...)`, plus metadata, auth checks, redirects, caching, and route-owned `@rpc()` actions | The route exists | `main.py`, `.venv/Lib/site-packages/casp/layout.py` |
| `layout.py` | Shared shell for a route subtree: `layout()` returns the wrapper template (and optionally props/metadata) | Multiple child routes share wrapper markup, props, or metadata defaults | `.venv/Lib/site-packages/casp/layout.py`, `routing.md`, `metadata.md` |
| `loading.html` | Route-scope loading UI used during SPA navigation | A section or page needs an immediate loading state before the next route finishes rendering | `.venv/Lib/site-packages/casp/loading.py`, `public/js/pp-reactive-v2.min.js` |
| `not-found.html` | Global 404 page | The app needs a branded fallback for unmatched URLs | `main.py` |
| `error.html` | Global 500 page | The app needs a safe fallback for unhandled exceptions | `main.py` |

## Authored Markup Rule

For route markup, layout templates, and the special files, keep one top-level parent HTML element or one imported `x-*` component root.

- Keep any owned plain `<script>` inside that root, not after it.
- Do not handwrite `pp-component`; keep owned component logic in a plain `<script>` inside the single root.
- Components used as `<x-*>` tags resolve from the Python imports at the top of the owning module.

This is a runtime requirement for **components**, which fail render when multi-rooted. For a `.py` route or layout it is the recommended shape rather than a hard limit: sibling top-level nodes are wrapped in a layout-neutral boundary host instead of raising. See [pulsepoint.md](./pulsepoint.md) "Multi-root pages and layouts".

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

## `loading.html`

`loading.html` provides route-scope loading UI for SPA navigation.

The runtime indexes `src/app/**/loading.html` separately from routes and layouts. During navigation, the browser runtime looks for the closest matching loading file by URL scope and falls back up the path toward `/`.

Examples:

- `src/app/loading.html` can act as the root fallback loader
- `src/app/dashboard/loading.html` applies to `/dashboard` and its descendants unless a closer loading file exists
- `src/app/(marketing)/loading.html` applies to the grouped subtree even though `(marketing)` does not appear in the URL

To preserve the surrounding shell during navigation, put `pp-loading-content="true"` on the live container that should be swapped during loading, usually the main content pane in a layout.

Example layout shell:

```html
<main class="docs-content" pp-loading-content="true" pp-reset-scroll="true">
  <slot />
</main>
```

Example loader:

```html
<div class="space-y-4 p-6">
  <div class="h-4 rounded bg-muted animate-pulse"></div>
  <div class="h-24 rounded bg-muted/70 animate-pulse"></div>
</div>
```

If no `pp-loading-content="true"` container exists, the browser runtime falls back to `document.body`.

## `not-found.html`

`src/app/not-found.html` is the global 404 page.

When `main.py` catches a `404`, it looks for this exact root-level file, renders it through the nested layout pipeline, and returns it with a 404 status.

Important scope rule:

- this is a global root-level file, not a per-folder `not-found.html` convention in the current runtime

The runtime passes `request` into the template context and sets default page metadata for the response.

Use this file for branded invalid-URL fallbacks instead of leaving the app on the generic FastAPI or Starlette message.

## `error.html`

`src/app/error.html` is the global 500 page for unhandled exceptions.

When `main.py` catches a general exception, it looks for this exact root-level file, renders it through the nested layout pipeline, and returns it with a 500 status.

The runtime provides these template values:

- `request`
- `error_message`
- `error_trace`

In production, `error_trace` is suppressed. Keep this page safe for users and avoid exposing sensitive internals outside development environments.

If rendering `error.html` fails, the runtime falls back to a minimal plain HTML 500 response.

## Decision Rule

Use this order when deciding where a concern belongs:

1. Every route is one `index.py` with `page()` returning `html(...)` markup.
2. Keep route-specific logic in that same `index.py`; move logic into `src/lib/**` only when it is actually shared.
3. Put shared subtree wrapper markup, props, and metadata defaults in `layout.py`.
4. Add `loading.html` when SPA navigation needs an immediate scoped loading state.
5. Use root `not-found.html` for unmatched URLs.
6. Use root `error.html` for unhandled exceptions.

Use [routing.md](./routing.md) for the broader file-based routing model, [metadata.md](./metadata.md) for page and layout metadata, [cache.md](./cache.md) for route caching in `index.py`, and [pulsepoint.md](./pulsepoint.md) for the authored template contract and browser runtime behavior.

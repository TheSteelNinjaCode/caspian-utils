---
title: File Conventions
description: Use this page when deciding what belongs in `index.html`, `index.py`, `layout.html`, `layout.py`, `loading.html`, `not-found.html`, or `error.html` in a Caspian app.
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

Use it when a task names `index.html`, `index.py`, `layout.html`, `layout.py`, `loading.html`, `not-found.html`, or `error.html`, or when the question is where a page shell, route logic, loading UI, 404 page, or 500 page should live.

Treat `caspian.config.json` and the actual project tree as the source of truth for which features exist in the current workspace. For runtime ownership, verify these rules against:

- `main.py` for global `not-found.html` and `error.html` handling
- `.venv/Lib/site-packages/casp/layout.py` for `render_page(...)`, `render_layout(...)`, and nested layout behavior
- `.venv/Lib/site-packages/casp/loading.py` plus `public/js/pp-reactive-v2.js` for `loading.html` collection and SPA loading behavior
- `.venv/Lib/site-packages/casp/caspian_config.py` for how `index.*`, `layout.html`, and `loading.html` are indexed under `src/app`

## Quick Map

| File | Purpose | Add it when | Verify against |
| --- | --- | --- | --- |
| `index.html` | Authored visible page template for a route | The route renders UI | `src/app/**`, `routing.md`, `pulsepoint.md` |
| `index.py` | Backend companion for route logic and metadata | The route needs `page()`, metadata, auth checks, redirects, caching, or route-owned `@rpc()` actions | `main.py`, `.venv/Lib/site-packages/casp/layout.py` |
| `layout.html` | Shared shell for a route subtree | Multiple child routes share wrapper markup | `.venv/Lib/site-packages/casp/layout.py`, `routing.md` |
| `layout.py` | Shared synchronous props and metadata defaults for a subtree | The shared shell needs Python-provided values or metadata | `.venv/Lib/site-packages/casp/layout.py`, `metadata.md` |
| `loading.html` | Route-scope loading UI used during SPA navigation | A section or page needs an immediate loading state before the next route finishes rendering | `.venv/Lib/site-packages/casp/loading.py`, `public/js/pp-reactive-v2.js` |
| `not-found.html` | Global 404 page | The app needs a branded fallback for unmatched URLs | `main.py` |
| `error.html` | Global 500 page | The app needs a safe fallback for unhandled exceptions | `main.py` |

## Authored HTML Rule

For authored route, layout, loading, not-found, and error HTML files, keep exactly one top-level parent HTML element or one imported `x-*` component root.

- Keep any `<!-- @import ... -->` directives above that root.
- Keep any owned plain `<script>` inside that root, not after it.
- Do not handwrite `pp-component` or `type="text/pp"`.

This is a runtime requirement, not a style preference.

## `index.html`

`index.html` is the authored page template for a route.

Use it for:

- visible page markup
- HTML-first `x-*` component usage
- PulsePoint state, refs, effects, and directives that belong to that route
- route-local plain `<script>` blocks that stay inside the same root

For any route that renders UI, keep the visible markup here even when the route also has an `index.py` companion.

Example:

```html
<!-- @import Button from "../../components/Button.py" -->

<section class="space-y-4 p-6">
  <h1 class="text-2xl font-semibold">Dashboard</h1>

  <x-button onclick="setFilter('open')">
    Show Open
  </x-button>

  <script>
    const [filter, setFilter] = pp.state("all");
  </script>
</section>
```

Use `index.html` by itself when the route is UI-only.

## `index.py`

`index.py` is the backend companion for a route.

Add it when the route needs:

- `page()`
- metadata
- auth checks or redirects
- route-level `Cache(...)`
- route-owned `@rpc()` actions
- server-side context before rendering the sibling template

For UI routes, `index.py` does not replace `index.html`. It prepares data and calls `render_page(__file__, ...)` so Caspian renders the sibling template.

Example:

```python
from casp.layout import Metadata, render_page

metadata = Metadata(
    title="Dashboard | Caspian",
    description="Overview page for the dashboard.",
)


async def page(params: dict, request=None):
    return render_page(__file__, {
        "slug": params.get("slug"),
        "has_request": request is not None,
    })
```

Current route-parameter behavior:

- path params arrive as a single `params` dict
- query params can be injected by name
- `request` is injected by keyword when declared

When one page needs to influence a wrapping layout, `page()` can return `(page_html, layout_props_dict)`.

## `layout.html`

`layout.html` is the shared wrapper for a route subtree.

Use it for:

- shared shell markup such as sidebars, headers, docs rails, or dashboard frames
- the `[[children|safe]]` insertion point for child routes
- shared layout props consumed as `[[ layout.* ]]`
- shared metadata fields consumed as `[[ metadata.* ]]`

Example:

```html
<div class="docs-shell">
  <aside class="docs-nav">Docs navigation</aside>

  <main class="docs-content" pp-reset-scroll="true">
    [[children|safe]]
  </main>
</div>
```

Use nested `layout.html` files for sections like `dashboard/`, `docs/`, `account/`, or route groups such as `(marketing)/`.

In grouped shells with separate shell and content scrolling, put `pp-reset-scroll="true"` on the content pane that should reset on child-route navigation. Leave persistent shell scrollers such as sidebars unmarked when they should keep their own scroll position.

## `layout.py`

`layout.py` is the Python companion for `layout.html`.

Use it for:

- shared synchronous props
- metadata defaults for everything below that folder
- small shared layout decisions that belong to the subtree rather than one page

Return `render_layout(__file__), props` so the sibling `layout.html` stays the authored wrapper.

Example:

```python
from casp.layout import Metadata, render_layout

metadata = Metadata(
    title="Docs Section | Caspian",
    description="Shared metadata for the docs subtree.",
)


def layout():
    return render_layout(__file__), {
        "shell_class": "docs-shell",
        "content_class": "docs-shell__content",
    }
```

Important runtime detail: `layout()` is synchronous in the installed runtime. Put async I/O in route `page()` functions or route-owned `@rpc()` actions instead of awaiting inside `layout.py`.

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
  [[children|safe]]
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

1. Put visible route markup in `index.html`.
2. Add `index.py` only when the same route needs backend work.
3. Put shared subtree wrapper markup in `layout.html`.
4. Add `layout.py` only when that shared shell needs synchronous Python props or metadata.
5. Add `loading.html` when SPA navigation needs an immediate scoped loading state.
6. Use root `not-found.html` for unmatched URLs.
7. Use root `error.html` for unhandled exceptions.

Use [routing.md](./routing.md) for the broader file-based routing model, [metadata.md](./metadata.md) for page and layout metadata, [cache.md](./cache.md) for route caching in `index.py`, and [pulsepoint.md](./pulsepoint.md) for the authored template contract and browser runtime behavior.

---
title: Routing
description: Use this page when the task mentions `src/app`, `index.py`, `index.html`, `layout.py`, dynamic routes, route groups, nested layouts, or file-based routing in Caspian.
related:
  title: Related docs
  description: Read the structure guide first, then use the components guide for reusable UI, the metadata guide for SEO fields, the cache guide for route-level HTML reuse, and the PulsePoint runtime guide for interactive route templates.
  links:
    - /docs/project-structure
    - /docs/components
    - /docs/cache
    - /docs/file-uploads
    - /docs/metadata
    - /docs/pulsepoint
    - /docs/index
---

Caspian follows the same mental model as the Next.js App Router: routes live under `src/app`, folders define URL segments, layouts nest automatically, and special folder names control grouping and dynamic matching.

The main difference is the file types. Instead of `page.tsx` and `layout.tsx`, Caspian uses `index.html`, `index.py`, `layout.html`, and optional Python companions for server-side logic.

## Overview

Caspian uses a high-performance file-system router built on top of FastAPI. Your directory structure becomes your URL structure.

Start with these rules:

- Put application routes in `src/app/`.
- For any route that renders a page, put the markup in `index.html`.
- If a route is UI-only, `index.html` by itself is enough.
- Add `index.py` only when the same route needs metadata, `page()`, `@rpc()` actions, auth checks, caching, redirects, or other server-side logic.
- Use a standalone `index.py` only for non-visual routes such as redirects or action-only handlers.
- Use `layout.html` to wrap child routes.
- Use `layout.py` when a layout needs shared synchronous props or metadata before rendering.

Use [cache.md](./cache.md) when an `index.py` route also needs declarative page caching via `Cache(...)`.

Framework internals note:

- Caspian's layout and route-resolution internals live in `.venv/Lib/site-packages/casp/layout.py`.
- Treat that file as framework code. Read it when the task is about routing internals, layout resolution, or metadata behavior inside Caspian itself.

See [metadata.md](./metadata.md) when a page or layout needs SEO fields.

## Next.js App Router Mapping

If you already know the Next.js App Router, use this translation layer:

| Next.js concept | Caspian equivalent |
| --- | --- |
| `app/` | `src/app/` |
| `page.tsx` | `index.html` plus optional `index.py` companion |
| `layout.tsx` | `layout.html` and optional `layout.py` |
| `[id]` | `[id]` |
| `[...slug]` | `[...slug]` |
| `(group)` | `(group)` |

This means most App Router habits carry over directly:

- Keep route logic close to the route folder.
- Model the URL with folders instead of a central route table.
- Use nested layouts for shared wrappers.
- Use route groups to organize code without changing the public path.

## Core Concepts

Every route lives inside `src/app`. For routes that render UI, `index.html` owns the markup and `index.py` is only an optional companion for server logic or metadata.

Use this decision rule when creating routes:

| Route shape | Files |
| --- | --- |
| UI only | `index.html` |
| UI plus backend logic or metadata | `index.html` and `index.py` |
| No rendered page, backend only | `index.py` |

Examples:

| Files | URL |
| --- | --- |
| `src/app/index.html` | `/` |
| `src/app/about/index.html` | `/about` |
| `src/app/dashboard/index.html` and `src/app/dashboard/index.py` | `/dashboard` |
| `src/app/blog/posts/index.html` | `/blog/posts` |
| `src/app/(auth)/signout/index.py` | `/signout` |

### `index.html`

Use `index.html` for the route template. This is the route's view layer.

If a route renders visible page content, that content belongs here even when the route also has an `index.py` companion.

Route templates can import reusable Python components with `<!-- @import ... -->` comments and render them with HTML-first `x-*` tags such as `<x-button />`. Use [components.md](./components.md) for the component authoring rules.

Place those import comments at the top of the file, above the authored root element. They are file-level directives, not children of the route root.

Do not manually add `pp-component="..."` to the route root. The Python render pipeline injects that attribute onto the route's single top-level lowercase HTML element.

Do not manually add `type="text/pp"` to a route-owned script either. In source templates, write a plain `<script>` inside the root and let `main.py` call `transform_scripts(...)` before the browser runtime sees the HTML.

That means route templates follow the same single-root discipline as component templates: one root element, no sibling roots, and PulsePoint scripts kept inside that root when needed.

For AI-generated route templates, treat `src/app/**/index.html` the same way you would a React component body: return one parent element that contains the entire route markup.

Good:

```html
<section class="dashboard-shell">
  <h1>Dashboard</h1>

  <script>
    const [filter, setFilter] = pp.state("all");
  </script>
</section>
```

Bad:

```html
<section class="dashboard-shell">
  <h1>Dashboard</h1>
</section>

<script>
  const [filter, setFilter] = pp.state("all");
</script>
```

Also bad:

```html
<section class="dashboard-shell">
  <!-- @import StatsCard from "../components" -->
  <x-stats-card title="Users" value="42" />
</section>
```

Example authored route template:

```html
<!-- @import StatsCard from "../components" -->

<section class="dashboard-shell">
  <x-stats-card title="Users" value="42" />

  <script>
    const [filter, setFilter] = pp.state("all");
  </script>
</section>
```

Rendered shape at runtime:

```html
<section pp-component="page_a1b2c3d4" class="dashboard-shell">
  <div pp-component="statscard_e5f6g7h8" title="Users" value="42">
    ...
  </div>

  <script type="text/pp">
    const [filter, setFilter] = pp.state("all");
  </script>
</section>
```

Write the first form. Caspian produces the second form by injecting `pp-component` on the root and `type="text/pp"` on the owned script.

### `index.py`

Use `index.py` as the backend companion when a route needs metadata or async server-side logic. For routes that render UI, keep the markup in the sibling `index.html` and let `page()` call `render_page(__file__, ...)`. Do not inline route HTML inside `index.py`.

Use `index.py` by itself only for non-visual routes such as redirects or action-only handlers. Because Caspian runs on FastAPI, the page entry should be async when it performs async work.

Example:

```python
from casp.layout import Metadata, render_page

metadata = Metadata(
    title="Caspian Documentation | The Native Python Web Framework",
    description="Explore the comprehensive documentation for Caspian.",
)

async def page():
    return render_page(__file__)
```

Use this pattern when the route needs to fetch data, compute metadata, or do other non-blocking server work before rendering the sibling `index.html`.

`page()` may also return a 2-item tuple: `(page_html, layout_props_dict)`.

- The first item is the rendered page HTML, usually `render_page(__file__, page_context)`.
- The second item must be a dict. Its keys are merged into the wrapping layout context and become available to parent layouts as `[[ layout.* ]]`.

Use that tuple form when one route needs to influence a wrapper without turning that value into a section-wide default. A common example is a dashboard page that needs to lock the root body with `overflow-hidden` while the rest of the app keeps normal scrolling.

Example root layout:

```html
<body class="[[ layout.dashboard_body_class | default('') ]]">
  [[ children | safe ]]
</body>
```

Example route:

```python
from casp.layout import render_page

async def page():
  return (
    render_page(__file__),
    {"dashboard_body_class": "w-screen h-screen overflow-hidden"},
  )
```

The key name is arbitrary, but it must match exactly between the dict returned from `page()` and the `[[ layout.some_key ]]` lookup in `layout.html`.

Use distinct names for those layout props. In the current router, the second dict is merged into the full layout context after path params and `request`, so a key such as `slug` or `request` can shadow an existing value.

When a route owns a file manager or upload UI, keep the owning upload and delete `@rpc()` actions in that same `index.py` and move reusable filesystem or Prisma helpers into `src/lib/`. Do not move ordinary upload behavior into `main.py`. See [file-uploads.md](./file-uploads.md).

For static and dynamic metadata rules, inheritance order, and social card fields, see [metadata.md](./metadata.md).

If the rendered HTML for that route is public and safe to reuse, declare route-level caching in the same file with `Cache(...)`. See [cache.md](./cache.md).

## Dynamic Routes

Caspian supports dynamic URL segments using the same bracket syntax used by the Next.js App Router.

### Dynamic Segments

Wrap a folder name in brackets to make it variable.

| File | Example URL |
| --- | --- |
| `src/app/users/[id]/index.html` | `/users/123` |

These segments are compiled into FastAPI path parameters for efficient route matching.

Add a sibling `index.py` when the dynamic route needs params during render, metadata, or other backend logic.

In the current `main.py` router, path params are collected into a single dict and passed as the first positional argument to `page()`. Matching query params can still be injected by name, and `request` is passed by keyword when declared.

Example:

```python
from casp.layout import render_page

async def page(params: dict):
    user_id = params["id"]
    return render_page(__file__, {"user_id": user_id})
```

### Catch-all Segments

Use an ellipsis inside brackets to match multiple path parts.

| File | Example URL |
| --- | --- |
| `src/app/docs/[...slug]/index.html` | `/docs/getting-started/setup` |

Use catch-all routes when the number of path segments is not fixed ahead of time.

Add a sibling `index.py` when that catch-all route also needs backend logic or metadata.

## Route Groups

Wrap a folder name in parentheses to organize code without adding that segment to the URL.

Examples:

| Files | URL |
| --- | --- |
| `src/app/(auth)/signin/index.html` and `src/app/(auth)/signin/index.py` | `/signin` |
| `src/app/(auth)/signup/index.html` and `src/app/(auth)/signup/index.py` | `/signup` |

Route groups are useful when you want to:

- Keep related features together, such as auth or dashboard flows.
- Split routes under different layout boundaries.
- Improve project organization without changing the public URL structure.

## Layouts And Nesting

Layouts work like the Next.js App Router layout system. A `layout.html` file wraps the routes beneath its folder, and nested layouts compose automatically.

When a layout imports components, keep each `<!-- @import ... -->` comment above the layout's authored wrapper element, such as the root `<section>` in a nested layout or the root `<html>` in the app layout.

Resolved SEO fields are exposed to layouts as `[[ metadata.* ]]`, while values returned from `layout.py` are exposed separately as `[[ layout.* ]]`.

For example, a page inside `/dashboard/settings` is wrapped by the root layout first and then by the dashboard layout.

Example root layout:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>[[ metadata.title ]]</title>
    <meta name="description" content="[[ metadata.description ]]" />
  </head>
  <body>
    <NavBar />
    [[ children | safe ]]
  </body>
</html>
```

### `layout.py`

If a layout needs shared synchronous props or metadata, add a `layout.py` file next to the HTML layout.

Example:

```python
from casp.auth import auth

def layout(context_data):
    user = auth.get_payload()

    return {
        "user": user,
    "dashboard_body_class": "w-screen h-screen overflow-hidden",
        "theme": "dark",
    }
```

`context_data` includes URL parameters such as dynamic route values.

In the common case, return a dict and let the sibling `layout.html` read those values through `[[ layout.* ]]`.

`layout()` currently supports these result shapes:

- `dict`: load the sibling `layout.html` and expose the dict as `[[ layout.* ]]`
- `str`: use that string as the layout content
- `(layout_html, props_dict)`: use the first item as the layout content and expose the second dict as `[[ layout.* ]]`
- `None`: fall back to the sibling `layout.html` with no extra layout props

If you intentionally want to render `layout.html` immediately with direct local variables instead of the `layout.*` namespace, call `render_layout(__file__, {...})` and reference those keys directly in the template.

Example:

```python
from casp.layout import render_layout

def layout():
  return render_layout(__file__, {"my_class": "size-8"})
```

In that pattern, the matching `layout.html` reads `[[ my_class ]]`, not `[[ layout.my_class ]]`, because the template string was already rendered before the nested layout pipeline continues.

If you need both a custom layout string and standard `[[ layout.* ]]` props, return a tuple:

```python
from casp.layout import render_layout

def layout():
  return (
    render_layout(__file__),
    {"dashboard_body_class": "w-screen h-screen overflow-hidden"},
  )
```

`layout()` currently runs synchronously in `casp.layout`. If you need async I/O, load it in `page()` instead of `layout.py`.

Use [metadata.md](./metadata.md) when a layout also needs SEO defaults. Return dictionaries from `layout()` for visual or template props, and use `Metadata(...)` for title, description, and social tags.

## Recommended Structure

This example shows a typical nested routing setup:

```text
src/
  app/
    layout.html
    index.html
    about/
      index.html
    users/
      [id]/
        index.html
        index.py
    docs/
      [...slug]/
        index.html
        index.py
    (auth)/
      login/
        index.html
        index.py
      register/
        index.html
        index.py
      signout/
        index.py
    dashboard/
      layout.html
      settings/
        index.html
        index.py
```

## AI Retrieval Notes

If an AI agent is choosing where to add or update route code, apply these rules first.

- Treat `src/app/` as the routing source of truth.
- Use folder names to model URL segments.
- If a route renders UI, create or update `index.html` for the markup.
- Add `index.py` only when the same route needs metadata or server behavior; do not place route HTML in `index.py`.
- Use [cache.md](./cache.md) when an `index.py` route should opt into page-level HTML caching.
- Use `layout.html` for shared wrappers and `layout.py` for layout-level synchronous props or metadata.
- When one route needs to change a parent layout, return `(render_page(__file__, ...), {"dashboard_body_class": ...})` from `page()` and read that value as `[[ layout.dashboard_body_class ]]` in the wrapping `layout.html`.
- Use `layout.py` for layout props that should apply across an entire subtree. Use `render_layout(__file__, {...})` only when the layout should consume direct local variables such as `[[ my_class ]]` instead of the standard `[[ layout.* ]]` namespace.
- Keep `<!-- @import ... -->` directives at the top of `index.html` and `layout.html`, above the single authored root element.
- Use [metadata.md](./metadata.md) when a route or layout needs SEO fields.
- Use `[segment]` for single dynamic parameters.
- Use `[...segment]` for catch-all route matching.
- Use `(group)` folders for organization when the folder should not appear in the URL.
- Use `.venv/Lib/site-packages/casp/layout.py` only when the task is about Caspian core routing, layout, or metadata internals.
- If you know the Next.js App Router, follow the same routing mental model but generate Caspian file names instead of React files.

Check [project-structure.md](./project-structure.md) when you need the full Caspian directory map.

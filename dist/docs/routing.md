---
title: Routing
description: Understand Caspian's Next.js App Router-style file-based routing, including src/app conventions, index files, dynamic segments, route groups, nested layouts, and component-friendly route templates.
related:
  title: Related docs
  description: Read the structure guide first, then use the components guide for reusable UI, the metadata guide for SEO fields, the cache guide for route-level HTML reuse, and the PulsePoint runtime guide for interactive route templates.
  links:
    - /docs/project-structure
    - /docs/components
    - /docs/cache
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
- Use `index.html` for a template-only route.
- Use `index.py` when the route needs metadata or async server-side logic.
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
| `page.tsx` | `index.html` or `index.py` |
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

Every route lives inside `src/app`. A route is a folder that contains either an `index.html` file, an `index.py` file, or both as part of the route implementation.

Examples:

| File | URL |
| --- | --- |
| `src/app/index.html` | `/` |
| `src/app/about/index.py` | `/about` |
| `src/app/blog/posts/index.html` | `/blog/posts` |

### `index.html`

Use `index.html` for the route template. This is the route's view layer.

Route templates can import reusable Python components with `<!-- @import ... -->` comments and render them with JSX-style tags such as `<Button />`. Use [components.md](./components.md) for the component authoring rules.

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
  <StatsCard title="Users" value="42" />
</section>
```

Example authored route template:

```html
<!-- @import StatsCard from "../components" -->

<section class="dashboard-shell">
  <StatsCard title="Users" value="42" />

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

Use `index.py` when the route needs metadata or async server-side logic. Because Caspian runs on FastAPI, the page entry should be async.

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

Use this pattern when the route needs to fetch data, compute metadata, or do other non-blocking server work before rendering.

For static and dynamic metadata rules, inheritance order, and social card fields, see [metadata.md](./metadata.md).

If the rendered HTML for that route is public and safe to reuse, declare route-level caching in the same file with `Cache(...)`. See [cache.md](./cache.md).

## Dynamic Routes

Caspian supports dynamic URL segments using the same bracket syntax used by the Next.js App Router.

### Dynamic Segments

Wrap a folder name in brackets to make it variable.

| File | Example URL |
| --- | --- |
| `src/app/users/[id]/index.py` | `/users/123` |

These segments are compiled into FastAPI path parameters for efficient route matching.

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
| `src/app/docs/[...slug]/index.py` | `/docs/getting-started/setup` |

Use catch-all routes when the number of path segments is not fixed ahead of time.

## Route Groups

Wrap a folder name in parentheses to organize code without adding that segment to the URL.

Examples:

| File | URL |
| --- | --- |
| `src/app/(auth)/login/index.py` | `/login` |
| `src/app/(auth)/register/index.py` | `/register` |

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
        "theme": "dark",
    }
```

`context_data` includes URL parameters such as dynamic route values.

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
        index.py
    docs/
      [...slug]/
        index.py
    (auth)/
      login/
        index.py
      register/
        index.py
    dashboard/
      layout.html
      settings/
        index.py
```

## AI Routing Notes

If an AI agent is choosing where to add or update route code, apply these rules first.

- Treat `src/app/` as the routing source of truth.
- Use folder names to model URL segments.
- Use `index.html` for route templates and `index.py` for route-level async logic.
- Use [cache.md](./cache.md) when an `index.py` route should opt into page-level HTML caching.
- Use `layout.html` for shared wrappers and `layout.py` for layout-level synchronous props or metadata.
- Keep `<!-- @import ... -->` directives at the top of `index.html` and `layout.html`, above the single authored root element.
- Use [metadata.md](./metadata.md) when a route or layout needs SEO fields.
- Use `[segment]` for single dynamic parameters.
- Use `[...segment]` for catch-all route matching.
- Use `(group)` folders for organization when the folder should not appear in the URL.
- Use `.venv/Lib/site-packages/casp/layout.py` only when the task is about Caspian core routing, layout, or metadata internals.
- If you know the Next.js App Router, follow the same routing mental model but generate Caspian file names instead of React files.

Check [project-structure.md](./project-structure.md) when you need the full Caspian directory map.

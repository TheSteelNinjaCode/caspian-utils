---
title: Routing
description: Understand Caspian's Next.js App Router-style file-based routing, including src/app conventions, index files, dynamic segments, route groups, and nested layouts.
related:
  title: Related docs
  description: Read the structure guide first, then use the metadata guide for SEO fields and the PulsePoint runtime guide for interactive route templates.
  links:
    - /docs/project-structure
    - /docs/metadata
    - /docs/pulsepoint
    - /docs/index
---

Caspian follows the same mental model as the Next.js App Router: routes live under `src/app`, folders define URL segments, layouts nest automatically, and special folder names control grouping and dynamic matching.

The main difference is the file types. Instead of `page.tsx` and `layout.tsx`, Caspian uses `index.html`, `index.py`, `layout.html`, and optional Python companions for async server-side logic.

## Overview

Caspian uses a high-performance file-system router built on top of FastAPI. Your directory structure becomes your URL structure.

Start with these rules:

- Put application routes in `src/app/`.
- Use `index.html` for a template-only route.
- Use `index.py` when the route needs metadata or async server-side logic.
- Use `layout.html` to wrap child routes.
- Use `layout.py` when a layout needs async data before rendering.

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

## Dynamic Routes

Caspian supports dynamic URL segments using the same bracket syntax used by the Next.js App Router.

### Dynamic Segments

Wrap a folder name in brackets to make it variable.

| File | Example URL |
| --- | --- |
| `src/app/users/[id]/index.py` | `/users/123` |

These segments are compiled into FastAPI path parameters for efficient route matching.

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

If a layout needs async data, add a `layout.py` file next to the HTML layout.

Example:

```python
from casp.auth import get_current_user

async def layout(context_data):
    user = await get_current_user()

    return {
        "user": user,
        "theme": "dark",
    }
```

`context_data` includes URL parameters such as dynamic route values.

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
- Use `layout.html` for shared wrappers and `layout.py` for layout-level async data.
- Use [metadata.md](./metadata.md) when a route or layout needs SEO fields.
- Use `[segment]` for single dynamic parameters.
- Use `[...segment]` for catch-all route matching.
- Use `(group)` folders for organization when the folder should not appear in the URL.
- Use `.venv/Lib/site-packages/casp/layout.py` only when the task is about Caspian core routing, layout, or metadata internals.
- If you know the Next.js App Router, follow the same routing mental model but generate Caspian file names instead of React files.

Check [project-structure.md](./project-structure.md) when you need the full Caspian directory map.

---
title: Metadata & SEO
description: Use this page when the task mentions metadata, SEO, page titles, descriptions, Open Graph, Twitter cards, or layout-to-page metadata inheritance in Caspian.
related:
  title: Related docs
  description: Use the routing guide to place layout.py and index.py files correctly, then use the structure guide when deciding where metadata defaults belong.
  links:
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page explains how Caspian handles document metadata, SEO fields, and social sharing tags.

At render time, Caspian resolves metadata through the layout engine and exposes the merged result to templates as `[[ metadata.* ]]`.

## Overview

Caspian uses a cascading metadata model similar to the Next.js App Router. Define defaults high in the tree, then override them closer to the final route.

Use the `Metadata` class from `casp.layout` when you need to set:

- Page titles
- Meta descriptions
- Open Graph tags
- Twitter card tags
- Other head metadata through the `extra` dictionary

Metadata is typically defined in one of two files:

- `layout.py` for defaults shared by everything below that folder
- `index.py` for route-specific metadata

On routes that render UI, keep the page markup in the sibling `index.html`. `index.py` is the metadata and backend companion, not the place to store route HTML. If a route needs no metadata or backend behavior, omit `index.py` and keep the page as `index.html` only.

The current `Metadata` implementation supports three fields:

- `title`
- `description`
- `extra`

## Static Definition

For metadata that does not depend on route params or fetched data, define a module-level `metadata` variable.

Example:

```python
from casp.layout import Metadata, render_page

metadata = Metadata(
    title="About Us | My App",
    description="Learn more about our team and mission.",
    extra={
        "og:title": "About Us | My App",
        "og:description": "Learn more about our team and mission.",
        "og:image": "/assets/og-about.jpg",
        "twitter:card": "summary_large_image",
    },
)

async def page():
    return render_page(__file__)
```

Use static metadata for:

- Marketing pages
- Docs pages
- Any route where the SEO copy is fixed at build or authoring time

The explicit `metadata = Metadata(...)` form is the clearest pattern and should be preferred in project code.

Implementation note:

- If you call `Metadata(...)` at module scope without assigning it, the current layout engine auto-registers that instance as the module's `metadata` value.
- That shortcut works because `Metadata.__post_init__()` inspects the caller frame and writes to module globals.
- Prefer the explicit assignment anyway so the intent is obvious to readers and tools.

## Dynamic Generation

For metadata that depends on a dynamic segment or fetched content, instantiate `Metadata(...)` inside the route function. Runtime metadata overrides the static defaults inherited from layouts.

Example:

```python
from casp.layout import Metadata, render_page

async def page(params: dict):
    slug = params["slug"]
    product_name = f"Product {slug.capitalize()}"

    Metadata(
        title=f"{product_name} | Store",
        description=f"Buy {product_name} at the best price.",
        extra={
            "og:title": f"{product_name} | Store",
            "twitter:title": f"{product_name} | Store",
        },
    )

    return render_page(__file__, {"name": product_name})
```

In the current router, dynamic path params arrive as a single `params` dict passed to `page()`.

Set dynamic metadata before calling `render_page(...)` so the route renders with the correct head values.

Inside the same module, runtime `Metadata(...)` overrides matching keys from the module-level static `metadata` object.

## How Inheritance Works

Caspian merges metadata from the root layout down to the final page. Lower levels override higher levels when they define the same field.

Priority order:

1. `src/app/layout.py` sets site-wide defaults.
2. Nested `layout.py` files override those defaults for a section.
3. `index.py` metadata or runtime `Metadata(...)` inside the route has the highest priority.

Within a single file, the layout engine applies static metadata first and then applies runtime metadata from `Metadata(...)`, so runtime values win for matching keys.

If a child route overrides only the title, the description and other fields continue to fall through from the nearest parent that defines them.

Example structure:

```text
src/
  app/
    layout.py
    blog/
      layout.py
      [slug]/
        index.py
```

Typical result:

- Root layout defines the site-wide title template and default description.
- Blog layout overrides section-level fields for `/blog/*`.
- Blog page overrides the final post title and any page-specific social tags.

## Template Access

Resolved metadata is passed into layout rendering as a `metadata` object.

Example:

```html
<title>[[ metadata.title ]]</title>
<meta name="description" content="[[ metadata.description ]]" />
<meta property="og:image" content="[[ metadata['og:image'] ]]" />
<meta name="twitter:card" content="[[ metadata['twitter:card'] ]]" />
```

Use bracket access for `extra` keys that contain characters such as `:`.

## Social Cards With `extra`

Use the `extra` dictionary for Open Graph and Twitter card tags.

Common keys include:

- `og:title`
- `og:description`
- `og:image`
- `twitter:card`
- `twitter:title`
- `twitter:description`
- `twitter:image`

Use stable, publicly reachable image paths for social cards so crawlers can fetch them reliably.

## Layout Props vs Metadata

Keep visual layout data and SEO metadata separate.

- Values returned from `layout()` are exposed as `[[ layout.* ]]`.
- The second dict returned from `page()` as `(page_html, layout_props_dict)` is also exposed to wrapping layouts as `[[ layout.* ]]`.
- SEO values are exposed as `[[ metadata.* ]]`.
- Do not return `title` or `description` from `layout()` expecting SEO changes.
- The layout engine explicitly strips `title` and `description` from layout props to avoid mixing visual props with metadata.

## Where To Put Defaults

- Put site-wide defaults in `src/app/layout.py`.
- Put section defaults in nested `layout.py` files such as `src/app/blog/layout.py`.
- Put page-specific static metadata in `index.py`.
- Put dynamic metadata inside `page()` after you have route params or fetched data.

## AI Retrieval Notes

If an AI agent is deciding where to put SEO fields, apply these rules first.

- Use `Metadata` from `casp.layout` for Caspian metadata.
- Prefer module-level `metadata = Metadata(...)` for static routes.
- Instantiate `Metadata(...)` inside `page()` when metadata depends on params, fetched records, or generated content.
- Put shared defaults in `layout.py` and let leaf pages override only what they need.
- If a single route only needs to tweak a wrapping layout, return `(render_page(__file__, ...), {"dashboard_body_class": ...})` from `page()` instead of moving that prop into metadata.
- Use `extra` for Open Graph and Twitter card tags.
- Access `extra` values in templates with bracket syntax such as `metadata['og:image']`.
- Keep `layout()` return data in `[[ layout.* ]]` and keep SEO fields in `Metadata(...)`.
- Check [routing.md](./routing.md) when deciding whether metadata belongs in a layout or a specific route folder.

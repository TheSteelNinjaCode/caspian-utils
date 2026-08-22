---
title: Routing
description: Use this page when the task mentions `src/app`, `index.py`, `layout.py`, dynamic routes, route groups, nested layouts, or file-based routing in Caspian.
related:
  title: Related docs
  description: Read the structure guide first, then use the components guide for reusable UI, the metadata guide for SEO fields, the cache guide for route-level HTML reuse, and the PulsePoint runtime guide for interactive route templates.
  links:
    - /docs/core-runtime-map
    - /docs/project-structure
    - /docs/components
    - /docs/cache
    - /docs/file-uploads
    - /docs/metadata
    - /docs/pulsepoint
    - /docs/pulsepoint-runtime-map
    - /docs/index
---

Caspian follows the same mental model as the Next.js App Router: routes live under `src/app`, folders define URL segments, layouts nest automatically, and special folder names control grouping and dynamic matching.

The main difference is the file types. Instead of `page.tsx` and `layout.tsx`, Caspian uses exactly one Python file per concern: `index.py` for a route and `layout.py` for a subtree shell. The markup lives inside those files as a triple-quoted template handed to `html(...)` (routes) or returned raw from `layout()` (layouts).

## Overview

Caspian uses a high-performance file-system router built on top of FastAPI. Your directory structure becomes your URL structure.

Start with these rules:

- Put application routes in `src/app/`.
- Every route is one `index.py` with a `page()` function. A UI route returns `html(r"""...""")` markup; a non-visual route (redirects, action-only handlers) returns a `Response`.
- Keep route-specific server logic in that route's `index.py`. Move logic into `src/lib/**` only when it is shared across routes, components, integrations, or features.
- When a folder owns child routes, add `layout.py` whose `layout()` returns the shared wrapper template. This is the default pattern for dashboards, admin sections, account areas, settings trees, and route groups.
- In grouped shells with separate shell and content scrolling, put `pp-reset-scroll="true"` on the content pane in the layout template when that pane should reset on child-route navigation while the shell sidebar or rail keeps its own scroll.
- Components used as `<x-*>` tags in a page or layout template resolve from the Python imports at the top of that module.
- Treat every authored route and layout template like a React component body **for root counting only**: normally one top-level parent HTML element or one imported `x-*` root, with any owned plain `<script>` inside that same root. Sibling top-level nodes are permitted in a `.py` route or layout and get a compiler-supplied boundary host — see "Template Root Shape" below. The markup itself is plain HTML, never JSX — see [pulsepoint.md](./pulsepoint.md) "PulsePoint Is Not JSX".
- For route and layout interactivity, use PulsePoint in the authored markup first: native `on*` event attributes, `pp.state(...)`, refs, effects, directives, and `pp.rpc()`. Do not create standard JavaScript event systems with ids, `data-*` state, `querySelector`, `addEventListener`, or manual `innerHTML` for normal first-party UI.

## Template Root Shape

Single-root is the shape to write for a route or layout by default.

- Use exactly one authored parent node.
- Keep any owned plain `<script>` inside that root, not after it.
- Do not leave sibling top-level HTML tags, sibling component tags, or stray top-level text.

A `.py` route or layout that does end up with sibling top-level nodes is **not** a render failure: the compiler wraps the whole template in a layout-neutral `<div pp-component style="display: contents">` boundary host. Use that when a page's sections are genuinely siblings and a wrapper `<div>` would be meaningless; keep the single owned `<script>` inside the template as usual. See [pulsepoint.md](./pulsepoint.md) "Multi-root pages and layouts".

The relaxation stops at routes and layouts. A **component** with sibling top-level nodes does not get this host — it becomes a fragment instead, and can no longer receive props. `TemplateRootError` (`must have exactly one top-level HTML element so Caspian can inject pp-component`) is now raised only when a component template has no root at all, or when its only root is an unresolvable `x-*` tag — see [components.md](./components.md) "Single-Root Rule".

Use [cache.md](./cache.md) when an `index.py` route also needs declarative page caching via `Cache(...)`.

Framework internals note:

- Caspian's layout and route-resolution internals live in `.venv/Lib/site-packages/casp/layout.py`; `html(...)` lives in `.venv/Lib/site-packages/casp/component_decorator.py`.
- Treat those files as framework code. Read them when the task is about routing internals, layout resolution, or metadata behavior inside Caspian itself.
- Use [core-runtime-map.md](./core-runtime-map.md) when a routing task crosses `main.py` route registration, parameter injection, and installed layout internals.

See [metadata.md](./metadata.md) when a page or layout needs SEO fields.

## Next.js App Router Mapping

If you already know the Next.js App Router, use this translation layer:

| Next.js concept | Caspian equivalent |
| --- | --- |
| `app/` | `src/app/` |
| `page.tsx` | `index.py` with `page()` returning `html(...)` |
| `layout.tsx` | `layout.py` with `layout()` returning the wrapper template |
| `[id]` | `[id]` |
| `[...slug]` | `[...slug]` |
| `(group)` | `(group)` |

This means most App Router habits carry over directly:

- Keep route logic close to the route folder.
- Model the URL with folders instead of a central route table.
- Use nested layouts for shared wrappers.
- Use route groups to organize code without changing the public path.

## Section Layout Pattern

Treat this section as the canonical grouped-subtree structure rule for the packaged docs. Other pages should point here instead of repeating the full folder pattern.

When a user asks for a dashboard, admin area, account section, docs section, or any grouped set of child routes, model it exactly like a Next.js App Router subtree.

- Create a parent folder for the section.
- Add `layout.py` in that folder for the shared shell.
- Put each child page in its own route folder with its `index.py`.
- Use a normal folder name such as `dashboard/` when the section name should appear in the URL.
- Use a route-group folder such as `(marketing)/` when the folder should organize code and own a layout without adding a URL segment.

Examples:

```text
src/
  app/
    dashboard/
      layout.py
      index.py
      settings/
        index.py
      users/
        index.py
```

This produces `/dashboard`, `/dashboard/settings`, and `/dashboard/users`, all wrapped by the `dashboard/layout.py` shell.

```text
src/
  app/
    (marketing)/
      layout.py
      pricing/
        index.py
      about/
        index.py
```

This produces `/pricing` and `/about`, both wrapped by the `(marketing)/layout.py` shell even though `(marketing)` does not appear in the URL.

Canonical end-to-end example:

```text
src/
  app/
    layout.py
    (marketing)/
      layout.py
      pricing/
        index.py
      about/
        index.py
    dashboard/
      layout.py
      loading.py        # optional
      index.py
      settings/
        index.py
      reports/
        index.py
```

`dashboard/loading.py` is optional and most sections do not have one. Include it only when the section is meant to show a loading state during child-route navigation; when it is, that file is the shipped mechanism for the whole `/dashboard/*` subtree.

Use this pattern when the app has a public grouped section that should stay out of the URL and a private dashboard section that should appear in the URL. The root layout owns app-wide chrome, `(marketing)/layout.py` owns the shared public marketing shell, and `dashboard/layout.py` owns the shared dashboard shell for `/dashboard/*` child routes.

A section shell is also where navigation loading would go, if the section wants one. It is optional — a subtree with no loader navigates with a plain fade, which is the common case. When one is wanted, add `loading.py` beside the section's `layout.py` and mark the pane it should replace with `pp-loading-content="true"` in the layout; the closest ancestor `loading.py` wins, so one file covers the whole subtree. Do not build a spinner component or an `isLoading` state for this; see [file-conventions.md](./file-conventions.md#loadingpy).

When a shared shell has its own scrollable sidebar or rail plus a separate page-content scroller, keep the persistent shell scrollers unmarked and put `pp-reset-scroll="true"` on the page-content scroller in the layout template. That gives grouped sections the common app-router behavior where the main pane resets on child-route navigation while the sidebar keeps its scroll position. Use `body[pp-reset-scroll="true"]` only when the target route should reset every scrollable surface.

## Core Concepts

Every route lives inside `src/app` as one `index.py`.

Examples:

| Files | URL |
| --- | --- |
| `src/app/index.py` | `/` |
| `src/app/about/index.py` | `/about` |
| `src/app/dashboard/index.py` | `/dashboard` |
| `src/app/blog/posts/index.py` | `/blog/posts` |
| `src/app/(auth)/signout/index.py` | `/signout` |

### `index.py`

`index.py` is the route: its `page()` returns the page markup through `html(...)`, and the same module owns metadata, auth checks, redirects, caching, and route-owned `@rpc()` actions.

Route templates import reusable Python components with normal Python imports and render them with HTML-first `x-*` tags such as `<x-button />`. Use [components.md](./components.md) for the component authoring rules.

Keep route templates as composition, not as the place where all section markup accumulates. A route with tabs, dashboard panels, forms, tables, or repeated cards should import focused components for those responsibilities and assemble them with `x-*` tags. For example, a dashboard route can import `<x-dashboard-header />`, `<x-metric-strip />`, `<x-activity-tab />`, and `<x-settings-tab />` rather than placing every tab panel's full markup in one giant template.

Route templates follow the same authored-vs-runtime contract documented in [pulsepoint.md](./pulsepoint.md) and the same single-root discipline documented in [components.md](./components.md): keep one authored parent node, use a plain `<script>` inside that root when needed, and do not handwrite `pp-component`. That root may be a native HTML element or a single imported `x-*` component tag, but after expansion it must resolve to one final HTML root.

When a route needs button clicks, form submits, input changes, filters, tabs, menus, uploads, polling, or reactive list updates, author those interactions as PulsePoint behavior in the template. Use `onclick`, `oninput`, `onchange`, `onsubmit`, `pp.state(...)`, `pp-for`, refs, effects, and `pp.rpc(...)` instead of starting with `id` attributes plus `document.querySelector(...)` or `addEventListener(...)`. For simple form submits, prefer `onsubmit="{submitForm(event)}"` and `Object.fromEntries(new FormData(event.currentTarget).entries())` over per-input `pp-ref` payload collection.

Example:

```python
from casp.component_decorator import html
from casp.layout import Metadata
from src.components.StatsCard import StatsCard

metadata = Metadata(
    title="Dashboard | Caspian",
    description="Overview page for the dashboard.",
)


async def page():
    return html(r"""
<section class="dashboard-shell">
  <h1>Dashboard</h1>

  <x-stats-card title="Users" value="42" />

  <script>
    const [filter, setFilter] = pp.state("all");
  </script>
</section>
""")
```

Good template shape (one root, script inside it):

```html
<section class="dashboard-shell">
  <h1>Dashboard</h1>

  <script>
    const [filter, setFilter] = pp.state("all");
  </script>
</section>
```

Bad (script outside the root, or sibling top-level tags):

```html
<section class="dashboard-shell">
  <h1>Dashboard</h1>
</section>

<script>
  const [filter, setFilter] = pp.state("all");
</script>
```

If the logic belongs only to this route, keep it in the same `index.py`. Examples include this page's first-render query, route-owned RPC actions, redirect decisions, route-specific validation, route-specific filters, and route-specific response shaping. Move code into `src/lib/**` when the same logic is reused or intentionally shared; do not extract one-route orchestration into a library just because it is Python code.

`page()` may also return a 2-item tuple: `(page_html, layout_props_dict)`.

- The first item is the rendered page markup, usually `html(r"""...""", **context)`.
- The second item must be a dict. Its keys are merged into the wrapping layout context and become available to parent layouts as `{{ layout.* }}`.

Use that tuple form when one route needs to influence a wrapper without turning that value into a section-wide default. A common example is a dashboard page that needs to lock the root body with `overflow-hidden` while the rest of the app keeps normal scrolling.

Example root layout template fragment:

```html
<body class="{{ layout.dashboard_body_class | default('') }}">
  <slot />
</body>
```

Example route:

```python
from casp.component_decorator import html

async def page():
    return (
        html(r"""<div class="dashboard">...</div>"""),
        {"dashboard_body_class": "w-screen h-screen overflow-hidden"},
    )
```

The key name is arbitrary, but it must match exactly between the dict returned from `page()` and the `{{ layout.some_key }}` lookup in the layout template.

Use distinct names for those layout props. In the current router, the second dict is merged into the full layout context after path params and `request`, so a key such as `slug` or `request` can shadow an existing value.

When a route owns a file manager or upload UI, keep the owning upload and delete `@rpc()` actions in that same `index.py` and move reusable filesystem or Prisma helpers into `src/lib/`. Do not move ordinary upload behavior into `main.py`. See [file-uploads.md](./file-uploads.md). When Prisma is enabled, database reads and writes in this route logic or its shared helpers should use the generated Prisma Python ORM.

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
from casp.component_decorator import html

async def page(params: dict):
    user_id = params["id"]
    return html(r"""
<section>
  <h1>User {{ user_id }}</h1>
</section>
""", user_id=user_id)
```

### Catch-all Segments

Use an ellipsis inside brackets to match multiple path parts.

| File | Example URL |
| --- | --- |
| `src/app/docs/[...slug]/index.py` | `/docs/getting-started/setup` |

Use catch-all routes when the number of path segments is not fixed ahead of time.

If the project uses static export (SSG) and you want a dynamic or catch-all route pre-rendered into the static build, export a `static_paths` provider from that route's `index.py` (Caspian's `getStaticPaths` equivalent). Without it, the static exporter skips the route by design. See [static-export.md](./static-export.md).

## Route Groups

Wrap a folder name in parentheses to organize code without adding that segment to the URL.

Examples:

| Files | URL |
| --- | --- |
| `src/app/(auth)/signin/index.py` | `/signin` |
| `src/app/(auth)/signup/index.py` | `/signup` |

Route groups are useful when you want to:

- Keep related features together, such as auth or dashboard flows.
- Split routes under different layout boundaries.
- Improve project organization without changing the public URL structure.

If the group needs a shared wrapper, put `layout.py` inside the `(group)` folder. That layout applies to every child route in the group while the group name stays out of the URL.

Use this decision rule:

- Use `dashboard/`, `account/`, or another normal folder when that segment should be part of the public path.
- Use `(dashboard)`, `(marketing)`, or another parenthesized folder when you want shared organization or a shared layout without adding a public path segment.

## Layouts And Nesting

Layouts work like the Next.js App Router layout system. A `layout.py` file wraps the routes beneath its folder, and nested layouts compose automatically.

In practice, this means a dashboard is usually a folder-level layout, not a single oversized page. Put the shared sidebar, header, and frame in `dashboard/layout.py`, then create child routes such as `dashboard/settings/index.py` and `dashboard/reports/index.py` beneath it.

Resolved SEO fields are exposed to layout templates as `{{ metadata.* }}`, while props returned from `layout()` are exposed separately as `{{ layout.* }}`.

### `layout.py`

`layout()` returns the shared wrapper markup for its subtree through `html(...)`, the same single markup entrypoint pages and components use.

A layout is the one caller whose markup cannot be rendered when the function runs: `children` is the page beneath it, which has not been composed yet. So `html(...)` called from a `layout()` does **not** render — it hands the template back unrendered, and the runtime renders it once later with `children`, `{{ layout.* }}` and `{{ metadata.* }}` merged into whatever context you passed. Those three names are engine-owned and win over your context, so a layout cannot accidentally shadow its own `children`.

The deferral is keyed on the `layout()` function itself, not on "anything that runs during a layout": a component the layout calls, or a helper in the same file, still renders eagerly and immediately.

A layout must place its children somewhere — `<slot />` or `{{ children }}`. One that does neither raises `LayoutChildrenError` instead of silently serving a shell with no page in it.

Follow the same authoring contract used by route templates: one authored parent node, a plain `<script>` inside the root when needed, and no handwritten `pp-component`. `<x-*>` tags in the returned template resolve from the Components imported at the top of `layout.py`. See [pulsepoint.md](./pulsepoint.md) for the canonical authored-vs-runtime explanation.

Place child routes with a plain HTML `<slot />` tag. Caspian replaces that layout slot with the current child route or nested layout while rendering.

For example, a page inside `/dashboard/settings` is wrapped by the root layout first and then by the dashboard layout.

Example root layout (`src/app/layout.py`):

```python
def layout():
    return html(r"""
<!DOCTYPE html>
<html>
  <head>
    <title>{{ metadata.title }}</title>
    <meta name="description" content="{{ metadata.description }}" />
  </head>
  <body>
    <slot />
  </body>
</html>
""")
```

`layout()` currently supports these result shapes:

- `html(...)`: the layout template for the subtree, plus any context you pass — **the form to write**
- `(html(...), props_dict)`: the template plus `{{ layout.* }}` props
- `dict`: a passthrough `<slot />` shell plus `{{ layout.* }}` props
- `None`: a passthrough `<slot />` shell with no extra layout props
- `str`: the legacy raw-template form, still supported — `html(...)` in a layout produces exactly this string plus its context

Example with shared props:

```python
from casp.auth import auth

def layout(context_data):
    user = auth.get_payload()

    return r"""
<div class="dashboard-shell">
  <aside>Signed in as {{ layout.user.name }}</aside>
  <main pp-reset-scroll="true"><slot /></main>
</div>
""", {
        "user": user,
        "theme": "dark",
    }
```

`context_data` includes URL parameters such as dynamic route values.

`layout()` may be synchronous or async in the installed runtime. Keep async layout work focused on shared subtree props or metadata; use `page()` or `@rpc()` when the work belongs to one route or a browser-triggered user action.

Use [metadata.md](./metadata.md) when a layout also needs SEO defaults. Return template/props from `layout()` for visual concerns, and use `Metadata(...)` for title, description, and social tags.

## Recommended Structure

This example shows a typical nested routing setup:

```text
src/
  app/
    layout.py
    index.py
    about/
      index.py
    (marketing)/
      layout.py
      pricing/
        index.py
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
      signout/
        index.py
    dashboard/
      layout.py
      index.py
      settings/
        index.py
      reports/
        index.py
```

## AI Retrieval Notes

If an AI agent is choosing where to add or update route code, apply these rules first.

- Treat `src/app/` as the routing source of truth.
- Use folder names to model URL segments.
- Every route is one `index.py`; `page()` returns `html(r"""...""")` markup for UI routes and a `Response` for non-visual routes.
- Keep route-specific backend logic in the route's own `index.py`; extract to `src/lib/**` only for genuinely shared logic.
- Keep the page template a short assembly of focused `x-*` components. When a route area has its own responsibility, such as a tab panel, settings form, data table, toolbar, or metric section, create a component for that area and pass route data into it as props.
- Import every component used as an `<x-*>` tag at the top of the owning Python module.
- Use PulsePoint as the first-party interaction model for route and layout markup. Avoid custom DOM wiring for normal events and reactivity.
- When the user asks for a dashboard, admin area, account section, or any grouped subtree of child routes, create a parent folder with `layout.py` and place the child routes beneath it. Follow the same mental model as the Next.js App Router.
- Use a normal folder such as `dashboard/` when the segment should appear in the URL. Use `(group)/` only when the folder should organize or wrap child routes without adding a path segment.
- Use [cache.md](./cache.md) when an `index.py` route should opt into page-level HTML caching.
- For grouped shells with separate shell and content scrolling, put `pp-reset-scroll="true"` on the content pane instead of the whole shell when only the page content should reset between child routes.
- When one route needs to change a parent layout, return `(html(...), {"dashboard_body_class": ...})` from `page()` and read that value as `{{ layout.dashboard_body_class }}` in the wrapping layout template.
- Use [metadata.md](./metadata.md) when a route or layout needs SEO fields.
- Use `[segment]` for single dynamic parameters.
- Use `[...segment]` for catch-all route matching.
- Use `(group)` folders for organization when the folder should not appear in the URL.
- Use `.venv/Lib/site-packages/casp/layout.py` only when the task is about Caspian core routing, layout, or metadata internals.
- If you know the Next.js App Router, follow the same routing mental model but generate Caspian file names instead of React files.

Check [project-structure.md](./project-structure.md) when you need the full Caspian directory map.

---
title: Project Structure
description: Understand the default Caspian project layout so AI agents place routes, reusable components, PulsePoint templates, RPC actions, validation helpers, auth code, MCP files, configuration, and database changes in the correct directories.
related:
  title: Related docs
  description: Start with installation for new apps, then use the component guide for reusable UI, the auth guide for bootstrap and session wiring, the MCP guide for server layout, the routing guide to map URLs correctly, and the cache guide when route HTML should be reused safely.
  links:
    - /docs/installation
    - /docs/components
    - /docs/auth
    - /docs/mcp
    - /docs/routing
    - /docs/cache
    - /docs/database
    - /docs/index
---

This page explains the default layout of a Caspian application, where Caspian core files live, and which paths AI agents should treat as project code versus framework internals.

## Overview

Caspian uses a lean project layout that keeps application code in `src`, database files in `prisma`, static assets in `public`, configuration in `caspian.config.json`, and framework internals in the installed package.

As an app grows, keep reusable rendered UI in `src/components/`, keep reusable non-UI support code in `src/lib/`, and keep route-owned files in `src/app/`. That split keeps page composition separate from shared component and service code.

In that layout, the default stack is Python components for reusable UI, PulsePoint in templates for reactive browser behavior, RPC for browser-triggered server calls, and `casp.validate` for input validation at route and action boundaries.

For public pages that can safely reuse rendered HTML, Caspian also supports route-level page caching through `casp.cache_handler`.

Before an AI agent decides which Caspian features are available in a workspace, it should read `./caspian.config.json` almost immediately. That file is the feature gate for project capabilities such as `backendOnly`, `tailwindcss`, `mcp`, `prisma`, `typescript`, and `componentScanDirs`.

Treat `caspian.config.json` as the single source of truth for optional feature enablement. Use feature-specific files and docs only after the matching flag is confirmed as enabled. If a feature is disabled and the user wants it, ask first, then update `caspian.config.json` and follow the update workflow in `commands.md`.

## Top-Level Areas

- `src/` contains routes, page templates, styles, reusable components, and shared libraries.
- `src/components/` contains reusable application UI components and optional same-name HTML templates.
- `src/lib/` contains reusable non-UI code such as helpers, services, validators, adapters, and shared support modules.
- `src/lib/auth/auth_config.py` contains auth-specific configuration for the app.
- `src/lib/mcp/` contains the app-owned FastMCP server and nested FastMCP config when MCP is enabled.
- `prisma/` contains the Prisma schema and seed scripts.
- `public/` contains static assets served directly.
- `main.py` is the application entry point.
- `caspian.config.json` is the core feature configuration file.
- `.venv/Lib/site-packages/casp/` contains the installed Caspian framework core.
- `node_modules/caspian-utils/dist/docs/` contains the packaged Caspian docs.

## Example Layout

```text
my-app/
  main.py
  caspian.config.json
  prisma/
    schema.prisma
    seed.ts
  public/
  src/
    app/
      layout.html
      index.py
      index.html
      globals.css
    components/
      Container.py
      ui/
        Button.py
        Button.html
    lib/
      auth/
        auth_config.py
      mcp/
        fastmcp.json
        mcp_server.py
      prisma/
        __init__.py
        db.py
        models.py
  .venv/
    Lib/
      site-packages/
        casp/
  node_modules/
    caspian-utils/
      dist/
        docs/
```

Optional directories such as `src/lib/mcp/` appear only when the relevant feature flag is enabled in `caspian.config.json`.

## Directory Breakdown

### `src/`

This is the main application area. It contains route files, templates, styles, and shared code used across the app.

### `src/app/`

This directory handles file-based routing. Route templates and route-specific backend logic live here.

When authoring a route HTML file such as `src/app/**/index.html`, keep the whole template inside exactly one top-level lowercase HTML element. Treat it the same way you would a React component returning one parent element: wrap the route markup and any owned PulsePoint script in the same root.

Do not handwrite `pp-component="..."` or `type="text/pp"` in those source templates. Author a plain `<script>` inside the root when you need PulsePoint logic, and let Caspian add the runtime attributes during render.

See `routing.md` for the full App Router-style rules for dynamic segments, route groups, and nested layouts.

### `src/components/`

Use this folder for reusable UI components that should be imported into route templates or other component templates.

As the app grows, default to `src/components/` for application-level UI that will be shared across routes or features. Keep page-only markup close to the route in `src/app/`, but move shared cards, forms, shells, navigation, and other reusable visual building blocks into `src/components/`.

The common Caspian pattern is a Python file such as `Button.py` with `@component`, optionally paired with a same-name HTML file such as `Button.html` when the component has richer markup or PulsePoint behavior.

One Python file can also export multiple related `@component` functions. When that happens, import those tags from that exact file path in HTML, for example `<!-- @import { Breadcrumb, BreadcrumbItem, BreadcrumbList } from "../components/Breadcrumb.py" -->`, instead of assuming each tag has its own sibling `.py` file.

For component HTML files, follow the same one-parent rule as route HTML files: one top-level lowercase HTML element only, with no sibling roots and no top-level script sitting outside that root.

Do not handwrite `pp-component="..."` or `type="text/pp"` in component source templates either. Write plain `<script>` inside the single root and let the render pipeline inject the runtime shape.

This workspace's component tooling scans `src/` based on `caspian.config.json`, so `src/components/` is a conventionally clean location, not a hard-coded runtime requirement.

### `src/lib/`

Use this folder for shared helpers, reusable validators, RPC-facing service wrappers, data-access helpers, formatting utilities, and other app-level support code that is not itself a reusable rendered component.

If the code is primarily rendered UI that will be imported as a component tag, prefer `src/components/`. If the code is a helper, service, adapter, validator, or other non-visual support module, prefer `src/lib/`.

This workspace already includes an app-owned Python database layer under `src/lib/prisma/`. Reuse that package for Python-side data access and keep any additional shared database helpers in `src/lib/`.

When MCP is enabled in the current workspace, this folder also contains the app-owned FastMCP server under `src/lib/mcp/`.

### Shared Database Helpers

If your Python routes or RPC actions need reusable database access code, keep that helper layer under `src/lib/` and extend the existing `src/lib/prisma/` package.

In this workspace, Prisma schema and seed files live under `prisma/`, while the Python-side adapter is application-owned code under `src/lib/prisma/`.

### `src/lib/auth/`

Use this folder for authentication-specific project code. The main auth configuration file lives at `src/lib/auth/auth_config.py`.

### `src/lib/mcp/`

Use this folder for app-owned Model Context Protocol server files.

When `caspian.config.json` has `mcp: true`:

- `src/lib/mcp/mcp_server.py` defines `mcp = FastMCP(...)` and the current tool set.
- `src/lib/mcp/fastmcp.json` is the default FastMCP config file for any workspace-defined MCP launcher, such as `npm run mcp` when that script exists.

Keep MCP tool definitions here instead of placing them in route files, `main.py`, or framework internals.

In the current workspace, `mcp: false`, so do not assume this folder exists until the feature is enabled and the update workflow has run.

### `prisma/`

This folder contains your database model definitions in `schema.prisma` and any seed logic such as `seed.ts`.

### `public/`

Store static assets here, including images, fonts, and generated frontend assets that should be served directly.

## Key Files

### `main.py`

The application entry point for the project.

In the current Caspian app shape, this file is where startup wiring happens:

- load environment variables
- call `configure_auth(build_auth_settings())`
- register OAuth providers with `Auth.set_providers(...)`
- create the FastAPI app
- register routes and RPC handlers
- add `SessionMiddleware`, CSRF middleware, auth middleware, and RPC middleware

Use `main.py` for auth bootstrap and middleware-order changes. Use `src/lib/auth/auth_config.py` for auth policy values such as public routes, redirects, and RBAC maps.

### `caspian.config.json`

The core feature configuration file for the application.

AI agents should read this file before making almost any feature-level decision. Use it to confirm which capabilities are enabled, which code generation paths make sense, and which directories should be scanned for components or other project assets.

In the current workspace, `caspian.config.json` shows `backendOnly: false`, `tailwindcss: true`, `mcp: false`, `prisma: true`, `typescript: false`, and `componentScanDirs: ["src"]`.

### `src/lib/auth/auth_config.py`

The project auth configuration file. Use this path when changing authentication behavior.

### `src/lib/mcp/mcp_server.py`

When `caspian.config.json` has `mcp: true`, this is the app-owned FastMCP server module. It exports the `mcp` server instance and should be the default place for workspace MCP tools.

### `src/lib/mcp/fastmcp.json`

When `caspian.config.json` has `mcp: true`, this is the default FastMCP config file for the workspace.

Because this file is nested under `src/lib/mcp/`, direct FastMCP commands should pass the explicit path, for example `fastmcp run src/lib/mcp/fastmcp.json`, unless the launch working directory changes.

### `src/app/layout.html`

The root layout shared across pages.

If this layout imports reusable components, place each `<!-- @import ... -->` comment above the authored root `<html>` element instead of nesting the comment inside `<html>` or `<body>`.

### `src/app/index.py`

The backend logic for the route. This is where route behavior can load first-render data, expose `@rpc()` actions, and import framework features such as auth helpers, `casp.validate`, and route-level `Cache(...)` declarations.

### `src/app/index.html`

The route template. It supports standard HTML, `<!-- @import ... -->` component imports, and PulsePoint directives, and it should be the default place for reactive frontend behavior in Caspian.

Treat import comments as top-of-file directives that belong above the route's single authored root element.

Use `components.md` when the task involves authoring reusable `<MyComponent />` tags backed by Python files.

### `src/app/globals.css`

Global application styles.

### `.venv/Lib/site-packages/casp/`

The installed Caspian framework package. This is framework internals, not normal application code.

Notable internal files include:

- `.venv/Lib/site-packages/casp/rpc.py` for the RPC decorator and related server-action internals
- `.venv/Lib/site-packages/casp/layout.py` for layout rendering, metadata handling, and routing-related internals
- `.venv/Lib/site-packages/casp/cache_handler.py` for route-level cache declarations, cache manifest handling, disk-backed HTML writes, and invalidation internals
- `.venv/Lib/site-packages/casp/validate.py` for direct validators, rule-based validation, sanitization, and file-validation internals

### `node_modules/caspian-utils/dist/docs/`

The packaged Caspian documentation location distributed with the current toolchain.

## AI Routing Notes

If an AI agent is deciding where to make changes, use these rules first.

- Read `caspian.config.json` almost immediately before making feature, tooling, or file-placement decisions. It tells you which Caspian features are enabled in the current workspace.
- Treat `caspian.config.json` as the single source of truth for optional feature enablement. Use feature-specific docs and file paths only when the matching flag is enabled.
- If an optional feature is disabled and the user wants it, ask first, then update `caspian.config.json` and use `npx casp update project` before assuming feature-managed files exist.
- Treat `package.json` scripts as opt-in operations. Do not run `npm run dev` or `npm run build` unless the user explicitly asks, the task genuinely requires that exact script, or deployment prep needs `npm run build`.
- Treat `__pycache__/` directories, `.pyc` files, `public/css/styles.css`, `settings/component-map.json`, and `settings/files-list.json` as generated artifacts when the local stack is intentionally running. They are not authored source files.
- Inspect `settings/component-map.json` and `settings/files-list.json` when you need the generated component or route inventory, but do not hand-edit them. The workspace regenerates them from `settings/component-map.ts` and `settings/files-list.ts`.
- Put route templates and route-specific backend logic in `src/app/`.
- As the app grows, keep route-owned code in `src/app/`, reusable rendered UI in `src/components/`, and reusable non-UI support code in `src/lib/`.
- Check [routing.md](./routing.md) when you need URL segment rules, layout nesting behavior, or dynamic route conventions.
- Put reusable component files in `src/components/` and check [components.md](./components.md) for `@component`, `render_html(__file__)`, import comments, and single-root template rules.
- When deciding between `src/components/` and `src/lib/`, use `src/components/` for anything rendered as reusable UI and `src/lib/` for helpers, services, validators, adapters, and shared business logic.
- Use [mcp.md](./mcp.md) only when `caspian.config.json` enables MCP and the task involves FastMCP tool definitions, nested config discovery, or local MCP commands.
- Keep `<!-- @import ... -->` comments above the single authored root element in route, layout, and component HTML files.
- For route and component HTML files, always emit one top-level lowercase HTML element. Good: one wrapper containing the content and a plain `<script>` when needed. Bad: a wrapper element followed by a sibling top-level `<script>`, or handwritten `pp-component="..."` and `type="text/pp"` attributes in source.
- Put shared helpers and reusable libraries in `src/lib/`.
- Put app-owned FastMCP code in `src/lib/mcp/` only when `caspian.config.json` enables MCP.
- Use PulsePoint conventions in route templates for reactive frontend behavior.
- Use `@rpc()` in route/backend code and `pp.rpc()` in client code for browser-triggered data flows.
- Use `casp.cache_handler` when a route's first-render HTML should be reused safely across requests.
- Use `casp.validate` at route and RPC boundaries, and move reusable validators into `src/lib/` when multiple routes share them.
- Use `database.md` when the task involves Prisma schema changes, migrations, seed logic, or workspace-specific database helper conventions.
- Put authentication configuration in `src/lib/auth/auth_config.py`.
- Put auth bootstrap, session middleware, and provider registration changes in `main.py`.
- Put database models and seed logic in `prisma/`.
- Put static files in `public/`.
- Put entry-point changes in `main.py`.
- Put feature and framework-level project configuration in `caspian.config.json`.
- Treat `.venv/Lib/site-packages/casp/` as framework internals unless the task is specifically about Caspian core behavior.
- Use `.venv/Lib/site-packages/casp/rpc.py` when investigating or documenting Caspian RPC internals.
- Use `.venv/Lib/site-packages/casp/layout.py` when investigating or documenting Caspian routing, layout, or metadata internals.
- Use `.venv/Lib/site-packages/casp/cache_handler.py` when investigating or documenting Caspian page-caching internals.
- Use `.venv/Lib/site-packages/casp/validate.py` when investigating or documenting Caspian validation internals.
- Look in `node_modules/caspian-utils/dist/docs/` when you need the packaged framework docs.

Check [index.md](./index.md) first if you need to choose between local Caspian docs.

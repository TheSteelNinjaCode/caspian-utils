---
title: Project Structure
description: Understand the default Caspian project layout so AI agents place routes, reusable components, `src/lib` helpers, auth code, MCP files, public assets, and database changes in the correct directories. Use when deciding where project files belong.
related:
  title: Related docs
  description: Start with installation for new apps, then use the component guide for reusable UI, the auth guide for bootstrap and session wiring, the MCP guide for server layout, the routing guide to map URLs correctly, and the cache guide when route HTML should be reused safely.
  links:
    - /docs/installation
    - /docs/core-runtime-map
    - /docs/components
    - /docs/auth
    - /docs/mcp
    - /docs/routing
    - /docs/cache
    - /docs/database
    - /docs/file-uploads
    - /docs/index
---

This page explains the default layout of a Caspian application, where Caspian core files live, and which paths AI agents should treat as project code versus framework internals.

Treat it as a framework guide for Caspian projects. Use `caspian.config.json` and the actual repository tree to confirm which optional directories and files exist in the current project.

## Overview

Caspian uses a lean project layout that keeps application code in `src`, database files in `prisma`, static assets in `public`, configuration in `caspian.config.json`, and framework internals in the installed package.

As an app grows, keep reusable rendered UI in `src/components/`, keep reusable non-UI support code in `src/lib/`, and keep route-owned files in `src/app/`. That split keeps page composition separate from shared component and service code.

In that layout, the default stack is Python components for reusable UI, PulsePoint in templates for reactive browser behavior, RPC for CRUD operations and browser-triggered backend reads, and `casp.validate` for input validation at route and action boundaries.

For public pages that can safely reuse rendered HTML, Caspian also supports route-level page caching through `casp.cache_handler`.

Before an AI agent decides which Caspian features are available in a workspace, it should read `./caspian.config.json` almost immediately. That file is the feature gate for project capabilities such as `backendOnly`, `tailwindcss`, `mcp`, `prisma`, `typescript`, `websocket`, and `componentScanDirs`.

Treat `caspian.config.json` as the single source of truth for optional feature enablement. Use feature-specific files and docs only after the matching flag is confirmed as enabled. If a feature is disabled and the user wants it, ask first, then update `caspian.config.json` and follow the update workflow in `commands.md`.

## Top-Level Areas

- `src/` contains routes, page templates, styles, reusable components, and shared libraries.
- `src/components/` contains reusable application UI components and optional same-name HTML templates.
- `src/lib/` contains reusable non-UI code such as helpers, services, validators, adapters, and shared support modules.
- `src/lib/auth/auth_config.py` contains auth-specific configuration for the app.
- `src/lib/mcp/` contains the app-owned FastMCP server and nested FastMCP config when MCP is enabled.
- `src/lib/websocket/` contains reusable socket helpers when WebSockets are enabled and the project includes shared session, auth, connection, or broadcast utilities.
- `prisma/` contains the Prisma schema and seed scripts.
- `public/` contains static assets served directly.
- `settings/` contains BrowserSync, build, restart, and generated-project helper files.
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
  settings/
    bs-config.ts
    build.ts
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
      websocket/
        websocket_security.py
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

Optional directories such as `src/lib/mcp/` and `src/lib/websocket/` appear only when the relevant feature flag is enabled in `caspian.config.json` and the project needs that app-owned surface.

## Directory Breakdown

### `src/`

This is the main application area. It contains route files, templates, styles, and shared code used across the app.

### `src/app/`

This directory handles file-based routing. Route templates and route-specific backend logic live here.

For any route that renders UI, keep that markup in `src/app/**/index.html`. If the route is UI-only, `index.html` alone is enough. Add `src/app/**/index.py` only as a companion when the same route needs metadata, `page()`, `@rpc()` actions, auth checks, caching, redirects, or other server-side behavior. Keep shared wrappers in `layout.html` and use `layout.py` only for shared props or metadata. Use a lone `index.py` only for non-visual routes such as redirect-only or action-only handlers.

Keep backend logic in the owning route when it is route-specific. Move Python code into `src/lib/**` only when it is shared across routes, components, integrations, or features.

When a folder represents a section with child routes, such as `dashboard`, `account`, `settings`, or `docs`, create `layout.html` in that folder and let the child routes live beneath it. See [routing.md](./routing.md) for the canonical section layout pattern.

If the shared wrapper should not add a URL segment, use a parenthesized route-group folder such as `(marketing)/layout.html` instead of a normal folder name. Use a normal folder such as `dashboard/` only when that segment should be part of the public URL.

When authoring route or layout HTML, follow the authoring contract documented in [routing.md](./routing.md) and [pulsepoint.md](./pulsepoint.md): keep one authored root, keep `<!-- @import ... -->` directives above that root, use a plain `<script>` inside the root when needed, and do not handwrite `pp-component` or `type="text/pp"`.

See `routing.md` for the full App Router-style rules for dynamic segments, route groups, and nested layouts.

### `src/components/`

Use this folder for reusable UI components that should be imported into route templates or other component templates.

As the app grows, default to `src/components/` for application-level UI that will be shared across routes or features. Keep page-only markup close to the route in `src/app/`, but move shared cards, forms, shells, navigation, and other reusable visual building blocks into `src/components/`.

For page-specific chunks that are still substantial, a route-local component folder is also acceptable. Use this when a component is owned by one route but deserves its own focused file, such as one tab panel, one settings form, one analytics section, or one table toolbar. The route should still read as a short assembly of named `x-*` tags.

The common Caspian pattern is a Python file such as `Button.py` with `@component`, optionally paired with a same-name HTML file such as `Button.html` when the component has richer markup or PulsePoint behavior. In authored HTML, that component is consumed as `<x-button />`.

One Python file can also export multiple related `@component` functions. When that happens, import those tags from that exact file path in HTML, for example `<!-- @import { Breadcrumb, BreadcrumbItem, BreadcrumbList } from "../components/Breadcrumb.py" -->`, and render them as `<x-breadcrumb />`, `<x-breadcrumb-item />`, and `<x-breadcrumb-list />` instead of assuming each tag has its own sibling `.py` file.

For component HTML files, follow the component authoring rules in [components.md](./components.md): one top-level parent node, no sibling roots, plain `<script>` inside that root when needed, and no handwritten `pp-component` or `type="text/pp"`.

The directories listed in `componentScanDirs` determine where component tooling scans. When that list includes `src/`, `src/components/` is a conventionally clean location, not a hard-coded runtime requirement.

### `src/lib/`

Use this folder for shared helpers, reusable validators, RPC-facing service wrappers, data-access helpers, formatting utilities, and other app-level support code that is not itself a reusable rendered component.

If the code is primarily rendered UI that will be imported as a component tag, prefer `src/components/`. If the code is a helper, service, adapter, validator, or other non-visual support module, prefer `src/lib/`.

For file upload and manager flows, keep route-owned `@rpc()` actions in `src/app/**/index.py` and keep shared storage, naming, filesystem, and Prisma-backed persistence helpers in `src/lib/`.

When a project includes an app-owned Python database layer under `src/lib/prisma/`, reuse that package for Python-side data access and keep any additional shared database helpers in `src/lib/`.

When MCP is enabled for the project, this folder also contains the app-owned FastMCP server under `src/lib/mcp/`.

Do not add `src/lib/security/runtime_security.py` for normal app work. Runtime security helpers for safe public-file serving, production session-secret enforcement, production-safe error messages, fail-closed environment resolution, and baseline response headers including the Content-Security-Policy are package-owned by `casp.runtime_security`.

### Shared Database Helpers

If your Python routes or RPC actions need reusable database access code, keep that helper layer under `src/lib/` and extend the existing `src/lib/prisma/` package.

In a Prisma-enabled Caspian project, schema and seed files typically live under `prisma/`, while the Python-side adapter lives in application-owned code under `src/lib/prisma/` when that layer exists. When `caspian.config.json` has `prisma: true`, Python-side database reads and writes should use that generated Prisma Python ORM instead of a custom fetch, raw driver, JSON active store, or second app-owned database abstraction.

### `src/lib/auth/`

Use this folder for authentication-specific project code. The main auth configuration file lives at `src/lib/auth/auth_config.py`.

### `src/lib/mcp/`

Use this folder for app-owned Model Context Protocol server files.

When `caspian.config.json` has `mcp: true`:

- `src/lib/mcp/mcp_server.py` defines `mcp = FastMCP(...)` and the current tool set.
- `src/lib/mcp/fastmcp.json` is the default FastMCP config file for any workspace-defined MCP launcher, such as `npm run mcp` when that script exists.

Keep MCP tool definitions here instead of placing them in route files, `main.py`, or framework internals.

If `caspian.config.json` has `mcp: false`, do not assume this folder exists until the feature is enabled and the update workflow has run.

### `prisma/`

This folder contains your database model definitions in `schema.prisma` and any seed logic such as `seed.ts`.

### `public/`

Store static assets here, including images, fonts, and generated frontend assets that should be served directly.

The public root maps directly to the URL root. Any existing nested file is
available at the same root-relative URL—for example,
`public/icons/app.png` is served as `/icons/app.png`. Adding a public
subdirectory does not require a matching route or mount in `main.py`.

Runtime-uploaded public blobs can also live here. Confirm the actual upload path in the project code, commonly `public/uploads/`, and keep that directory aligned with any BrowserSync ignore rules.

If the upload directory is created only at runtime, create it on demand in the owning upload helper instead of assuming it is already committed.

Public means browser-accessible, not automatically safe to render inline.
`PublicFilesMiddleware.inline_safe_subdirectories` must include every top-level
public directory that receives untrusted runtime uploads. Unsafe types in a
configured upload directory, notably HTML and SVG, are sent as attachments with
`nosniff`; trusted first-party assets are served inline. The standard
`public/uploads/**` location is protected only when `main.py` configures the
top-level `uploads` entry.

If the local BrowserSync stack is running, keep that upload directory in `settings/bs-config.ts` `PUBLIC_IGNORE_DIRS` so new uploads do not force a full browser reload.

## Key Files

### `main.py`

The application entry point for the project.

In the current Caspian app shape, this file is where startup wiring happens:

- load environment variables
- call `configure_auth(build_auth_settings())`
- register OAuth providers with `Auth.set_providers(...)`
- create the FastAPI app
- register routes and RPC handlers
- add response headers middleware, `SessionMiddleware`, CSRF middleware, auth middleware, and RPC middleware

Use `main.py` for auth bootstrap and middleware-order changes. Use `src/lib/auth/auth_config.py` for auth policy values such as public routes, redirects, and RBAC maps.

### `.venv/Lib/site-packages/casp/runtime_security.py`

The installed Caspian package owns runtime security helpers that `main.py` can import as `casp.runtime_security`. Keep this file in the package instead of copying it into `src/lib/`, because normal app users should not need to edit it.

This helper is not a registry for third-party browser resources. Do not add Google, YouTube, CDN, image-host, API, or iframe origins here just because the browser or server uses them. If an app later chooses to enforce a Content Security Policy, document and implement that as an explicit project policy rather than assuming `runtime_security.py` owns a domain allowlist.

Use this package module when the task is about:

- safe serving of files from `public/**`
- production session-secret enforcement
- user-facing vs production-safe exception messaging helpers
- baseline response headers such as permissions, referrer, MIME sniffing, framing, or HSTS behavior

Because this file is framework-owned, edit it only for Caspian runtime work or when documentation must match the installed package. App-specific auth policy still belongs in `src/lib/auth/auth_config.py`, and app-specific upload or storage behavior should live in route-owned code or other `src/lib/**` helpers.

`PublicFilesMiddleware` serves only existing `GET`/`HEAD` files that resolve
inside the configured public root and falls through for every other request.
Install it inside the security-header layer and outside rate limiting, sessions,
CSRF, auth, RPC, and page routing. Preserve a restricted inline-media policy for
user-upload directories so executable uploads such as HTML and SVG download
instead of rendering with the application's origin.

### `caspian.config.json`

The core feature configuration file for the application.

AI agents should read this file before making almost any feature-level decision. Use it to confirm which capabilities are enabled, which code generation paths make sense, and which directories should be scanned for components or other project assets.

Read the actual values in `caspian.config.json` instead of inferring feature state from this doc.

### `settings/bs-config.ts`

The BrowserSync watcher configuration for the local stack lives here.

When runtime uploads write into `public/uploads/`, keep the public-root-relative entry `uploads` in `PUBLIC_IGNORE_DIRS`.

`settings/bs-config.ts` matches ignore entries against paths relative to the `public/` root, so nested uploaded files do not trigger reloads.

### `settings/bs-config.json`

When the local BrowserSync stack is running, this generated file is the active URL snapshot.

Use it when AI needs the current local, external, or UI URL instead of the configured defaults, and use it when tracing the development cookie scope logic in `main.py` because `_dev_cookie_scope()` can read the active port from this file.

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

Nested section layouts follow the same pattern in child folders such as `src/app/dashboard/layout.html` or `src/app/(marketing)/layout.html`. Use those nested layouts for shared dashboard shells, grouped page wrappers, sidebars, headers, or section-level metadata defaults.

Keep visible wrapper markup in `layout.html`, not in `layout.py`.

### `src/app/layout.py`

The backend companion for a layout. Use this file for shared props, metadata defaults, and other server-side preparation for the sibling `layout.html`.

Do not store layout HTML in `layout.py`. Keep the authored wrapper in `layout.html` and let `layout.py` return props or metadata.

### `src/app/index.py`

The backend logic companion for the route. Use this file when the same route needs first-render data, metadata, `@rpc()` actions, auth helpers, `casp.validate`, route-level `Cache(...)` declarations, redirects, or other server behavior.

If the route renders UI, keep the markup in the sibling `index.html` and let `index.py` call `render_page(__file__, ...)` with whatever context the template needs. Do not store route HTML in `index.py`. A lone `index.py` should be reserved for non-visual routes.

When a route owns a file manager or upload UI, keep the owning upload and delete `@rpc()` actions in that route's `index.py` and move reusable filesystem or Prisma helpers into `src/lib/`. Do not move ordinary upload behavior into `main.py`.

### `src/app/index.html`

The route template. It supports standard HTML, `<!-- @import ... -->` component imports, and PulsePoint directives, and it should be the default place for reactive frontend behavior in Caspian.

If a route renders UI and needs no backend behavior, this file alone is sufficient. If the route also has an `index.py` companion, keep the visible page markup here.

Treat import comments as top-of-file directives that belong above the route's single authored root element.

That authored root may be a native HTML element or a single imported `x-*` component tag, but after component expansion it must still resolve to one final HTML root.

Use `components.md` when the task involves authoring reusable `<x-my-component />` tags backed by Python files.

### `src/app/globals.css`

Global application styles.

### `.venv/Lib/site-packages/casp/`

The installed Caspian framework package. This is framework internals, not normal application code.

Use [core-runtime-map.md](./core-runtime-map.md) when you need the fastest concern-to-file lookup across this directory.

Notable internal files include:

- `.venv/Lib/site-packages/casp/rpc.py` for the RPC decorator and related server-action internals
- `.venv/Lib/site-packages/casp/layout.py` for layout rendering, metadata handling, and routing-related internals
- `.venv/Lib/site-packages/casp/cache_handler.py` for route-level cache declarations, cache manifest handling, disk-backed HTML writes, and invalidation internals
- `.venv/Lib/site-packages/casp/validate.py` for direct validators, rule-based validation, sanitization, and file-validation internals
- `.venv/Lib/site-packages/casp/auth.py` for auth settings, route-protection checks, and OAuth internals
- `.venv/Lib/site-packages/casp/state_manager.py` for request-scoped server state and persistence caveats
- `.venv/Lib/site-packages/casp/component_decorator.py` for `@component`, `render_html(...)`, and component loading
- `.venv/Lib/site-packages/casp/components_compiler.py` for `@import` parsing, `x-*` resolution, root validation, and `pp-component` injection
- `.venv/Lib/site-packages/casp/scripts_type.py` for converting authored `<script>` tags to `type="text/pp"`
- `.venv/Lib/site-packages/casp/caspian_config.py` for config loading and route inventory parsing
- `.venv/Lib/site-packages/casp/streaming.py` for `SSE` and streamed response helpers

### `node_modules/caspian-utils/dist/docs/`

The packaged Caspian documentation location distributed with the current toolchain.

## AI Retrieval Notes

If an AI agent is deciding where to make changes, use these rules first.

- Read `caspian.config.json` almost immediately before making feature, tooling, or file-placement decisions. It tells you which Caspian features are enabled in the current project.
- Treat `caspian.config.json` as the single source of truth for optional feature enablement. Use feature-specific docs and file paths only when the matching flag is enabled.
- If an optional feature is disabled and the user wants it, ask first, then update `caspian.config.json` and use `npx casp update project` before assuming feature-managed files exist.
- Treat `package.json` scripts as opt-in operations. Do not run `npm run dev` or `npm run build` unless the user explicitly asks, the task genuinely requires that exact script, or deployment prep needs `npm run build`.
- Treat `__pycache__/` directories, `.pyc` files, `public/css/styles.css`, `settings/component-map.json`, and `settings/files-list.json` as generated artifacts when the local stack is intentionally running. They are not authored source files.
- Inspect `settings/component-map.json` and `settings/files-list.json` when you need the generated component or route inventory, but do not hand-edit them. The workspace regenerates them from `settings/component-map.ts` and `settings/files-list.ts`.
- Put route templates and route-specific backend logic in `src/app/`.
- Put only genuinely shared helpers, services, adapters, and validation logic in `src/lib/`.
- As the app grows, keep route-owned code in `src/app/`, reusable rendered UI in `src/components/`, and reusable non-UI support code in `src/lib/`.
- When the user asks for a dashboard, admin area, account area, or any grouped set of child routes, create a parent folder in `src/app/` with `layout.html` and place the child routes beneath it. Use `(group)/layout.html` only when that parent should not appear in the URL.
- Read [file-uploads.md](./file-uploads.md) when the task involves upload widgets, media libraries, or file-manager flows.
- Check [routing.md](./routing.md) when you need URL segment rules, layout nesting behavior, or dynamic route conventions.
- Put reusable component files in `src/components/` and check [components.md](./components.md) for `@component`, `render_html(__file__)`, `x-*` component tags, import comments, and single-root template rules.
- Split UI by component responsibility before files become large. A tab panel, settings form, table section, toolbar, or repeated card/list item can be its own component even when it is only used by one route.
- When deciding between `src/components/` and `src/lib/`, use `src/components/` for anything rendered as reusable UI and `src/lib/` for helpers, services, validators, adapters, and shared business logic.
- Use [mcp.md](./mcp.md) only when `caspian.config.json` enables MCP and the task involves FastMCP tool definitions, nested config discovery, or local MCP commands.
- Keep `<!-- @import ... -->` comments above the single authored root element in route, layout, and component HTML files. Do not nest them inside `<html>`, `<body>`, `<section>`, or any other parent wrapper.
- Follow the single-root authoring contract in [routing.md](./routing.md), [components.md](./components.md), and [pulsepoint.md](./pulsepoint.md): one authored root, any owned `<script>` inside that root, and no handwritten `pp-component` or `type="text/pp"` in source templates.
- Put shared helpers and reusable libraries in `src/lib/`.
- Use `settings/bs-config.ts` when uploaded public assets should not trigger BrowserSync reloads during the local stack.
- Put app-owned FastMCP code in `src/lib/mcp/` only when `caspian.config.json` enables MCP.
- Use PulsePoint conventions in route templates for reactive frontend behavior.
- Use `@rpc()` in route/backend code and `pp.rpc()` in client code for browser-triggered data flows, especially CRUD operations and interactive backend reads.
- Use `casp.cache_handler` when a route's first-render HTML should be reused safely across requests.
- Use `casp.validate` at route and RPC boundaries, and move reusable validators into `src/lib/` when multiple routes share them.
- Use `database.md` when the task involves Prisma schema changes, migrations, seed logic, or project-specific database helper conventions.
- Put authentication configuration in `src/lib/auth/auth_config.py`.
- Put auth bootstrap, session middleware, and provider registration changes in `main.py`.
- Put database models and seed logic in `prisma/`.
- Put static files in `public/`.
- Put entry-point changes in `main.py`.
- Put feature and framework-level project configuration in `caspian.config.json`.
- Read [core-runtime-map.md](./core-runtime-map.md) when the task touches `main.py` or installed `casp` internals and the controlling file is not obvious yet.
- Treat `.venv/Lib/site-packages/casp/` as framework internals unless the task is specifically about Caspian core behavior.
- Use `.venv/Lib/site-packages/casp/rpc.py` when investigating or documenting Caspian RPC internals.
- Use `.venv/Lib/site-packages/casp/layout.py` when investigating or documenting Caspian routing, layout, or metadata internals.
- Use `.venv/Lib/site-packages/casp/cache_handler.py` when investigating or documenting Caspian page-caching internals.
- Use `.venv/Lib/site-packages/casp/validate.py` when investigating or documenting Caspian validation internals.
- Look in `node_modules/caspian-utils/dist/docs/` when you need the packaged framework docs.

Check [index.md](./index.md) first if you need to choose between local Caspian docs.

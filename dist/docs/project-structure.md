---
title: Project Structure
description: Understand the default Caspian project layout so AI agents place files in the correct directories before generating PulsePoint templates, RPC actions, validation helpers, auth code, configuration, or database changes.
related:
  title: Related docs
  description: Start with installation for new apps, then use the auth guide for bootstrap and session wiring, the routing guide to map URLs correctly, and the cache guide when route HTML should be reused safely.
  links:
    - /docs/installation
    - /docs/auth
    - /docs/routing
    - /docs/cache
    - /docs/database
    - /docs/index
---

This page explains the default layout of a Caspian application, where Caspian core files live, and which paths AI agents should treat as project code versus framework internals.

## Overview

Caspian uses a lean project layout that keeps application code in `src`, database files in `prisma`, static assets in `public`, configuration in `caspian.config.json`, and framework internals in the installed package.

In that layout, the default stack is PulsePoint in route templates, RPC for browser-triggered server calls, and `casp.validate` for input validation at route and action boundaries.

For public pages that can safely reuse rendered HTML, Caspian also supports route-level page caching through `casp.cache_handler`.

## Top-Level Areas

- `src/` contains routes, page templates, styles, and shared libraries.
- `src/lib/auth/auth_config.py` contains auth-specific configuration for the app.
- `prisma/` contains the Prisma schema and seed scripts.
- `public/` contains static assets served directly.
- `main.py` is the application entry point.
- `caspian.config.json` is the core feature configuration file.
- `.venv/Lib/site-packages/casp/` contains the installed Caspian framework core.
- `node_modules/caspian/dist/docs/` contains the packaged Caspian docs.

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
    lib/
      auth/
        auth_config.py
      prisma/
        __init__.py
        db.py
        models.py
  .venv/
    Lib/
      site-packages/
        casp/
  node_modules/
    caspian/
      dist/
        docs/
```

## Directory Breakdown

### `src/`

This is the main application area. It contains route files, templates, styles, and shared code used across the app.

### `src/app/`

This directory handles file-based routing. Route templates and route-specific backend logic live here.

See `routing.md` for the full App Router-style rules for dynamic segments, route groups, and nested layouts.

### `src/lib/`

Use this folder for shared helpers, reusable validators, RPC-facing service wrappers, reusable UI utilities, and app-level support code.

When Prisma ORM is enabled, this folder also contains the shared database client imports under `src/lib/prisma/`.

### `src/lib/prisma/`

When a Caspian app includes Prisma support, this folder contains the shared ORM imports used by route code and RPC actions.

Typical files include:

- `src/lib/prisma/db.py` for the shared Prisma client instance
- `src/lib/prisma/models.py` for generated Python model types
- `src/lib/prisma/__init__.py` for package-level re-exports such as `prisma` and `models`

See `database.md` for the Prisma-specific workflow, generation step, and usage patterns.

### `src/lib/auth/`

Use this folder for authentication-specific project code. The main auth configuration file lives at `src/lib/auth/auth_config.py`.

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

### `src/lib/auth/auth_config.py`

The project auth configuration file. Use this path when changing authentication behavior.

### `src/app/layout.html`

The root layout shared across pages. Author it like a layout component with one top-level wrapper around `[[ children | safe ]]`; at the app root, that wrapper is usually `<html>`.

### `src/app/index.py`

The backend logic for the route. This is where route behavior can load first-render data, expose `@rpc()` actions, and import framework features such as auth helpers, `casp.validate`, and route-level `Cache(...)` declarations.

### `src/app/index.html`

The route template. It supports standard HTML, PulsePoint directives, and component imports, and it should be the default place for reactive frontend behavior in Caspian. Keep it focused on page content and move shared wrappers into `layout.html`.

### `src/app/globals.css`

Global application styles.

### `.venv/Lib/site-packages/casp/`

The installed Caspian framework package. This is framework internals, not normal application code.

Notable internal files include:

- `.venv/Lib/site-packages/casp/rpc.py` for the RPC decorator and related server-action internals
- `.venv/Lib/site-packages/casp/layout.py` for layout rendering, metadata handling, and routing-related internals
- `.venv/Lib/site-packages/casp/cache_handler.py` for route-level cache declarations, cache manifest handling, disk-backed HTML writes, and invalidation internals
- `.venv/Lib/site-packages/casp/validate.py` for direct validators, rule-based validation, sanitization, and file-validation internals

### `node_modules/caspian/dist/docs/`

The packaged Caspian documentation location distributed with the framework.

## AI Routing Notes

If an AI agent is deciding where to make changes, use these rules first.

- Put route templates and route-specific backend logic in `src/app/`.
- Check [routing.md](./routing.md) when you need URL segment rules, layout nesting behavior, or dynamic route conventions.
- Put shared helpers and reusable libraries in `src/lib/`.
- Use PulsePoint conventions in route templates for reactive frontend behavior.
- Use `@rpc()` in route/backend code and `pp.rpc()` in client code for browser-triggered data flows.
- Use `casp.cache_handler` when a route's first-render HTML should be reused safely across requests.
- Use `casp.validate` at route and RPC boundaries, and move reusable validators into `src/lib/` when multiple routes share them.
- Use `database.md` when the task involves Prisma schema changes, generated client usage, or the shared files under `src/lib/prisma/`.
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
- Look in `node_modules/caspian/dist/docs/` when you need the packaged framework docs.

Check [index.md](./index.md) first if you need to choose between local Caspian docs.

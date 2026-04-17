---
title: Project Structure
description: Understand the default Caspian project layout so AI agents place files in the correct directories before generating routes, templates, auth code, configuration, or database changes.
related:
  title: Related docs
  description: Start with installation for new apps, then use the routing guide to map URLs to files correctly.
  links:
    - /docs/installation
    - /docs/routing
    - /docs/index
---

This page explains the default layout of a Caspian application, where Caspian core files live, and which paths AI agents should treat as project code versus framework internals.

## Overview

Caspian uses a lean project layout that keeps application code in `src`, database files in `prisma`, static assets in `public`, configuration in `caspian.config.json`, and framework internals in the installed package.

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

Use this folder for shared helpers, reusable UI utilities, and app-level support code.

### `src/lib/auth/`

Use this folder for authentication-specific project code. The main auth configuration file lives at `src/lib/auth/auth_config.py`.

### `prisma/`

This folder contains your database model definitions in `schema.prisma` and any seed logic such as `seed.ts`.

### `public/`

Store static assets here, including images, fonts, and generated frontend assets that should be served directly.

## Key Files

### `main.py`

The application entry point for the project.

### `caspian.config.json`

The core feature configuration file for the application.

### `src/lib/auth/auth_config.py`

The project auth configuration file. Use this path when changing authentication behavior.

### `src/app/layout.html`

The root layout shared across pages.

### `src/app/index.py`

The backend logic for the route. This is where route behavior can import framework features such as RPC and auth helpers from `casp`.

### `src/app/index.html`

The route template. It supports standard HTML, PulsePoint directives, and component imports.

### `src/globals.css`

Global application styles.

### `.venv/Lib/site-packages/casp/`

The installed Caspian framework package. This is framework internals, not normal application code.

Notable internal files include:

- `.venv/Lib/site-packages/casp/rpc.py` for the RPC decorator and related server-action internals
- `.venv/Lib/site-packages/casp/layout.py` for layout rendering, metadata handling, and routing-related internals
- `.venv/Lib/site-packages/casp/validate.py` for direct validators, rule-based validation, sanitization, and file-validation internals

### `node_modules/caspian/dist/docs/`

The packaged Caspian documentation location distributed with the framework.

## AI Routing Notes

If an AI agent is deciding where to make changes, use these rules first.

- Put route templates and route-specific backend logic in `src/app/`.
- Check [routing.md](./routing.md) when you need URL segment rules, layout nesting behavior, or dynamic route conventions.
- Put shared helpers and reusable libraries in `src/lib/`.
- Put authentication configuration in `src/lib/auth/auth_config.py`.
- Put database models and seed logic in `prisma/`.
- Put static files in `public/`.
- Put entry-point changes in `main.py`.
- Put feature and framework-level project configuration in `caspian.config.json`.
- Treat `.venv/Lib/site-packages/casp/` as framework internals unless the task is specifically about Caspian core behavior.
- Use `.venv/Lib/site-packages/casp/rpc.py` when investigating or documenting Caspian RPC internals.
- Use `.venv/Lib/site-packages/casp/layout.py` when investigating or documenting Caspian routing, layout, or metadata internals.
- Use `.venv/Lib/site-packages/casp/validate.py` when investigating or documenting Caspian validation internals.
- Look in `node_modules/caspian/dist/docs/` when you need the packaged framework docs.

Check [index.md](./index.md) first if you need to choose between local Caspian docs.

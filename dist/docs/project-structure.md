---
title: Project Structure
description: Understand the default Caspian project layout so AI agents place files in the correct directories before generating routes, templates, shared libraries, or database code.
related:
  title: Related docs
  description: Start with installation for new apps or return to the local docs index to choose the right Caspian guide.
  links:
    - /docs/installation
    - /docs/index
---

# Project Structure

This page explains the default layout of a Caspian application and where each major kind of code should live.

## Overview

Caspian uses a lean project layout that keeps application code in `src`, database files in `prisma`, and static assets in `public`, while framework utilities come from the installed Caspian package.

## Top-Level Areas

- `src/` contains routes, page templates, styles, and shared libraries.
- `prisma/` contains the Prisma schema and seed scripts.
- `public/` contains static assets served directly.
- `main.py` is the application entry point and the place for auth configuration.
- `caspian.config.json` contains project-level configuration.

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
```

## Directory Breakdown

### `src/`

This is the main application area. It contains route files, templates, styles, and shared code used across the app.

### `src/app/`

This directory handles file-based routing. Route templates and route-specific backend logic live here.

### `src/lib/`

Use this folder for shared helpers, reusable UI utilities, and app-level support code.

### `prisma/`

This folder contains your database model definitions in `schema.prisma` and any seed logic such as `seed.ts`.

### `public/`

Store static assets here, including images, fonts, and generated frontend assets that should be served directly.

## Key Files

### `main.py`

The application entry point. Caspian auth configuration belongs here.

### `caspian.config.json`

The central project configuration file for the application.

### `src/app/layout.html`

The root layout shared across pages.

### `src/app/index.py`

The backend logic for the route. This is where route behavior can import framework features such as RPC and auth helpers from `casp`.

### `src/app/index.html`

The route template. It supports standard HTML, PulsePoint directives, and component imports.

### `src/globals.css`

Global application styles.

## AI Routing Notes

If an AI agent is deciding where to make changes, use these rules first.

- Put route templates and route-specific backend logic in `src/app/`.
- Put shared helpers and reusable libraries in `src/lib/`.
- Put database models and seed logic in `prisma/`.
- Put static files in `public/`.
- Put entry-point and auth setup changes in `main.py`.

Check [index.md](./index.md) first if you need to choose between local Caspian docs.
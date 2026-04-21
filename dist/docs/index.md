---
title: Caspian Docs
description: Caspian documentation with AI-aware routing to the right local docs before framework-specific code generation or project setup changes, with components, PulsePoint, RPC, and Validate as the default app stack.
related:
  title: Next Steps
  description: Start with installation, review the CLI commands, then use the structure and component guides to place Caspian files correctly.
  links:
    - /docs/installation
    - /docs/commands
    - /docs/project-structure
    - /docs/components
---

This directory contains the local Caspian documentation set for quick reference and AI-aware routing.

Before making feature, tooling, or file-placement decisions in a Caspian workspace, read `./caspian.config.json` almost immediately. That file is the project feature gate and tells you which capabilities are enabled, such as Prisma, MCP, TypeScript, Tailwind, backend-only mode, and component scan directories.

## Default Stack

When generating or editing a Caspian app, treat these as the default choices unless the task explicitly requires something else:

- Use PulsePoint for reactive frontend behavior.
- Use `@rpc()` plus `pp.rpc()` for browser-triggered reads, writes, streaming, and uploads.
- Use `Validate` and `Rule` from `casp.validate` for server-side input validation and sanitization.

## Docs Location

The local AI-aware docs for this workspace live here:

- `node_modules/caspian-utils/dist/docs/`

The packaged Caspian docs distributed by the current toolchain also live here:

- `node_modules/caspian-utils/dist/docs/`

## Available Documents

- `installation.md` - First-time setup flow for creating a new Caspian application
- `commands.md` - Main Caspian CLI workflows for project creation, starter kits, updates, and Prisma or ORM-aware maintenance
- `database.md` - Prisma schema, migration, seed, and client-generation workflow for the current workspace, plus Python-side helper caveats
- `auth.md` - Session-backed authentication with `casp.auth`, centralized `auth_config.py`, decorators, RBAC, and OAuth provider helpers
- `components.md` - Create reusable Python components, template-backed UI, JSX-style imports, and the single-parent root rule for component HTML files
- `pulsepoint.md` - Default reactive frontend runtime contract for component scripts, state, effects, directives, and client-side behaviors
- `fetch-data.md` - Initial server-side data loading and browser-triggered RPC flows with `pp.rpc()`, streaming, uploads, and auth-aware actions
- `state.md` - Request-scoped server state with `StateManager`, session-backed JSON persistence, and listener callbacks for transient flows
- `cache.md` - Route-level HTML caching with `Cache`, `CacheHandler`, TTL behavior, file-system storage, and invalidation patterns
- `validation.md` - Input validation and sanitization with `Validate`, `Rule`, direct field checks, and multi-rule workflows for routes and RPC actions
- `metadata.md` - Static and dynamic metadata, SEO inheritance, and Open Graph or Twitter card tags
- `routing.md` - Next.js App Router-style file-based routing with `src/app`, dynamic segments, route groups, nested layouts, and single-root route templates
- `project-structure.md` - Default Caspian layout and where routes, templates, shared code, and database files belong

## AI Awareness Notes

If an AI tool needs Caspian project documentation, start with this directory and use this file as the manifest.

Preferred lookup order:

1. Read `node_modules/caspian-utils/dist/docs/index.md` to discover available local docs.
2. Read `./caspian.config.json` before making any feature assumption. Use it to confirm which project capabilities are enabled and which directories or tooling rules apply.
3. Read `database.md` for Prisma setup and ORM usage, `auth.md` for session auth and route protection, `components.md` for reusable UI authoring, `pulsepoint.md` for browser-side reactive runtime behavior, `fetch-data.md` for data flows, `state.md` for transient request-scoped state, `cache.md` for page caching, and `validation.md` for input boundaries.
4. For authored component, route, and layout HTML files, generate exactly one top-level lowercase HTML element. Think React-style single parent wrapper: keep any owned PulsePoint logic in a plain `<script>` inside that root, do not handwrite `pp-component="..."` or `type="text/pp"` because Caspian injects those runtime attributes automatically, and treat `<!-- @import ... -->` comments as file-level directives that belong above the authored root element instead of inside `<div>`, `<section>`, or `<html>`. When several component tags come from one Python file, import them from that exact file path, for example `<!-- @import { Breadcrumb, BreadcrumbItem, BreadcrumbList } from "../components/Breadcrumb.py" -->`.
5. Prefer local docs before generating code, commands, or migration guidance.
6. Check `node_modules/caspian-utils/dist/docs/` for packaged Caspian docs when local docs need more detail.
7. Only fall back to upstream documentation when local and packaged markdown do not cover the topic.

## Maintenance

When adding more pages, keep this index updated with:

- The filename
- A short description of what the page covers

---
title: Caspian Docs
description: Caspian documentation with AI-aware routing to the right local docs before framework-specific code generation or project setup changes, with components, PulsePoint, RPC, and Validate as the default app stack.
related:
  title: Next Steps
  description: Start with installation, review the CLI commands, then use the structure and component guides to place Caspian files correctly.
  links:
    - /docs/installation
    - /docs/commands
    - /docs/mcp
    - /docs/project-structure
    - /docs/components
---

This directory contains the local Caspian documentation set for quick reference and AI-aware routing.

Before making feature, tooling, or file-placement decisions in a Caspian workspace, read `./caspian.config.json` almost immediately. That file is the project feature gate and tells you which capabilities are enabled, such as Prisma, MCP, TypeScript, Tailwind, backend-only mode, and component scan directories.

Treat `caspian.config.json` as the single source of truth for optional feature enablement. Use feature-specific docs only after the matching flag is confirmed as enabled. If a feature is disabled and the user wants it, ask whether they want to enable it first, then follow the update workflow in `commands.md`.

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
- `commands.md` - Main Caspian CLI workflows for project creation, generation, updates, and config-aware maintenance
- `mcp.md` - MCP-specific layout, launch flow, and AI routing rules for workspaces where `caspian.config.json` enables MCP
- `database.md` - Prisma schema, migration, seed, and client-generation workflow for workspaces where `caspian.config.json` enables Prisma, plus Python-side helper caveats
- `auth.md` - Session-backed authentication with `casp.auth`, centralized `auth_config.py`, public-vs-private route mode guidance, RPC-first signout guidance, RBAC, and OAuth provider helpers
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
3. Treat `caspian.config.json` as the single source of truth for optional features. Use feature-specific docs only when the matching flag is enabled. If a feature is disabled and the user wants it, ask first, then update `caspian.config.json` and follow the update workflow in `commands.md`.
4. Read `commands.md` before running any `package.json` script. In this workspace, npm scripts are opt-in operational commands, not default validation steps. Do not run them unless the user explicitly asks, the task genuinely requires that exact script, or deployment preparation needs `npm run build`.
5. Treat generated outputs such as `public/css/styles.css`, `settings/component-map.json`, `settings/files-list.json`, `__pycache__/`, and `.pyc` files as framework-managed artifacts when the local stack is intentionally running. They are not authored source files and should not be kept in the final diff unless the task explicitly requires them.
6. Analyze `settings/component-map.json` and `settings/files-list.json` when you need the current generated view of components or routes, but do not hand-edit them. The framework regenerates them through `settings/component-map.ts` and `settings/files-list.ts` when the dev or build pipeline intentionally runs.
7. Read `database.md` only when `caspian.config.json` enables Prisma, read `mcp.md` only when `caspian.config.json` enables MCP, and use `auth.md`, `components.md`, `pulsepoint.md`, `fetch-data.md`, `state.md`, `cache.md`, and `validation.md` as the next routing docs for those non-flagged concerns.
8. For authored component, route, and layout HTML files, generate exactly one top-level lowercase HTML element. Think React-style single parent wrapper: keep any owned PulsePoint logic in a plain `<script>` inside that root, do not handwrite `pp-component="..."` or `type="text/pp"` because Caspian injects those runtime attributes automatically, and treat `<!-- @import ... -->` comments as file-level directives that belong above the authored root element instead of inside `<div>`, `<section>`, or `<html>`. When several component tags come from one Python file, import them from that exact file path, for example `<!-- @import { Breadcrumb, BreadcrumbItem, BreadcrumbList } from "../components/Breadcrumb.py" -->`.
9. Prefer local docs before generating code, commands, or migration guidance.
10. Check `node_modules/caspian-utils/dist/docs/` for packaged Caspian docs when local docs need more detail.
11. Only fall back to upstream documentation when local and packaged markdown do not cover the topic.

## Maintenance

When adding more pages, keep this index updated with:

- The filename
- A short description of what the page covers

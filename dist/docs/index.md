---
title: Caspian Docs
description: Packaged Caspian documentation manifest for AI task routing, feature discovery, and file placement. Use this page when deciding which Caspian doc to read first or which feature guide applies.
related:
  title: Next Steps
  description: Start with installation, review the CLI commands, then use the structure and component guides to place Caspian files correctly.
  links:
    - /docs/installation
    - /docs/commands
    - /docs/mcp
    - /docs/file-uploads
    - /docs/project-structure
    - /docs/components
---

This directory contains the packaged Caspian documentation set for AI-aware feature discovery, task routing, and file-placement guidance.

Treat these docs as reusable Caspian feature guidance. Treat `./caspian.config.json` as the single source of truth for which optional features are enabled in the project being analyzed.

The docs can mention optional features even when those features are disabled in a project. Their job is to explain how a feature works, when a doc applies, and which files to inspect next once the feature is confirmed as relevant.

Before making feature, tooling, or file-placement decisions in a Caspian project, read `./caspian.config.json` almost immediately. That file tells you which optional capabilities are enabled, such as Prisma, MCP, TypeScript, Tailwind, backend-only mode, and component scan directories.

## Default Stack

When generating or editing a Caspian app, treat these as the default choices unless the task explicitly requires something else:

- Use PulsePoint for reactive frontend behavior.
- When `caspian.config.json` has `tailwindcss: true`, use Python `merge_classes(...)` plus browser `twMerge(...)` as the only supported Tailwind class-merging path.
- Use `@rpc()` plus `pp.rpc()` for browser-triggered reads, writes, streaming, and uploads.
- Use `Validate` and `Rule` from `casp.validate` for server-side input validation and sanitization.

## Docs Location

The packaged Caspian docs referenced by this index live here:

- `node_modules/caspian-utils/dist/docs/`

## Available Documents

- `installation.md` - First-time setup flow for creating a new Caspian application
- `commands.md` - Main Caspian CLI workflows for project creation, generation, updates, and config-aware maintenance
- `mcp.md` - MCP-specific layout, launch flow, and AI routing rules for projects where `caspian.config.json` enables MCP
- `database.md` - Prisma schema, migration, seed, and client-generation workflow for projects where `caspian.config.json` enables Prisma, plus Python-side helper caveats
- `auth.md` - Session-backed authentication with `casp.auth`, centralized `auth_config.py`, public-vs-private route mode guidance, RPC-first signout guidance, RBAC, and OAuth provider helpers
- `components.md` - Create reusable Python components, template-backed UI, HTML-first `x-*` component tags, the single-parent root rule for component HTML files, and the Python-side `merge_classes(...)` contract when Tailwind CSS is enabled
- `pulsepoint.md` - Default reactive frontend runtime contract for component scripts, state, effects, directives, client-side behaviors, and direct browser `twMerge(...)` usage when Tailwind CSS is enabled
- `fetch-data.md` - Initial server-side data loading and browser-triggered RPC flows with `pp.rpc()`, streaming, uploads, and auth-aware actions
- `file-uploads.md` - Route-local file uploads and file-manager flows with `@rpc()`, `pp.rpc()`, Prisma metadata, public asset storage, and BrowserSync ignore rules
- `state.md` - Request-scoped server state with `StateManager`, session-backed JSON persistence, and listener callbacks for transient flows
- `cache.md` - Route-level HTML caching with `Cache`, `CacheHandler`, TTL behavior, file-system storage, and invalidation patterns
- `validation.md` - Input validation and sanitization with `Validate`, `Rule`, direct field checks, and multi-rule workflows for routes and RPC actions
- `metadata.md` - Static and dynamic metadata, SEO inheritance, and Open Graph or Twitter card tags
- `routing.md` - Next.js App Router-style file-based routing with `src/app`, dynamic segments, route groups, nested layouts, and single-root route templates
- `project-structure.md` - Default Caspian layout and where route files, reusable UI in `src/components/`, reusable non-UI code in `src/lib/`, and database files belong

## AI Retrieval Notes

If an AI tool needs Caspian documentation, start with this directory and use this file as the manifest.

Preferred lookup order:

1. Read `node_modules/caspian-utils/dist/docs/index.md` to discover available local docs.
2. Read `./caspian.config.json` before making any feature assumption. A doc existing in the package does not mean that feature is enabled in the current project.
3. Treat `caspian.config.json` as the single source of truth for optional features. Use feature-specific docs only when the matching flag is enabled. If a feature is disabled and the user wants it, ask first, then update `caspian.config.json` and follow the update workflow in `commands.md`.
4. After the feature is confirmed, inspect the actual project files that decide behavior, such as `package.json`, `main.py`, `src/app/**`, `src/lib/**`, `settings/**`, `prisma/**`, and the installed `casp` runtime.
5. Use `commands.md` for scaffold and update workflows, `project-structure.md` for placement decisions, and the feature docs such as `mcp.md`, `database.md`, `auth.md`, `fetch-data.md`, and `file-uploads.md` for task-specific guidance.
6. Prefer packaged Caspian docs before upstream documentation when generating code, commands, or migration guidance.
7. Keep `index.md` and cross-links aligned so AI can quickly discover the right doc.

## Maintenance

When adding more pages, keep this index updated with:

- The filename
- A short description of what the page covers

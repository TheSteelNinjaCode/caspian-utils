---
title: Caspian Docs
description: Caspian documentation with AI-aware routing to the right local docs before framework-specific code generation or project setup changes, with PulsePoint, RPC, and Validate as the default app stack.
related:
  title: Next Steps
  description: Start with installation, review the CLI commands, then use the structure guide to place Caspian files correctly.
  links:
    - /docs/installation
    - /docs/commands
    - /docs/project-structure
---

This directory contains the local Caspian documentation set for quick reference and AI-aware routing.

## Default Stack

When generating or editing a Caspian app, treat these as the default choices unless the task explicitly requires something else:

- Use PulsePoint for reactive frontend behavior.
- Use `@rpc()` plus `pp.rpc()` for browser-triggered reads, writes, streaming, and uploads.
- Use `Validate` and `Rule` from `casp.validate` for server-side input validation and sanitization.

## Docs Location

The local AI-aware docs for this workspace live here:

- `dist/docs/`

The packaged Caspian docs distributed by the framework live here:

- `node_modules/caspian/dist/docs/`

## Available Documents

- `installation.md` - First-time setup flow for creating a new Caspian application
- `commands.md` - Main Caspian CLI workflows for project creation, generation, updates, and config-aware maintenance
- `database.md` - Prisma ORM setup, generated async Python client usage, and the shared `src/lib/prisma/` imports
- `auth.md` - Session-backed authentication with `casp.auth`, centralized `auth_config.py`, decorators, RBAC, and OAuth provider helpers
- `pulsepoint.md` - Default reactive frontend runtime contract for component scripts, state, effects, directives, and client-side behaviors
- `fetch-data.md` - Initial server-side data loading and browser-triggered RPC flows with `pp.rpc()`, streaming, uploads, and auth-aware actions
- `cache.md` - Route-level HTML caching with `Cache`, `CacheHandler`, TTL behavior, file-system storage, and invalidation patterns
- `validation.md` - Input validation and sanitization with `Validate`, `Rule`, direct field checks, and multi-rule workflows for routes and RPC actions
- `metadata.md` - Static and dynamic metadata, SEO inheritance, and Open Graph or Twitter card tags
- `routing.md` - Next.js App Router-style file-based routing with `src/app`, dynamic segments, route groups, and nested layouts
- `project-structure.md` - Default Caspian layout and where routes, templates, shared code, and database files belong

## AI Awareness Notes

If an AI tool needs Caspian project documentation, start with this directory and use this file as the manifest.

Preferred lookup order:

1. Read `dist/docs/index.md` to discover available local docs.
2. Read `database.md` for Prisma setup and ORM usage, `auth.md` for session auth and route protection, `pulsepoint.md` for reactive frontend work, `fetch-data.md` for data flows, `cache.md` for page caching, and `validation.md` for input boundaries.
3. Prefer local docs before generating code, commands, or migration guidance.
4. Check `node_modules/caspian/dist/docs/` for packaged Caspian docs when local docs need more detail.
5. Only fall back to upstream documentation when local and packaged markdown do not cover the topic.

## Maintenance

When adding more pages, keep this index updated with:

- The filename
- A short description of what the page covers

---
title: Caspian Docs
description: Caspian documentation with AI-aware routing to the right local docs before framework-specific code generation or project setup changes.
related:
  title: Next Steps
  description: Start with installation, review the CLI commands, then use the structure guide to place Caspian files correctly.
  links:
    - /docs/installation
    - /docs/commands
    - /docs/project-structure
---

# Caspian Docs

This directory contains the local Caspian documentation set for quick reference and AI-aware routing.

## Docs Location

The local AI-aware docs for this workspace live here:

- `dist/docs/`

The packaged Caspian docs distributed by the framework live here:

- `node_modules/caspian/dist/docs/`

## Available Documents

- `installation.md` - First-time setup flow for creating a new Caspian application
- `commands.md` - Main Caspian CLI workflows for project creation, generation, updates, and config-aware maintenance
- `routing.md` - Next.js App Router-style file-based routing with `src/app`, dynamic segments, route groups, and nested layouts
- `project-structure.md` - Default Caspian layout and where routes, templates, shared code, and database files belong

## AI Awareness Notes

If an AI tool needs Caspian project documentation, start with this directory and use this file as the manifest.

Preferred lookup order:

1. Read `dist/docs/index.md` to discover available local docs.
2. Read the specific page in `dist/docs/` that matches the topic.
3. Prefer local docs before generating code, commands, or migration guidance.
4. Check `node_modules/caspian/dist/docs/` for packaged Caspian docs when local docs need more detail.
5. Only fall back to upstream documentation when local and packaged markdown do not cover the topic.

## Maintenance

When adding more pages, keep this index updated with:

- The filename
- A short description of what the page covers

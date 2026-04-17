---
title: Commands
description: Use the Caspian CLI reference to scaffold projects, generate code, and update framework files without confusing first-time setup with existing-project maintenance.
related:
  title: Related docs
  description: Start with installation for new apps, then use the routing and structure guides to place generated files correctly.
  links:
    - /docs/installation
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page summarizes the main Caspian CLI workflows for creating projects, generating code, and updating an existing app.

## Overview

Use Caspian CLI commands for three main tasks:

- Create a new application.
- Generate code from your Prisma schema.
- Update framework-managed project files.

Before running update commands, review `caspian.config.json` because it controls overwrite behavior.

## Project Creation

Use the interactive scaffold when starting a new Caspian app:

```bash
npx create-caspian-app@latest
```

You can also create a project with starter-kit flags:

```bash
npx create-caspian-app my-app --starter-kit=fullstack
```

For a custom source setup, the CLI also supports a custom starter flow:

```bash
npx create-caspian-app my-tool --starter-kit=custom ...
```

After scaffolding, use `routing.md` to understand how Caspian maps `src/app` folders to URLs.

## Installation Flags

Common scaffold flags include:

- `--backend-only` for API-only projects without frontend assets
- `--tailwindcss` to install Tailwind CSS support
- `--typescript` to add TypeScript and Vite support
- `--mcp` to initialize a Model Context Protocol server for AI agents
- `--prisma` to initialize Prisma ORM support

## Code Generation

When your Prisma schema changes, use the generation command shown in the Caspian CLI docs:

```bash
npx ppy generate
```

This flow generates Python data classes based on `prisma/schema.prisma`.

## Update Project

Use the project update command for existing Caspian apps:

```bash
npx casp update project
```

This command updates framework-managed project files and can overwrite entry points, styles, or configuration files if they are not protected.

## Configuration

The CLI reads `caspian.config.json` to decide how it should interact with the project.

Important configuration areas include:

- Project identity and root path
- Feature toggles such as backend-only, Tailwind, MCP, Prisma, and TypeScript
- Component scan directories
- `excludeFiles` rules for protecting local changes during updates

## `excludeFiles` Strategy

Use `excludeFiles` in `caspian.config.json` to prevent the update command from overwriting files you have customized.

This is useful for protecting:

- Stylesheets
- Configuration files
- Entry-point files
- Other locally modified framework-managed files

If you exclude a file, it will be preserved during updates, but you may need to merge future framework changes into that file manually.

## AI Routing Notes

If an AI agent is deciding which command flow to use, apply these rules first.

- Use `npx create-caspian-app@latest` when the user is creating a new project.
- Use `npx casp update project` only for an existing Caspian project.
- Read `caspian.config.json` before running update commands.
- Use `npx ppy generate` after Prisma schema changes.
- Check [routing.md](./routing.md) before generating or modifying route folders under `src/app/`.
- Check [project-structure.md](./project-structure.md) before placing generated files into the project.

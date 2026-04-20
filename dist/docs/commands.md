---
title: Commands
description: Use the Caspian CLI reference to scaffold projects, generate code, and update framework files without confusing first-time setup with existing-project maintenance.
related:
  title: Related docs
  description: Start with installation for new apps, then use the routing and structure guides to place generated files correctly.
  links:
    - /docs/installation
    - /docs/database
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page summarizes the main Caspian CLI workflows for creating projects, generating code, and updating an existing app.

## Overview

The current workspace includes a local `prisma` binary, but it does not include local `create-caspian-app`, `casp`, or `ppy` binaries under `node_modules/.bin`. Treat the scaffold and project-update commands below as external `npx` workflows rather than project-local executables.

Use Caspian CLI commands for three main tasks:

- Create a new application.
- Regenerate Node and Python ORM code from your Prisma schema.
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
- `--typescript` to add TypeScript support and any scaffold-specific TypeScript build/config files
- `--mcp` to initialize a Model Context Protocol server for AI agents
- `--prisma` to initialize Prisma ORM support

## Code Generation

When `prisma/schema.prisma` changes in this workspace, use this command flow:

```bash
npx prisma migrate dev

# If the change requires refreshed seed data:
npx prisma generate
npx prisma db seed

npx ppy generate
```

Use `npx prisma migrate dev` first so the development database and migration history stay aligned. If the updated schema affects seed data, run `npx prisma generate` before `npx prisma db seed` so Node-side tooling such as `prisma/seed.ts` uses the current generated client.

Always finish with `npx ppy generate`. That is the required way to refresh the Python ORM classes used by the app after a schema change.

This workspace already includes an app-owned Python database layer in `src/lib/prisma/`, so reuse that package for Python-side database access instead of creating a second helper. Do not hand-edit generated ORM classes.

See `database.md` for the full Prisma workflow, including `.env`, `prisma.config.ts`, migrations, and async usage patterns.

## Update Project

Use the project update command for existing Caspian apps:

```bash
npx casp update project
```

This command updates framework-managed project files and can overwrite entry points, styles, or configuration files if they are not protected. The `casp` binary is not bundled locally in this workspace, so verify the external CLI package before automating this command.

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
- When `prisma/schema.prisma` changes, run `npx prisma migrate dev` first.
- If the change requires seed data, run `npx prisma generate` and then `npx prisma db seed`.
- Run `npx ppy generate` after every Prisma schema change to refresh the Python ORM classes.
- Never hand-edit generated Prisma or Python ORM classes.
- Check `database.md` when the task involves Prisma setup, schema updates, or the current workspace's schema, migration, and seed tooling.
- Check [routing.md](./routing.md) before generating or modifying route folders under `src/app/`.
- Check [project-structure.md](./project-structure.md) before placing generated files into the project.

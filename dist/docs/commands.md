---
title: Commands
description: Use this Caspian CLI reference to choose the right create, starter-kit, update, and Prisma-to-Python ORM regeneration flow for the current workspace.
related:
  title: Related docs
  description: Start with installation for new apps, then use database and structure docs when commands affect schema, generated ORM files, or project layout.
  links:
    - /docs/installation
    - /docs/database
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page documents the current Caspian command families used with this workspace and the surrounding docs toolchain.

## Overview

The current workspace includes a local `prisma` binary, but it does not include local `create-caspian-app`, `casp`, or `ppy` binaries under `node_modules/.bin`. Treat project creation, project update, and Python ORM generation as external `npx` workflows rather than project-local executables.

Examples below use the bare package name for readability. If you want to force the latest published scaffold package, you can add `@latest`, for example `npx create-caspian-app@latest my-app`.

Use these command families for different tasks:

| Task | Command | When to use |
| --- | --- | --- |
| Create a new project | `npx create-caspian-app my-app` | First-time scaffold for a brand new Caspian app. |
| Update an existing project | `npx casp update project` | Refresh framework-managed files inside an existing Caspian project. |
| Create from a starter kit | `npx create-caspian-app my-app --starter-kit=fullstack` | Start from a built-in or custom template instead of the default scaffold. |
| List starter kits | `npx create-caspian-app --list-starter-kits` | Inspect the starter catalog before choosing a preset. |
| Regenerate ORM after schema change | `npx prisma migrate dev` then optional seed commands, then `npx ppy generate` | Keep Prisma migrations, seed flow, and generated Python ORM files aligned after `prisma/schema.prisma` changes. |

Before running `npx casp update project`, read `caspian.config.json` because it controls feature flags and `excludeFiles` overwrite protection. Before running Prisma or `ppy` commands, read [database.md](./database.md).

## Supported Flags

The scaffold CLI exposes these feature flags:

| Flag | Meaning | When to use |
| --- | --- | --- |
| `-y` | Non-interactive mode | Use in CI, automation, or scripted setup. |
| `--backend-only` | Backend-only project | Use for APIs, services, or apps with no browser UI. |
| `--tailwindcss` | Enable Tailwind CSS | Use for frontend or full-stack styling. |
| `--typescript` | Enable TypeScript frontend tooling | Use when you want frontend TypeScript assets. |
| `--mcp` | Enable MCP support | Use when the app needs Model Context Protocol support. |
| `--prisma` | Enable Prisma setup | Use when the app needs schema, migration, and ORM tooling. |
| `--starter-kit=<kit>` | Use a built-in or named starter kit | Use to bootstrap from a preset instead of the default scaffold. |
| `--starter-kit-source=<url>` | Pull a starter kit from a Git source | Use when the starter kit lives in an external repository. |

Add `-y` to most create or update examples below when you want the same flow to run without prompts.

## Create a New Project

### Interactive create

```bash
npx create-caspian-app my-app
```

Use when you want the normal interactive scaffold flow. The CLI can prompt for project name and feature toggles.

### Non-interactive create

```bash
npx create-caspian-app my-app -y
```

Use when you already know the desired feature flags or when automation should avoid prompts.

Without extra flags, the non-interactive default behaves like this:

- `backendOnly: false`
- `tailwindcss: false`
- `typescript: false`
- `mcp: false`
- `prisma: false`

## Backend-only Create Combinations

When `--backend-only` is enabled, the meaningful feature toggles are `--mcp` and `--prisma`. Frontend-oriented flags are not part of the normal backend-only prompt flow.

| Goal | Example command | When to use |
| --- | --- | --- |
| Minimal backend-only | `npx create-caspian-app my-app --backend-only` | Simple API or service without frontend assets. |
| Backend-only + MCP | `npx create-caspian-app my-app --backend-only --mcp` | Backend service plus MCP support. |
| Backend-only + Prisma | `npx create-caspian-app my-app --backend-only --prisma` | API or service with database and ORM support. |
| Backend-only + MCP + Prisma | `npx create-caspian-app my-app --backend-only --mcp --prisma` | Full backend-only setup with both integrations. |

You can append `-y` to any backend-only example above.

### Accepted but not recommended in backend-only mode

```bash
npx create-caspian-app my-app --backend-only --tailwindcss
npx create-caspian-app my-app --backend-only --typescript
npx create-caspian-app my-app --backend-only --tailwindcss --typescript
```

Avoid them in normal use. Backend-only mode strips frontend assets, so Tailwind and frontend TypeScript flags do not have a practical effect in the resulting scaffold.

## Full-stack and Frontend-enabled Create Combinations

When `--backend-only` is not used, the feature flags `--tailwindcss`, `--typescript`, `--mcp`, and `--prisma` can be combined in any subset. The table below lists the common command matrix.

| Flags | Example command | When to use |
| --- | --- | --- |
| none | `npx create-caspian-app my-app` | Default full-stack scaffold. |
| `--tailwindcss` | `npx create-caspian-app my-app --tailwindcss` | Styled frontend with Tailwind. |
| `--typescript` | `npx create-caspian-app my-app --typescript` | Frontend TypeScript assets. |
| `--mcp` | `npx create-caspian-app my-app --mcp` | Full-stack app with MCP support. |
| `--prisma` | `npx create-caspian-app my-app --prisma` | Full-stack app with Prisma support. |
| `--tailwindcss --typescript` | `npx create-caspian-app my-app --tailwindcss --typescript` | Styled frontend with TypeScript. |
| `--tailwindcss --mcp` | `npx create-caspian-app my-app --tailwindcss --mcp` | Styled UI plus MCP support. |
| `--tailwindcss --prisma` | `npx create-caspian-app my-app --tailwindcss --prisma` | Styled UI plus database tooling. |
| `--typescript --mcp` | `npx create-caspian-app my-app --typescript --mcp` | Frontend TypeScript plus MCP. |
| `--typescript --prisma` | `npx create-caspian-app my-app --typescript --prisma` | Frontend TypeScript plus Prisma. |
| `--mcp --prisma` | `npx create-caspian-app my-app --mcp --prisma` | Both major backend integrations without forcing frontend styling choices. |
| `--tailwindcss --typescript --mcp` | `npx create-caspian-app my-app --tailwindcss --typescript --mcp` | Rich frontend plus MCP. |
| `--tailwindcss --typescript --prisma` | `npx create-caspian-app my-app --tailwindcss --typescript --prisma` | Rich frontend plus Prisma. |
| `--tailwindcss --mcp --prisma` | `npx create-caspian-app my-app --tailwindcss --mcp --prisma` | Styled UI plus both major integrations. |
| `--typescript --mcp --prisma` | `npx create-caspian-app my-app --typescript --mcp --prisma` | Frontend TypeScript plus both major integrations. |
| `--tailwindcss --typescript --mcp --prisma` | `npx create-caspian-app my-app --tailwindcss --typescript --mcp --prisma` | Most feature-complete default full-stack setup. |

You can append `-y` to any frontend-enabled example above.

## Starter Kit Commands

### List available starter kits

```bash
npx create-caspian-app --list-starter-kits
```

Use when you want to inspect the built-in starter kit catalog before creating a project.

### Built-in starter kits

The current command reference includes these built-in starter kits:

| Starter kit | Example command | Preset features | When to use |
| --- | --- | --- | --- |
| `basic` | `npx create-caspian-app my-app --starter-kit=basic` | `backendOnly=true`, `tailwindcss=false`, `prisma=false`, `mcp=false` | Minimal backend-oriented scaffold. |
| `fullstack` | `npx create-caspian-app my-app --starter-kit=fullstack` | `backendOnly=false`, `tailwindcss=true`, `prisma=true`, `mcp=false` | Complete full-stack web app preset. |
| `api` | `npx create-caspian-app my-app --starter-kit=api` | `backendOnly=true`, `tailwindcss=false`, `prisma=true`, `mcp=false` | Backend API preset with Prisma. |
| `realtime` | `npx create-caspian-app my-app --starter-kit=realtime` | `backendOnly=false`, `tailwindcss=true`, `prisma=true`, `mcp=true` | Realtime-oriented preset with both UI and integration support. |

If you want frontend TypeScript, add `--typescript` explicitly rather than assuming it is implied by a starter kit preset.

Append `-y` to any starter kit example when you want the same preset without prompts.

### Starter kit overrides

```bash
npx create-caspian-app my-app --starter-kit=basic --prisma
npx create-caspian-app my-app --starter-kit=fullstack --typescript
npx create-caspian-app my-app --starter-kit=api --mcp
npx create-caspian-app my-app --starter-kit=realtime --backend-only
```

Use overrides when a starter kit is close to the target app but you need to force one or more feature flags.

### Custom starter kit source

```bash
npx create-caspian-app my-app --starter-kit=custom --starter-kit-source=https://github.com/user/repo
```

You can combine extra flags with the custom source:

```bash
npx create-caspian-app my-app --starter-kit=custom --starter-kit-source=https://github.com/user/repo --typescript --prisma
```

Use this flow when the scaffold should come from an external repository instead of a built-in preset.

## Update an Existing Project

The updater wrapper recognizes the `update project` family:

```bash
npx casp update project
```

Use this only inside an existing Caspian project that already has `caspian.config.json` in the current directory.

### Update to latest

```bash
npx casp update project
```

Use when you want the normal refresh to the latest framework-managed files.

### Update to a specific tag

```bash
npx casp update project @v4-alpha
```

Use when you want a prerelease or named tag instead of the latest stable release.

### Update to a specific version

```bash
npx casp update project @1.2.3
npx casp update project 1.2.3
```

Use when you want a deterministic pinned version.

### Non-interactive update

```bash
npx casp update project -y
npx casp update project @v4-alpha -y
npx casp update project @1.2.3 -y
npx casp update project 1.2.3 -y
```

Use when automation, CI, or scripted maintenance should skip confirmation prompts.

### Update safety notes

The update flow is config-aware. Review `caspian.config.json` before running it, especially these areas:

- feature flags such as backend-only, Tailwind, TypeScript, MCP, and Prisma
- `componentScanDirs`
- `excludeFiles`, which protects customized files from overwrite

If you exclude a file, the updater preserves it, but you are responsible for merging future framework changes into that file manually.

## Prisma and Python ORM Regeneration

The create and update commands above are not the whole maintenance story for this workspace. When `prisma/schema.prisma` changes, follow the ORM flow below so the TypeScript Prisma client, database state, and Python ORM layer stay aligned.

### Required order after `prisma/schema.prisma` changes

```bash
npx prisma migrate dev

# Only when seed flow or prisma/seed.ts depends on the new schema:
npx prisma generate
npx prisma db seed

npx ppy generate
```

Use this order:

1. Run `npx prisma migrate dev` first so migrations and the development database stay aligned.
2. If `prisma/seed.ts` or seed data depends on the new schema, run `npx prisma generate` and then `npx prisma db seed`.
3. Run `npx ppy generate` last so the Python ORM layer matches the updated Prisma schema.

Do not manually create or edit these generated files:

- `src/lib/prisma/__init__.py`
- `src/lib/prisma/db.py`
- `src/lib/prisma/models.py`
- `settings/prisma-schema.json`

`npx ppy generate` owns those files. Treat `src/lib/prisma/` as the app's generated Python database layer, not as a manual editing surface.

See [database.md](./database.md) for the full schema, migration, seed, and async usage guide.

## Practical Recommendation Matrix

| Goal | Recommended command |
| --- | --- |
| Minimal backend without database | `npx create-caspian-app my-app --backend-only -y` |
| Small API or service with Prisma | `npx create-caspian-app my-app --backend-only --prisma -y` |
| Full-stack app with styled UI and Prisma | `npx create-caspian-app my-app --tailwindcss --prisma -y` |
| Rich full-stack app with TypeScript, Tailwind, and Prisma | `npx create-caspian-app my-app --tailwindcss --typescript --prisma -y` |
| Full-stack app with everything enabled | `npx create-caspian-app my-app --tailwindcss --typescript --mcp --prisma -y` |
| Realtime preset | `npx create-caspian-app my-app --starter-kit=realtime -y` |
| Update current project to latest | `npx casp update project` |
| Update current project in automation | `npx casp update project -y` |
| Update current project to an exact version | `npx casp update project @1.2.3 -y` |

## AI Routing Notes

If an AI agent is deciding which command flow to use, apply these rules first.

- Use `npx create-caspian-app ...` only when the user is creating a new Caspian project.
- Use `npx casp update project ...` only when the user is already inside an existing Caspian project.
- Add `-y` when the user wants automation, CI, or a no-prompt flow.
- Treat `--tailwindcss` and `--typescript` as frontend-oriented flags. They are not meaningful in normal backend-only scaffolds.
- Treat starter kit defaults as a base layer that can be overridden by explicit flags.
- When `prisma/schema.prisma` changes, run `npx prisma migrate dev` first, then any required seed refresh, then `npx ppy generate`.
- Never hand-edit generated Python ORM files under `src/lib/prisma/` or `settings/prisma-schema.json`.
- Read [database.md](./database.md) before generating Prisma schema, migration, seed, or ORM guidance.
- Read [routing.md](./routing.md) before generating or modifying route folders under `src/app/`.
- Read [project-structure.md](./project-structure.md) before placing generated files into the project.

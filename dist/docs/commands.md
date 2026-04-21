---
title: Commands
description: Use the current Caspian CLI reference to choose the right scaffold, starter-kit, update, and Prisma regeneration workflow for the current workspace.
related:
  title: Related docs
  description: Start with installation for new apps, then use database, MCP, and structure docs when commands affect schema, server tooling, generated ORM files, or project layout.
  links:
    - /docs/installation
    - /docs/database
    - /docs/mcp
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page documents the current Caspian command families used with this workspace and the surrounding docs toolchain.

## Overview

The current workspace includes a local `prisma` binary, but it does not include local `create-caspian-app`, `casp`, or `ppy` binaries under `node_modules/.bin`. Treat project creation, project update, and Python ORM generation as external `npx` workflows rather than project-local executables.

The current workspace also includes a local MCP run path through `npm run mcp`, which starts the app-owned FastMCP server via `settings/restart-mcp.ts`.

Examples below use `npx create-caspian-app` for readability. If you want to force the latest published scaffold package explicitly, you can use `npx create-caspian-app@latest` instead.

This updated reference includes the newer updater behavior and the current scaffold behavior:

- positional version or tag values such as `beta` or `1.2.3`
- named options such as `--tag beta`, `--tag=beta`, `--version 1.2.3`, and `--version=1.2.3`
- conflict detection when more than one version source is provided
- PowerShell-safe tagging syntax with `--tag beta`
- Windows-safe execution that resolves `npx.cmd` on Win32
- scaffold behavior that reuses an existing `.venv` instead of recreating it every time

Before running update commands, read `caspian.config.json` because it controls feature flags and `excludeFiles` overwrite protection. In the current workspace that config shows `backendOnly: false`, `tailwindcss: true`, `mcp: true`, `prisma: true`, `typescript: false`, and `componentScanDirs: ["src"]`.

## 1. Main Command Families

### Create a new project

```bash
npx create-caspian-app my-app
```

Use when you want to scaffold a new Caspian project from scratch.

### Create a project from a starter kit

```bash
npx create-caspian-app my-app --starter-kit=fullstack
npx create-caspian-app my-app --starter-kit=custom --starter-kit-source=https://github.com/user/repo
```

Use when you want a preset structure instead of the blank default scaffold.

### List available starter kits

```bash
npx create-caspian-app --list-starter-kits
```

Use when you want to inspect the built-in starter kits before choosing one.

### Update an existing project

```bash
npx casp update project
```

Use when you are inside an existing Caspian project and want to refresh framework-managed files using the project's `caspian.config.json`.

### Run the local MCP server

```bash
npm run mcp
```

Use when you want the workspace's app-owned FastMCP server only.

### Run the full local stack including MCP

```bash
npm run dev
```

Use when you want BrowserSync, Tailwind, the Python app server, and MCP together.

### Inspect the local MCP config directly

```powershell
.venv\Scripts\fastmcp.exe inspect src/lib/mcp/fastmcp.json
```

Use when you are debugging the nested FastMCP config or checking which tools the current MCP server exposes.

### Run FastMCP directly with the nested config

```powershell
.venv\Scripts\fastmcp.exe run src/lib/mcp/fastmcp.json --no-banner
```

Use when you want the raw FastMCP path without the npm wrapper.

### Regenerate ORM after schema changes

```bash
npx prisma migrate dev

# If the change requires refreshed seed data:
npx prisma generate
npx prisma db seed

npx ppy generate
```

Use when `prisma/schema.prisma` changes and you need migrations, seed flow, and the generated Python ORM layer to stay aligned.

Because this workspace keeps `fastmcp.json` under `src/lib/mcp/`, plain `fastmcp run` from the project root does not auto-detect the config. Use `npm run mcp` or pass the explicit config path.

## 2. Supported Flags And Options

### For `create-caspian-app`

| Flag | Meaning | When to use |
| --- | --- | --- |
| `-y` | Non-interactive mode | Use in scripts, CI, or when you want to skip prompts. |
| `--backend-only` | Backend-only project | Use for APIs or services without frontend assets. |
| `--tailwindcss` | Enable Tailwind CSS | Use for frontend or full-stack UI styling. |
| `--typescript` | Enable TypeScript frontend tooling | Use when you want the scaffold's TypeScript frontend path. |
| `--mcp` | Enable MCP support | Use when the project needs Model Context Protocol support. |
| `--prisma` | Enable Prisma-related setup | Use when the project needs ORM and database tooling. |
| `--starter-kit=<kit>` | Select a starter kit | Use when you want a built-in preset or named kit. |
| `--starter-kit-source=<url>` | External starter kit source | Use when the starter kit lives in a Git repository. |
| `--list-starter-kits` | List built-in starter kits | Use when you want to inspect the starter catalog before scaffold. |

### For `npx casp update project`

| Option | Meaning | When to use |
| --- | --- | --- |
| `-y` | Non-interactive mode | Use for automated updates without prompts. |
| positional tag | Example: `beta` | Use for a quick tag or channel update. |
| positional version | Example: `1.2.3` | Use for exact version pinning. |
| `--tag <value>` | Named tag option | Use especially in PowerShell or other shells where explicit spacing is safer. |
| `--tag=<value>` | Inline named tag option | Same as above in inline syntax. |
| `--version <value>` | Named version option | Use for explicit version selection. |
| `--version=<value>` | Inline named version option | Same as above in inline syntax. |

The updater throws an error if conflicting version sources are provided.

## 3. Create Command Combinations

### 3.1 Base create command

#### Interactive create

```bash
npx create-caspian-app my-app
```

Use when you want the standard guided setup.

#### Non-interactive create

```bash
npx create-caspian-app my-app -y
```

Use when you want a quick scaffold without prompts.

In skip-prompt mode, the default feature values are:

- `backendOnly: false`
- `tailwindcss: false`
- `typescript: false`
- `mcp: false`
- `prisma: false`

### 3.2 Backend-only combinations

In backend-only mode, the main meaningful toggles are `--mcp` and `--prisma`. Frontend-oriented flags may still appear in raw CLI args, but they are not part of the normal backend-only feature flow.

#### Minimal backend-only

```bash
npx create-caspian-app my-app --backend-only
npx create-caspian-app my-app --backend-only -y
```

Use when you want a simple backend project without frontend assets.

#### Backend-only plus MCP

```bash
npx create-caspian-app my-app --backend-only --mcp
npx create-caspian-app my-app --backend-only --mcp -y
```

Use when you want a backend service with MCP enabled.

#### Backend-only plus Prisma

```bash
npx create-caspian-app my-app --backend-only --prisma
npx create-caspian-app my-app --backend-only --prisma -y
```

Use when you want a backend or API project with database tooling.

#### Backend-only plus MCP plus Prisma

```bash
npx create-caspian-app my-app --backend-only --mcp --prisma
npx create-caspian-app my-app --backend-only --mcp --prisma -y
```

Use when you want the fullest backend-only setup.

#### Accepted but usually not useful for backend-only mode

```bash
npx create-caspian-app my-app --backend-only --tailwindcss
npx create-caspian-app my-app --backend-only --typescript
npx create-caspian-app my-app --backend-only --tailwindcss --typescript
```

These flags can be present in raw CLI args, but backend-only mode removes frontend assets and disables the normal TypeScript and frontend path, so they are not practical combinations.

### 3.3 Full-stack combinations

These apply when `--backend-only` is not used.

#### Minimal full-stack

```bash
npx create-caspian-app my-app
npx create-caspian-app my-app -y
```

Use when you want the default scaffold and will add features later.

#### Full-stack plus Tailwind CSS

```bash
npx create-caspian-app my-app --tailwindcss
npx create-caspian-app my-app --tailwindcss -y
```

Use when you want Tailwind-based UI styling.

#### Full-stack plus TypeScript

```bash
npx create-caspian-app my-app --typescript
npx create-caspian-app my-app --typescript -y
```

Use when you want the scaffold's TypeScript frontend tooling.

#### Full-stack plus MCP

```bash
npx create-caspian-app my-app --mcp
npx create-caspian-app my-app --mcp -y
```

Use when you want MCP support in a full-stack app.

#### Full-stack plus Prisma

```bash
npx create-caspian-app my-app --prisma
npx create-caspian-app my-app --prisma -y
```

Use when you want ORM and database support.

#### Full-stack plus Tailwind plus TypeScript

```bash
npx create-caspian-app my-app --tailwindcss --typescript
npx create-caspian-app my-app --tailwindcss --typescript -y
```

Use when you want the typical richer frontend setup.

#### Full-stack plus Tailwind plus MCP

```bash
npx create-caspian-app my-app --tailwindcss --mcp
npx create-caspian-app my-app --tailwindcss --mcp -y
```

Use when you want styled UI plus MCP.

#### Full-stack plus Tailwind plus Prisma

```bash
npx create-caspian-app my-app --tailwindcss --prisma
npx create-caspian-app my-app --tailwindcss --prisma -y
```

Use when you want styled UI plus database tooling.

#### Full-stack plus TypeScript plus MCP

```bash
npx create-caspian-app my-app --typescript --mcp
npx create-caspian-app my-app --typescript --mcp -y
```

Use when you want TypeScript assets plus MCP.

#### Full-stack plus TypeScript plus Prisma

```bash
npx create-caspian-app my-app --typescript --prisma
npx create-caspian-app my-app --typescript --prisma -y
```

Use when you want TypeScript assets plus database tooling.

#### Full-stack plus MCP plus Prisma

```bash
npx create-caspian-app my-app --mcp --prisma
npx create-caspian-app my-app --mcp --prisma -y
```

Use when you want both integration layers without forcing frontend preferences.

#### Full-stack plus Tailwind plus TypeScript plus MCP

```bash
npx create-caspian-app my-app --tailwindcss --typescript --mcp
npx create-caspian-app my-app --tailwindcss --typescript --mcp -y
```

Use when you want a rich frontend plus MCP.

#### Full-stack plus Tailwind plus TypeScript plus Prisma

```bash
npx create-caspian-app my-app --tailwindcss --typescript --prisma
npx create-caspian-app my-app --tailwindcss --typescript --prisma -y
```

Use when you want a rich frontend plus database tooling.

#### Full-stack plus Tailwind plus MCP plus Prisma

```bash
npx create-caspian-app my-app --tailwindcss --mcp --prisma
npx create-caspian-app my-app --tailwindcss --mcp --prisma -y
```

Use when you want styled UI and both integrations.

#### Full-stack plus TypeScript plus MCP plus Prisma

```bash
npx create-caspian-app my-app --typescript --mcp --prisma
npx create-caspian-app my-app --typescript --mcp --prisma -y
```

Use when you want TypeScript assets plus both integrations.

#### Full-stack plus Tailwind plus TypeScript plus MCP plus Prisma

```bash
npx create-caspian-app my-app --tailwindcss --typescript --mcp --prisma
npx create-caspian-app my-app --tailwindcss --typescript --mcp --prisma -y
```

Use when you want the most complete default full-stack configuration.

## 4. Starter Kit Command Combinations

### Built-in starter kits

#### Basic starter kit

```bash
npx create-caspian-app my-app --starter-kit=basic
npx create-caspian-app my-app --starter-kit=basic -y
```

Use when you want the minimal backend preset.

Preset features:

- `backendOnly = true`
- `tailwindcss = false`
- `prisma = false`
- `mcp = false`

#### Fullstack starter kit

```bash
npx create-caspian-app my-app --starter-kit=fullstack
npx create-caspian-app my-app --starter-kit=fullstack -y
```

Use when you want the built-in full-stack preset.

Preset features:

- `backendOnly = false`
- `tailwindcss = true`
- `prisma = true`
- `mcp = false`

#### API starter kit

```bash
npx create-caspian-app my-app --starter-kit=api
npx create-caspian-app my-app --starter-kit=api -y
```

Use when you want a backend API preset with ORM and database support.

Preset features:

- `backendOnly = true`
- `tailwindcss = false`
- `prisma = true`
- `mcp = false`

#### Realtime starter kit

```bash
npx create-caspian-app my-app --starter-kit=realtime
npx create-caspian-app my-app --starter-kit=realtime -y
```

Use when you want the built-in realtime preset.

Preset features:

- `backendOnly = false`
- `tailwindcss = true`
- `prisma = true`
- `mcp = true`

### Starter kit with feature overrides

Starter kit defaults can still be overridden by explicit feature flags because the CLI reapplies raw args after loading starter kit defaults.

Examples:

```bash
npx create-caspian-app my-app --starter-kit=basic --prisma
npx create-caspian-app my-app --starter-kit=fullstack --typescript
npx create-caspian-app my-app --starter-kit=api --mcp
npx create-caspian-app my-app --starter-kit=realtime --backend-only
```

Use when a starter kit is close to what you want but still needs a few forced overrides.

### Custom starter kit source

```bash
npx create-caspian-app my-app --starter-kit=custom --starter-kit-source=https://github.com/user/repo
npx create-caspian-app my-app --starter-kit=custom --starter-kit-source=https://github.com/user/repo -y
```

You can combine feature overrides too:

```bash
npx create-caspian-app my-app --starter-kit=custom --starter-kit-source=https://github.com/user/repo --typescript --prisma
```

Use when the scaffold should come from an external Git repository. The CLI clones the repository, removes `.git`, and updates project config for the new project.

## 5. Update Command Combinations

The updater recognizes only the `update project` command family. Anything outside that family is rejected by the wrapper.

### Update to latest

```bash
npx casp update project
```

Use when you want the newest release and do not need a pinned tag or version.

### Update to a specific tag using positional syntax

```bash
npx casp update project beta
```

Use when you want a named release channel such as `beta`.

### Update to a specific version using positional syntax

```bash
npx casp update project 1.2.3
```

Use when you want an exact version.

### Update using named tag option

```bash
npx casp update project --tag beta
npx casp update project --tag=beta
```

Use when you want clearer syntax or PowerShell-safe argument parsing.

### Update using named version option

```bash
npx casp update project --version 1.2.3
npx casp update project --version=1.2.3
```

Use when you want explicit version selection with named syntax.

### Non-interactive latest update

```bash
npx casp update project -y
```

Use for scripted or CI updates to latest.

### Non-interactive tagged update

```bash
npx casp update project beta -y
npx casp update project --tag beta -y
npx casp update project --tag=beta -y
```

Use for scripted or shell-safe tag-based upgrades.

### Non-interactive versioned update

```bash
npx casp update project 1.2.3 -y
npx casp update project --version 1.2.3 -y
npx casp update project --version=1.2.3 -y
```

Use for automated pinned upgrades.

## 6. Invalid Or Conflicting Update Cases

These cases matter because the newer updater parsing is stricter.

### Missing value

```bash
npx casp update project --tag
npx casp update project --version
```

Result: parsing error because the option is missing its value.

### Too many positional arguments

```bash
npx casp update project beta extra
```

Result: parsing error because `update project` accepts at most one extra positional version or tag value.

### Conflicting version sources

```bash
npx casp update project beta --tag latest
npx casp update project 1.2.3 --version 2.0.0
```

Result: parsing error because more than one version source was provided.

## 7. Prisma And Python ORM Regeneration

The create and update commands above are not the whole maintenance story for this workspace. When `prisma/schema.prisma` changes, follow the ORM flow below so the TypeScript Prisma client, database state, and Python ORM layer stay aligned.

### Required order after schema changes

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
3. Run `npx ppy generate` last so the generated Python ORM layer matches the updated Prisma schema.

Do not manually create or edit these generated files:

- `src/lib/prisma/__init__.py`
- `src/lib/prisma/db.py`
- `src/lib/prisma/models.py`
- `settings/prisma-schema.json`

`npx ppy generate` owns those files. Treat `src/lib/prisma/` as the app's generated Python database layer, not as a manual editing surface.

See [database.md](./database.md) for the full schema, migration, seed, and async usage guide.

## 8. Practical Recommendation Matrix

| Goal | Recommended command |
| --- | --- |
| Minimal backend service | `npx create-caspian-app my-app --backend-only -y` |
| Backend API with database support | `npx create-caspian-app my-app --backend-only --prisma -y` |
| Full-stack app with styled UI and database support | `npx create-caspian-app my-app --tailwindcss --prisma -y` |
| Full-stack app with richer frontend tooling and Prisma | `npx create-caspian-app my-app --tailwindcss --typescript --prisma -y` |
| Full-stack app with everything enabled | `npx create-caspian-app my-app --tailwindcss --typescript --mcp --prisma -y` |
| Realtime preset | `npx create-caspian-app my-app --starter-kit=realtime -y` |
| Update current project to latest | `npx casp update project` |
| Update current project to beta safely in PowerShell | `npx casp update project --tag beta` |
| Update current project to an exact version | `npx casp update project --version 1.2.3 -y` |
| Regenerate Python ORM after schema changes | `npx prisma migrate dev` then optional seed commands, then `npx ppy generate` |

## 9. Configuration Notes

The CLI reads `caspian.config.json` to decide how it should interact with the project.

Important configuration areas include:

- project identity and root path
- feature toggles such as backend-only, Tailwind, MCP, Prisma, and TypeScript
- component scan directories
- `excludeFiles`, which protects customized files from overwrite during updates

Use `excludeFiles` in `caspian.config.json` to prevent the update command from overwriting files you have customized.

This is useful for protecting:

- stylesheets
- configuration files
- entry-point files
- other locally modified framework-managed files

If you exclude a file, the updater preserves it, but you are responsible for merging future framework changes into that file manually.

## 10. Operational Notes

1. On Windows, the updater resolves `npx.cmd` instead of plain `npx`, which makes execution more reliable on Win32.
2. The updater still requires `caspian.config.json` in the current directory before running an update.
3. The creator reuses an existing `.venv` if one is already present instead of recreating it every time.
4. Starter kits still support later overrides via explicit CLI flags.
5. Frontend-oriented flags are only fully meaningful when `backendOnly` is false.

## AI Routing Notes

If an AI agent is deciding which command flow to use, apply these rules first.

- Use `npx create-caspian-app ...` only when the user is creating a new Caspian project.
- Use `npx casp update project ...` only when the user is already inside an existing Caspian project.
- Add `-y` when the user wants automation, CI, or a no-prompt flow.
- Treat `--tailwindcss` and `--typescript` as frontend-oriented flags. They are not meaningful in normal backend-only scaffolds.
- Treat starter kit defaults as a base layer that can be overridden by explicit flags.
- Read `caspian.config.json` before running update commands.
- When `prisma/schema.prisma` changes, run `npx prisma migrate dev` first, then any required seed refresh, then `npx ppy generate`.
- Never hand-edit generated Python ORM files under `src/lib/prisma/` or `settings/prisma-schema.json`.
- Read [database.md](./database.md) before generating Prisma schema, migration, seed, or ORM guidance.
- Read [routing.md](./routing.md) before generating or modifying route folders under `src/app/`.
- Read [project-structure.md](./project-structure.md) before placing generated files into the project.

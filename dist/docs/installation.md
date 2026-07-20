---
title: Installation
description: Use this page when creating a new Caspian application, scaffolding a project, running first-time setup, or choosing optional features during install.
related:
  title: Related docs
  description: Continue with the routing, structure, and MCP guides after scaffold so the new app follows Caspian conventions and keeps optional FastMCP files in the right place.
  links:
    - /docs/commands
    - /docs/mcp
    - /docs/file-uploads
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page documents the first-time Caspian setup flow for new applications.

## Overview

Caspian provides a scaffold flow for new apps with a FastAPI backend and a PulsePoint-based reactive frontend workflow.

These packaged docs can exist inside an already-created Caspian project, but scaffold commands themselves are typically resolved through external `npx` packages rather than from this docs directory.

## Default App Stack

The scaffolded Caspian baseline is:

- PulsePoint for reactive frontend behavior.
- `@rpc()` plus `pp.rpc()` for browser-triggered data fetching and mutations.
- `casp.validate` with `Validate` and `Rule` for input validation and sanitization.

## System Requirements

Install the supported runtimes before creating a project.

- Node.js with `npm` and `npx` available
- Python `v3.14.0` or newer

You can verify your environment with:

```bash
node -v
python -V
```

## Create a New App

Run the scaffold command:

```bash
npx create-caspian-app@latest
```

The interactive wizard walks through the main project options, including:

- Project name
- Feature toggles such as backend-only mode, Tailwind CSS, Prisma, MCP, and TypeScript
- Other scaffold options exposed by the current CLI version

After scaffold, read `caspian.config.json` and treat it as the single source of truth for which optional features are enabled in that project. Use feature-specific docs only after the matching flag is confirmed as enabled.

If a feature is disabled and the user wants it later, ask whether they want to enable it first, then update `caspian.config.json` and run `npx casp update project` so framework-managed files align with the new feature set.

If the project enables MCP, use `mcp.md` after scaffold to place the app-owned FastMCP server and config files correctly for that project's conventions.

## Recommended VS Code Setup

For the best development experience, use these VS Code extensions:

- [Caspian Official Framework Support](https://marketplace.visualstudio.com/items?itemName=JeffersonAbrahamOmier.caspian)
- [Python](https://marketplace.visualstudio.com/items?itemName=ms-python.python) — ships [Pylance](https://marketplace.visualstudio.com/items?itemName=ms-python.vscode-pylance) as the default language server, which is Pyright under the hood and therefore agrees with the `npm run check` gate (recommended).
- [Pyright](https://marketplace.visualstudio.com/items?itemName=ms-pyright.pyright) — optional if you want the standalone Pyright server and CLI on top of Pylance; otherwise Pylance already covers editor parity.
- [Prisma](https://marketplace.visualstudio.com/items?itemName=Prisma.prisma)
- [Tailwind CSS IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss)

The Caspian extension is the key piece for Python component snippets and autocomplete.

## Start the Development Server

After scaffolding, move into the generated project and start the development server:

```bash
cd my-app
npm run dev
```

Confirm what `npm run dev` does in the generated project's `package.json`. Many Caspian projects use BrowserSync plus PostCSS watchers rather than a Vite dev server, but the actual script wins.

For AI agents and other automated helpers, this is an opt-in local-stack command, not a default validation step. Do not run `package.json` scripts just because a route, feature, or doc changed.

If `npm run dev` is intentionally running, let that stack own generated outputs such as `public/css/styles.css`, `settings/component-map.json`, `settings/files-list.json`, `__pycache__/`, and `.pyc` files. Use `npm run build` only for deployment prep or when the user explicitly asks for a build.

Inspect `settings/component-map.json` and `settings/files-list.json` when you need the generated component or route inventory, but do not hand-edit them. The framework refreshes them from `settings/component-map.ts` and `settings/files-list.ts`.

If runtime uploads write into a public upload directory, keep that directory in `settings/bs-config.ts` `PUBLIC_IGNORE_DIRS` so BrowserSync does not reload on every uploaded file.

## After Setup

Once the project is scaffolded:

- Read `database.md` when the app includes Prisma ORM support or needs database changes.
- Read `mcp.md` when the app includes MCP support or needs FastMCP server, config, or launch-flow changes.
- Read `auth.md` before choosing public-vs-private route mode, wiring session config, sign-in or signout flows, route guards, or OAuth providers.
- Read `pulsepoint.md` before generating interactive frontend behavior.
- Read `fetch-data.md` before adding browser-triggered reads, writes, uploads, or streams.
- Read `file-uploads.md` before building file pickers, media libraries, or Prisma-backed file-manager flows.
- Read `validation.md` before handling forms, auth input, or RPC payloads.
- Read `routing.md` to learn how `src/app` folders map to URLs.
- Read `project-structure.md` to place route code, shared libraries, config, and database files in the correct directories.

## AI Retrieval Notes

If an AI agent is reading this page, treat it as the source for new-project installation steps.

- Use this workflow when the user is creating a Caspian app from scratch.
- Do not use existing-project migration or update commands unless the project already exists.
- After scaffold, default to PulsePoint for interactive UI, RPC for browser-to-server data flow, and `casp.validate` for validation.
- Treat `package.json` scripts as opt-in commands after scaffold. Do not auto-run `npm run dev` or `npm run build` for ordinary source edits.
- Read `caspian.config.json` after scaffold and treat it as the single source of truth for optional features before using feature-specific docs.
- If an optional feature is disabled and the user wants it, ask first, then update `caspian.config.json` and use `npx casp update project` before assuming the framework-managed files exist.
- If MCP is enabled, read [mcp.md](./mcp.md) before editing FastMCP files or assuming where the server config should live.
- Once the app exists, check [routing.md](./routing.md) before creating or changing routes under `src/app/`.
- Check [index.md](./index.md) first when deciding which local doc to follow.

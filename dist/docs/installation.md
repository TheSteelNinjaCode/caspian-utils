---
title: Installation
description: Learn how to create a new Caspian application so AI agents use the first-time setup flow instead of assuming an existing project is already in place.
related:
  title: Related docs
  description: Continue with the routing, structure, and MCP guides after scaffold so the new app follows Caspian conventions and keeps optional FastMCP files in the right place.
  links:
    - /docs/commands
    - /docs/mcp
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page documents the first-time Caspian setup flow for new applications.

## Overview

Caspian provides a scaffold flow for new apps with a FastAPI backend and a PulsePoint-based reactive frontend workflow.

In this workspace, the local `caspian-utils` package ships documentation only, so scaffold commands are resolved through external `npx` packages rather than project-local binaries.

## Default App Stack

The scaffolded Caspian baseline is:

- PulsePoint for reactive frontend behavior.
- `@rpc()` plus `pp.rpc()` for browser-triggered data fetching and mutations.
- `casp.validate` with `Validate` and `Rule` for input validation and sanitization.

## System Requirements

Install the supported runtimes before creating a project.

- Node.js with `npm` and `npx` available
- Python `v3.11.0` or newer

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

If the project enables MCP, use `mcp.md` after scaffold to place the app-owned FastMCP server and config files correctly for the current workspace conventions.

## Recommended VS Code Setup

For the best development experience, use these VS Code extensions:

- [Caspian Official Framework Support](https://marketplace.visualstudio.com/items?itemName=JeffersonAbrahamOmier.caspian)
- [Python](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
- [Pyright](https://marketplace.visualstudio.com/items?itemName=ms-pyright.pyright)
- [Pyrefly](https://marketplace.visualstudio.com/items?itemName=tamasfe.pyrefly)
- [Prisma](https://marketplace.visualstudio.com/items?itemName=Prisma.prisma)
- [Tailwind CSS IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss)

The Caspian extension is the key piece for Python component snippets and autocomplete.

## Start the Development Server

After scaffolding, move into the generated project and start the development server:

```bash
cd my-app
npm run dev
```

In this workspace, `npm run dev` is backed by BrowserSync plus PostCSS watchers, not a Vite dev server.

## After Setup

Once the project is scaffolded:

- Read `database.md` when the app includes Prisma ORM support or needs database changes.
- Read `mcp.md` when the app includes MCP support or needs FastMCP server, config, or launch-flow changes.
- Read `auth.md` before choosing public-vs-private route mode, wiring session config, sign-in or signout flows, route guards, or OAuth providers.
- Read `pulsepoint.md` before generating interactive frontend behavior.
- Read `fetch-data.md` before adding browser-triggered reads, writes, uploads, or streams.
- Read `validation.md` before handling forms, auth input, or RPC payloads.
- Read `routing.md` to learn how `src/app` folders map to URLs.
- Read `project-structure.md` to place route code, shared libraries, config, and database files in the correct directories.

## AI Routing Notes

If an AI agent is reading this page, treat it as the source for new-project installation steps.

- Use this workflow when the user is creating a Caspian app from scratch.
- Do not use existing-project migration or update commands unless the project already exists.
- After scaffold, default to PulsePoint for interactive UI, RPC for browser-to-server data flow, and `casp.validate` for validation.
- If MCP is enabled, read [mcp.md](./mcp.md) before editing FastMCP files or assuming where the server config should live.
- Once the app exists, check [routing.md](./routing.md) before creating or changing routes under `src/app/`.
- Check [index.md](./index.md) first when deciding which local doc to follow.

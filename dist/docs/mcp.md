---
title: MCP
description: Use the workspace FastMCP setup through src/lib/mcp/mcp_server.py, src/lib/mcp/fastmcp.json, and settings/restart-mcp.ts, with npm-driven startup and explicit manual paths when running FastMCP directly.
related:
  title: Related docs
  description: Start with project structure for file placement, then use commands for local run flows and index for documentation routing.
  links:
    - /docs/project-structure
    - /docs/commands
    - /docs/fetch-data
    - /docs/index
---

This page documents the Model Context Protocol workflow present in this workspace.

The current repo enables MCP in `caspian.config.json`, keeps the app-owned FastMCP server at `src/lib/mcp/mcp_server.py`, keeps the FastMCP config at `src/lib/mcp/fastmcp.json`, and starts it through `npm run mcp`, which runs `settings/restart-mcp.ts`.

## Overview

Use these rules as the default MCP workflow in this workspace:

1. Read `caspian.config.json` and confirm `mcp: true` before assuming MCP files should exist or be started.
2. Edit `src/lib/mcp/mcp_server.py` when changing FastMCP tools, server instructions, or server-owned helper logic.
3. Edit `src/lib/mcp/fastmcp.json` when changing the transport, host, port, path, or server entrypoint.
4. Edit `settings/restart-mcp.ts` when changing discovery order, environment overrides, or MCP log filtering.
5. Use `npm run mcp` as the normal local launch flow.

## Current Workspace Layout

- `src/lib/mcp/mcp_server.py` defines `mcp = FastMCP(...)` and the current workspace tools.
- `src/lib/mcp/fastmcp.json` points to `src/lib/mcp/mcp_server.py` and configures `streamable-http` on `127.0.0.1:5101/mcp`.
- `settings/restart-mcp.ts` is the npm-facing launcher. It resolves MCP config paths, starts FastMCP, and trims noisy banner or warning output down to essential lines.
- `package.json` defines `npm run mcp` as `tsx settings/restart-mcp.ts` and includes `mcp` in `npm run dev`.
- `caspian.config.json` is the feature gate. If `mcp` is false, the MCP runner exits cleanly without starting a server.

## Current Tools

The app-owned MCP server currently exposes these read-only tools:

- `project_info` returns the project name, root path, version metadata, browser sync target, feature flags, and component scan directories.
- `workspace_files` returns the generated workspace file list, optionally filtered to `all`, `app`, or `public`.
- `component_inventory` returns the latest generated component map from `settings/component-map.json`.

Add or change tools in `src/lib/mcp/mcp_server.py`.

## Running The Server

### Preferred local commands

```bash
npm run mcp
npm run dev
```

Use `npm run mcp` when you want the MCP server only. Use `npm run dev` when you want BrowserSync, Tailwind, the Python app server, and MCP together.

### Direct FastMCP commands

On this Windows workspace, the direct FastMCP commands are:

```powershell
.venv\Scripts\fastmcp.exe inspect src/lib/mcp/fastmcp.json
.venv\Scripts\fastmcp.exe run src/lib/mcp/fastmcp.json --no-banner
```

Because the config file lives under `src/lib/mcp/`, plain `fastmcp run` from the project root does not auto-detect it. Use `npm run mcp` or pass the explicit config path.

## Config And Discovery Rules

`settings/restart-mcp.ts` currently resolves MCP server specs in this order:

1. `MCP_SERVER_SPEC`
2. `src/lib/mcp/fastmcp.json`
3. `fastmcp.json`
4. `mcp.json`

The runner also forwards these optional overrides to FastMCP:

- `MCP_TRANSPORT`
- `MCP_HOST`
- `MCP_PORT`
- `MCP_PATH`
- `MCP_LOG_LEVEL`

Keep the `source.path` value in `src/lib/mcp/fastmcp.json` root-relative as `src/lib/mcp/mcp_server.py`. Do not shorten it to `mcp_server.py` unless the launch working directory changes too, because FastMCP resolves that path from the current working directory.

## AI Routing Notes

If an AI tool is working on MCP in this workspace, use this order:

1. Read `caspian.config.json` to confirm `mcp: true`.
2. Read `node_modules/caspian-utils/dist/docs/mcp.md` for the current workspace MCP rules.
3. Read `src/lib/mcp/mcp_server.py` for tool definitions and server instructions.
4. Read `src/lib/mcp/fastmcp.json` for transport and entrypoint configuration.
5. Read `settings/restart-mcp.ts` and `package.json` for local start behavior.

Default placement rules:

- Put app-owned FastMCP tools in `src/lib/mcp/mcp_server.py`.
- Keep the default FastMCP config in `src/lib/mcp/fastmcp.json`.
- Keep npm-facing launch behavior in `settings/restart-mcp.ts`.
- If the MCP file layout changes, update this page, `AGENTS.md`, `.github/copilot-instructions.md`, and any workspace runner logic together.

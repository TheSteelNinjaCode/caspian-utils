---
title: MCP
description: Check `caspian.config.json` first. When it enables MCP, this page becomes the relevant guide for reviewing, fixing, launching, and extending the workspace MCP feature.
related:
  title: Related docs
  description: Start with project structure for file placement, then use commands for feature enablement and local run flows, and index for documentation routing.
  links:
    - /docs/project-structure
    - /docs/commands
    - /docs/fetch-data
    - /docs/index
---

This page documents the Model Context Protocol workflow for Caspian workspaces.

Treat `caspian.config.json` as the single source of truth for whether MCP is enabled. If `caspian.config.json` confirms `"mcp": true`, this page is relevant, the MCP-specific files in the workspace are the correct files to analyze, and the feature can be reviewed, fixed, launched, or extended here.

If `caspian.config.json` has `"mcp": false`, treat this page as reference material only. Do not assume `src/lib/mcp/**`, `settings/restart-mcp.ts`, or MCP-related npm scripts exist until the user chooses to enable MCP and the update workflow has run.

## Enablement Workflow

1. Read `caspian.config.json` and confirm `mcp: true` before assuming MCP files or scripts should exist.
2. If `mcp` is false and the user wants MCP, ask for confirmation first.
3. After the user confirms, update `caspian.config.json` and run `npx casp update project` so framework-managed MCP files align with the new feature set.
4. After the update, inspect the actual `package.json`, `src/lib/mcp/`, and any launcher files that the workspace now contains.

## MCP-Enabled Workspace Layout

When `caspian.config.json` has `mcp: true`, these are the main MCP surfaces to inspect:

- `src/lib/mcp/mcp_server.py` for the app-owned FastMCP server, tool definitions, instructions, and MCP-specific helper logic.
- `src/lib/mcp/fastmcp.json` for the default FastMCP config, transport, host, port, path, and server entrypoint.
- `settings/restart-mcp.ts` for workspace-specific launcher logic, discovery order, environment overrides, or log filtering when that file is present.
- `package.json` for the actual MCP-related scripts that the current workspace defines.

If these files exist in the workspace, they are the right files to analyze when reviewing or fixing MCP behavior.

## What To Review

When `mcp: true` and an MCP issue needs investigation, inspect the files in this order:

1. `caspian.config.json` to confirm the feature is enabled.
2. `package.json` to see which MCP scripts the workspace actually exposes.
3. `src/lib/mcp/mcp_server.py` for tool implementation and server behavior.
4. `src/lib/mcp/fastmcp.json` for config and entrypoint details.
5. `settings/restart-mcp.ts` when the workspace includes it and launcher behavior is part of the issue.

## Running MCP

Only when `caspian.config.json` has `mcp: true` and the relevant files or scripts exist:

- Use the workspace-defined `npm run mcp` command when `package.json` provides it.
- Use `npm run dev` only if the workspace wires MCP into the full local stack and the user explicitly wants that full stack running.
- Use direct FastMCP commands against `src/lib/mcp/fastmcp.json` when that config file exists, for example:

```powershell
.venv\Scripts\fastmcp.exe inspect src/lib/mcp/fastmcp.json
.venv\Scripts\fastmcp.exe run src/lib/mcp/fastmcp.json --no-banner
```

Because script names and launcher wiring can vary by workspace version, always confirm the actual scripts in `package.json` instead of assuming them from docs alone.

## Current Workspace Status

- `caspian.config.json` currently has `mcp: false`.
- `package.json` does not currently define `npm run mcp`.
- The current repo tree should not be treated as if `src/lib/mcp/**` is guaranteed to exist.

## AI Routing Notes

1. Read `caspian.config.json` first.
2. If `mcp` is false, do not infer MCP files, scripts, or launch flow from generic Caspian examples.
3. If the user wants MCP while it is disabled, ask first, then update `caspian.config.json` and use `npx casp update project` before continuing.
4. If `mcp` is true, this page becomes the relevant MCP guide and the MCP-specific files in the workspace become the correct analysis surface.
5. Inspect the actual generated files and scripts in the workspace before editing or starting anything.

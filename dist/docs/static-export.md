---
title: Static Export
description: Use this guide when a Caspian app is exported to static HTML (SSG, like Next.js `output: export`) and previewed over HTTP. Use when the task mentions `npm run static`, `npm run static:serve`, `settings/build-static.py`, `settings/serve-static.py`, static-site generation, `static_paths`, a static preview server, or "the default port is already in use".
related:
  title: Related docs
  description: Static export walks the route/component index and boots the app, so pair it with routing, components, commands, and the metadata guides.
  links:
    - /docs/commands
    - /docs/routing
    - /docs/components
    - /docs/metadata
    - /docs/project-structure
    - /docs/testing
    - /docs/index
---

This page covers exporting a Caspian app to a folder of static HTML (SSG, the equivalent of Next.js `output: export`) and previewing that folder over HTTP with a robust, port-aware local server.

## When This Doc Applies

Read this doc when the task is about:

- exporting the app to static HTML for a static host (Netlify, Vercel, GitHub Pages, nginx, S3, any CDN),
- previewing the exported `static/` folder over HTTP locally,
- pre-rendering dynamic routes for a static build,
- troubleshooting a failing static build or a preview server whose port is already in use.

## Feature Status And Gate

Static export is an **app-owned build convention layered on top of Caspian**, the same category as the `npm run check` quality gate in [testing.md](./testing.md). The framework itself ships no static exporter and no preview server. There is **no `caspian.config.json` flag** for it.

Because it is not a shipped feature, do not assume it exists in every Caspian project. Confirm it in the project first:

- `package.json` defines the `static` and `static:serve` scripts, and
- `settings/build-static.py` (the exporter) and `settings/serve-static.py` (the preview server) are present.

If those are absent, the project has not adopted static export; do not invent the scripts unless the user asks you to add them.

When `caspian.config.json` has `tailwindcss: true`, the export compiles Tailwind as part of the build (see the build order below). In `backendOnly` projects a static HTML export is usually not meaningful.

## The Two Commands

```bash
# 1. Export the app to static/  (build metadata + Tailwind, then render every static route)
npm run static

# 2. Preview the exported static/ folder over HTTP (robust, auto-selects a free port)
npm run static:serve
```

### `npm run static` — the exporter

The script is composed so the export always runs against a **fresh route/component index**:

```
"static": "npm run build && uv run python settings/build-static.py"
```

`npm run build` is `tailwind:build` **plus** `projectName`. The `projectName` step (`settings/project-name.ts`) regenerates `settings/files-list.json` and `settings/component-map.json` and clears stale `.casp/` and `caches/`. This matters: `settings/build-static.py` boots the app and iterates `get_files_index()`, which reads `settings/files-list.json` — **the route index that decides what gets exported**. If the export ran after only `tailwind:build` (skipping `projectName`), a newly added route or component could be missing from the index and silently never exported, and a removed one could error. Always regenerate metadata before exporting; reuse `npm run build` rather than duplicating just the Tailwind half.

`settings/build-static.py` then:

- boots the real ASGI app in-process and requests every route through Starlette's `TestClient`, so exported HTML is byte-identical to what the dev server serves (layouts, components, PulsePoint deferral, security headers — the whole pipeline),
- writes each page to `static/<route>/index.html` (`/` → `static/index.html`),
- mirrors public asset trees (`css`, `js`, `assets`, `uploads`, `favicon.ico`) into `static/`,
- runs in `APP_ENV=development` so the build does not require production secrets.

**Scope policy is "warn & skip"** — anything that cannot be fully static is reported and NOT written, so nothing broken ships silently:

- **Dynamic routes** (`[id]` / `[...slug]`) are skipped unless their `index.py` exports `static_paths` — Caspian's `getStaticPaths` equivalent. It may be a list of dicts (`[{"id": 1}]`), a list of scalars (`[1, 2]`, mapped onto the route's single param), or a sync/async callable returning either.
- **Auth-gated routes** redirect instead of returning a page, so they are skipped.
- **Non-200 or non-HTML** responses are skipped.

**Caveats that hold for ANY static build of a Caspian app** (state them when a user exports an interactive app): `pp.rpc()` server actions, auth/sessions, WebSockets, streaming, and per-request server data do NOT work without the Python backend. Pages that depend on them still render, but those interactions are dead in the output; the exporter flags pages that appear to use `pp.rpc()`. Asset URLs are root-absolute (`/css/...`, `/js/...`), so serve `static/` at a domain root or rewrite the base path for a subdirectory deploy.

### `npm run static:serve` — the preview server

```
"static:serve": "uv run python settings/serve-static.py"
```

`settings/serve-static.py` is a hardened local preview server for the exported folder. It exists specifically so an occupied port never aborts the preview — it replaces the fragile `python -m http.server ... 8000` one-liner, which crashes with `address already in use` the moment the port is taken. Its behavior:

- **Auto-selects a free port.** It tries a preferred start port (default `8000`) and walks upward until one binds, then prints the actual URL it landed on. Detection is by genuine bind (no probe race), and it disables `SO_REUSEADDR` so that on Windows a port another process is actively listening on truly fails to bind instead of silently colliding.
- **Serves only `static/`** — no traversal outside the exported folder — and adds `X-Content-Type-Options: nosniff` and `Cache-Control: no-store`.
- **Binds loopback `127.0.0.1` by default**, so the preview is not exposed to the local network. Exposing is opt-in via `HOST=0.0.0.0` (only then does it print the LAN URL).
- **Fails fast** with a clear "build it first" message if `static/` was never exported.
- **Shuts down cleanly** on Ctrl+C.

Environment overrides (all optional): `HOST` (bind address), `PORT` (preferred start port), `PORT_TRIES` (how many ports to try, default 50). Invalid values fall back to the defaults with a warning instead of crashing.

Preview over HTTP — do **not** double-click `static/index.html` (`file://`): root-absolute asset paths break and browsers block ES module scripts from `file://` origins.

> The preview server's port has nothing to do with the dev BrowserSync ports in `./settings/bs-config.json`. `bs-config.json` is the source of truth for the running **dev** stack; the static preview is a separate process on its own auto-selected port. Read the port the serve command actually prints, not `bs-config.json`.

## Files AI Usually Inspects

- `package.json` — confirm the `static` / `static:serve` scripts and their composition (the export must run `npm run build`, not just `tailwind:build`).
- `settings/build-static.py` — the exporter: route walking, `static_paths` resolution, skip policy, asset copy.
- `settings/serve-static.py` — the preview server: port-walk, loopback binding, headers, env overrides.
- `settings/project-name.ts` — regenerates `settings/files-list.json` and `settings/component-map.json` that the exporter walks.
- `src/app/**/index.py` — add `static_paths` here to pre-render a dynamic route.
- `main.py` and the installed `casp` runtime — own the actual render pipeline the exporter drives via `TestClient`; see [core-runtime-map.md](./core-runtime-map.md).

## Verify Before Editing Or Explaining

- Confirm the scripts and both `settings/*.py` files exist before describing static export as available in the project.
- Confirm the `static` script runs `npm run build` (metadata + Tailwind) before `settings/build-static.py`; a build that only runs `tailwind:build` exports from a stale route index.
- For any dynamic route the user expects in the output, confirm its `index.py` exports `static_paths`; otherwise it is skipped by design.
- Read the port the serve command prints; do not assume `8000` or read `settings/bs-config.json` for it.
- Treat `static/`, `settings/files-list.json`, and `settings/component-map.json` as generated outputs — do not hand-edit them.

## AI Retrieval Notes

- Use this doc when the task names static export, SSG, `output: export`, `npm run static`, `npm run static:serve`, `static_paths`, a static preview server, or a "port already in use" preview problem.
- Static export is app-owned tooling, not a shipped, config-gated Caspian feature — verify the scripts exist rather than assuming them.
- To pre-render dynamic routes, add `static_paths` to the route's `index.py` (the `getStaticPaths` equivalent); read [routing.md](./routing.md) for route/segment shape.
- Warn users that `pp.rpc()`, auth, WebSockets, streaming, and per-request data are inert in a static export; those need the Python backend.
- The preview server auto-selects a free port and binds loopback by default; exposing on the network (`HOST=0.0.0.0`) is opt-in.
- See [commands.md](./commands.md) for how `static` relates to `npm run build`, and [testing.md](./testing.md) for the sibling app-owned-convention pattern.

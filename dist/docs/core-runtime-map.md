---
title: Core Runtime Map
description: Use this page when the task touches `main.py`, installed `casp` internals, or when AI needs to know which core file controls a behavior and which packaged doc explains it.
related:
  title: Related docs
  description: Start with the structure guide, then jump from the runtime file to the matching feature doc.
  links:
    - /docs/project-structure
    - /docs/routing
    - /docs/auth
    - /docs/fetch-data
    - /docs/websockets
    - /docs/pulsepoint
    - /docs/pulsepoint-runtime-map
    - /docs/index
---

This page maps the app entry point and the installed Caspian runtime modules to the packaged docs that explain them.

Use it when you already know a behavior is controlled by `main.py` or `.venv/Lib/site-packages/casp/**` and you need the fastest path from symptom to source file.

## When To Read This Page

- a packaged doc mentions a runtime contract and you need the exact file that implements it
- `main.py` or the installed `casp` package disagrees with older notes or examples
- the task is about framework internals rather than normal app-owned route or component code
- AI needs to know which packaged doc should be kept aligned with a core-file change

## App Entry Point

| Concern | Core file | Read first | Why it matters |
| --- | --- | --- | --- |
| App bootstrap and request flow | [main.py](../../../../main.py) plus imported runtime helpers such as `casp.runtime_security` | [project-structure.md](./project-structure.md), [routing.md](./routing.md), [auth.md](./auth.md) | FastAPI app creation, middleware wiring, route registration, cache check and save, exception handlers, and final HTML transforms live here. Package-owned helpers imported by `main.py` may own safe public-file serving, baseline response headers including the Content-Security-Policy, production-safe error messages, fail-closed environment resolution, or production session-secret enforcement. |
| Browser runtime, SPA navigation, and scroll restoration | [public/js/pp-reactive-v2.min.js](../../../../public/js/pp-reactive-v2.min.js) | [pulsepoint.md](./pulsepoint.md), [fetch-data.md](./fetch-data.md) | The shipped `pp` runtime, same-origin SPA interception, per-history-entry scroll state, `pp-reset-scroll` behavior, and browser RPC helpers live here. |

Important current `main.py` behaviors AI should keep in mind:

- In development, `_scoped_cookie_name(...)` appends the active BrowserSync or dev port to both the session cookie name and the CSRF cookie name. The scope is resolved from `CASPIAN_BROWSER_SYNC_PORT`, then from the `local` URL in `settings/bs-config.json` (only when its host is `localhost` or `127.0.0.1`). There is no further fallback: a non-numeric or unresolved scope yields `""`, so the unsuffixed base names are used. `PORT` is not consulted.
- `register_single_route(...)` passes path params to `page()` as one positional dict, injects matching query params by name, and injects `request` by keyword when declared.
- The render pipeline transforms components, renders nested layouts, and finally defers outermost component roots inside inert templates.
- Route-level generators returned from `page()` are wrapped in `SSE(...)` before the response is sent.
- Safe public-file helpers live in `casp.runtime_security`. `PublicFilesMiddleware` maps every existing nested file under `public/**` to the same root-relative URL without per-directory routes, rejects traversal and symlink escape, handles only `GET`/`HEAD`, and falls through when no file exists so page and mounted-app routing still works. User-upload directories must retain their restricted inline-media policy.
- Session middleware secrets may be resolved through `casp.runtime_security` so production can fail fast when `AUTH_SECRET` is missing or still on a default placeholder.
- Baseline response headers are built by `casp.runtime_security` and attached through `SecurityHeadersMiddleware`. That set now includes a Content-Security-Policy from `build_content_security_policy()`, which the `CONTENT_SECURITY_POLICY` environment variable replaces wholesale. The policy must keep `'unsafe-eval'` and `'unsafe-inline'` in `script-src`, because the PulsePoint runtime compiles templates with `new Function`; read the current helper before tightening it. Outside production the helper also adds loopback origins to `connect-src` so the BrowserSync live-reload client, which polls a different port than the proxied page, is not blocked; an environment override replaces that too.
- Middleware is added in source order as `RPCMiddleware`, `AuthMiddleware`, `CSRFMiddleware`, `SessionMiddleware`, `BodySizeLimitMiddleware`, `RateLimitMiddleware`, `PublicFilesMiddleware`, `SecurityHeadersMiddleware`, plus `RequestDiagnosticsMiddleware` outside production. The effective production request order is reversed: security headers, public files, rate limit, body limit, session, CSRF, auth, then RPC and routing. Development diagnostics are outermost. Existing public-file `GET`/`HEAD` requests stop at `PublicFilesMiddleware`; missing paths fall through. Verify the current list in `main.py` rather than trusting an older order.
- `public/js/pp-reactive-v2.min.js` saves scroll positions per history entry, resets window scroll on push navigation, and uses `pp-reset-scroll="true"` to opt specific containers or the whole body into reset behavior.

## Caspian Core Feature Map

Use this table when the task names a framework feature but the owning file is not obvious yet.

| Core feature | App-owned entry points | Runtime owner | Packaged docs |
| --- | --- | --- | --- |
| Feature flags and generated surface area | `caspian.config.json` | `casp.caspian_config` | [commands.md](./commands.md), [project-structure.md](./project-structure.md) |
| File-based routing | `src/app/**/index.py` | `main.py`, `casp.layout`, `casp.caspian_config` | [routing.md](./routing.md) |
| Nested layouts | `src/app/**/layout.py` | `casp.layout`, `main.py` | [routing.md](./routing.md), [metadata.md](./metadata.md) |
| Metadata | route or layout `metadata`, runtime metadata helpers | `casp.layout`, `main.py` | [metadata.md](./metadata.md), [routing.md](./routing.md) |
| Component imports and `x-*` tags | `src/components/**`, `src/app/**/*.html` | `casp.components_compiler`, `casp.component_decorator` | [components.md](./components.md), [routing.md](./routing.md) |
| Auth and route protection | `src/lib/auth/auth_config.py`, `main.py`, route decorators | `casp.auth`, `main.py` middleware | [auth.md](./auth.md) |
| Baseline response headers and safe public-file serving | `main.py` imports from `casp.runtime_security` | `.venv/Lib/site-packages/casp/runtime_security.py` | [auth.md](./auth.md), [project-structure.md](./project-structure.md) |
| RPC and server actions | route or component Python modules with `@rpc()` | `casp.rpc`, `main.py` middleware | [fetch-data.md](./fetch-data.md), [file-uploads.md](./file-uploads.md) |
| Streaming | route `page()` generators, RPC generators | `casp.streaming`, `casp.rpc`, `main.py` | [fetch-data.md](./fetch-data.md) |
| WebSockets (named sockets) | `src/lib/websocket/sockets.py` when `caspian.config.json` has `websocket: true`, route- or lib-owned `@socket()` functions, `pp.socket(...)` clients in `src/app/**` component scripts | `src/lib/websocket/sockets.py` owns the whole surface — the `@socket()` registry, `Socket`/`SocketSender`/`SocketPool`, origin allow-list, connection cap, auth delegation to `casp/auth.py`, and per-connection size/rate/idle limits; `main.py` wires the single `@app.websocket(SOCKET_PATH)` endpoint; browser half is `public/js/pp-reactive-v2.min.js` | [websockets.md](./websockets.md), [auth.md](./auth.md), [pulsepoint.md](./pulsepoint.md) |
| Server state | request handlers and RPC actions | `casp.state_manager`, `main.py` middleware | [state.md](./state.md) |
| Page caching | route-level `Cache(...)` declarations | `casp.cache_handler`, `main.py` cache check/save | [cache.md](./cache.md) |
| Validation | route and RPC input boundaries | `casp.validate` | [validation.md](./validation.md) |
| Application timezone and calendar dates | route handlers, RPC actions, and anything asking "what day is it" | `casp.app_time`, resolved once at boot by `main.py` | see the `casp.app_time` entry below |
| Tailwind class merge | Python components and PulsePoint templates | `casp.html_attrs`, browser `twMerge(...)` | [components.md](./components.md), [pulsepoint.md](./pulsepoint.md) |
| Prisma persistence | `prisma/**`, `src/lib/prisma/**`, route/RPC code | generated Prisma and Python clients | [database.md](./database.md), [fetch-data.md](./fetch-data.md) |
| MCP tools | `src/lib/mcp/**` when enabled | app-owned FastMCP server | [mcp.md](./mcp.md), [commands.md](./commands.md) |

For browser-side PulsePoint details, use [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) instead of expanding this Python-runtime map.

## Task-To-File Examples

Protected dashboard route:

1. Check `caspian.config.json` for enabled features such as Prisma and Tailwind.
2. Read [routing.md](./routing.md) for the section layout shape.
3. Read [auth.md](./auth.md) for route privacy and decorators.
4. Update `src/app/dashboard/layout.py` for the shared shell and child `index.py` files for page-specific behavior.
5. Verify auth bootstrap and middleware ownership in `main.py` only if route protection behaves unexpectedly.

Interactive CRUD page:

1. Read [fetch-data.md](./fetch-data.md) for the `page()` plus `@rpc()` split.
2. Read [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) for browser-side `pp.rpc(...)` behavior.
3. Keep first-render data and the page markup in `src/app/**/index.py`, reusable helpers in `src/lib/**`, and database schema changes in `prisma/**`.

## Installed Runtime Map

| Runtime file | Primary responsibility | Read these docs |
| --- | --- | --- |
| [.venv/Lib/site-packages/casp/layout.py](../../../../.venv/Lib/site-packages/casp/layout.py) | nested layout discovery, `layout()` result resolution (string / tuple / dict, sync or async), metadata merge, layout-module component scope stashing, and parser-based `<slot />` replacement | [routing.md](./routing.md), [metadata.md](./metadata.md) |
| [.venv/Lib/site-packages/casp/auth.py](../../../../.venv/Lib/site-packages/casp/auth.py) | `AuthSettings`, route privacy checks, session payloads, OAuth providers, CSRF helper behavior, and redirect logic | [auth.md](./auth.md) |
| [.venv/Lib/site-packages/casp/runtime_security.py](../../../../.venv/Lib/site-packages/casp/runtime_security.py) | safe public-file serving (with optional attachment mode for user uploads), baseline response headers including the CSP, production-safe error messages, fail-closed `APP_ENV` resolution via `is_production_environment()`, and production session-secret enforcement used by `main.py` | [project-structure.md](./project-structure.md), [auth.md](./auth.md) |
| [.venv/Lib/site-packages/casp/rpc.py](../../../../.venv/Lib/site-packages/casp/rpc.py) | `@rpc()` registration, rate limits, request handling, auth-aware action checks, and streamed RPC responses | [fetch-data.md](./fetch-data.md) |
| [.venv/Lib/site-packages/casp/streaming.py](../../../../.venv/Lib/site-packages/casp/streaming.py) | `SSE`, `ServerSentEvent`, and generator-to-event-stream wrapping | [fetch-data.md](./fetch-data.md) |
| [.venv/Lib/site-packages/casp/state_manager.py](../../../../.venv/Lib/site-packages/casp/state_manager.py) | request-scoped state, session bucket persistence, listener lifecycle, and wire-request reset behavior | [state.md](./state.md) |
| [.venv/Lib/site-packages/casp/cache_handler.py](../../../../.venv/Lib/site-packages/casp/cache_handler.py) | `Cache`, cache manifest handling, filename generation, disk-backed HTML storage, and invalidation | [cache.md](./cache.md) |
| [.venv/Lib/site-packages/casp/validate.py](../../../../.venv/Lib/site-packages/casp/validate.py) | `Validate`, `Rule`, sanitization, and file-validation internals | [validation.md](./validation.md) |
| [.venv/Lib/site-packages/casp/component_decorator.py](../../../../.venv/Lib/site-packages/casp/component_decorator.py) | `@component`, the `html(...)` markup entrypoint (with module-scope capture for `x-*` resolution, Jinja-callable imported components, and direct-call composition), and prop normalization before component calls | [components.md](./components.md) |
| [.venv/Lib/site-packages/casp/components_compiler.py](../../../../.venv/Lib/site-packages/casp/components_compiler.py) | `x-*` component resolution (from a component's Python module imports and the scope captured by `html(...)` and layout modules), parent-scope expansion of slot content, single-root validation, and `pp-component` injection | [components.md](./components.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md) |
| [.venv/Lib/site-packages/casp/html_attrs.py](../../../../.venv/Lib/site-packages/casp/html_attrs.py) | `get_attributes(...)`, alias normalization, and the Python-side `merge_classes(...)` contract | [components.md](./components.md) |
| [.venv/Lib/site-packages/casp/caspian_config.py](../../../../.venv/Lib/site-packages/casp/caspian_config.py) | typed config loading, feature-flag reads, `settings/files-list.json` parsing, and route rule derivation | [project-structure.md](./project-structure.md), [commands.md](./commands.md), [routing.md](./routing.md) |
| [.venv/Lib/site-packages/casp/app_time.py](../../../../.venv/Lib/site-packages/casp/app_time.py) | `APP_TIMEZONE` resolution and the calendar helpers built on it: `get_app_timezone()`, `now()`, `today()`, `to_app_time(...)`, `day_bounds_utc(...)`, `as_naive_utc(...)` | see below |

Secondary helpers such as `html_native.py`, `loading.py`, and `string_helpers.py` support the modules above. `html_native.py` owns BeautifulSoup-backed fragment parsing used by component/root transforms and layout slot replacement. Read these helpers only when a higher-level runtime file is still insufficient to explain the behavior you are tracing.

### Application time (`casp.app_time`)

**Never call a bare `datetime.now()` in app code.** With no argument it returns the *server's* local wall clock, which is a deployment accident rather than a decision: the same query answers differently on a developer laptop and on a UTC container, and comparing that naive value against a UTC-stored column shifts every result by the server's offset. Use `casp.app_time` instead.

`APP_TIMEZONE` (an IANA name such as `UTC` or `America/New_York`, defaulting to `UTC`) sets the application **calendar** — which wall-clock day an instant belongs to.

| Function | Returns |
| --- | --- |
| `get_app_timezone()` | The configured `ZoneInfo`. Raises `InvalidAppTimezoneError` on an unknown name |
| `now()` | Timezone-aware current time in the application timezone |
| `today()` | Current calendar date in the application timezone |
| `to_app_time(dt)` | A stored timestamp read back in the application timezone; a naive input is treated as UTC |
| `day_bounds_utc(day)` | The UTC instants bounding one local calendar day, as a **half-open** `[start, end)` range |
| `as_naive_utc(dt)` | Naive UTC, for a driver that rejects aware datetimes |

Three rules hold this together:

- **Storage stays UTC.** The setting changes which day an instant is *reported* under, never the stored value.
- **Absolute time ignores it, deliberately.** Session expiry (`casp.auth`) stays on `datetime.now(timezone.utc)` and cache TTLs (`casp.cache_handler`) on `time.time()`. If either followed the app timezone, moving a deployment to UTC+14 would hand every live session fourteen extra hours.
- **An unknown zone raises rather than falling back to UTC**, and `main.py` resolves the zone once at import so the failure lands at boot instead of on whichever request first formats a date. A silent fallback would leave every date quietly wrong with nothing in the logs to explain it.

Query a local calendar day against UTC-stored columns with `day_bounds_utc`, and pair it with `gte`/`lt` — not `gte`/`lte`. A closed range built on `time.max` double-counts the endpoint shared with the next range, and millisecond-resolution columns can round a late-evening row onto the following day.

`zoneinfo` reads the operating system's tz database, which **Windows does not have**. A deployment or developer machine without one needs the `tzdata` package, or every zone except `UTC` raises `ZoneInfoNotFoundError`.

## Verification Focus

Use these behavior checkpoints when AI needs the fastest verification path for a runtime behavior before widening the search.

| Runtime area | Verify these behaviors |
| --- | --- |
| `main.py` routing and request flow | route registration, path and query injection, generic public-root file handling and middleware placement, session-middleware wiring, response-header middleware, and exception rendering |
| `main.py` WebSockets | `cfg.websocket` gates one route: the named-socket endpoint at `SOCKET_PATH` (`/__pulsepoint/ws`), a pure wiring point into `src/lib/websocket/sockets.py`. That module owns the whole layer: the `@socket()` registry, the origin check run before `accept()`, the connection ceiling, per-socket auth (`require_auth`/`allowed_roles` delegated to `Auth`), and the per-connection limits (message size, rate, idle timeout). The HTTP middleware stack skips `scope["type"] == "websocket"`, so socket auth is enforced by the socket layer, not by `AuthMiddleware`. There are no separate public/private channel endpoints |
| `public/js/pp-reactive-v2.min.js` browser runtime | SPA interception, history-vs-push scroll behavior, `pp-reset-scroll` semantics, and browser-side `pp.rpc()` behavior |
| `casp.auth` | auth settings, signin and signout flow, provider wiring, and page protection behavior |
| `casp.rpc` and streamed RPC responses | middleware interception, CSRF and session expectations, registry behavior, and helper-level RPC contracts |
| `casp.layout` | layout discovery, metadata merge, root handling, and layout rendering rules |
| `casp.components_compiler`, `casp.component_decorator`, `main.py` template-root deferral, and the browser runtime | `x-*` expansion, Python-module-import tag resolution, parent-scope slot resolution, `html(...)` rendering and children-safe handling, deterministic root keys, `pp-component` injection, plain component-script preservation, and safe script capture during materialization |
| `casp.state_manager` | wire vs non-wire reset behavior, request-state persistence assumptions, and AttributeDict access |
| `casp.cache_handler` | filename generation, manifest writes, TTL handling, and invalidation behavior |
| `casp.caspian_config` | config parsing, files index building, and Next.js-style route inventory behavior |
| `casp.app_time` | `APP_TIMEZONE` resolution and its fail-loud behavior on an unknown zone, half-open day bounds, and that session expiry and cache TTLs still resolve in UTC |

When a runtime or packaged-doc change touches one of these areas, verify the relevant behavior in the owning source file before inferring behavior from memory.

## Decision Rule

- Start with the packaged feature doc that matches the task.
- Use this runtime map when that doc needs to point AI to the controlling source file.
- Prefer app code and `main.py` over packaged docs when the task is about the current workspace behavior.
- Do not modify `main.py` or `.venv/Lib/site-packages/casp/**` just because a packaged doc mentions them. Edit those files only when the user asks for a runtime change or when the task is explicitly about framework behavior.

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
    - /docs/pulsepoint
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
| App bootstrap and request flow | [main.py](../../../../main.py) | [project-structure.md](./project-structure.md), [routing.md](./routing.md), [auth.md](./auth.md) | FastAPI app creation, static asset routes, middleware order, route registration, cache check and save, exception handlers, and final HTML transforms all live here. |

Important current `main.py` behaviors AI should keep in mind:

- In development, `_scoped_cookie_name(...)` appends the active BrowserSync or dev port to both the session cookie name and the CSRF cookie name. The scope is resolved from `CASPIAN_BROWSER_SYNC_PORT`, then `settings/bs-config.json`, then `PORT`.
- `register_single_route(...)` passes path params to `page()` as one positional dict, injects matching query params by name, and injects `request` by keyword when declared.
- The render pipeline is `transform_components(...)`, then `render_with_nested_layouts(...)`, then `transform_scripts(...)`.
- Route-level generators returned from `page()` are wrapped in `SSE(...)` before the response is sent.
- Middleware is added in source order as `RPCMiddleware`, `AuthMiddleware`, `CSRFMiddleware`, `SessionMiddleware`, so the effective request order is reversed at runtime.

## Installed Runtime Map

| Runtime file | Primary responsibility | Read these docs |
| --- | --- | --- |
| [.venv/Lib/site-packages/casp/layout.py](../../../../.venv/Lib/site-packages/casp/layout.py) | `render_page(...)`, `render_layout(...)`, nested layout discovery, metadata merge, and the synchronous `layout()` contract | [routing.md](./routing.md), [metadata.md](./metadata.md) |
| [.venv/Lib/site-packages/casp/auth.py](../../../../.venv/Lib/site-packages/casp/auth.py) | `AuthSettings`, route privacy checks, session payloads, OAuth providers, CSRF helper behavior, and redirect logic | [auth.md](./auth.md) |
| [.venv/Lib/site-packages/casp/rpc.py](../../../../.venv/Lib/site-packages/casp/rpc.py) | `@rpc()` registration, rate limits, request handling, auth-aware action checks, and streamed RPC responses | [fetch-data.md](./fetch-data.md) |
| [.venv/Lib/site-packages/casp/streaming.py](../../../../.venv/Lib/site-packages/casp/streaming.py) | `SSE`, `ServerSentEvent`, and generator-to-event-stream wrapping | [fetch-data.md](./fetch-data.md) |
| [.venv/Lib/site-packages/casp/state_manager.py](../../../../.venv/Lib/site-packages/casp/state_manager.py) | request-scoped state, session bucket persistence, listener lifecycle, and wire-request reset behavior | [state.md](./state.md) |
| [.venv/Lib/site-packages/casp/cache_handler.py](../../../../.venv/Lib/site-packages/casp/cache_handler.py) | `Cache`, cache manifest handling, filename generation, disk-backed HTML storage, and invalidation | [cache.md](./cache.md) |
| [.venv/Lib/site-packages/casp/validate.py](../../../../.venv/Lib/site-packages/casp/validate.py) | `Validate`, `Rule`, sanitization, and file-validation internals | [validation.md](./validation.md) |
| [.venv/Lib/site-packages/casp/component_decorator.py](../../../../.venv/Lib/site-packages/casp/component_decorator.py) | `@component`, `render_html(...)`, component loading, and prop normalization before component calls | [components.md](./components.md) |
| [.venv/Lib/site-packages/casp/components_compiler.py](../../../../.venv/Lib/site-packages/casp/components_compiler.py) | `@import` parsing, `x-*` component resolution, single-root validation, and `pp-component` injection | [components.md](./components.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md) |
| [.venv/Lib/site-packages/casp/scripts_type.py](../../../../.venv/Lib/site-packages/casp/scripts_type.py) | rewriting authored body `<script>` tags to `type="text/pp"` in rendered HTML | [pulsepoint.md](./pulsepoint.md), [routing.md](./routing.md), [components.md](./components.md) |
| [.venv/Lib/site-packages/casp/html_attrs.py](../../../../.venv/Lib/site-packages/casp/html_attrs.py) | `get_attributes(...)`, alias normalization, and the Python-side `merge_classes(...)` contract | [components.md](./components.md) |
| [.venv/Lib/site-packages/casp/caspian_config.py](../../../../.venv/Lib/site-packages/casp/caspian_config.py) | typed config loading, feature-flag reads, `settings/files-list.json` parsing, and route rule derivation | [project-structure.md](./project-structure.md), [commands.md](./commands.md), [routing.md](./routing.md) |
| [.venv/Lib/site-packages/casp/syntax_compiler.py](../../../../.venv/Lib/site-packages/casp/syntax_compiler.py) | transpiling Caspian `<[[ ... ]]>` and `<template casp-*>` syntax before template render | [components.md](./components.md), [routing.md](./routing.md) |

Secondary helpers such as `html_native.py`, `loading.py`, and `string_helpers.py` support the modules above. Read them only when a higher-level runtime file is still insufficient to explain the behavior you are tracing.

## Verification Focus

Use these behavior checkpoints when AI needs the fastest verification path for a runtime behavior before widening the search.

| Runtime area | Verify these behaviors |
| --- | --- |
| `main.py` routing and request flow | route registration, path and query injection, static asset handling, and exception rendering |
| `casp.auth` | auth settings, signin and signout flow, provider wiring, and page protection behavior |
| `casp.rpc` and streamed RPC responses | middleware interception, CSRF and session expectations, registry behavior, and helper-level RPC contracts |
| `casp.layout` | layout discovery, metadata merge, root handling, and layout rendering rules |
| `casp.components_compiler`, `casp.component_decorator`, `casp.scripts_type`, and template-root injection | `@import` parsing, `x-*` expansion, syntax transpilation, deterministic root keys, `pp-component` injection, and authored-script rewriting to `type="text/pp"` |
| `casp.state_manager` | wire vs non-wire reset behavior, request-state persistence assumptions, and AttributeDict access |
| `casp.cache_handler` | filename generation, manifest writes, TTL handling, and invalidation behavior |
| `casp.caspian_config` | config parsing, files index building, and Next.js-style route inventory behavior |

When a runtime or packaged-doc change touches one of these areas, verify the relevant behavior in the owning source file before inferring behavior from memory.

## Decision Rule

- Start with the packaged feature doc that matches the task.
- Use this runtime map when that doc needs to point AI to the controlling source file.
- Prefer app code and `main.py` over packaged docs when the task is about the current workspace behavior.
- Do not modify `main.py` or `.venv/Lib/site-packages/casp/**` just because a packaged doc mentions them. Edit those files only when the user asks for a runtime change or when the task is explicitly about framework behavior.

---
title: Cache
description: Use this page when the task mentions `Cache`, `CacheHandler`, page caching, route HTML reuse, TTLs, or invalidation in Caspian.
related:
  title: Related docs
  description: Use the routing guide to place route-level cache declarations correctly, then pair cache with fetch-data when deciding what can be safely served from reused HTML.
  links:
    - /docs/core-runtime-map
    - /docs/routing
    - /docs/fetch-data
    - /docs/project-structure
    - /docs/index
---

This page explains the current Caspian page-cache API for route-level HTML caching, disk-backed cache storage, TTL handling, and manual invalidation.

Treat `casp.cache_handler` as the default page-caching layer in Caspian app code. Use it for public, read-heavy routes whose rendered HTML can be safely reused across requests. Do not cache personalized, auth-sensitive, or rapidly changing HTML unless you fully control invalidation and visibility.

## Overview

Caspian exposes page caching through `casp.cache_handler`.

Import the API like this:

```python
from casp.cache_handler import Cache, CacheHandler
```

The current installed implementation lives in `.venv/Lib/site-packages/casp/cache_handler.py`.

Use [core-runtime-map.md](./core-runtime-map.md) when a cache task crosses `main.py` request flow, cache lookup timing, and the installed cache-handler internals.

## Source Of Truth

- Route cache intent belongs in `src/app/**/index.py` through module-level `cache_settings = Cache(...)`.
- Cache lookup and save timing are controlled by `main.py`.
- Disk-backed cache behavior lives in `.venv/Lib/site-packages/casp/cache_handler.py`.
- Cache invalidation after writes usually belongs in the owning route `@rpc()` action or a shared helper under `src/lib/**`.

The real API surface is:

- declarative route config with `Cache(ttl=..., enabled=...)`
- disk-backed HTML lookup with `CacheHandler.serve_cache(...)`
- disk-backed HTML writes with `CacheHandler.save_cache(...)`
- selective invalidation with `CacheHandler.invalidate_by_uri(...)`
- full or targeted resets with `CacheHandler.reset_cache(...)`

Use caching when a route is expensive to render but the final HTML is stable enough to reuse, such as marketing pages, blog indexes, docs pages, and public content pages.

## Default Caching Rule

- Use `Cache(...)` at module scope in a route's `index.py` when you want that route to participate in Caspian's page cache.
- Cache public HTML that is safe to share between visitors.
- Avoid caching personalized section shells such as authenticated dashboards, auth callbacks, request-specific error states, flash-message flows, or any page whose HTML changes per user.
- If a dashboard, account area, admin area, or grouped subtree uses a parent `layout.py`, evaluate each child route separately. Public child pages may still be cacheable, but personalized section layouts generally are not.
- Invalidate cached pages after writes, publishes, deletes, or admin actions that change what those routes should render.
- Treat page caching as HTML reuse, not as a replacement for database caching or RPC response caching.

## Declarative Route Config

The `Cache` dataclass is the project-facing API you define in route code.

Current fields:

- `ttl: int = 600`
- `enabled: bool = True`

Example:

```python
from casp.component_decorator import html
from casp.cache_handler import Cache

cache_settings = Cache(ttl=3600, enabled=True)

async def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")
```

You can also rely on the current auto-registration behavior and call `Cache(...)` bare at module scope:

```python
from casp.component_decorator import html
from casp.cache_handler import Cache

Cache(ttl=7200, enabled=True)

async def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")
```

Implementation notes from the installed file:

- `Cache.__post_init__()` inspects the caller frame.
- When `Cache(...)` is called at module scope, it auto-registers itself as `cache_settings` in that module's globals.
- That magic registration only happens for module-level calls where the caller code object name is `<module>`.

Prefer the explicit `cache_settings = Cache(...)` form in maintained project code because it makes the route intent obvious to readers and tooling.

To disable caching for a route explicitly:

```python
from casp.cache_handler import Cache

cache_settings = Cache(enabled=False)
```

Use this for routes that are public but too volatile or sensitive to serve from cached HTML.

## How The Disk Cache Works

`CacheHandler` stores cached HTML on disk under the current working directory.

Current paths:

- cache directory: `caches/`
- manifest file: `caches/cache_manifest.json`

The request lifecycle is conceptually:

1. A request arrives for a URI.
2. `CacheHandler.serve_cache(uri, default_ttl=600)` checks `cache_manifest.json` for that exact URI.
3. If the manifest entry is missing, the file is missing, or the entry has expired, the request is treated as a cache miss.
4. On a hit, Caspian returns the cached HTML string directly.
5. On a miss, the route renders normally and the framework can persist the rendered HTML with `CacheHandler.save_cache(...)`.

Filename generation is path-based:

- `/` becomes `index.html`
- other paths are lowercased, non-alphanumeric characters are replaced with `_`, and the result is saved as `*.html`

Examples:

- `/docs/cache` becomes `docs_cache.html`
- `/blog/post-1` becomes `blog_post-1.html`

## `CacheHandler` Methods

The installed `CacheHandler` class currently provides these main helpers:

| Method | Purpose | Return value |
| --- | --- | --- |
| `ensure_cache_dir()` | Creates `caches/` and `cache_manifest.json` if needed. | None |
| `serve_cache(uri, default_ttl=600)` | Returns cached HTML when the entry exists and is still fresh. | Cached `str` or `None` |
| `save_cache(uri, content, ttl=0)` | Writes the HTML file and updates the manifest entry. | None |
| `invalidate_by_uri(uris)` | Removes one or more cached URIs and their HTML files. | None |
| `reset_cache(uri=None)` | Removes a specific cached URI or clears the whole cache directory. | None |

Example manual usage:

```python
from casp.cache_handler import CacheHandler

html = CacheHandler.serve_cache("/docs/cache", default_ttl=600)
if html is not None:
    return html

CacheHandler.invalidate_by_uri("/docs/cache")
CacheHandler.invalidate_by_uri([
    "/blog",
    "/blog/python",
])

CacheHandler.reset_cache("/docs/cache")
# Or clear everything:
# CacheHandler.reset_cache()
```

In normal route code, `CacheHandler.serve_cache(...)` and `save_cache(...)` are usually framework-driven internals. App code most often interacts with `Cache(...)` and with invalidation helpers after content changes.

## Invalidate After Mutations

If a cached page depends on mutable content, invalidate it after the write succeeds.

Example:

```python
from casp.rpc import rpc
from casp.cache_handler import CacheHandler

@rpc(require_auth=True)
async def publish_post(slug: str):
    # Save post changes here.

    CacheHandler.invalidate_by_uri([
        "/blog",
        f"/blog/{slug}",
    ])

    return {"success": True}
```

Use this pattern when:

- an RPC action updates a blog post, docs page, or product page
- an admin publish flow changes public route output
- a route index depends on records that were just created or deleted

For browser-triggered writes and invalidation flows, pair this page with [fetch-data.md](./fetch-data.md).

## TTL Behavior

The installed implementation stores a per-entry TTL in the manifest and compares it to the saved timestamp.

Important details:

- `Cache.ttl` defaults to `600` seconds.
- `CacheHandler.save_cache(...)` records the `created_at` timestamp and the provided `ttl` value.
- `CacheHandler.serve_cache(...)` uses the saved TTL when it is greater than `0`.
- If the saved TTL is `0` or negative, `serve_cache(...)` falls back to its `default_ttl` argument.
- Expired entries are invalidated automatically and then treated as cache misses.

This means `ttl=0` does not mean "cache forever" in the current implementation. It means "use the caller's default TTL fallback."

## Upstream Global Config Note

The upstream Caspian cache page also describes application-wide environment settings such as `CACHE_ENABLED` and `CACHE_TTL`.

Those settings are not parsed inside `casp.cache_handler.py` itself. Treat `Cache(...)` as the route-level override layer and treat any environment-driven defaults as framework configuration that is handled elsewhere in the stack for your specific Caspian version.

## Current Implementation Notes

- The manifest file name is currently `cache_manifest.json`, not `manifest.json`.
- `get_filename(uri)` strips the query string before generating the HTML filename.
- The manifest itself is keyed by the exact URI string passed into `save_cache(...)` and `serve_cache(...)`.
- Because filenames ignore the query string, different query variants of the same path can map to the same HTML file name.
- `invalidate_by_uri(...)` normalizes each URI to start with `/`, so use leading-slash URIs consistently.
- `save_cache(...)`, invalidation, and write failures log to stdout with `[Cache] ...` messages.
- `reset_cache()` removes files from `caches/` and then rewrites an empty manifest.
- `Cache(...)` auto-registration only occurs at module scope. Instantiating it inside a function does not declaratively register route cache settings.

## Recommended Usage Pattern

Validate whether the HTML is shareable first, then add cache configuration close to the route boundary.

Common placement patterns are:

- module-level `cache_settings = Cache(...)` inside `src/app/**/index.py`
- explicit invalidation inside `@rpc()` actions or admin flows after a successful write
- occasional maintenance resets through `CacheHandler.reset_cache()` in operational scripts or debugging sessions

Good candidates for page caching are:

- landing pages
- public docs pages
- blog indexes and published article pages
- public category or catalog pages whose HTML changes on a predictable cadence

Poor candidates are:

- authenticated dashboards and other personalized section layouts
- per-user feeds or notification views
- forms whose server-rendered HTML includes request-specific or sensitive state
- pages that need sub-second freshness unless you have reliable invalidation hooks

## AI Retrieval Notes

If an AI agent is deciding whether or how to cache a Caspian route, apply these rules first.

- Use `Cache` from `casp.cache_handler` for route-level page caching.
- Prefer `cache_settings = Cache(...)` at module scope over a bare call when writing maintained project code.
- Use `enabled=False` for pages whose HTML is personalized, auth-sensitive, or too volatile to reuse safely.
- When a user asks about caching a dashboard, admin area, account area, or grouped subtree, check whether the shared `layout.py` is personalized before caching any child route. Shared authenticated shells are usually poor cache candidates even when some public child pages are cacheable.
- Use `CacheHandler.invalidate_by_uri(...)` after writes that change cached public content.
- Use `CacheHandler.reset_cache()` only for targeted maintenance, debugging, or full cache clears.
- Use leading-slash URIs consistently when interacting with `CacheHandler`.
- Treat `caches/` and `caches/cache_manifest.json` as framework cache artifacts, not as source files.
- Pair this page with [fetch-data.md](./fetch-data.md) when deciding how cached first-render HTML relates to interactive RPC updates.
- Check [routing.md](./routing.md) before deciding where a route's cache declaration belongs.

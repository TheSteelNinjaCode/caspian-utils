---
title: Fetch Data
description: Fetch first-render and interactive data in Caspian with async route functions, `@rpc()` actions, `pp.rpc()`, streaming, and uploads, with RPC as the default browser-to-server data path.
related:
  title: Related docs
  description: Use the routing guide to place route logic correctly, then use the auth guide for protected actions, the MCP guide for AI-facing tools, the state guide for transient request-scoped mutation state, the cache guide for reusable first-render HTML, and the PulsePoint runtime guide for client-side `pp.rpc()` details.
  links:
    - /docs/auth
    - /docs/mcp
    - /docs/state
    - /docs/database
    - /docs/file-uploads
    - /docs/cache
    - /docs/routing
    - /docs/pulsepoint
    - /docs/project-structure
    - /docs/index
---

This page explains how data fetching works in Caspian. Use route functions for initial page data and use RPC actions for browser-triggered reads, writes, streams, and uploads.

Treat RPC as the default way for browser code to talk to Python in Caspian. Do not reach for ad hoc fetch calls to custom JSON endpoints, alternate transport layers, or older helper names unless the task explicitly requires that shape.

MCP is a separate integration surface. Do not place app-owned FastMCP tools in route `index.py` files or treat `@rpc()` actions as a replacement for MCP tools. Use `mcp.md` and `src/lib/mcp/` only when `caspian.config.json` has `mcp: true`. In this workspace, `mcp: false`, so do not assume those files exist.

## Overview

Caspian has two main data-loading paths:

- `page()` for initial-render data, plus `layout()` for synchronous shared props or metadata during the render
- `@rpc()` plus `pp.rpc()` for interactive fetches after the page is already loaded

In practice, most pages use both:

1. Load the first screen of data in `index.py`.
2. Render that data into `index.html`.
3. Call `pp.rpc()` for refreshes, form submits, toggles, infinite scroll, or streamed updates.

## Default Data Rule

- Use `page()` for async or route-level data required before HTML renders, and use `layout()` only for synchronous shared props or metadata.
- Use `@rpc()` on the server and `pp.rpc()` in PulsePoint code for all browser-triggered data work after first render.
- Keep custom REST or other endpoint patterns as explicit exceptions, not the baseline Caspian approach.

## Initial Data In `index.py`

Use the route's backend file for data that should exist before the template is rendered.

Example:

```python
from casp.layout import render_page
from src.lib.prisma import prisma

async def page():
    todos = await prisma.todo.find_many()

    return render_page(__file__, {
        "todos": [todo.to_dict() for todo in todos],
    })
```

Use this pattern for:

- SEO-sensitive page content
- The first render of lists, dashboards, and detail views
- Data that layouts or templates need before the browser runs any client code

If a route's first-render HTML is public and stable enough to reuse across requests, pair this pattern with [cache.md](./cache.md) for route-level page caching and invalidation guidance.

Notes:

- Prefer `async def page()` when your database or API client is async-capable.
- Put shared section-level props in `layout.py` when multiple child routes need the same synchronous payload. The current layout engine does not await `layout()`.
- Keep reusable database or API clients under `src/lib/`; keep route-specific orchestration in `src/app/`.

If the data source is Prisma, confirm `caspian.config.json` has `prisma: true`, then see `database.md` for the current workspace's schema, migration, and generation workflow. In this workspace, Prisma is enabled and the app-owned Python database layer lives under `src/lib/prisma/`.

## Interactive Data With RPC

Use RPC when the browser needs to call Python after the initial page load.

Framework internals note:

- The `@rpc()` decorator and Caspian RPC bridge internals live in `.venv/Lib/site-packages/casp/rpc.py`.
- Treat that file as framework code. Read it when you need to understand or debug Caspian's RPC behavior, not when you are adding normal app-level actions.

Define a server action with `@rpc()`:

```python
from casp.rpc import rpc
from src.lib.prisma import prisma

@rpc()
async def list_todos():
    return await prisma.todo.find_many()

@rpc(require_auth=True, limits="20/minute")
async def create_todo(title: str):
    return await prisma.todo.create(
        data={"title": title, "completed": False}
    )
```

Call it from the client with `pp.rpc()`:

```html
<script>
  async function refreshTodos() {
    const todos = await pp.rpc("list_todos");
    console.log(todos);
  }

  async function addTodo(title) {
    const todo = await pp.rpc("create_todo", { title });
    console.log(todo);
  }
</script>
```

Use RPC for:

- Button-triggered refreshes
- Form submissions and mutations
- Polling or background refreshes
- Filters, sorting, and pagination
- Any browser-to-server interaction that should not require a full page navigation

Validate incoming form and mutation payloads before persisting them. See [validation.md](./validation.md).

If an RPC action needs transient request-scoped success or error state beyond its direct return payload, read [state.md](./state.md) and verify the current wire-request lifecycle before depending on persistence across navigations or full-page reloads.

Important:

- `pp.rpc()` posts to the current route RPC bridge.
- Older docs may refer to `pp.fetchFunction()`. In this repo's current runtime, the supported helper is `pp.rpc()`.

## Streaming Responses

Caspian RPC supports streaming via Server-Sent Events when the action yields values.

Example:

```python
from casp.rpc import rpc
import asyncio

@rpc(limits="10/minute")
async def stream_summary(prompt: str):
    response = f"Streaming a summary for: {prompt}"

    for word in response.split():
        yield word + " "
        await asyncio.sleep(0.1)
```

Client example:

```html
<script>
  async function runStream(prompt) {
    await pp.rpc("stream_summary", { prompt }, {
      onStream: (chunk) => {
        console.log("chunk", chunk);
      },
      onStreamComplete: () => {
        console.log("stream complete");
      },
    });
  }
</script>
```

Use streaming when the user should see partial progress instead of waiting for a single final payload.

## File Uploads

RPC actions can accept FastAPI upload types, but the preferred Caspian pattern is route-local upload actions in `src/app/**/index.py`, shared persistence helpers in `src/lib/`, and reactive client updates through `pp.rpc()`.

Example route-local upload action:

```python
from fastapi import File, UploadFile
from casp.rpc import rpc

from src.lib.dashboard.file_manager import (
    build_file_manager_payload,
    save_file_manager_upload,
)

@rpc(require_auth=True)
async def upload_file(file: UploadFile = File(...), collection: str = "auto"):
    user_id = current_user_id()
    content = await file.read()

    uploaded = await save_file_manager_upload(
        user_id,
        file_name=file.filename or "file",
        content=content,
        mime_type=file.content_type,
        desired_group=collection,
    )

    return {
        "success": True,
        "uploaded": uploaded,
        "data": await build_file_manager_payload(user_id),
    }
```

Client example:

```html
<script>
  async function uploadSelectedFiles(fileList, collection = "auto") {
    for (const file of Array.from(fileList ?? [])) {
      const response = await pp.rpc("upload_file", { file, collection }, {
        onUploadProgress: (progress) => {
          console.log(progress.percentage ?? 0);
        },
      });

      if (response?.data) {
        setFileManagerData(response.data);
      }
    }
  }
</script>
```

Use this pattern for real file managers:

- Keep upload and delete actions in the owning route `index.py`, not in `main.py`.
- Use `page()` to render the initial manager payload.
- Store durable metadata in Prisma and store the browser-accessible blob separately under `public/assets/file-manager/`.
- Use `pp.state(...)` plus `pp-for` for the list UI instead of manual `innerHTML` writes.
- Add `public/assets/file-manager` to `PUBLIC_IGNORE_DIRS` in `settings/bs-config.ts` so runtime uploads do not trigger BrowserSync reloads during `npm run dev`.

Read [file-uploads.md](./file-uploads.md) for the complete file-manager pattern.

## Auth, Roles, And Limits

The `@rpc()` decorator also controls access and abuse protection.

Example:

```python
from casp.rpc import rpc

@rpc(
    require_auth=True,
    allowed_roles=["admin"],
    limits="5/minute",
)
def delete_user(user_id: int):
    return {"deleted": user_id}
```

Use these options for privileged reads, destructive mutations, and endpoints that could be abused if left unbounded.

Use [auth.md](./auth.md) when the action should also participate in centralized session config, page decorators, guest-only flows, RBAC route policy, or OAuth provider handling.

According to the upstream Caspian RPC docs, actions are private by default until decorated with `@rpc()`, and the framework includes CSRF protection plus origin validation for exposed actions.

## Serialization Rules

For RPC responses, Caspian can automatically serialize common Python return types such as:

- dictionaries and lists
- Pydantic models
- dataclasses
- Prisma objects

For initial route rendering with `render_page(...)`, prefer passing plain template-ready dictionaries or lists so the HTML layer gets explicit data shapes.

## Recommended Decision Rule

Use `page()` when async or route-specific data is part of the page render. Use `layout()` for synchronous shared props or metadata. Use `@rpc()` plus `pp.rpc()` when the browser needs to ask the server for more data after the page is already visible.

A common pattern is:

- Fetch the first page of data in `index.py`
- Render that data into the HTML template
- Use RPC for subsequent filters, mutations, refreshes, streaming, or uploads

If the first-render HTML is expensive to produce and safe to share between visitors, add route-level caching with [cache.md](./cache.md) and invalidate affected URIs after successful mutations.

## AI Routing Notes

If an AI agent is choosing how to load data in Caspian, apply these rules first.

- Put first-render data loading in `src/app/**/index.py`.
- Put shared section props in `layout.py` only when multiple child routes need the same synchronous data.
- Keep async I/O in `page()` or `@rpc()` because the current layout engine does not await `layout()`.
- Treat RPC as the default read and write layer between PulsePoint code and Python route logic.
- Use `@rpc()` for backend functions that should be callable from the browser.
- Use `pp.rpc()` for client-side calls; do not prefer older `pp.fetchFunction()` wording.
- Prefer route-render data plus RPC over inventing parallel REST endpoints for normal Caspian page interactions.
- Read [file-uploads.md](./file-uploads.md) when the task involves file pickers, upload progress, media libraries, or file-manager UI.
- Use [cache.md](./cache.md) when a route's initial HTML should be reused across requests and invalidated after writes.
- Use [state.md](./state.md) when an RPC mutation needs transient request-scoped success or error state outside the direct response payload.
- Use `onStream` for streamed responses and `onUploadProgress` for upload-aware client calls.
- Add `require_auth`, `allowed_roles`, and `limits` to sensitive RPC actions.
- Keep reusable database clients and service wrappers in `src/lib/`.
- Use `.venv/Lib/site-packages/casp/rpc.py` only when the task is about Caspian core RPC internals or framework debugging.
- Check [routing.md](./routing.md) before adding new route files and [pulsepoint.md](./pulsepoint.md) before generating raw client runtime code.

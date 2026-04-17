---
title: Fetch Data
description: Fetch first-render and interactive data in Caspian with async route functions, `@rpc()` actions, `pp.rpc()`, streaming, and uploads.
related:
  title: Related docs
  description: Use the routing guide to place route logic correctly, then use the PulsePoint runtime guide for client-side `pp.rpc()` details.
  links:
    - /docs/routing
    - /docs/pulsepoint
    - /docs/project-structure
    - /docs/index
---

This page explains how data fetching works in Caspian. Use route functions for initial page data and use RPC actions for browser-triggered reads, writes, streams, and uploads.

## Overview

Caspian has two main data-loading paths:

- `page()` or `layout()` for server-side data needed during the initial render
- `@rpc()` plus `pp.rpc()` for interactive fetches after the page is already loaded

In practice, most pages use both:

1. Load the first screen of data in `index.py`.
2. Render that data into `index.html`.
3. Call `pp.rpc()` for refreshes, form submits, toggles, infinite scroll, or streamed updates.

## Initial Data In `index.py`

Use the route's backend file for data that should exist before the template is rendered.

Example:

```python
from casp.layout import render_page
from src.lib.prisma import prisma

async def page():
    todos = prisma.todo.find_many()

    return render_page(__file__, {
        "todos": [todo.to_dict() for todo in todos],
    })
```

Use this pattern for:

- SEO-sensitive page content
- The first render of lists, dashboards, and detail views
- Data that layouts or templates need before the browser runs any client code

Notes:

- Prefer `async def page()` when your database or API client is async-capable.
- Put shared section-level data in `layout.py` when multiple child routes need the same payload.
- Keep reusable database or API clients under `src/lib/`; keep route-specific orchestration in `src/app/`.

## Interactive Data With RPC

Use RPC when the browser needs to call Python after the initial page load.

Define a server action with `@rpc()`:

```python
from casp.rpc import rpc
from src.lib.prisma import prisma

@rpc()
def list_todos():
    return prisma.todo.find_many()

@rpc(require_auth=True, limits="20/minute")
def create_todo(title: str):
    return prisma.todo.create(
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

RPC actions can accept FastAPI upload types.

Example:

```python
from fastapi import File, UploadFile
from casp.rpc import rpc
import shutil

@rpc()
def upload_file(file: UploadFile = File(...)):
    with open(f"uploads/{file.filename}", "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)

    return {
        "status": "success",
        "filename": file.filename,
        "size": file.size,
    }
```

On the client, call `pp.rpc()` and pass `onUploadProgress` when the UI needs progress updates.

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

According to the upstream Caspian RPC docs, actions are private by default until decorated with `@rpc()`, and the framework includes CSRF protection plus origin validation for exposed actions.

## Serialization Rules

For RPC responses, Caspian can automatically serialize common Python return types such as:

- dictionaries and lists
- Pydantic models
- dataclasses
- Prisma objects

For initial route rendering with `render_page(...)`, prefer passing plain template-ready dictionaries or lists so the HTML layer gets explicit data shapes.

## Recommended Decision Rule

Use `page()` or `layout()` when the data is part of the page render. Use `@rpc()` plus `pp.rpc()` when the browser needs to ask the server for more data after the page is already visible.

A common pattern is:

- Fetch the first page of data in `index.py`
- Render that data into the HTML template
- Use RPC for subsequent filters, mutations, refreshes, streaming, or uploads

## AI Routing Notes

If an AI agent is choosing how to load data in Caspian, apply these rules first.

- Put first-render data loading in `src/app/**/index.py`.
- Put shared section data in `layout.py` when multiple child routes need it.
- Use `@rpc()` for backend functions that should be callable from the browser.
- Use `pp.rpc()` for client-side calls; do not prefer older `pp.fetchFunction()` wording.
- Use `onStream` for streamed responses and `onUploadProgress` for upload-aware client calls.
- Add `require_auth`, `allowed_roles`, and `limits` to sensitive RPC actions.
- Keep reusable database clients and service wrappers in `src/lib/`.
- Check [routing.md](./routing.md) before adding new route files and [pulsepoint.md](./pulsepoint.md) before generating raw client runtime code.

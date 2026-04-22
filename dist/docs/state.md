---
title: State Management
description: Use this page when the task mentions `StateManager`, `request.state.session`, request-scoped state, listeners, or session-backed JSON state in Caspian.
related:
  title: Related docs
  description: Pair server request state with auth and RPC flows, then read the PulsePoint guide when the state really belongs in the browser instead of the request lifecycle.
  links:
    - /docs/auth
    - /docs/fetch-data
    - /docs/pulsepoint
    - /docs/project-structure
    - /docs/index
---

This page explains the current installed Caspian state manager API for request-scoped state, session-backed persistence, listener callbacks, and AttributeDict reads.

Treat `casp.state_manager` as the transient server-state layer in Caspian. Use it for short-lived state that should be available during the current request and, where the middleware flow allows it, across a tightly scoped follow-up request. Do not treat it as a database, a long-lived session model, or a replacement for PulsePoint browser state.

## Overview

Caspian exposes request state through `casp.state_manager`.

Import the public API like this:

```python
from casp.state_manager import StateManager
```

The current installed implementation lives in `.venv/Lib/site-packages/casp/state_manager.py`.

The real API surface is:

- request initialization with `StateManager.init(request)`
- top-level state reads with `StateManager.get_state(...)`
- top-level state writes with `StateManager.set_state(...)`
- reset operations with `StateManager.reset_state(...)`
- request-local subscriptions with `StateManager.subscribe(...)`

Use this layer when middleware, route handlers, auth flows, or RPC actions need a shared transient state bag without explicitly threading the request object through every helper.

## Default State Rule

- Use `StateManager` for short-lived server state such as flash-style messages, transient form state, or middleware-to-route coordination.
- Keep values JSON-serializable because every write persists the full state dict with `json.dumps(...)`.
- Read and write top-level keys. The current implementation does not provide deep merge, nested path helpers, or delete helpers.
- Let middleware initialize the manager. App code usually should not call `StateManager.init(...)` directly.
- Use PulsePoint for browser reactivity and persistent data stores for durable application state.

## Request-Scoped Storage Model

The installed implementation stores state in `ContextVar` containers:

- `_state_data` holds the current request's state dict
- `_state_listeners` holds the current request's listener callbacks
- `_current_request` holds the current request object

That means the state manager is request-scoped, not process-global. One request does not share in-memory state or listeners with another request.

The session persistence layer is separate from the in-memory request copy. `save_state()` serializes the current dict to JSON and stores it under the constant key `app_state_cL7y4KirLp`.

Important implementation detail:

- the current code reads and writes `request.state.session`, not `request.session`

In other words, cross-request persistence depends on the surrounding Caspian middleware keeping `request.state.session` available and in sync with session storage.

Inspect the current project's `main.py`: if `SessionMiddleware` exposes `request.session` but the middleware stack does not mirror that into `request.state.session`, treat persistence as opt-in until you add that bridge explicitly.

## Request Lifecycle

The current request lifecycle is:

1. `StateManager.init(request)` stores the request object.
2. It clears the in-memory state dict and listener list for the new request.
3. It calls `load_state()` and tries to decode any previously saved JSON from the session bucket.
4. It checks the `X-PulsePoint-Wire` header.
5. When that header is missing or falsy, it immediately calls `reset_state()`.

That last step matters. In the current implementation, non-wire requests start by clearing the loaded state and writing the empty state back to the session bucket.

Treat the current behavior in this repo as request-local state plus partially wired transient persistence for wire-driven flows. If you expect full-page redirect flash semantics or cross-request state reuse, verify the exact middleware bridge in your local Caspian version before depending on it.

You usually do not initialize this manually in route code. The current auth middleware docs already show `StateManager.init(request)` running during request setup.

## Reading State

Use `StateManager.get_state(...)` to read the whole state bag or a single top-level key.

Behavior:

- `StateManager.get_state()` returns an `AttributeDict` wrapping the full top-level state dict
- `StateManager.get_state("key", initial_value)` returns the stored value or the provided default
- when the returned value is a dict, the direct return value is wrapped in `AttributeDict`

Example:

```python
from casp.state_manager import StateManager

StateManager.set_state("flash", {
    "type": "success",
    "message": "Profile updated.",
})

flash = StateManager.get_state("flash")
if flash is not None:
    print(flash.type)
    print(flash.message)

state = StateManager.get_state()
print(state.flash["message"])
```

Implementation note:

- `AttributeDict` is shallow in the current file

`StateManager.get_state("flash")` wraps that one dict, so `flash.message` works. `StateManager.get_state()` wraps only the top-level state object, so nested dict values remain plain dicts unless you fetch that key directly.

Do not assume recursive dot notation such as `StateManager.get_state().flash.message` unless your stored nested value has already been wrapped elsewhere.

## Writing State

Use `StateManager.set_state(key, value)` to replace a top-level key.

Current behavior:

- updates the current in-memory dict in place
- notifies every registered listener
- serializes the entire state dict into the session bucket

Example:

```python
from casp.rpc import rpc
from casp.state_manager import StateManager

@rpc(require_auth=True)
async def save_profile(data: dict):
    StateManager.set_state("flash", {
        "type": "success",
        "message": "Profile updated.",
    })

    return {"success": True}
```

Keep writes JSON-safe. The current `save_state()` implementation does not catch `TypeError` from `json.dumps(...)`, so non-serializable values such as file handles, request objects, response objects, or arbitrary class instances can break the write path.

## Resetting State

Use `StateManager.reset_state(...)` to clear either one key or the whole state dict.

Example:

```python
from casp.state_manager import StateManager

StateManager.reset_state("flash")
StateManager.reset_state()
```

Current behavior is slightly surprising:

- `StateManager.reset_state()` replaces the whole state dict with `{}`
- `StateManager.reset_state("flash")` does not delete the key; it sets the existing key to `None`
- both forms notify listeners and save the updated state back to the session bucket

If you need true key removal, there is no public delete helper in the current implementation.

## Request-Local Listeners

`StateManager.subscribe(listener)` registers a callback for the current request context.

Behavior:

- appends the listener to the current request's listener list
- immediately invokes the listener with the current state dict
- returns an `unsubscribe()` function that removes that listener from the current request list

Example:

```python
from casp.state_manager import StateManager

def log_changes(state):
    if state.get("error"):
        print(f"Error detected: {state['error']}")

unsubscribe = StateManager.subscribe(log_changes)

StateManager.set_state("error", "Invalid credentials")

unsubscribe()
```

Treat these listeners as request-local callbacks, not as a cross-request event bus.

The current implementation resets `_state_listeners` inside `StateManager.init(...)`, so subscriptions do not survive into later requests.

## API Reference

| Method | Purpose | Current behavior |
| --- | --- | --- |
| `init(request)` | Prepare request-bound state | Stores the request, clears in-memory state and listeners, loads saved JSON state, then clears state again for non-wire requests. |
| `get_state(key=None, initial_value=None)` | Read all state or one key | Returns a shallow `AttributeDict` for dict values and the provided default for missing keys. |
| `set_state(key, value)` | Replace one top-level key | Mutates the state dict, notifies listeners, and JSON-saves the full state bag. |
| `reset_state(key=None)` | Clear one key or everything | Sets one key to `None` or replaces the full dict with `{}`. |
| `subscribe(listener)` | Listen for request-local changes | Immediately invokes the listener and returns an unsubscribe closure. |
| `save_state()` | Persist current state | Serializes the full dict into the session bucket under `APP_STATE_KEY`. |
| `load_state()` | Rehydrate saved state | Loads JSON when the session bucket exists and ignores malformed JSON. |

## Current Implementation Notes

- The persistence bucket key is `StateManager.APP_STATE_KEY = "app_state_cL7y4KirLp"`.
- `load_state()` only catches `json.JSONDecodeError`; malformed JSON is ignored silently.
- `save_state()` writes the entire state dict every time `set_state()` or `reset_state()` runs.
- `set_state()` replaces a top-level key value. There is no deep merge behavior.
- `notify_listeners()` runs on every `set_state()`, successful `load_state()`, and every `reset_state()` call.
- Listener callbacks receive the raw state dict, not an `AttributeDict` wrapper.
- `AttributeDict.__getattr__` raises `AttributeError` for missing keys and `__setattr__` writes back into the dict.
- Because persistence uses JSON, keep stored values simple: strings, numbers, booleans, lists, dicts, and `None`.
- The current lifecycle resets loaded state on non-wire requests, so verify redirect and flash behavior against the actual request type you are using.
- In this repo's current middleware stack, `request.state.session` is not populated from `request.session`, so saved state may not survive into a later request unless you add that bridge.

## Recommended Usage Pattern

Use `StateManager` close to the request boundary and keep the stored payloads small.

Common placement patterns are:

- route handlers that need to share transient data with helpers in the same request
- `@rpc()` actions that record short-lived success or error state during a wire-driven flow
- auth-aware flows that need request-bound coordination between middleware and route logic
- shared helpers under `src/lib/` when multiple routes use the same transient-state conventions

Good candidates for this state layer are:

- success or error messages
- temporary redirect or workflow flags
- lightweight request-bound metadata
- small structured dicts that can be serialized safely

Poor candidates are:

- ORM models or request objects
- file handles and uploads
- large payloads or cached query results
- durable business data that belongs in a database
- browser interaction state that should live in PulsePoint

For browser-triggered writes and route actions, pair this page with [fetch-data.md](./fetch-data.md). For middleware and session-aware auth flows, pair it with [auth.md](./auth.md).

## AI Retrieval Notes

If an AI agent is deciding how to use transient state in a Caspian app, apply these rules first.

- Use `StateManager` from `casp.state_manager` for request-scoped server state, not for durable application records.
- Assume the current API is top-level-key based only.
- Keep stored values JSON-serializable.
- Prefer `StateManager.get_state("key")` when you want dot notation on a dict payload.
- Do not assume recursive `AttributeDict` wrapping for nested dicts.
- Treat `subscribe(...)` listeners as request-local and short-lived.
- Be careful with full-page redirect assumptions because `init(request)` clears loaded state on non-wire requests.
- Do not assume cross-request persistence in this repo until middleware copies `request.session` into `request.state.session`.
- Use PulsePoint state for client interactivity and `StateManager` for server request flows.
- Check [auth.md](./auth.md) for middleware ordering and request initialization details.
- Check [project-structure.md](./project-structure.md) before deciding whether a transient-state helper belongs next to a route or in `src/lib/`.

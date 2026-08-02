---
title: WebSockets
description: Use this page when the task mentions named sockets (`@socket()` / `pp.socket`), FastAPI WebSocket endpoints, live channels, broadcast managers or pools, browser `WebSocket`, origin checks, or authenticated socket sessions in Caspian.
related:
  title: Related docs
  description: Use routing and PulsePoint for client placement, auth for session policy, and the core runtime map for endpoint ownership in `main.py`.
  links:
    - /docs/routing
    - /docs/pulsepoint
    - /docs/auth
    - /docs/fetch-data
    - /docs/core-runtime-map
    - /docs/index
---

WebSockets in Caspian are gated by `caspian.config.json` and delivered as **named sockets**: `@socket()` functions in Python consumed by `pp.socket(...)` in the browser, all sharing one endpoint. They are not the default data path.

Use `@rpc()` plus `pp.rpc()` for ordinary browser-triggered reads, writes, uploads, and Server-Sent Event streams. Use WebSockets only when the task needs a bidirectional, long-lived channel such as live chat, collaboration, multiplayer state, server push that is not a good SSE fit, or presence.

Treat `caspian.config.json` as the single source of truth for whether the WebSocket feature is enabled. If `websocket` is false, this page is reference material only; do not assume `@app.websocket(...)` endpoints, `src/lib/websocket/**`, or socket demo routes exist until the user chooses to enable the feature and the update workflow has run.

Named sockets are the socket counterpart of `@rpc()`/`pp.rpc()` and the only shipped socket layer. Auth policy is declared per socket (`@socket(require_auth=True, allowed_roles=[...])`), so there are no separate public and private channel endpoints, no per-channel guard calls, and no shared "channel transport loop" — apps that once had a `/ws/live` + `/ws/public` pair migrate each channel to one `@socket()` function. A hand-written `@app.websocket(...)` route with a native browser `WebSocket` remains possible as an escape hatch for wires the JSON-frame contract cannot carry (binary frames, non-JSON protocols), and then the app owns every check itself.

## Named Sockets

An rpc is a question with one answer, and an rpc stream is an answer that arrives in pieces. A named socket is the third shape: both sides may speak, at any time, for as long as the page is open. When the app ships `src/lib/websocket/sockets.py`, a live channel is one decorated function:

```python
from src.lib.websocket.sockets import Socket, socket

@socket()
async def echo(label: str, socket: Socket):
    while (text := await socket.recv()) is not None:
        if not await socket.send(f"{label}: {text}"):
            break  # The browser is gone.
```

From the browser, inside a PulsePoint component script:

```html
<script>
    const sock = pp.ref(null);

    pp.effect(() => {
        sock.current = pp.socket("echo", { label: "you" }, {
            onMessage: (value) => append(value),
            onError: (error) => setStatus(error.message),
        });
        return () => sock.current.close();
    }, []);

    function say(text) {
        sock.current.send(text);
    }
</script>
```

### The wire

- Every socket connects to **one endpoint**: `/__pulsepoint/ws` (`SOCKET_PATH` in `src/lib/websocket/sockets.py`, wired in `main.py`). The path carries the PulsePoint runtime's name rather than any server framework's, because `pp.socket(...)` is the same client whichever backend serves it — the client's default path and `SOCKET_PATH` must stay identical. The function is named in the `name` query parameter. Names are therefore application-wide: the function's own name, unique across the app, checked at registration.
- The arguments do not travel in the URL — a URL is logged by every proxy on the way — but as the connection's **first frame**: one JSON object, exactly the payload `pp.rpc` would have posted. The client sends it automatically on open, `{}` included.
- Every frame after that is **one JSON value**, in either direction.
- There is no status line inside an open connection, so **failure is a frame**: `{"error": "..."}` — that key alone — followed by a close. The client routes it to `onError`, never `onMessage`, and the server refuses to send that shape as an ordinary message.

### The handler contract

- A `@socket()` function is `async def`, declares its arguments as normal parameters, plus one parameter named `socket` that receives the open `Socket`. The payload is filtered against the signature exactly as rpc payloads are: a parameter is client-settable only when declared, and `socket` can never be supplied from the wire.
- `await socket.recv()` returns the next JSON value, or `None` when the connection closes — the natural loop is `while (value := await socket.recv()) is not None:`. `recv_text()` hands over the raw frame. A frame that is not valid JSON raises `ValueError`, which travels back to the browser as the error frame.
- `await socket.send(value)` returns `False` when the browser is gone — navigated away, closed the tab. That is the signal to stop, not an error to report. Values go through the same serializer as rpc results, so dataclasses and models travel identically over both wires.
- When the handler returns, the connection closes: a handler that returns is a conversation that ends. A raised `ValueError` becomes a readable error frame; any other exception becomes a generic error frame in production.

### Broadcast: senders and pools

`socket.sender()` returns a `SocketSender` — the sending half of the connection, detached from the conversation, safe to hold in shared state or another task. A room is a `SocketPool` of senders:

```python
from src.lib.websocket.sockets import Socket, SocketPool, socket

_room = SocketPool()

@socket()
async def chat_room(name: str, socket: Socket):
    sender = socket.sender()
    _room.add(sender)
    try:
        await _room.broadcast({"from": "room", "text": f"{name} joined"})
        while (said := await socket.recv()) is not None:
            await _room.broadcast({"from": name, "text": str(said.get("text", ""))[:1000]})
    finally:
        _room.discard(sender)
        await _room.broadcast({"from": "room", "text": f"{name} left"})
```

`SocketPool.broadcast(...)` prunes connections whose browser is gone as it walks. Keep authenticated and guest traffic in separate pools, exactly as with raw connection managers.

### Auth and security

- `@socket(require_auth=True)` refuses the connection unless the request carries an authenticated session; `@socket(allowed_roles=[...])` adds RBAC — both delegate to Caspian's `Auth`, the same source of truth as HTTP routes and rpc. Refusals travel as the error frame, so the browser gets a readable message.
- The endpoint keeps the shared protections: the anti-CSWSH origin check runs before the handshake upgrades, and every connection is subject to the connection cap, message-size limit, per-connection message rate, and idle timeout (outbound traffic counts as liveness, so a passive listener in an active room is not "idle"). The socket session is read-only — session mutations are not persisted back to the cookie over a WebSocket.

### Registration timing

A `@socket()` in a route's `index.py` registers when that module is first imported, which happens when the route first renders — so the page that opens `pp.socket(...)` has always registered its own sockets. A socket shared by several routes belongs in `src/lib/**`, imported by each owning route. After a server restart, a browser that reconnects without reloading may briefly see "no socket named ..." until the owning route renders once.

## Source Of Truth

Before adding, debugging, or documenting a WebSocket feature:

- Read `caspian.config.json` first and confirm `websocket: true`.
- If `websocket` is false and the user wants live channels, ask for confirmation before enabling it, then update the config and follow the project update workflow.
- Inspect `src/lib/websocket/sockets.py` for the layer itself: the `@socket()` registry, `SOCKET_PATH`, limits, close codes, and the endpoint handler wired in `main.py`.
- Inspect `src/lib/websocket/**` for the origin check and connection ceiling, and for sockets shared across routes.
- Inspect the owning route under `src/app/**` for its `@socket()` functions and the `pp.socket(...)` client in the component script.
- Check `settings/bs-config.json` before testing development URLs through BrowserSync or a proxy.

Packaged docs should describe this workflow generally. Keep endpoint names, route names, ports, and demo inventory project-owned; do not treat one app's route tree as the framework convention.

## Where Socket Code Lives

- The layer itself — the `@socket()` registry, `Socket`/`SocketSender`/`SocketPool`, limits, and the endpoint handler — lives in `src/lib/websocket/sockets.py`, wired in `main.py` as the single `@app.websocket(SOCKET_PATH)` route.
- A socket owned by one route is declared in that route's `index.py`, next to its `@rpc()` actions. A socket shared by several routes lives in `src/lib/**`, imported by each owning route.
- Route-specific browser UI stays in the page template inside `src/app/**/index.py`. Routes do not pass `websocket_path`/`websocket_url` into templates: `pp.socket(...)` already knows the shared endpoint, so a route only names its function.

## Raw Endpoints (escape hatch only)

For a wire the JSON-frame contract cannot carry — binary frames, a non-JSON protocol — an app may still hand-write an `@app.websocket(...)` route in `main.py` and pair it with a native browser `WebSocket` kept in `pp.ref(...)` with listeners inside the owning component script. Such an endpoint owns every check the named-socket endpoint would have done for it: run the origin check (`is_websocket_origin_allowed`) before accepting, delegate auth to Caspian's `Auth`, and enforce its own size/rate/idle/capacity limits. Do not build parallel public/private channel endpoints for ordinary JSON traffic — that is what per-socket auth flags replaced.

## Auth And Sessions

HTTP route privacy does not automatically make a WebSocket endpoint private. The HTTP middleware stack (`AuthMiddleware`, `CSRFMiddleware`, `RPCMiddleware`, body-size, security-header, and diagnostics layers) early-returns on every non-`http` scope, so a WebSocket handshake (`scope["type"] == "websocket"`) is never seen by `AuthMiddleware`. The only middleware that runs on socket scopes is `SessionMiddleware`, which exposes `websocket.session`. The socket layer therefore authorizes each connection itself.

For named sockets that is declarative: `@socket(require_auth=True)` refuses guests, `@socket(allowed_roles=[...])` adds RBAC, and refusals travel as the `{"error": "..."}` frame before the close so the browser gets a readable message. Under the hood — and for any raw escape-hatch endpoint, which must do this itself — the check delegates to Caspian's `Auth` instead of re-implementing session/`exp`/payload parsing. `Auth` methods read only `request.session`, and a `WebSocket` exposes `.session`, so binding the socket as the request context reuses the exact HTTP auth logic as a single source of truth:

```python
from casp.auth import Auth

Auth.set_request(websocket)            # WebSocket exposes .session
auth = Auth.get_instance()
if auth.is_authenticated():
    payload = auth.get_payload() or {}
    if roles and not auth.check_role(payload, roles):
        ...                            # forbidden -> close 1008
else:
    ...                                # unauthenticated -> close 1008 if required
```

Use close code `1008` for policy violations such as failed origin checks or missing authentication. If the endpoint accepts before rejecting, send a small JSON error payload first so the browser UI can show a useful message.

Treat the socket auth check as read-only. `SessionMiddleware` only writes the session cookie on `http.response.start`, which never fires for a WebSocket, so session mutations made during a connection (including `refresh_session` / sliding `exp`) are not persisted back to the cookie. Do not rely on the socket to extend a session; verify identity and, if needed, drive session refresh from an HTTP request. Do not assume HTTP `AuthMiddleware` behavior applies to `scope["type"] == "websocket"`.

Path-based `Auth` helpers (`is_private_route`, `get_required_roles`) read the HTTP route configuration and do not know about the socket endpoint's path. For socket RBAC, declare the roles on the socket (`@socket(allowed_roles=[...])`) rather than asking `Auth` to classify the path.

## Origin And Proxy Checks

WebSocket endpoints should validate the `Origin` header before accepting connections, especially in production.

Common allowed origins:

- same-origin derived from the socket URL
- explicit values from environment variables such as `WEBSOCKET_ALLOWED_ORIGINS`, `CORS_ALLOWED_ORIGINS`, `APP_BASE_URL`, `PUBLIC_BASE_URL`, or `SITE_URL`
- localhost HTTP origins in development when the app intentionally permits local BrowserSync or proxy testing

When testing locally, read `settings/bs-config.json` first and build `ws://` or `wss://` URLs from its active `local` value. Do not assume the BrowserSync port is the default.

## Message Contract

The named-socket wire is fixed by the layer; per-socket contracts are only about the *values* inside the frames:

- The first frame is the arguments — one JSON object, sent by `pp.socket(...)` automatically. Every later frame is one JSON value, in either direction.
- `{"error": "..."}` (that key alone) is reserved for failures and routed to `onError`; the server refuses to send it as an ordinary message.
- Size, rate, idle, and capacity limits are enforced by the layer — a handler does not re-implement them, but should still sanitize and trim outbound text so one client cannot broadcast unbounded data.
- A `send` that answers `False` (Python) or `false` (browser) means the other side is gone: stop, and let `SocketPool.broadcast` prune.

Document each socket's value shapes (what a frame means) next to the `@socket()` function and the component that opens it.

## WebSockets Vs RPC/SSE

Choose the transport deliberately:

| Need | Preferred transport |
| --- | --- |
| CRUD, form submits, button actions, upload progress | `@rpc()` plus `pp.rpc()` |
| Server sends one-way progress chunks for a request | RPC streaming / SSE |
| Browser and server both send messages over a persistent channel | Named socket (`@socket()` + `pp.socket`) |
| Presence, collaboration, live chat, multiplayer state | Named socket (`@socket()` + `pp.socket`) |
| Binary frames, non-JSON wires, exotic handshakes | Raw `@app.websocket(...)` endpoint |

Do not replace normal Caspian RPC with WebSockets just because a page is interactive.

## AI Retrieval Notes

If an AI agent is working on WebSockets in a Caspian app:

- Start with `caspian.config.json`. Continue only after confirming `websocket: true`, unless the task is specifically to enable the feature.
- Check for `src/lib/websocket/sockets.py` next. When it exists, add live channels as `@socket()` functions consumed by `pp.socket(...)` instead of hand-writing `@app.websocket(...)` endpoints and native `WebSocket` clients.
- Use [core-runtime-map.md](./core-runtime-map.md) to connect `main.py` behavior back to this doc.
- Use [routing.md](./routing.md) before changing the route files that render the client.
- Use [pulsepoint.md](./pulsepoint.md) for authored template rules and component lifecycle cleanup.
- Use [auth.md](./auth.md) before changing authenticated socket behavior.
- Keep reusable connection/session helpers in `src/lib/websocket/**` when they are shared; keep route UI in whichever `src/app/**` route owns the live experience; keep app-level ASGI endpoints in `main.py` unless the runtime intentionally provides another registration layer.
- Verify local test URLs from `settings/bs-config.json`.

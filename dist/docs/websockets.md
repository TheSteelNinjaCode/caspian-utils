---
title: WebSockets
description: Use this page when the task mentions FastAPI WebSocket endpoints, live channels, broadcast managers, browser `WebSocket`, origin checks, or authenticated socket sessions in Caspian.
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

WebSockets in Caspian are gated by `caspian.config.json` and implemented as app-owned FastAPI endpoints, not the default RPC data path.

Use `@rpc()` plus `pp.rpc()` for ordinary browser-triggered reads, writes, uploads, and Server-Sent Event streams. Use WebSockets only when the task needs a bidirectional, long-lived channel such as live chat, collaboration, multiplayer state, server push that is not a good SSE fit, or presence.

Treat `caspian.config.json` as the single source of truth for whether the WebSocket feature is enabled. If `websocket` is false, this page is reference material only; do not assume `@app.websocket(...)` endpoints, `src/lib/websocket/**`, or socket demo routes exist until the user chooses to enable the feature and the update workflow has run.

## Source Of Truth

Before adding, debugging, or documenting a WebSocket feature:

- Read `caspian.config.json` first and confirm `websocket: true`.
- If `websocket` is false and the user wants live channels, ask for confirmation before enabling it, then update the config and follow the project update workflow.
- Inspect `main.py` for `@app.websocket(...)` routes, origin checks, close codes, message limits, idle timeout, and middleware interactions.
- Inspect `src/lib/**` for app-owned socket helpers such as session extraction, auth payload checks, connection managers, or broadcast utilities.
- Inspect the owning route under `src/app/**` for the browser client, first-render URL selection, and PulsePoint state around the native `WebSocket`.
- Check `settings/bs-config.json` before testing development URLs through BrowserSync or a proxy.

Packaged docs should describe this workflow generally. Keep endpoint names, route names, ports, and demo inventory project-owned; do not treat one app's route tree as the framework convention.

## Recommended Shape

When `caspian.config.json` has `websocket: true`, put socket endpoints in `main.py` when they are application-level ASGI routes:

```python
from fastapi import WebSocket, WebSocketDisconnect, status

WEBSOCKET_PATH = "/ws/channel"

@app.websocket(WEBSOCKET_PATH)
async def websocket_live_endpoint(websocket: WebSocket):
    # ONE guard authorizes the connection (origin + auth) and returns the
    # identity, or None after closing. Pass require_auth / roles per channel.
    if await authorize_websocket(websocket, require_auth=True) is None:
        return

    await manager.connect(websocket)
    try:
        while True:
            raw_message = await websocket.receive_text()
            ...
    except WebSocketDisconnect:
        return
    finally:
        manager.disconnect(websocket)
```

Keep the authorization gate in one reusable helper rather than re-deriving auth in each endpoint. A public and a private channel should differ only by their `require_auth`/`roles` policy and their broadcast pool, not by a second copy of origin and session logic. Keep authenticated and guest traffic in separate connection managers so a private broadcast can never fan out to a guest connection.

Keep reusable socket helpers in `src/lib/**` when they are shared by more than one endpoint or route. Good candidates include connection managers, session helpers, auth-payload extraction, payload normalization, and shared broadcast code.

Keep route-specific browser UI in `src/app/**/index.html` and route-specific first-render values in the matching `index.py`. For example, an `index.py` can pass `websocket_path` and a development-only BrowserSync-aware `websocket_url` into the template with `render_page(__file__, {...})`.

## Browser Client Pattern

Use PulsePoint for the component state and lifecycle, while using the native browser `WebSocket` object for the socket itself.

```html
<main>
  <button onclick="{connectSocket()}" disabled="{isConnected}">Connect</button>
  <button onclick="{disconnectSocket()}" disabled="{!isConnected}">Disconnect</button>

  <script>
    const websocketPath = "{{ websocket_path }}";
    const socketRef = pp.ref(null);
    const [status, setStatus] = pp.state("idle");
    const isConnected = status === "connected";

    function buildWebSocketUrl(path) {
      const protocol = window.location.protocol === "https:" ? "wss:" : "ws:";
      return new URL(`${protocol}//${window.location.host}${path}`, window.location.href).toString();
    }

    function connectSocket() {
      const socket = new WebSocket(buildWebSocketUrl(websocketPath));
      socketRef.current = socket;

      socket.addEventListener("message", (event) => {
        const payload = JSON.parse(event.data);
        ...
      });
    }

    function disconnectSocket() {
      socketRef.current?.close(1000, "Client disconnect");
      socketRef.current = null;
    }

    pp.effect(() => {
      return () => socketRef.current?.close(1000, "Page disposed");
    }, []);
  </script>
</main>
```

This is one of the narrow cases where direct browser APIs and event listeners are expected. Keep them inside the owning PulsePoint component script, store the socket in `pp.ref(...)`, and keep rendered UI state in `pp.state(...)`.

## Auth And Sessions

HTTP route privacy does not automatically make a WebSocket endpoint private. The HTTP middleware stack (`AuthMiddleware`, `CSRFMiddleware`, `RPCMiddleware`, body-size, security-header, and diagnostics layers) early-returns on every non-`http` scope, so a WebSocket handshake (`scope["type"] == "websocket"`) is never seen by `AuthMiddleware`. The only middleware that runs on socket scopes is `SessionMiddleware`, which exposes `websocket.session`. Each authenticated socket endpoint must therefore authorize itself.

Align that self-check with the rest of the app by delegating to Caspian's `Auth` instead of re-implementing session/`exp`/payload parsing. `Auth` methods read only `request.session`, and a `WebSocket` exposes `.session`, so binding the socket as the request context reuses the exact HTTP auth logic as a single source of truth:

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

Path-based `Auth` helpers (`is_private_route`, `get_required_roles`) read the HTTP route configuration and do not know about `/ws/*` paths. For socket RBAC, pass the required roles explicitly into the guard rather than asking `Auth` to classify the socket path.

## Origin And Proxy Checks

WebSocket endpoints should validate the `Origin` header before accepting connections, especially in production.

Common allowed origins:

- same-origin derived from the socket URL
- explicit values from environment variables such as `WEBSOCKET_ALLOWED_ORIGINS`, `CORS_ALLOWED_ORIGINS`, `APP_BASE_URL`, `PUBLIC_BASE_URL`, or `SITE_URL`
- localhost HTTP origins in development when the app intentionally permits local BrowserSync or proxy testing

When testing locally, read `settings/bs-config.json` first and build `ws://` or `wss://` URLs from its active `local` value. Do not assume the BrowserSync port is the default.

## Message Contract

Document the JSON message contract near the endpoint and client that own it.

Useful baseline conventions:

- Clients send JSON objects, usually `{ "type": "message", "text": "..." }` or `{ "type": "ping" }`.
- Servers send `ready`, `message`, `error`, and `pong` payloads as JSON.
- Enforce a maximum incoming message size before parsing or broadcasting.
- Close idle sockets after a configured timeout when the app does not need long silent connections.
- Sanitize or trim outbound text so one client cannot broadcast unbounded data.
- Disconnect stale sockets when sends fail.

## WebSockets Vs RPC/SSE

Choose the transport deliberately:

| Need | Preferred transport |
| --- | --- |
| CRUD, form submits, button actions, upload progress | `@rpc()` plus `pp.rpc()` |
| Server sends one-way progress chunks for a request | RPC streaming / SSE |
| Browser and server both send messages over a persistent channel | WebSocket |
| Presence, collaboration, live chat, multiplayer state | WebSocket |

Do not replace normal Caspian RPC with WebSockets just because a page is interactive.

## AI Retrieval Notes

If an AI agent is working on WebSockets in a Caspian app:

- Start with `caspian.config.json`. Continue only after confirming `websocket: true`, unless the task is specifically to enable the feature.
- Use [core-runtime-map.md](./core-runtime-map.md) to connect `main.py` behavior back to this doc.
- Use [routing.md](./routing.md) before changing the route files that render the client.
- Use [pulsepoint.md](./pulsepoint.md) for authored template rules and component lifecycle cleanup.
- Use [auth.md](./auth.md) before changing authenticated socket behavior.
- Keep reusable connection/session helpers in `src/lib/websocket/**` when they are shared; keep route UI in whichever `src/app/**` route owns the live experience; keep app-level ASGI endpoints in `main.py` unless the runtime intentionally provides another registration layer.
- Verify local test URLs from `settings/bs-config.json`.

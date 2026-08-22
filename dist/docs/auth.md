---
title: Authentication
description: Use this page when the task mentions `auth_config.py`, sign-in, signout, protected routes, `require_auth`, RBAC, auth decorators, or Google or GitHub OAuth in Caspian.
related:
  title: Related docs
  description: Use the fetch-data guide when sign-in or signout happens through RPC, then use the state guide for transient auth-adjacent request state, validation to guard credentials, and routing or project structure to place auth files correctly.
  links:
    - /docs/core-runtime-map
    - /docs/fetch-data
    - /docs/state
    - /docs/validation
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page explains the current Caspian authentication API based on the centralized app config pattern in `src/lib/auth/auth_config.py` and the installed `casp.auth` implementation.

Treat `casp.auth` as the default authentication layer in Caspian app code. Do not build parallel session helpers, direct cookie-writing utilities, or route-by-route auth conventions when `AuthSettings`, `configure_auth(...)`, `auth`, and the built-in decorators already define the access boundary.

## Overview

Caspian authentication has two main layers:

- app-level policy, controlled by `src/lib/auth/auth_config.py`
- framework runtime behavior, implemented by `.venv/Lib/site-packages/casp/auth.py`

The main public API includes:

- `AuthSettings` for centralized app auth policy
- `configure_auth(...)` and `get_auth_settings()` for app startup and reads
- the global `auth` object for session lifecycle work
- `require_auth`, `require_role`, and `guest_only` for page-level protection
- `@rpc(require_auth=True, allowed_roles=[...])` for action-level protection
- `GoogleProvider` and `GithubProvider` for OAuth provider setup

Import the common API like this:

```python
from casp.auth import (
    AuthSettings,
    configure_auth,
    auth,
    require_auth,
    require_role,
    guest_only,
)
```

## Default Auth Rule

- Define app-wide auth behavior in `src/lib/auth/auth_config.py` through `build_auth_settings()` and apply it once at startup with `configure_auth(...)`.
- Use `auth.sign_in(...)` and `auth.sign_out(...)` instead of setting or clearing session keys directly.
- Prefer `pp.rpc(...)` plus `@rpc(require_auth=True)` for signout buttons or menus rendered in pages or components. Use a dedicated signout route only when you need a plain HTML form POST or a no-JavaScript fallback.
- Use `@require_auth`, `@require_role`, and `@guest_only` for page access rules.
- Use `@rpc(require_auth=True, allowed_roles=[...])` for browser-triggered actions that need protection.
- Use `StateManager` only for transient auth-adjacent request state; keep the authenticated session itself owned by `casp.auth`.
- Keep secrets and provider credentials in `.env`; keep route visibility, redirects, and RBAC policy in `src/lib/auth/auth_config.py`.
- Do not implement custom `next` parsing or post-login redirect selection inside sign-in UI flows when using the built-in Caspian auth stack. The runtime and centralized auth config already own guest redirects, auth-route redirects, and the default post-login target.
- Validate login, signup, reset-password, and profile-mutation inputs before hitting the database or external providers.

## Framework Internals Note

- The centralized app auth policy controller is `src/lib/auth/auth_config.py`.
- The installed framework implementation lives in `.venv/Lib/site-packages/casp/auth.py`.
- Treat `auth_config.py` as project code and `casp/auth.py` as framework runtime code. Do not edit `casp/auth.py` to control app route privacy, redirects, or RBAC.
- If upstream docs and the installed implementation disagree, prefer the installed implementation for local project guidance.
- Use [core-runtime-map.md](./core-runtime-map.md) when an auth task crosses `main.py` bootstrap behavior such as development cookie scoping or middleware ownership.

## Centralized Auth Settings

Keep application auth policy in `src/lib/auth/auth_config.py`. This file is the controller for route privacy, redirects, and RBAC.

Example:

```python
from __future__ import annotations
from casp.auth import AuthSettings


def build_auth_settings() -> AuthSettings:
    """
    Centralized app auth policy controller.

    Keep secrets (AUTH_SECRET, AUTH_COOKIE_NAME) in .env.
    Keep app-level session settings in .env (SESSION_LIFETIME_HOURS, etc).
    Decide route privacy, redirects, and RBAC here at app setup time instead of
    changing Caspian core runtime files.
    """

    return AuthSettings(
        # Token settings
        default_token_validity="1h",
        token_auto_refresh=False,

        # Route protection
        is_all_routes_private=False,
        public_routes=["/"],
        auth_routes=["/signin", "/signup"],
        private_routes=[],  # unused when all-routes-private is True

        # Role-based access
        is_role_based=False,
        role_identifier="role",

        # RBAC policy is app-owned here; the runtime expects ROUTE/PATTERN -> [ROLES].
        role_based_routes={},

        # Redirects / prefixes
        default_signin_redirect="/dashboard",
        default_signout_redirect="/signin",
        api_auth_prefix="/api/auth",
    )
```

Important behavior from the current implementation:

- `default_token_validity` is parsed by the installed auth runtime with the format `^\d+(s|m|h|d)$`. Use values such as `30m`, `1h`, or `7d`.
- `token_auto_refresh` only changes behavior when the request lifecycle calls `auth.refresh_session()`. In the installed `auth.py`, the flag alone does not refresh expiry by itself.
- The framework `AuthSettings` dataclass defaults `is_all_routes_private=True`, but the project example above explicitly changes that to `False`.
- In generated app-owned starter config, `src/lib/auth/auth_config.py` starts with `is_all_routes_private=False`, so routes are public by default until the app chooses stricter route protection.
- `public_routes`, `auth_routes`, `private_routes`, and `role_based_routes` are matched as route **scopes**, not exact paths, in the installed `Auth` methods. An entry covers the path itself *and* everything nested under it, so `/admin` also matches `/admin/users`. Segment boundaries are respected (`/admin` does not match `/administration`), and dynamic segments work (`/orders/[id]` matches `/orders/42` and below). Root `/` is the one exception: it matches only `/`, so listing it in `public_routes` does not make the whole site public. Verify the exact rule in `_route_scope_matches_request` before relying on an entry to protect or expose a subtree.
- `private_routes` matters only when `is_all_routes_private=False`.
- `role_based_routes` currently expects `PATH -> [ROLES]`, not role names keyed to paths the other way around.
- `role_identifier` defaults to `role`, and the current `auth.sign_in(...)` flow also normalizes `userRole` into `role` when possible.
- `api_auth_prefix` defaults to `/api/auth` and should stay centralized here rather than being hard-coded across routes.

## Redirect Ownership

Treat redirect behavior as framework-owned plus config-owned, not sign-in-page-owned.

- Keep the redirect policy in `src/lib/auth/auth_config.py`, especially `default_signin_redirect` and `default_signout_redirect`.
- In the current runtime, unauthenticated requests to protected routes are redirected by middleware and decorators to `/signin?next=...`.
- In the current runtime, authenticated users who hit an auth route such as `/signin` are redirected by the auth layer to `default_signin_redirect`, which defaults to `/dashboard`.
- `guest_only()` also prefers a safe `next` query value for already-authenticated users and otherwise falls back to `default_signin_redirect`.
- Because this behavior already exists in `main.py`, `.venv/Lib/site-packages/casp/auth.py`, and `src/lib/auth/auth_config.py`, sign-in implementations should not add their own redirect-decision layer, duplicate `next` handling, or hard-code a dashboard redirect inside the sign-in action.
- When building sign-in flows, focus on credential validation, calling `auth.sign_in(...)` or the relevant provider flow, and rendering errors. Let Caspian decide where the user goes next.

Use this rule of thumb for AI-generated sign-in work:

- If the task is only to build or style the sign-in page, do not add redirect support.
- If the task is to change where authenticated users land after sign-in, update `default_signin_redirect` in `src/lib/auth/auth_config.py` instead of editing the sign-in route logic.
- If the task is to change how protected routes send guests to sign-in, inspect the installed auth runtime and middleware instead of adding local redirect code to the sign-in page.

## Choosing Public vs Private Route Mode

Make this decision at app setup time in `src/lib/auth/auth_config.py`.

- In the default app-owned starter pattern, routes start public because `is_all_routes_private=False` in `src/lib/auth/auth_config.py`.
- If the application has only a few public pages and most routes require auth, set `is_all_routes_private=True` and list the exceptions in `public_routes`.
- If the application has many public pages and only a few protected areas, keep `is_all_routes_private=False` and list only the protected routes in `private_routes`.
- In the current runtime, `auth_routes=["/signin", "/signup"]` stays public by default, and most apps do not need to change it unless the user explicitly asks for different auth routes.
- In all-private mode, the default `public_routes=["/"]` keeps the home page public unless you change that list.
- `token_auto_refresh=True` does not make routes private. It only enables sliding-session refresh when the request lifecycle calls `auth.refresh_session()`.
- Do not modify Caspian core files for this decision. Keep the policy in `src/lib/auth/auth_config.py`.
- If you customize `src/lib/auth/auth_config.py`, add it to `excludeFiles` in `caspian.config.json` so update commands do not overwrite your local auth policy.

Example all-private setup with a few public exceptions:

```python
return AuthSettings(
    default_token_validity="1h",
    token_auto_refresh=False,
    is_all_routes_private=True,
    public_routes=["/", "/pricing", "/about"],
    auth_routes=["/signin", "/signup"],
    private_routes=[],
)
```

Example mixed app with many public routes and only a few protected areas:

```python
return AuthSettings(
    default_token_validity="1h",
    token_auto_refresh=False,
    is_all_routes_private=False,
    public_routes=["/"],
    auth_routes=["/signin", "/signup"],
    private_routes=["/dashboard", "/settings", "/billing"],
)
```

## Environment Variables

The installed auth code reads several values from `.env` when explicit values are not passed.

- `AUTH_SECRET` backs `AuthSettings.secret_key`.
- App-owned startup helpers may also validate `AUTH_SECRET` and refuse production startup when the value is missing or still on a default placeholder.
- `AUTH_COOKIE_NAME` backs `AuthSettings.cookie_name`.
- `SESSION_LIFETIME_HOURS` controls `SessionMiddleware.max_age` in the current `main.py` bootstrap.
- `APP_ENV` selects the environment, resolved **fail-closed** by `casp.runtime_security.is_production_environment()`: only an explicit development value (`dev`, `development`, `local`, `staging`, `test`, `testing`) turns the development relaxations on, so an unset or misspelled value is treated as production. Production enables secure session cookies and the `Secure` flag on the CSRF cookie.
- `CASPIAN_BROWSER_SYNC_PORT` can override the development cookie scope suffix used by the current `main.py` bootstrap.
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, and `GOOGLE_REDIRECT_URI` back `GoogleProvider`.
- `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, and `GITHUB_REDIRECT_URI` back `GithubProvider`.
- `GITHUB_REDIRECT_URI` is optional and behaves differently from the Google one. Google is skipped entirely without `GOOGLE_REDIRECT_URI`; GitHub still signs in without it, because an authorize request that omits `redirect_uri` lands on the **first** redirect URI registered on the GitHub App. That is why registering a second URI (a localhost one beside production) changes nothing on its own — each environment has to name the URI it wants. When the variable is empty and `APP_BASE_URL` is set, the provider derives `<APP_BASE_URL><api_auth_prefix>/callback/github`; when both are empty it sends no `redirect_uri` and GitHub applies its first-registered default. A resolved value is sent on the authorize request **and** repeated in the token exchange, as GitHub requires.

Keep these secrets out of route files and out of committed source.

In the current bootstrap, set `AUTH_COOKIE_NAME` explicitly in `.env`. The pasted `main.py` uses `session` as the fallback `SessionMiddleware` cookie name, while `AuthSettings` falls back to `auth_token` when the env var is absent.

In development, the current `main.py` does not always use those base cookie names directly. `_scoped_cookie_name(...)` appends the active BrowserSync or dev port to both the session cookie name and the CSRF cookie name when `APP_ENV` is not `production`. The scope is resolved from `CASPIAN_BROWSER_SYNC_PORT`, then from the `local` URL in `settings/bs-config.json` when its host is `localhost` or `127.0.0.1`. There is no third fallback — `PORT` is not read — and a scope that is missing or non-numeric yields `""`, leaving the base names unsuffixed. That means a local stack can emit names such as `session_5091` and `pp_csrf_5091` instead of the unsuffixed production names.

## Startup Wiring

Apply the centralized settings once during startup, before normal request handling begins.

Example:

```python
from casp.auth import configure_auth
from src.lib.auth.auth_config import build_auth_settings


configure_auth(build_auth_settings())
```

Use `get_auth_settings()` when other app code only needs to read the resolved settings.

## `main.py` Bootstrap

In the current app bootstrap, authentication is initialized in `main.py` before route registration and middleware execution.

Example:

```python
from casp.auth import (
    Auth,
    GoogleProvider,
    GithubProvider,
    configure_auth,
)
from src.lib.auth.auth_config import build_auth_settings


def setup_auth():
    configure_auth(build_auth_settings())
    Auth.set_providers(GithubProvider(), GoogleProvider())


setup_auth()
```

This does two things:

- applies the centralized settings once at startup
- registers OAuth providers once so `AuthMiddleware` can delegate signin and callback paths through `auth.auth_providers(...)`

The current `main.py` also wires the session middleware directly, and some apps delegate the secret lookup to an app-owned helper:

```python
from starlette.middleware.sessions import SessionMiddleware
from casp.runtime_security import get_session_secret, is_production_environment


SESSION_LIFETIME_HOURS = int(os.getenv("SESSION_LIFETIME_HOURS", 7))
IS_PRODUCTION = is_production_environment()


app.add_middleware(
    SessionMiddleware,
    secret_key=get_session_secret(is_production=IS_PRODUCTION),
    session_cookie=os.getenv("AUTH_COOKIE_NAME", "session"),
    max_age=SESSION_LIFETIME_HOURS * 3600,
    same_site="lax",
    https_only=IS_PRODUCTION,
    path="/",
)
```

Keep this wiring in `main.py`. Keep only the policy values in `src/lib/auth/auth_config.py`. Runtime security helpers for session-secret enforcement, safe public-file serving, production-safe error messages, and baseline response headers are package-owned by `casp.runtime_security`; keep the middleware attachment itself in `main.py`.

## Session Lifetime Rules

The current app bootstrap has two independent expiration controls:

- `default_token_validity` controls the auth payload expiration stored by `auth.sign_in(...)`
- `SESSION_LIFETIME_HOURS` controls the signed session cookie lifetime set by `SessionMiddleware`

The effective authenticated session lasts only while both are valid.

- If the session cookie expires first, the session disappears even if the auth payload expiration was longer.
- If the auth payload expires first, `auth.is_authenticated()` clears it even if the session cookie still exists.
- In apps where the request flow does not call `auth.refresh_session()`, `token_auto_refresh=True` still does nothing by itself.

Keep `SESSION_LIFETIME_HOURS` and `default_token_validity` aligned unless you intentionally want one boundary to end earlier than the other.

## Middleware Order

The current `main.py` adds middleware in this source order:

```python
app.add_middleware(RPCMiddleware)
app.add_middleware(AuthMiddleware)
app.add_middleware(CSRFMiddleware)
app.add_middleware(SessionMiddleware, ...)
app.add_middleware(BodySizeLimitMiddleware)
app.add_middleware(MissingPublicAssetMiddleware)
app.add_middleware(RateLimitMiddleware)
app.add_middleware(
    PublicFilesMiddleware,
    directory="public",
    inline_safe_subdirectories={
        "uploads": INLINE_SAFE_UPLOAD_MEDIA_TYPES,
    },
)
app.add_middleware(SecurityHeadersMiddleware)
# RequestDiagnosticsMiddleware is added last outside production.
```

Because Starlette runs the last-added middleware first, the effective request order is:

1. `RequestDiagnosticsMiddleware` outside production
2. `SecurityHeadersMiddleware`
3. `PublicFilesMiddleware`
4. `RateLimitMiddleware`
5. `MissingPublicAssetMiddleware`
6. `BodySizeLimitMiddleware`
7. `SessionMiddleware`
8. `CSRFMiddleware`
9. `AuthMiddleware`
10. `RPCMiddleware`
11. route handler or RPC endpoint

Current behavior by layer:

- `SecurityHeadersMiddleware` attaches baseline response headers from `casp.runtime_security`, including the Content-Security-Policy, `X-Content-Type-Options`, framing policy, referrer policy, permissions policy, and production HSTS, while preserving headers already set by the response. `CONTENT_SECURITY_POLICY` replaces the default CSP wholesale when the app needs a different policy.
- `PublicFilesMiddleware` serves any existing `GET`/`HEAD` file under `public/**` before rate limiting, sessions, CSRF, auth, RPC, or page routing. It falls through for missing files and preserves restricted inline-media handling for configured upload directories.
- `RateLimitMiddleware` applies the page budget to requests that were not already served as existing public files; `/health` is explicitly exempt.
- `MissingPublicAssetMiddleware` returns a 404 for a **missing** file whose first path segment is a real `public/` directory. `PublicFilesMiddleware` deliberately falls through on a miss so routing keeps working, but under `is_all_routes_private=True` that miss would otherwise reach `AuthMiddleware` and answer `303 -> /signin` — so a `<script src="/js/typo.js">` would receive the sign-in *page* as `200 text/html` and fail with a parse error naming the wrong file. It sits inside the rate limiter (a 404 flood is still a flood) and outside sessions, CSRF, and auth, so a missing asset costs no session decryption.
- `BodySizeLimitMiddleware` rejects oversized request bodies before session, auth, RPC, or route parsing.
- `SessionMiddleware` provides `request.session` for the rest of the stack.
- `CSRFMiddleware` ensures `request.session["csrf_token"]` exists and emits a scoped CSRF cookie based on `pp_csrf`, for example `pp_csrf_5091` in development or plain `pp_csrf` when no dev scope is active.
- `AuthMiddleware` sets request context with `Auth.set_request(request)`, initializes `StateManager`, runs provider callbacks, and enforces public, auth, private, and role-based route redirects. Existing public files never reach it because `PublicFilesMiddleware` serves them first.
- `RPCMiddleware` handles `POST` requests with `X-PP-RPC: true` and forwards them to Caspian's RPC handler after auth and session setup are already available.

Keep `SessionMiddleware` outside CSRF, auth, and RPC so those inner layers can access `request.session`. Keep `PublicFilesMiddleware` outside rate limiting, body parsing, sessions, CSRF, auth, and RPC, but inside `SecurityHeadersMiddleware`, so public files avoid per-user work and cookies while retaining security headers.

## Current `AuthMiddleware` Flow

The pasted `main.py` auth middleware currently behaves like this:

- receives no request for an existing public file because the outer `PublicFilesMiddleware` has already served it; missing paths continue through normal auth and routing behavior
- initializes request-bound auth state with `StateManager.init(request)` and `Auth.set_request(request)`
- runs OAuth provider signin and callback handling before public or private route checks
- lets public routes through immediately
- redirects authenticated users away from auth routes such as `/signin` and `/signup`
- applies role-based redirects before generic private-route redirects when `is_role_based=True`
- redirects unauthenticated private-route requests to `/signin?next=/current/path`

Because `AuthMiddleware` initializes `StateManager` on every request, auth flows can also use [state.md](./state.md) for transient request-scoped success or error state. Keep identity, session lifetime, and authorization decisions in `auth.sign_in(...)`, `auth.sign_out(...)`, and the auth decorators rather than moving them into the state manager.

`AuthMiddleware` only runs on `scope["type"] == "http"`; it early-returns on WebSocket scopes. A private page route therefore does not protect a WebSocket. To keep socket auth aligned with page auth, reuse the same `Auth` instance inside the socket guard: bind the socket with `Auth.set_request(websocket)` (a `WebSocket` exposes `.session`, the one piece `Auth` reads) and call `auth.is_authenticated()`, `auth.get_payload()`, and `auth.check_role(...)`. See [websockets.md](./websockets.md) for the full guard pattern and the read-only-session caveat.

## The `auth` Object

The global `auth` singleton owns the session lifecycle.

The current installed methods are:

| Method                                                       | Purpose                    | Current behavior                                                                                         |
| ------------------------------------------------------------ | -------------------------- | -------------------------------------------------------------------------------------------------------- |
| `auth.sign_in(data, token_validity=None, redirect_to=False)` | Create a session           | Stores the payload in the session, sets a CSRF token, and returns either `"ok"` or a `RedirectResponse`. |
| `auth.sign_out(redirect_to=None)`                            | Destroy a session          | Clears the session and redirects to the explicit target or `default_signout_redirect`.                   |
| `auth.is_authenticated()`                                    | Check current auth state   | Returns `False` when the payload is missing, malformed, or expired, and clears invalid session data.     |
| `auth.get_payload()`                                         | Read the signed-in payload | Returns the stored dict payload, or wraps non-dict payloads as `{"value": data}`.                        |
| `auth.refresh_session()`                                     | Extend expiration          | Only updates expiry when `token_auto_refresh=True`.                                                      |
| `auth.check_role(user, allowed_roles)`                       | Check RBAC access          | Reads the configured role field (a single role or a list of them) and grants access on any overlap.      |

Sign-in example:

```python
from casp.auth import auth, guest_only
from casp.component_decorator import html
from casp.rpc import rpc
from casp.validate import Rule, Validate
from src.lib.prisma import prisma
from werkzeug.security import check_password_hash


@guest_only()
def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")


@rpc()
async def do_login(email: str, password: str, next: str | None = None):
    clean_email = Validate.email(email)
    password_check = Validate.with_rules(password, [Rule.REQUIRED, Rule.min(8)])

    if clean_email is None:
        return {"error": "Invalid email address."}

    if password_check is not True:
        return {"error": password_check}

    user = await prisma.user.find_unique(
        where={"email": clean_email},
        include={"userRole": True},
    )

    if not user:
        return {"error": "Invalid credentials."}

    if not user.password or not check_password_hash(user.password, password):
        return {"error": "Invalid credentials."}

    user_data = user.to_dict(omit={"password": True})
    redirect_url = next or auth.settings.default_signin_redirect

    return auth.sign_in(user_data, redirect_to=redirect_url)
```

Implementation details that matter:

- `auth.sign_in(...)` accepts dict payloads and simpler values, but dict payloads are the normal choice for app auth.
- When a dict payload includes `userRole`, the installed implementation tries to copy that into a top-level `role` field so RBAC checks can work against the default `role_identifier`.
- `redirect_to=True` means use `default_signin_redirect`.
- Without a redirect target, `auth.sign_in(...)` returns the string `"ok"`.

## Protecting Routes And Actions

Use the smallest protection level that matches the behavior you need.

### RPC Action Level

Use action-level protection when a page can render publicly or partially, but a specific mutation or read must be authenticated.

```python
from casp.rpc import rpc


@rpc(require_auth=True, allowed_roles=["admin"], limits="5/minute")
async def delete_user(user_id: str):
    return {"deleted": user_id}
```

Use this pattern for destructive mutations, private data fetches, and admin-only buttons. See `fetch-data.md` for the broader RPC flow.

### Page Level

Use the page decorators from `casp.auth` when the whole route should enforce an access rule.

Authenticated-only page:

```python
from casp.auth import require_auth
from casp.component_decorator import html


@require_auth()
def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")
```

Role-protected page:

```python
from casp.auth import require_role
from casp.component_decorator import html


@require_role("admin", "superadmin")
def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")
```

Guest-only page:

```python
from casp.auth import guest_only
from casp.component_decorator import html


@guest_only()
def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")
```

Current decorator behavior from the installed `auth.py`:

- `@require_auth()` redirects unauthenticated users to `/signin?next=/requested/path` unless you pass a custom `redirect_to`.
- `@require_role(...)` redirects unauthenticated users to `/signin?next=/requested/path` and redirects authenticated but unauthorized users to `/unauthorized` unless you override `redirect_to`.
- `@guest_only()` redirects already-authenticated users to `auth.settings.default_signin_redirect` or the current request's `next` value.

### Central Route Rules

The current `Auth` implementation also exposes central route checks:

- `auth.is_public_route(path)`
- `auth.is_auth_route(path)`
- `auth.is_private_route(path)`
- `auth.get_required_roles(path)`

These helpers use exact string comparisons against the routes you configured in `AuthSettings`.

## Common Route Workflow

A typical credential-based auth setup often uses this route shape:

```text
src/
    app/
        (auth)/
            layout.py
            signin/
                index.py
                index.py
            signup/
                index.py
                index.py
            signout/
                index.py
        dashboard/
            index.py
            layout.py
            settings/
                index.py
                index.py
```

Use `guest_only()` on signin and signup routes, use `auth.sign_in(...)` inside the owning RPC action after validation and credential checks, prefer RPC for signout UI actions, and protect private destinations with `require_auth()` or central route policy.

When auth pages share a wrapper without adding a URL segment, use a route group such as `(auth)/layout.py`. When a protected area such as `dashboard` owns multiple child routes, apply the section layout pattern from [routing.md](./routing.md): put the shared shell in `dashboard/layout.py` and place child routes such as `dashboard/settings/index.py` beneath it.

Preferred signout pattern: auth-protected RPC from a page or component.

Use this when the logout trigger lives in a page, layout, header, dropdown, or reusable component. The HTML can live at page level or component level.

```html
<div>
    <button onclick="signout()">
        Sign out
    </button>

    <script>
        async function signout() {
            await pp.rpc("signout");
        }
    </script>
</div>
```

```python
from casp.auth import auth
from casp.rpc import rpc


@rpc(require_auth=True)
def signout():
    return auth.sign_out()
```

Fallback signout pattern: dedicated route.

Use this when you need a plain HTML form POST, a no-JavaScript fallback, or a full-route signout endpoint.

HTML:

```html
<form action="/signout" method="post">
    <button type="submit">
        <span>Log Out</span>
    </button>
</form>
```

Route: `src/app/(auth)/signout/index.py`

```python
from casp.auth import auth, require_auth


@require_auth()
def page():
    return auth.sign_out()
```

Simple protected dashboard example:

```python
from casp.auth import auth, require_auth
from casp.component_decorator import html


@require_auth()
def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""", **{
        "user": auth.get_payload(),
    })
```

If the protected destination is a section rather than a single page, prefer a folder-level layout for the shared shell.

```text
src/
    app/
        dashboard/
            layout.py
            index.py
            settings/
                index.py
                index.py
            billing/
                index.py
                index.py
```

In that pattern, `dashboard/layout.py` owns the shared sidebar, header, and frame, while each child route decides whether it needs its own `index.py` for auth checks, metadata, RPC actions, or server-side data.

For signup flows, use the same route ownership pattern as signin: validate the input, create the user, build a safe payload without password fields, and then call `auth.sign_in(...)` with the redirect target you want.

Signup example:

```python
from casp.auth import auth, guest_only
from casp.component_decorator import html
from casp.rpc import rpc
from casp.validate import Rule, Validate
from src.lib.prisma import prisma
from werkzeug.security import generate_password_hash


@guest_only()
def page():
    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")


@rpc()
async def do_signup(
    name: str,
    email: str,
    password: str,
    password_confirmation: str,
):
    clean_name = Validate.string(name)
    clean_email = Validate.email(email)
    password_check = Validate.with_rules(
        password,
        [Rule.REQUIRED, Rule.min(8), Rule.confirmed()],
        confirmation_value=password_confirmation,
    )

    if clean_name is None or len(clean_name) < 2:
        return {"error": "Name must be at least 2 characters."}

    if clean_email is None:
        return {"error": "Invalid email address."}

    if password_check is not True:
        return {"error": password_check}

    existing_user = await prisma.user.find_unique(
        where={"email": clean_email},
    )
    if existing_user:
        return {"error": "An account with that email already exists."}

    user = await prisma.user.create(
        data={
            "name": clean_name,
            "email": clean_email,
            "password": generate_password_hash(password),
        }
    )

    user_data = user.to_dict(omit={"password": True})
    return auth.sign_in(user_data, redirect_to=True)
```

Adjust the `create(...)` payload to match your actual Prisma schema. The important pattern is: validate first, hash the password before storage, omit sensitive fields from the session payload, and let `auth.sign_in(...)` create the session.

## Account Lifecycle Patterns

The installed `casp.auth` implementation covers session creation, session destruction, route protection, RBAC checks, CSRF token seeding, and Google or GitHub OAuth callbacks. It does not implement full account lifecycle flows such as:

- password reset token issuance and redemption
- email verification tokens
- forced signout across all active devices
- remember-device or multi-factor workflows

Build those flows in app code under `src/app/` and `src/lib/auth/`.

Recommended route placement:

```text
src/
  app/
    (auth)/
      forgot-password/
        index.py
        index.py
      reset-password/
        [token]/
          index.py
          index.py
      verify-email/
        [token]/
          index.py
          index.py
```

Recommended workflow:

- validate the incoming email, password, or token with `Validate` and `Rule`
- create a short-lived token in your database or a dedicated token table
- send the token link with your app's mailer or background job flow
- verify token expiry and ownership before updating the user record
- call `auth.sign_in(...)` only after the final credential or verification state is correct
- call `auth.sign_out()` when a reset or security event should invalidate the current session

Keep token generation, hashing helpers, and mail-sending wrappers in `src/lib/auth/` when multiple routes reuse them.

## Role-Based Access

When role checks are enabled in app policy, Caspian matches the current path against `role_based_routes` and checks the payload field named by `role_identifier`.

Example RBAC config:

```python
from casp.auth import AuthSettings


def build_auth_settings() -> AuthSettings:
    return AuthSettings(
        is_role_based=True,
        role_identifier="role",
        role_based_routes={
            "/report": ["admin"],
            "/admin": ["admin", "superadmin"],
        },
    )
```

In the installed implementation:

- `auth.check_role(...)` reads `user[role_identifier]` when the payload is a dict.
- If you pass a plain string instead of a dict, that string is treated as the role directly.
- That field may hold a single role or a list/tuple/set of them, so a user who is both `editor` and `auditor` passes a check for either. Access is granted when the user's roles and the allowed roles intersect.
- `role_based_routes` keys are matched as route *scopes*, not exact paths: `get_required_roles(path)` ranks every matching key by specificity and returns the most specific one. A key therefore also covers nested paths (`/admin` gates `/admin/users`), and dynamic segments such as `/orders/[id]` match. A static key wins over a dynamic one, which wins over a catch-all.
- The installed payload normalization helps when your Prisma include returns a `userRole` object and you want a plain `role` field in the session payload.

## OAuth Providers

The installed `casp.auth` file includes provider helpers for Google and GitHub OAuth.

In the app-owned starter bootstrap, both providers are already registered. Treat Google and GitHub sign-in as a shipped feature, not something to build from scratch. The current `main.py` calls:

```python
Auth.set_providers(GithubProvider(), GoogleProvider())
```

so `AuthMiddleware` already delegates the provider signin and callback paths through `auth.auth_providers(...)` on every request. To add a working social sign-in to a page, you usually need only two things:

- a link or button that navigates to the provider signin path, and
- the provider credentials in `.env`.

The signin and callback paths live under `api_auth_prefix` (default `/api/auth`):

- Google sign-in link target: `/api/auth/signin/google`
- GitHub sign-in link target: `/api/auth/signin/github`
- Google callback (provider-side redirect URI): `/api/auth/callback/google`
- GitHub callback (provider-side redirect URI): `/api/auth/callback/github`

Minimal sign-in button:

```html
<a href="/api/auth/signin/google">Continue with Google</a>
<a href="/api/auth/signin/github">Continue with GitHub</a>
```

Do not reinvent this flow. When a task asks for Google or GitHub login, do not add `authlib`, a hand-written `httpx2` token-exchange, a parallel session writer, or custom `/callback` route handlers. The shipped providers plus `auth.auth_providers(...)` already perform the redirect, the code exchange, payload normalization, `auth.sign_in(...)`, and the post-login redirect to `default_signin_redirect`. Reuse them and keep credentials in `.env`.

Available provider classes:

- `GoogleProvider(client_id, client_secret, redirect_uri, max_age="30d")`
- `GithubProvider(client_id, client_secret, max_age="30d")`

The current `auth.auth_providers(*providers)` implementation:

- redirects to the provider when the request path contains `signin/google` or `signin/github`
- handles callbacks when the path contains `callback/google` or `callback/github` and a `code` query parameter is present
- exchanges the code with the provider using `httpx2`
- builds a normalized user payload
- signs the user in with the provider's `max_age`
- redirects to `default_signin_redirect` on success

Minimal example:

```python
from casp.auth import GoogleProvider, GithubProvider, auth
from casp.component_decorator import html


google = GoogleProvider()
github = GithubProvider()


def page():
    response = auth.auth_providers(google, github)
    if response:
        return response

    return html(r"""
<section class="page">
  <!-- page markup -->
</section>
""")
```

Use this only when the route is actually handling an auth-provider signin or callback flow.

## CSRF Token Helper

The installed auth runtime also exposes `get_csrf_token()`.

Behavior:

- `auth.sign_in(...)` seeds `csrf_token` into the session.
- `get_csrf_token()` reads the existing token or creates one when a request session exists.
- If no request or session is available, it returns an empty string.

Use this helper when custom form or fetch flows need access to the session CSRF token.

## Current Implementation Notes

- The installed auth runtime is session-backed. It stores the auth payload and CSRF token in `request.session`.
- Expiration uses timestamps and the current duration parser only accepts `s`, `m`, `h`, and `d` units.
- Expired or malformed payloads are removed during `auth.is_authenticated()` checks.
- `auth.get_payload()` returns `None` for missing payloads, a dict for dict payloads, and `{"value": ...}` for non-dict payloads.
- OAuth callback helpers return `None` on failure, so calling routes should be prepared for a normal page render or explicit error state when provider login fails.
- The current app bootstrap may wrap auth and session handling in an outer `SecurityHeadersMiddleware`; auth-aware code still depends on `SessionMiddleware`, `CSRFMiddleware`, `AuthMiddleware`, and `RPCMiddleware` staying in the correct relative order so sessions and CSRF state exist before route checks or RPC handling.
- Password reset, email verification, and other account-recovery workflows are application responsibilities layered on top of `casp.auth`, not built-in auth runtime features.
- Route lists and RBAC maps are exact-path checks, not wildcard or prefix rules.

## Recommended Usage Pattern

Validate and authenticate close to the input boundary.

Common placement patterns are:

- put centralized policy in `src/lib/auth/auth_config.py`
- decide public-vs-private route mode at app start in `src/lib/auth/auth_config.py` instead of scattering route privacy logic across route files
- use `is_all_routes_private=True` only when most routes should require auth; otherwise keep `is_all_routes_private=False` and maintain `private_routes`
- keep public exceptions in `public_routes`, and leave `auth_routes=["/signin", "/signup"]` alone unless the app explicitly needs different auth endpoints
- call `configure_auth(build_auth_settings())` during startup
- keep sign-in, signup, and reset-password handlers in the owning route backend under `src/app/**/index.py`
- prefer signout through `pp.rpc("signout")` with `@rpc(require_auth=True)` when the trigger lives in page or component UI
- use a dedicated signout route only for no-JavaScript, form-post, or full-navigation edge cases
- use `@rpc()` for browser-triggered auth actions and page decorators for full-route protection
- use `Validate` and `Rule` from `casp.validate` before password checks, user lookup, persistence, or external provider flows
- keep reusable auth helpers in `src/lib/auth/`
- protect customized auth policy files from updater overwrite with `excludeFiles` in `caspian.config.json`

For browser-triggered auth forms, pair this page with `fetch-data.md`. For credential and form validation, pair it with `validation.md`.

## AI Retrieval Notes

If an AI agent is deciding how to handle auth in Caspian, apply these rules first.

- Treat `casp.auth` as the default authentication layer.
- Centralize route visibility, redirects, and RBAC policy in `src/lib/auth/auth_config.py`.
- Decide route mode early in `src/lib/auth/auth_config.py`: use `is_all_routes_private=True` when most routes should require auth, otherwise keep `is_all_routes_private=False` and list protected routes in `private_routes`.
- In app-owned starter config, treat `is_all_routes_private=False` as the default starting point, which means routes begin public until the app opts into stricter protection.
- Treat `public_routes` as the public exception list for all-private apps. In the current defaults, `/` stays public and `auth_routes=["/signin", "/signup"]` stays public too.
- For protected grouped sections, follow the section layout pattern in [routing.md](./routing.md) and keep auth-specific route policy in `src/lib/auth/auth_config.py`.
- Apply settings at startup in `main.py` with `configure_auth(build_auth_settings())`.
- Treat Google and GitHub OAuth as shipped: the starter `main.py` already registers both providers with `Auth.set_providers(GithubProvider(), GoogleProvider())`. For social sign-in, link to `/api/auth/signin/{google,github}` and set `.env` credentials; do not hand-roll OAuth, token exchange, or callback routes. Only call `Auth.set_providers(...)` yourself if you are changing which providers are registered.
- Use `auth.sign_in(...)` and `auth.sign_out(...)` for session lifecycle changes.
- Prefer `pp.rpc("signout")` plus `@rpc(require_auth=True)` for logout buttons or menus in pages and components.
- Use a dedicated `/signout` route only for plain HTML form POST, no-JavaScript fallback, or other full-navigation edge cases.
- Do not treat `token_auto_refresh` as a route-privacy switch. In the current app it only affects sliding-session refresh if `auth.refresh_session()` is called.
- Use `@require_auth`, `@require_role`, and `@guest_only` for page-level access rules.
- Use `@rpc(require_auth=True, allowed_roles=[...])` for protected browser-triggered actions.
- Keep `SessionMiddleware` immediately inside any outer response-header wrapper so auth, CSRF, and RPC handlers can read `request.session`.
- Use [state.md](./state.md) only for transient auth-adjacent request state, not as the auth session store.
- Align `SESSION_LIFETIME_HOURS` with `default_token_validity` unless you intentionally want different cookie and auth-payload expiry windows.
- Prefer exact route strings in `public_routes`, `auth_routes`, `private_routes`, and `role_based_routes`.
- Protect customized `src/lib/auth/auth_config.py` from updater overwrite by adding it to `excludeFiles` in `caspian.config.json`.
- Keep auth secrets and OAuth provider credentials in `.env`.
- Set `AUTH_COOKIE_NAME` explicitly so the session middleware cookie name and auth settings stay aligned.
- Use `.venv/Lib/site-packages/casp/auth.py` only when the task is about framework auth internals or debugging installed behavior.
- Use `main.py` plus `casp.runtime_security` when the task is about auth bootstrap, session middleware, safe public-file serving, response headers, or middleware execution order.
- Pair auth work with `fetch-data.md` for RPC forms and `validation.md` for credential validation.
- Check `routing.md` and `project-structure.md` before creating new auth routes or shared auth helpers.

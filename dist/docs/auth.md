---
title: Authentication
description: Manage session-backed authentication in Caspian with `casp.auth`, `AuthSettings`, the global `auth` object, centralized `auth_config.py`, page decorators, RPC protection, role-based routes, and optional Google or GitHub OAuth providers.
related:
  title: Related docs
  description: Use the fetch-data guide when sign-in happens through RPC, then use validation to guard credentials and routing or project structure to place auth files correctly.
  links:
    - /docs/fetch-data
    - /docs/validation
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page explains the current Caspian authentication API based on the centralized app config pattern in `src/lib/auth/auth_config.py` and the installed `casp.auth` implementation.

Treat `casp.auth` as the default authentication layer in Caspian app code. Do not build parallel session helpers, direct cookie-writing utilities, or route-by-route auth conventions when `AuthSettings`, `configure_auth(...)`, `auth`, and the built-in decorators already define the access boundary.

## Overview

Caspian authentication has two main layers:

- app-level policy in `src/lib/auth/auth_config.py`
- framework runtime behavior in `.venv/Lib/site-packages/casp/auth.py`

The main public API includes:

- `AuthSettings` for centralized auth configuration
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

- Define app-wide auth behavior in `build_auth_settings()` and apply it once at startup with `configure_auth(...)`.
- Use `auth.sign_in(...)` and `auth.sign_out(...)` instead of setting or clearing session keys directly.
- Use `@require_auth`, `@require_role`, and `@guest_only` for page access rules.
- Use `@rpc(require_auth=True, allowed_roles=[...])` for browser-triggered actions that need protection.
- Keep secrets and provider credentials in `.env`; keep route visibility, redirects, and RBAC policy in `src/lib/auth/auth_config.py`.
- Validate login, signup, reset-password, and profile-mutation inputs before hitting the database or external providers.

## Framework Internals Note

- The centralized app auth config belongs in `src/lib/auth/auth_config.py`.
- The installed framework implementation lives in `.venv/Lib/site-packages/casp/auth.py`.
- Treat `auth_config.py` as project code and `casp/auth.py` as framework code.
- If upstream docs and the installed implementation disagree, prefer the installed implementation for local project guidance.

## Centralized Auth Settings

Keep application auth policy in `src/lib/auth/auth_config.py`.

Example:

```python
from __future__ import annotations
from casp.auth import AuthSettings


def build_auth_settings() -> AuthSettings:
    """
    Centralized auth configuration.

    Keep secrets (AUTH_SECRET, AUTH_COOKIE_NAME) in .env.
    Keep app-level session settings in .env (SESSION_LIFETIME_HOURS, etc).
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

        # IMPORTANT: current casp.auth expects PATH -> [ROLES]
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
- `public_routes`, `auth_routes`, `private_routes`, and `role_based_routes` are exact path matches in the installed `Auth` methods.
- `private_routes` matters only when `is_all_routes_private=False`.
- `role_based_routes` currently expects `PATH -> [ROLES]`, not role names keyed to paths the other way around.
- `role_identifier` defaults to `role`, and the current `auth.sign_in(...)` flow also normalizes `userRole` into `role` when possible.
- `api_auth_prefix` defaults to `/api/auth` and should stay centralized here rather than being hard-coded across routes.

## Environment Variables

The installed auth code reads several values from `.env` when explicit values are not passed.

- `AUTH_SECRET` backs `AuthSettings.secret_key`.
- `AUTH_COOKIE_NAME` backs `AuthSettings.cookie_name`.
- `SESSION_LIFETIME_HOURS` controls `SessionMiddleware.max_age` in the current `main.py` bootstrap.
- `APP_ENV=production` enables secure session cookies and the `Secure` flag on the current CSRF cookie.
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, and `GOOGLE_REDIRECT_URI` back `GoogleProvider`.
- `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` back `GithubProvider`.

Keep these secrets out of route files and out of committed source.

In the current bootstrap, set `AUTH_COOKIE_NAME` explicitly in `.env`. The pasted `main.py` uses `session` as the fallback `SessionMiddleware` cookie name, while `AuthSettings` falls back to `auth_token` when the env var is absent.

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

The current `main.py` also wires the session middleware directly:

```python
from starlette.middleware.sessions import SessionMiddleware


SESSION_LIFETIME_HOURS = int(os.getenv("SESSION_LIFETIME_HOURS", 7))
IS_PRODUCTION = os.getenv("APP_ENV") == "production"


app.add_middleware(
    SessionMiddleware,
    secret_key=os.getenv("AUTH_SECRET", "change-me"),
    session_cookie=os.getenv("AUTH_COOKIE_NAME", "session"),
    max_age=SESSION_LIFETIME_HOURS * 3600,
    same_site="lax",
    https_only=IS_PRODUCTION,
    path="/",
)
```

Keep this wiring in `main.py`. Keep only the policy values in `src/lib/auth/auth_config.py`.

## Session Lifetime Rules

The current app bootstrap has two independent expiration controls:

- `default_token_validity` controls the auth payload expiration stored by `auth.sign_in(...)`
- `SESSION_LIFETIME_HOURS` controls the signed session cookie lifetime set by `SessionMiddleware`

The effective authenticated session lasts only while both are valid.

- If the session cookie expires first, the session disappears even if the auth payload expiration was longer.
- If the auth payload expires first, `auth.is_authenticated()` clears it even if the session cookie still exists.
- In the pasted `main.py`, `token_auto_refresh=True` still does nothing by itself because the request flow does not call `auth.refresh_session()`.

Keep `SESSION_LIFETIME_HOURS` and `default_token_validity` aligned unless you intentionally want one boundary to end earlier than the other.

## Middleware Order

The current `main.py` adds middleware in this source order:

```python
app.add_middleware(RPCMiddleware)
app.add_middleware(AuthMiddleware)
app.add_middleware(CSRFMiddleware)
app.add_middleware(SessionMiddleware, ...)
```

Because Starlette runs the last-added middleware first, the effective request order is:

1. `SessionMiddleware`
2. `CSRFMiddleware`
3. `AuthMiddleware`
4. `RPCMiddleware`
5. route handler or RPC endpoint

Current behavior by layer:

- `SessionMiddleware` provides `request.session` for the rest of the stack.
- `CSRFMiddleware` ensures `request.session["csrf_token"]` exists and emits a `pp_csrf` cookie.
- `AuthMiddleware` sets request context with `Auth.set_request(request)`, initializes `StateManager`, runs provider callbacks, skips configured static asset paths, and enforces public, auth, private, and role-based route redirects.
- `RPCMiddleware` handles `POST` requests with `X-PP-RPC: true` and forwards them to Caspian's RPC handler after auth and session setup are already available.

Keep `SessionMiddleware` outermost. If CSRF, auth, or RPC handling runs before it, `request.session` will not be available.

## Current `AuthMiddleware` Flow

The pasted `main.py` auth middleware currently behaves like this:

- bypasses auth checks for `/css/*`, `/js/*`, `/assets/*`, and `/favicon.ico`
- initializes request-bound auth state with `StateManager.init(request)` and `Auth.set_request(request)`
- runs OAuth provider signin and callback handling before public or private route checks
- lets public routes through immediately
- redirects authenticated users away from auth routes such as `/signin` and `/signup`
- applies role-based redirects before generic private-route redirects when `is_role_based=True`
- redirects unauthenticated private-route requests to `/signin?next=/current/path`

## The `auth` Object

The global `auth` singleton owns the session lifecycle.

The current installed methods are:

| Method | Purpose | Current behavior |
| --- | --- | --- |
| `auth.sign_in(data, token_validity=None, redirect_to=False)` | Create a session | Stores the payload in the session, sets a CSRF token, and returns either `"ok"` or a `RedirectResponse`. |
| `auth.sign_out(redirect_to=None)` | Destroy a session | Clears the session and redirects to the explicit target or `default_signout_redirect`. |
| `auth.is_authenticated()` | Check current auth state | Returns `False` when the payload is missing, malformed, or expired, and clears invalid session data. |
| `auth.get_payload()` | Read the signed-in payload | Returns the stored dict payload, or wraps non-dict payloads as `{"value": data}`. |
| `auth.refresh_session()` | Extend expiration | Only updates expiry when `token_auto_refresh=True`. |
| `auth.check_role(user, allowed_roles)` | Check RBAC access | Reads the configured role field from the payload and compares it to the allowed roles. |

Sign-in example:

```python
from casp.auth import auth, guest_only
from casp.layout import render_page
from casp.rpc import rpc
from casp.validate import Rule, Validate
from src.lib.prisma import prisma
from werkzeug.security import check_password_hash


@guest_only()
def page():
    return render_page(__file__)


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
from casp.layout import render_page


@require_auth()
def page():
    return render_page(__file__)
```

Role-protected page:

```python
from casp.auth import require_role
from casp.layout import render_page


@require_role("admin", "superadmin")
def page():
    return render_page(__file__)
```

Guest-only page:

```python
from casp.auth import guest_only
from casp.layout import render_page


@guest_only()
def page():
    return render_page(__file__)
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
            signin/
                index.py
                index.html
            signup/
                index.py
                index.html
        signout/
            index.py
        dashboard/
            index.py
            index.html
```

Use `guest_only()` on signin and signup routes, use `auth.sign_in(...)` inside the owning RPC action after validation and credential checks, and protect private destinations with `require_auth()` or central route policy.

Simple signout route example:

```python
from casp.auth import auth, require_auth


@require_auth()
def page():
    return auth.sign_out()
```

Simple protected dashboard example:

```python
from casp.auth import auth, require_auth
from casp.layout import render_page


@require_auth()
def page():
    return render_page(__file__, {
        "user": auth.get_payload(),
    })
```

For signup flows, use the same route ownership pattern as signin: validate the input, create the user, build a safe payload without password fields, and then call `auth.sign_in(...)` with the redirect target you want.

Signup example:

```python
from casp.auth import auth, guest_only
from casp.layout import render_page
from casp.rpc import rpc
from casp.validate import Rule, Validate
from src.lib.prisma import prisma
from werkzeug.security import generate_password_hash


@guest_only()
def page():
    return render_page(__file__)


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
        index.html
      reset-password/
        [token]/
          index.py
          index.html
      verify-email/
        [token]/
          index.py
          index.html
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
- `role_based_routes` is an exact-path lookup with `dict.get(path)`.
- The installed payload normalization helps when your Prisma include returns a `userRole` object and you want a plain `role` field in the session payload.

## OAuth Providers

The installed `casp.auth` file includes provider helpers for Google and GitHub OAuth.

Available provider classes:

- `GoogleProvider(client_id, client_secret, redirect_uri, max_age="30d")`
- `GithubProvider(client_id, client_secret, max_age="30d")`

The current `auth.auth_providers(*providers)` implementation:

- redirects to the provider when the request path contains `signin/google` or `signin/github`
- handles callbacks when the path contains `callback/google` or `callback/github` and a `code` query parameter is present
- exchanges the code with the provider using `httpx`
- builds a normalized user payload
- signs the user in with the provider's `max_age`
- redirects to `default_signin_redirect` on success

Minimal example:

```python
from casp.auth import GoogleProvider, GithubProvider, auth
from casp.layout import render_page


google = GoogleProvider()
github = GithubProvider()


def page():
    response = auth.auth_providers(google, github)
    if response:
        return response

    return render_page(__file__)
```

Use this only when the route is actually handling an auth-provider signin or callback flow.

## CSRF Token Helper

The installed auth runtime also exposes `get_csrf_token()`.

Behavior:

- `auth.sign_in(...)` seeds `csrf_token` into the session.
- `get_csrf_token()` reads the existing token or creates one when a request session exists.
- If no request or session is available, it returns an empty string.

Use this helper when custom form or fetch flows need access to the session CSRF token.

## Backwards Compatibility Alias

The installed auth file still includes `AuthConfig` as a compatibility alias.

It exposes:

- property proxies for `PUBLIC_ROUTES`, `PRIVATE_ROUTES`, `AUTH_ROUTES`, `IS_ALL_ROUTES_PRIVATE`, `DEFAULT_SIGNIN_REDIRECT`, and `DEFAULT_SIGNOUT_REDIRECT`
- `AuthConfig.check_auth_role(...)` as a proxy to `auth.check_role(...)`

Prefer `AuthSettings`, `configure_auth(...)`, and `auth.settings` in new code.

## Current Implementation Notes

- The installed auth runtime is session-backed. It stores the auth payload and CSRF token in `request.session`.
- Expiration uses timestamps and the current duration parser only accepts `s`, `m`, `h`, and `d` units.
- Expired or malformed payloads are removed during `auth.is_authenticated()` checks.
- `auth.get_payload()` returns `None` for missing payloads, a dict for dict payloads, and `{"value": ...}` for non-dict payloads.
- OAuth callback helpers return `None` on failure, so calling routes should be prepared for a normal page render or explicit error state when provider login fails.
- The current app bootstrap adds `SessionMiddleware`, `CSRFMiddleware`, `AuthMiddleware`, and `RPCMiddleware`; auth-aware code depends on that ordering so sessions and CSRF state exist before route checks or RPC handling.
- Password reset, email verification, and other account-recovery workflows are application responsibilities layered on top of `casp.auth`, not built-in auth runtime features.
- Route lists and RBAC maps are exact-path checks, not wildcard or prefix rules.

## Recommended Usage Pattern

Validate and authenticate close to the input boundary.

Common placement patterns are:

- put centralized policy in `src/lib/auth/auth_config.py`
- call `configure_auth(build_auth_settings())` during startup
- keep sign-in, signup, signout, and reset-password handlers in the owning route backend under `src/app/**/index.py`
- use `@rpc()` for browser-triggered auth actions and page decorators for full-route protection
- use `Validate` and `Rule` from `casp.validate` before password checks, user lookup, persistence, or external provider flows
- keep reusable auth helpers in `src/lib/auth/`

For browser-triggered auth forms, pair this page with `fetch-data.md`. For credential and form validation, pair it with `validation.md`.

## AI Routing Notes

If an AI agent is deciding how to handle auth in Caspian, apply these rules first.

- Treat `casp.auth` as the default authentication layer.
- Centralize route visibility, redirects, and RBAC policy in `src/lib/auth/auth_config.py`.
- Apply settings at startup in `main.py` with `configure_auth(build_auth_settings())`.
- Register provider instances in `main.py` with `Auth.set_providers(...)` when Google or GitHub OAuth is enabled.
- Use `auth.sign_in(...)` and `auth.sign_out(...)` for session lifecycle changes.
- Use `@require_auth`, `@require_role`, and `@guest_only` for page-level access rules.
- Use `@rpc(require_auth=True, allowed_roles=[...])` for protected browser-triggered actions.
- Keep `SessionMiddleware` outermost so auth, CSRF, and RPC handlers can read `request.session`.
- Align `SESSION_LIFETIME_HOURS` with `default_token_validity` unless you intentionally want different cookie and auth-payload expiry windows.
- Prefer exact route strings in `public_routes`, `auth_routes`, `private_routes`, and `role_based_routes`.
- Keep auth secrets and OAuth provider credentials in `.env`.
- Set `AUTH_COOKIE_NAME` explicitly so the session middleware cookie name and auth settings stay aligned.
- Use `.venv/Lib/site-packages/casp/auth.py` only when the task is about framework auth internals or debugging installed behavior.
- Use `main.py` when the task is about auth bootstrap, session middleware, or middleware execution order.
- Pair auth work with `fetch-data.md` for RPC forms and `validation.md` for credential validation.
- Check `routing.md` and `project-structure.md` before creating new auth routes or shared auth helpers.

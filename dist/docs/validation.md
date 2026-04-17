---
title: Validation
description: Validate and sanitize Caspian inputs with `casp.validate`, `Validate`, `Rule`, direct validators, rule-based checks, and file or date or ID validation before route or RPC logic persists data.
related:
  title: Related docs
  description: Use the fetch-data guide when validation runs inside RPC actions, then use the structure guide to place reusable validators in the right layer.
  links:
    - /docs/fetch-data
    - /docs/routing
    - /docs/project-structure
    - /docs/index
---

This page explains the current installed Caspian validation API for direct field checks, rule-based validation, sanitization, and reusable input guards.

## Overview

Caspian exposes validation through `casp.validate`.

Import the public API like this:

```python
from casp.validate import Validate, Rule
```

The current installed implementation lives in `.venv/Lib/site-packages/casp/validate.py`.

The real API surface is:

- direct single-value validation with `Validate.*(...)`
- multi-rule validation with `Validate.with_rules(...)`
- rule-string helpers built with `Rule`

Use validation at the boundary where untrusted data enters the app, especially in form handling, auth flows, and RPC mutations.

## Framework Internals Note

- The validation implementation lives in `.venv/Lib/site-packages/casp/validate.py`.
- Treat that file as framework code. Read it when the task is about validation internals, debugging framework behavior, or documenting the installed Caspian implementation.
- If upstream docs and the installed file disagree, prefer the installed file for local project guidance.

## Direct Validation

Use direct validators when you only need to check one value. These helpers usually return a sanitized or coerced value on success and `None` on failure.

The current `Validate` class exposes these main groups of helpers:

| Category | Methods | Notes |
| --- | --- | --- |
| Strings and identifiers | `string`, `email`, `url`, `ip`, `uuid`, `ulid`, `cuid`, `cuid2`, `nanoid`, `bytes`, `xml` | `string()` trims input and escapes HTML by default. |
| Numbers | `int`, `big_int`, `float`, `decimal` | `decimal()` returns a quantized `Decimal`, with `scale=30` by default. |
| Dates and times | `date`, `date_time` | Uses PHP-style format tokens such as `Y-m-d` and `Y-m-d H:i:s`. |
| Booleans | `boolean` | Accepts `1`, `true`, `on`, `yes`, `y` and the corresponding false forms. |
| Structured values | `json`, `is_json`, `enum`, `enum_class` | `json()` serializes non-strings and validates JSON strings. |
| Text utility | `emojis` | Replaces supported text tokens such as `:wave:` and `<3`. |

Example:

```python
from casp.validate import Validate

email = Validate.email("  User@Example.com ")
# Result: "User@Example.com"

identifier = Validate.cuid2("tz4a98xxat96iws9zmbrgj3a")
# Result: "tz4a98xxat96iws9zmbrgj3a"

invalid = Validate.url("not-a-url")
# Result: None

safe_name = Validate.string(" <Admin> ")
# Result: "&lt;Admin&gt;"

enabled = Validate.boolean("yes")
# Result: True
```

Implementation notes from the installed file:

- `Validate.string(value, escape_html=True)` trims and HTML-escapes by default.
- `Validate.date(...)` and `Validate.date_time(...)` use PHP-style format tokens mapped internally to Python datetime formats.
- `Validate.decimal(...)` returns a `Decimal` rounded with `ROUND_HALF_UP`.
- `Validate.cuid(...)` accepts either installed-library-backed legacy CUID or CUID2-like values depending on available packages.
- `Validate.is_json(...)` is the strict boolean check. `Validate.json(...)` returns serialized JSON for non-strings and parser text for invalid JSON strings.

Use this style for:

- one-off field checks
- simple preconditions before a query or mutation
- cases where `None` is enough to branch into an error response

## Rules Engine With `with_rules`

Use `Validate.with_rules(...)` when a field needs multiple constraints.

The current function signature is conceptually:

```python
Validate.with_rules(value, rules, confirmation_value=None)
```

It accepts two rule formats:

- a pipe-separated string such as `required|min:3|max:20|regex:^[a-zA-Z0-9_]+$`
- a list of rule strings, often created with `Rule`

Example:

```python
from casp.validate import Validate, Rule

def register_user(data):
    username = data.get("username")
    password = data.get("password")
    password_confirmation = data.get("password_confirmation")

    check = Validate.with_rules(
        username,
        "required|min:3|max:20|regex:^[a-zA-Z0-9_]+$",
    )

    check_safe = Validate.with_rules(username, [
        Rule.REQUIRED,
        Rule.min(3),
        Rule.max(20),
        Rule.regex(r"^[a-zA-Z0-9_]+$"),
        Rule.not_in_list(["admin", "root"]),
    ])

    password_check = Validate.with_rules(password, [
        Rule.REQUIRED,
        Rule.min(8),
        Rule.confirmed(),
    ], confirmation_value=password_confirmation)

    if check_safe is not True:
        return {"error": check_safe}

    if password_check is not True:
        return {"error": password_check}

    return {"success": True}
```

The current implementation applies rules in order and stops on the first failure.

Contract:

- `True` means the value passed all rules
- a non-`True` result is the first validation error message

Prefer the `Rule` form when the rules will live in maintained project code because it reduces string typos and gives better autocomplete, even though `Rule` ultimately returns rule strings.

## Implemented Rule Set

The installed `apply_rule(...)` implementation currently supports these rule names:

- Presence and length: `required`, `min`, `max`, `size`, `between`
- Prefix and suffix: `startsWith`, `endsWith`
- Confirmation: `confirmed`
- Format and identity: `email`, `url`, `ip`, `uuid`, `ulid`, `cuid`, `cuid2`, `nanoid`, `json`, `regex`
- Numeric and boolean: `int`, `float`, `boolean`, `digits`, `digitsBetween`
- Membership: `in`, `notIn`
- Dates: `date`, `dateFormat`, `before`, `after`
- Files: `file`, `extensions`, `mimes`

The `Rule` helper currently provides:

- Constants: `REQUIRED`, `EMAIL`, `URL`, `IP`, `UUID`, `ULID`, `CUID`, `CUID2`, `NANOID`, `INTEGER`, `FLOAT`, `BOOLEAN`, `JSON`, `FILE`
- Methods: `min`, `max`, `size`, `starts_with`, `ends_with`, `confirmed`, `in_list`, `not_in_list`, `between`, `digits`, `digits_between`, `date`, `date_format`, `before`, `after`, `regex`, `extensions`, `mimes`

Examples:

| Rule | Helper | Purpose |
| --- | --- | --- |
| `required` | `Rule.REQUIRED` | Fails for `None`, empty strings, and empty collections. |
| `email` | `Rule.EMAIL` | Validates email format. |
| `min` / `max` | `Rule.min(5)` / `Rule.max(20)` | Enforces string length constraints. |
| `confirmed` | `Rule.confirmed()` | Compares the value to `confirmation_value`. |
| `in` / `notIn` | `Rule.in_list([...])` / `Rule.not_in_list([...])` | Restricts or blocks specific values. |
| `regex` | `Rule.regex(r"^[a-z]+$")` | Matches a custom regular expression. |
| `extensions` / `mimes` | `Rule.extensions([...])` / `Rule.mimes([...])` | Validates upload extensions or MIME types. |

## File Validation

The installed file includes upload-oriented helpers and rules.

- `file` passes when the value is a filesystem path, a readable file-like object, or an object with `read(...)`.
- `extensions` validates file suffixes from a path or `filename` attribute.
- `mimes` uses `python-magic` when installed for MIME sniffing, then falls back to `mimetypes.guess_type(...)`.

Use these rules for RPC uploads and form submissions that accept files.

## Sanitization

Validation in Caspian is more than rule checking. The installed implementation also sanitizes or normalizes several values.

Documented behaviors include:

- trimming and cleaning string inputs during direct validation
- sanitization helpers for safer string handling
- normalization for formats such as dates and booleans

Concrete examples from the current file:

- `Validate.string(...)` escapes HTML unless `escape_html=False`
- `Validate.boolean(...)` normalizes common truthy and falsy text values
- `Validate.date_time(...)` accepts multiple parse paths before formatting the final string

This means validation should happen before application code persists, compares, or authorizes incoming values.

## Optional Dependency Behavior

The installed implementation uses optional packages when they are available:

- `email-validator` for stronger email validation
- `python-dateutil` for more flexible datetime parsing
- `python-magic` for content-based MIME detection
- `cuid2` and `cuid` packages to recognize specific ID formats

If those packages are missing, the module falls back to simpler behavior.

## Current Implementation Notes

- Unknown rule names currently fall through and pass because `apply_rule(...)` returns `True` when a rule is not recognized.
- The upstream docs show `alpha_num`, but the current installed `apply_rule(...)` implementation does not implement an `alpha_num` rule.
- Use `regex` for alphanumeric username rules until the framework adds a dedicated `alpha_num` rule.
- `Rule` is an IntelliSense and string-builder helper, not an enum-backed validator by itself.

## Recommended Usage Pattern

Validate close to the input boundary.

Common placement patterns are:

- inside `@rpc()` actions before database writes
- inside route-level backend files such as `src/app/**/index.py`
- in shared helpers under `src/lib/` when multiple routes or actions reuse the same rule sets

For browser-triggered writes and form submissions, pair this page with [fetch-data.md](./fetch-data.md).

Example RPC action:

```python
from casp.rpc import rpc
from casp.validate import Validate, Rule

@rpc()
def create_account(data: dict):
    email = Validate.email(data.get("email"))
    password = data.get("password")

    password_check = Validate.with_rules(password, [
        Rule.REQUIRED,
        Rule.min(8),
    ])

    if email is None:
        return {"error": "Invalid email address."}

    if password_check is not True:
        return {"error": password_check}

    return {"success": True}
```

## AI Routing Notes

If an AI agent is deciding how to validate input in Caspian, apply these rules first.

- Use direct `Validate.*(...)` calls for quick single-field checks.
- Use `Validate.with_rules(...)` for multi-constraint validation.
- Prefer `Rule` helpers in maintained project code instead of long rule strings.
- Validate request, form, and RPC payloads before database writes or auth-sensitive logic.
- Put reusable validation workflows in `src/lib/` when multiple routes need them.
- Keep route-specific validation next to the route or action that owns the input boundary.
- Use `.venv/Lib/site-packages/casp/validate.py` only when the task is about Caspian core validation internals or framework debugging.
- Prefer the implemented local rule set over upstream examples when they differ.
- Pair validation with [fetch-data.md](./fetch-data.md) when user input arrives through `pp.rpc()`.
- Check [project-structure.md](./project-structure.md) before deciding whether a validator belongs in `src/app/` or `src/lib/`.

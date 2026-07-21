---
title: Testing And Quality Gate
description: Use this page when the task mentions tests, pytest, type checking, pyright, linting, ruff, auto-fixing lint (ruff --fix / check:fix), unused-import (F401) removal, a quality gate, CI checks, or "make the code production-ready" for a Caspian app's own Python. Explains the recommended one-command gate over `main.py` and `src/**`, and why ruff must not auto-delete component imports used as `<x-*>` tags.
related:
  title: Related docs
  description: Pair the quality gate with the runtime map when a failing check points into core files, and with the structure and command docs when deciding where tests and tooling belong.
  links:
    - /docs/core-runtime-map
    - /docs/project-structure
    - /docs/commands
    - /docs/auth
    - /docs/index
---

This page describes the recommended way to add tests, type checking, and linting to a Caspian application's **own** Python (`main.py` and `src/**`).

Caspian does not ship a test runner, type checker, or linter. Quality tooling is an app-owned convention layered on top of the framework, so treat everything here as a recommended setup to scaffold per project, not as a built-in feature that already exists in every Caspian app. The framework runtime under `.venv/Lib/site-packages/casp/**` is out of scope for app tests.

## When Does This Doc Apply

- The task is to add or extend tests for app code, add type checking, add linting, wire a CI/pre-commit check, or prepare an app for production.
- A prior change touched `main.py`, `src/lib/**`, or route `index.py` files and needs verification.
- Not gated by any `caspian.config.json` flag. It applies to any Caspian app; it does not depend on `prisma`, `mcp`, `websocket`, or `typescript`.

## Recommended Shape: One Command

Expose a single gate command so an agent or CI has exactly one thing to run. The recommended command runs three tools in one pass and reports every problem with its exact location:

- **type check** — [pyright](https://github.com/microsoft/pyright) over `main.py` and `src/**` (Pylance, the default VS Code Python language server, is Pyright under the hood and reads the same `[tool.pyright]` config, so the editor and `npm run check` report the same thing)
- **lint** — [ruff](https://docs.astral.sh/ruff/)
- **tests** — [pytest](https://docs.pytest.org)

The command should print each problem as `path:line:col [tool:code] message` and exit non-zero when any check fails, so the file and line to fix are always explicit. Prefer a single `npm run check` script (backed by an app-owned orchestrator such as `settings/check.py`) over separate `test` / `lint` / `typecheck` scripts, so the surface stays minimal. Keep a per-tool escape hatch (for example `--only pyright`) for debugging rather than as additional npm scripts.

## Source Of Truth

- The gate command and its tool list are app-owned. Confirm the actual script name in the project's `package.json` and the orchestrator file (commonly `settings/check.py`) before assuming a command exists.
- Tooling versions and configuration are app-owned in `pyproject.toml`. Do not assume a tool is installed just because this doc mentions it; check the project's dependency group and `uv.lock`.
- App tests live in a top-level `tests/` directory. Framework internals under `.venv/Lib/site-packages/casp/**` are never the test target.

## Recommended Layout

```
tests/
  conftest.py            # put project root on sys.path; set safe dev env defaults
  test_*.py              # app-level unit + integration tests
settings/check.py        # orchestrator: runs the tools, prints path:line:col report
```

- `conftest.py` should add the project root to `sys.path` and set safe development defaults (for example `APP_ENV=development` and a throwaway `AUTH_SECRET`) so importing `main` never fails during collection.
- Test `main.py` through its pure helpers and through `starlette.testclient.TestClient` against `main.app` (for example the always-on `/health` route, which exercises the middleware stack). Test `src/lib/**` policy such as `auth_config.py` directly.

## Recommended `pyproject.toml` Configuration

Keep the dev tooling in a dependency group and install it with `uv sync --group dev`:

```toml
[dependency-groups]
dev = [
    "pyright>=1.1",
    "ruff>=0.6",
    "pytest>=8.0",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"

[tool.ruff]
# `settings/*.py` (not just `settings/check.py`) so every orchestrator script —
# `check.py`, `fix.py`, `_component_imports.py` — is linted, not just the entrypoint.
include = ["main.py", "src/**/*.py", "tests/**/*.py", "settings/*.py"]
extend-exclude = [".venv", "node_modules"]

[tool.ruff.lint]
# Correctness-focused; leave cosmetic rules off the generated starter code.
select = ["E", "F", "W"]
ignore = ["E501"]

[tool.pyright]
# Pylance (the VS Code Python extension) reads this same config, so the editor
# and `npm run check` report the same thing instead of disagreeing.
# `settings/*.py` is included so the orchestrator scripts are type-checked too.
include = ["main.py", "src", "settings/*.py"]
exclude = [".venv", "node_modules", "**/__pycache__"]
# `basic` is Pylance's default mode. It catches reportArgumentType-style bugs
# (e.g. passing a dict[str, str | None] to a TypedDict whose field is `str`)
# without flooding the gate with strict-mode noise. Note this `include`/`exclude`
# pair analyzes everything under `src`, including the generated Prisma Python ORM
# in `src/lib/prisma/**`; `basic` mode (plus the optional suppressions below)
# keeps its Optional-everywhere models from dominating the gate. If that ORM still
# produces too much noise, add `"src/lib/prisma"` to `exclude` explicitly.
pythonPlatform = "Windows"   # or Linux/Darwin to match the deploy target
typeCheckingMode = "basic"
# Mirror suppressions from older pyrefly-based setups if you migrated; otherwise
# leave these out. Re-enable per-rule when tightening.
# reportReturnType = "none"
# reportAssignmentType = "none"
```

## Auto-Fixing, And Why Ruff Must Not Delete Component Imports

The gate only **reports**. Ruff can also **fix** many lint findings in place, so a second command is worth exposing alongside the gate — recommended `npm run check:fix`, backed by a small orchestrator (for example `settings/fix.py`) that applies safe fixes and then re-runs the gate to print the authoritative report of what is left. Type errors (pyright) and failing tests (pytest) are never auto-fixed.

**Hazard — unused-import autofix (`F401`) breaks single-file components.** A single-file component imports its children and then uses them **only** as `<x-*>` tags inside an `html(...)` / `render_html(...)` template string, for example `from .Dialog import DialogContent` used as `<x-dialog-content>`. Ruff parses Python, not the template string, so it reports the import as unused (`F401`) and `--fix` would delete it. That import is load-bearing: Caspian resolves the tag from the **module's globals at render time** (see `component_decorator._attach_caller_scope`, which scans the caller module for `Component` instances). Deleting it breaks rendering at runtime, silently. This affects any component file, so it cannot be scoped by path — but it also must not disable dead-import cleanup for ordinary Python.

Handle it in three layers, all generic:

1. **Never let a raw `ruff check --fix` delete an import.** Mark `F401` unfixable so ruff reports but never strips it, project-wide — this protects anyone who runs ruff directly:

   ```toml
   [tool.ruff.lint]
   select = ["E", "F", "W"]
   ignore = ["E501"]
   unfixable = ["F401"]
   ```

2. **Let the fixer still clean genuinely dead imports.** The `check:fix` orchestrator lists `F401` findings under the project config, marks any file that contains an import used as an `<x-*>` tag as *component-guarded*, and removes dead imports only from the **non-guarded** files. Because the project config makes `F401` unfixable, re-enable removal for that step with an **isolated** ruff run scoped to those exact files: `ruff check <files> --select F401 --fix-only --isolated` (`--isolated` ignores the project config so `F401` is fixable again; passing explicit files keeps include/exclude moot). Guarded files are skipped whole and left for the gate to report, so a load-bearing import is never at risk.

3. **Keep the gate honest.** In the orchestrator (`settings/check.py`), drop the `F401` findings whose bound symbol is actually used as an `<x-*>` tag in the same file, and keep the rest — so the gate still fails on genuinely dead imports. Derive the tag exactly as the compiler does: `x-{camel_to_kebab(import_name)}` (mirror `casp.string_helpers.camel_to_kebab`; e.g. `DialogContent` → `<x-dialog-content`). Pull the symbol from the ruff JSON message (`` `.Dialog.DialogContent` imported but unused`` → last dotted segment, or the alias after ` as `), then match `<x-dialog-content` on a tag boundary in the file source. Share this detection between the fixer and the gate in one module (for example `settings/_component_imports.py`).

**Working rule:** when the gate reports an `F401` in a component-guarded file, remove that import by hand only after confirming it is not used as an `<x-*>` tag anywhere in the same file. Do not blanket-ignore `F401` or re-enable its autofix globally — it cannot see template usage.

## Things To Verify Before Editing Or Explaining

- Confirm the gate command name and orchestrator path in `package.json` and `settings/`, since they are app-owned and may differ per project.
- Confirm the dev tools are actually installed (`[dependency-groups]` in `pyproject.toml`, resolved in `uv.lock`) before telling a user to run the gate.
- Check `[tool.pyright]` in `pyproject.toml`. A project may set `typeCheckingMode = "basic"` (Pylance's default, recommended) and individually silence rules with `reportReturnType = "none"` / `reportAssignmentType = "none"` (commonly done to mirror older pyrefly-based setups). Any rule silenced there is then not reported by the gate, so do not assume every annotation mismatch is caught. Pylance in the editor reads this same config, so the IDE and `npm run check` agree.
- Keep the scope on app code. If a check points into `.venv/Lib/site-packages/casp/**`, use [core-runtime-map.md](./core-runtime-map.md) to understand the runtime, but do not add framework files to the app's test, lint, or type-check scope.

## Working Rule For Agents

When you add or change app-owned Python (`main.py`, `src/**`), add or extend the matching test under `tests/`, write annotated and type-checkable code, and run the single gate command until it passes before treating the change as done. Fix problems at the exact `path:line:col` the gate reports.

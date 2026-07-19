---
title: Testing And Quality Gate
description: Use this page when the task mentions tests, pytest, type checking, pyrefly, linting, ruff, a quality gate, CI checks, or "make the code production-ready" for a Caspian app's own Python. Explains the recommended one-command gate over `main.py` and `src/**`.
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

- **type check** — [pyrefly](https://pyrefly.org) over `main.py` and `src/**`
- **lint** — [ruff](https://docs.astral.sh/ruff/)
- **tests** — [pytest](https://docs.pytest.org)

The command should print each problem as `path:line:col [tool:code] message` and exit non-zero when any check fails, so the file and line to fix are always explicit. Prefer a single `npm run check` script (backed by an app-owned orchestrator such as `settings/check.py`) over separate `test` / `lint` / `typecheck` scripts, so the surface stays minimal. Keep a per-tool escape hatch (for example `--only pyrefly`) for debugging rather than as additional npm scripts.

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
    "pyrefly>=0.16",
    "ruff>=0.6",
    "pytest>=8.0",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"

[tool.ruff]
include = ["main.py", "src/**/*.py", "tests/**/*.py", "settings/check.py"]
extend-exclude = [".venv", "node_modules"]

[tool.ruff.lint]
# Correctness-focused; leave cosmetic rules off the generated starter code.
select = ["E", "F", "W"]
ignore = ["E501"]

[tool.pyrefly]
project-includes = ["main.py", "src/**"]
search-path = [".", "src"]
```

## Things To Verify Before Editing Or Explaining

- Confirm the gate command name and orchestrator path in `package.json` and `settings/`, since they are app-owned and may differ per project.
- Confirm the dev tools are actually installed (`[dependency-groups]` in `pyproject.toml`, resolved in `uv.lock`) before telling a user to run the gate.
- Check `[tool.pyrefly.errors]` in `pyproject.toml`. A project may suppress specific error kinds (commonly `bad-return` and `bad-assignment`); those are then not reported by the gate, so do not assume every annotation mismatch is caught.
- Keep the scope on app code. If a check points into `.venv/Lib/site-packages/casp/**`, use [core-runtime-map.md](./core-runtime-map.md) to understand the runtime, but do not add framework files to the app's test, lint, or type-check scope.

## Working Rule For Agents

When you add or change app-owned Python (`main.py`, `src/**`), add or extend the matching test under `tests/`, write annotated and type-checkable code, and run the single gate command until it passes before treating the change as done. Fix problems at the exact `path:line:col` the gate reports.

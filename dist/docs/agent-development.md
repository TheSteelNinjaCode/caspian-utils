---
title: Agent Development Workflow
description: Use this page when an AI agent edits a Caspian app while its development stack is already running. Explains the coordinated dev hold, the one-restart/one-browser-reload contract, Claude Code, GitHub Copilot, and Codex hook integration, manual fallback commands, and frontend verification order.
related:
  title: Related docs
  description: Pair the editing workflow with the app-owned quality gate and command reference, then use the feature guide for the code being changed.
  links:
    - /docs/testing
    - /docs/commands
    - /docs/ai-validation-checklist
    - /docs/index
---

# Agent Development Workflow

Caspian projects may include an app-owned development coordinator that turns an AI agent's whole editing run into **one Python restart and one browser reload**. Without it, the normal watcher settles after a short quiet period; tool round-trips are longer than that period, so a multi-file agent change can restart Python and reload every open browser tab once per edit.

This is scaffolded development tooling, not a `caspian.config.json` feature and not a guarantee for every Caspian project. Before relying on it, confirm the project contains:

- `settings/dev-hold.ts` and `settings/dev-hold-hook.ts`
- hold-aware change coordination in `settings/bs-config.ts`
- `dev:hold`, `dev:resume`, and `dev:hold:status` scripts in `package.json`
- at least one supported agent hook configuration, or use the manual fallback

## The one-refresh contract

Treat one editing run as one transaction:

1. Read the current frontend report before editing when the task changes UI or fixes a browser error.
2. Make all related file edits. The agent hook acquires or refreshes the hold before write-capable tools, while the dev stack queues file events.
3. Run read-only checks while the hold remains active. Do not reload the browser yet; it is still serving the pre-edit code.
4. When the edit phase is complete, run `npm run dev:resume` **once**. This releases the hold and lets the queued changes settle as one batch, producing one Python restart and one browser reload.
5. After the reload, exercise the affected route and interactions, then inspect the frontend report again.

Do not run `dev:resume` after each file, and do not alternate edit → reload → edit → reload. If verification reveals another required edit, that begins a new editing batch; finish that batch before releasing again.

The automatic `Stop` hook is a safety net for the end of a turn. It is too late for browser verification performed inside the same turn, so the agent must explicitly run `npm run dev:resume` after its final edit and before it tests the changed UI.

## Supported agent hosts

The standard scaffold points the same dependency-free hook bridge at three hosts:

| Host | Configuration | Acquire or refresh | Automatic release |
| --- | --- | --- | --- |
| Claude Code | `.claude/settings.json` | `PreToolUse` | `Stop`, `SessionEnd` |
| GitHub Copilot CLI and VS Code | `.github/hooks/dev-hold.json` | `PreToolUse` | `Stop`, `SessionEnd` |
| Codex CLI and compatible Codex clients | `.codex/hooks.json` | `PreToolUse` | `Stop` |

The hook normalizes host-specific write-tool names such as `Edit`, `apply_patch`, and `insert_edit_into_file`. Shell tools are inspected separately: commands that can write acquire the hold, while read-only commands and the hold/log controls do not. This prevents `npm run logs` or the quality gate from creating a new hold and making a current frontend report look stale.

Do not release on a subagent-stop event. A subagent finishing does not imply that the primary agent has completed the shared editing run.

## Manual controls and fallback

```bash
npm run dev:hold         # acquire or refresh before a manual editing run
npm run dev:hold:status  # inspect the current hold
npm run dev:resume       # release and apply all queued changes once
```

Use the manual acquire/release pair when the current agent host does not support repository hooks. `dev:resume` is safe to call when no hold exists. If it reports `No hold was active` after write tools were used, the current host's hook integration is absent or not firing; check the applicable configuration file and use the manual path for the next editing batch.

Never start a second `npm run dev` merely to force a refresh or see the existing stack's output. A second stack may clear development state, select different ports, and move the browser away from the session being verified. Use the project's existing BrowserSync URL and frontend reporting command instead.

## Failure behavior

The hold lives under `.casp/`, which a fresh development start recreates, so a new stack does not inherit an old hold. The standard coordinator also fails open:

- a hold with no refresh for 120 seconds becomes stale
- a continuously refreshed hold stops deferring after the 10-minute absolute cap
- a missing, corrupt, or structurally invalid hold is treated as inactive
- hook parsing or execution errors must exit successfully rather than blocking the agent's tool call

These valves favor an extra reload over a frozen development stack. They are recovery behavior, not the normal completion path; a healthy agent still releases explicitly before frontend verification.

## Frontend reports while held

If the project's frontend digest prints `DEV HOLD ACTIVE`, its route results describe the code from before the queued edits. Release the hold, allow the queued batch to settle, reload or confirm the automatic reload of the affected route, repeat the relevant interaction, and only then read the digest as current evidence.

For the full distinction between mount errors, interaction errors, historical entries, and routes that were never exercised, follow [Frontend verification for agents](./testing.md#frontend-verification-for-agents).

## Implementation ownership

When maintaining this workflow, verify behavior in the project rather than copying assumptions from this guide:

- `settings/dev-hold.ts` owns the hold file, lifecycle, stale timeout, absolute cap, and CLI commands.
- `settings/dev-hold-hook.ts` normalizes tool names, detects write-capable shell commands, and must fail open.
- `settings/bs-config.ts` and its batch worker own queued changes, the single restart, and the single browser reload.
- `.claude/settings.json`, `.github/hooks/dev-hold.json`, and `.codex/hooks.json` connect supported hosts.
- The project's Node tests should cover lifecycle, expiry, hook configuration, write-tool detection, and one-batch draining.

Keep the canonical workflow here. Other Caspian docs should summarize the one-refresh rule and link back instead of duplicating the implementation details.

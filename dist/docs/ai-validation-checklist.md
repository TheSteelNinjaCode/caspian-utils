---
title: AI Validation Checklist
description: Use this page when testing whether AI can find the right Caspian docs, core files, and verification checkpoints for a task after documentation or runtime changes.
related:
  title: Related docs
  description: Start with the manifest, use the runtime map when needed, then validate against the matching feature guide and owning runtime behavior.
  links:
    - /docs/index
    - /docs/core-runtime-map
    - /docs/routing
    - /docs/auth
    - /docs/fetch-data
    - /docs/pulsepoint
---

This page gives AI a small validation workflow for checking whether the packaged Caspian docs lead to the correct source files and runtime behaviors.

Use it after changing packaged docs, after changing `main.py` or installed `casp` internals, or when you want to verify that AI can resolve a task without drifting across unrelated files.

## Goal

The goal is not to restate feature docs. The goal is to verify four things quickly:

- AI starts from the correct manifest and feature gate.
- AI reaches the correct packaged feature doc.
- AI reaches the controlling app or runtime file.
- AI reaches a concrete behavior checkpoint before making assumptions about behavior.

## Fast Workflow

1. Read [index.md](./index.md) to discover the local Caspian docs.
2. Read `./caspian.config.json` before assuming any optional feature is enabled.
3. If the task touches `main.py` or `.venv/Lib/site-packages/casp/**`, read [core-runtime-map.md](./core-runtime-map.md).
4. Read the matching feature doc such as [routing.md](./routing.md), [auth.md](./auth.md), [fetch-data.md](./fetch-data.md), [pulsepoint.md](./pulsepoint.md), or [file-uploads.md](./file-uploads.md).
5. Verify the claim against the actual project code and the installed runtime.
6. Confirm the relevant behavior in the owning files surfaced by [core-runtime-map.md](./core-runtime-map.md) before treating the behavior as settled.

## Pass Criteria

A documentation path is in good shape when AI can do all of these without broad repo wandering:

- identify whether the feature is enabled from `caspian.config.json`
- choose the correct packaged doc from [index.md](./index.md)
- jump to the controlling runtime file through [core-runtime-map.md](./core-runtime-map.md) when needed
- distinguish app-owned files from installed framework internals
- identify the concrete behavior that must be verified in the owning file
- avoid suggesting changes to `main.py` or `.venv/Lib/site-packages/casp/**` unless the task truly requires runtime edits

## Representative Prompt Suite

Use prompts like these to check whether AI lands on the correct docs and files.

| Prompt | First docs AI should read | Controlling files AI should reach | Verification focus |
| --- | --- | --- | --- |
| Create a protected dashboard section with child routes and a shared shell. | [index.md](./index.md), [routing.md](./routing.md), [auth.md](./auth.md), [project-structure.md](./project-structure.md) | `src/app/**`, `src/lib/auth/auth_config.py`, `main.py`, `.venv/Lib/site-packages/casp/layout.py` | section layout ownership, route privacy mode, and child-route wrapping |
| Make a grouped shell keep sidebar scroll while resetting page content on child-route navigation. | [index.md](./index.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md), [core-runtime-map.md](./core-runtime-map.md) | `src/app/**/layout.html`, `public/js/pp-reactive-v2.js`, `main.py` | `pp-reset-scroll` placement, push-vs-history scroll behavior, and shared-shell ownership |
| Create a new contact page with interactive form behavior. | [index.md](./index.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md), [components.md](./components.md) | `src/app/**/index.html`, `main.py`, `.venv/Lib/site-packages/casp/components_compiler.py`, `.venv/Lib/site-packages/casp/scripts_type.py` | single-root template shape, script-inside-root authoring, and authored-vs-runtime boundaries |
| Add a button, filter, or form interaction to a route template. | [index.md](./index.md), [routing.md](./routing.md), [pulsepoint.md](./pulsepoint.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) | `src/app/**/index.html`, `public/js/pp-reactive-v2.js`, `main.py` | PulsePoint `on*` event attributes, `pp.state`, directives, `Object.fromEntries(new FormData(...).entries())` for simple form submits, and avoiding id-driven `querySelector` or `addEventListener` wiring |
| Add a file manager page with upload progress and persisted metadata. | [index.md](./index.md), [file-uploads.md](./file-uploads.md), [fetch-data.md](./fetch-data.md), [database.md](./database.md) | `src/app/**/index.html`, `src/app/**/index.py`, `src/lib/**`, `prisma/schema.prisma` when Prisma applies | route-owned upload actions, persisted metadata flow, and Prisma-backed storage boundaries |
| Explain why authored Caspian templates use a plain `<script>` instead of `type="text/pp"`. | [index.md](./index.md), [pulsepoint.md](./pulsepoint.md), [components.md](./components.md), [routing.md](./routing.md) | `main.py`, `.venv/Lib/site-packages/casp/scripts_type.py`, `.venv/Lib/site-packages/casp/components_compiler.py` | script rewriting, `pp-component` injection, and authored-vs-runtime boundaries |
| Debug why `StateManager` does not persist across a full redirect. | [index.md](./index.md), [state.md](./state.md), [auth.md](./auth.md), [core-runtime-map.md](./core-runtime-map.md) | `main.py`, `.venv/Lib/site-packages/casp/state_manager.py` | wire vs non-wire reset behavior and `request.state.session` dependency |
| Trace the auth cookie and CSRF cookie names used in development. | [index.md](./index.md), [auth.md](./auth.md), [core-runtime-map.md](./core-runtime-map.md) | `main.py`, `.venv/Lib/site-packages/casp/auth.py` | development cookie scoping and CSRF or session naming ownership |
| Find how route params and query params reach `page()`. | [index.md](./index.md), [routing.md](./routing.md), [core-runtime-map.md](./core-runtime-map.md) | `main.py`, `.venv/Lib/site-packages/casp/layout.py` | path-dict delivery, query coercion, and `request` injection |
| Decide whether a dashboard child route should use page caching. | [index.md](./index.md), [cache.md](./cache.md), [routing.md](./routing.md), [fetch-data.md](./fetch-data.md) | `main.py`, `.venv/Lib/site-packages/casp/cache_handler.py`, the route's `index.py` | route-level cache ownership, cacheable HTML scope, and invalidation points |
| Add a PulsePoint context provider to a reusable component. | [index.md](./index.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md), [pulsepoint.md](./pulsepoint.md), [components.md](./components.md) | `ts/TemplateCompiler.ts`, `public/js/pp-reactive-v2.js`, component `.html`, `.venv/Lib/site-packages/casp/components_compiler.py` | lowercase HTML-first `*.provider` usage, logical ancestry, single-root template shape |
| Add upload progress to an existing file manager. | [index.md](./index.md), [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md), [file-uploads.md](./file-uploads.md), [fetch-data.md](./fetch-data.md) | `src/app/**/index.py`, `src/app/**/index.html`, `src/lib/**`, `public/js/pp-reactive-v2.js` | `pp.rpc` upload options, state replacement, persisted metadata, BrowserSync ignore rules |
| Decide whether MCP files should be created. | [index.md](./index.md), [mcp.md](./mcp.md), [commands.md](./commands.md), [project-structure.md](./project-structure.md) | `caspian.config.json`, `settings/restart-mcp.ts`, `package.json`, `src/lib/mcp/**` when enabled | `mcp` feature gate, update workflow, nested FastMCP config ownership |

Treat the table as a prompt pack for spot checks, not as a full validation matrix.

## Common Failure Modes

- AI skips `caspian.config.json` and assumes an optional feature is enabled because a packaged doc exists.
- AI reads only the packaged feature doc and never checks `main.py` or the installed runtime.
- AI edits framework internals when the task only requires app-owned route or helper changes.
- AI treats runtime HTML examples as authored template examples.
- AI generates a route or component template with a valid-looking root element but leaves a sibling top-level `<script>` or second top-level element after it.
- AI starts first-party interactivity with ids, `data-*` attributes, `querySelector`, `addEventListener`, manual `fetch`, manual `innerHTML`, or per-input `pp-ref` payload collection instead of PulsePoint `on*` attributes, form submit events, state, directives, and `pp.rpc()`.
- AI puts `pp-reset-scroll="true"` on the whole shell or `body` when only the page-content pane should reset, causing persistent sidebars or rails to lose their scroll position.
- AI decides behavior from memory without checking the owning implementation details.

## Decision Rule

- If the task is feature-oriented, start from [index.md](./index.md) and the matching feature doc.
- If the task is behavior-oriented and the owner is unclear, jump to [core-runtime-map.md](./core-runtime-map.md).
- If the task is still ambiguous after that, verify the controlling file and the specific behavior in scope before widening the search.

---
title: Components
description: Use this page when the task mentions `@component`, reusable UI, HTML-first `x-*` component tags, component imports, same-name `.html` templates, `merge_classes(...)`, `twMerge(...)`, or where shared components belong in a Caspian project.
related:
  title: Related docs
  description: Use the structure guide for file placement, the routing guide for route templates, the PulsePoint guide for browser-side scripts, and the data guide for component-owned RPC flows.
  links:
    - /docs/project-structure
    - /docs/core-runtime-map
    - /docs/routing
    - /docs/pulsepoint
    - /docs/pulsepoint-runtime-map
    - /docs/fetch-data
    - /docs/index
---

Components in Caspian are implemented as Python functions decorated with `@component`.

In authored HTML, import them with an `@import` comment and render them with HTML-first kebab-cased `x-*` tags such as `<x-my-component />` or `<x-my-component>...</x-my-component>`.

Treat that `x-*` form as the current Caspian component contract for authored templates.

Component tooling scans Python files under the paths listed in `caspian.config.json`. When `componentScanDirs` includes `src/`, `src/components/` is the clean default location for reusable UI, even though any scanned path can work.

As the app grows, treat `src/components/` as the default home for reusable application UI. Keep route-owned markup in `src/app/`, and keep non-UI helpers or services in `src/lib/`.

## Mental Model

- Use a Python component when you want a reusable server-rendered UI building block.
- Return an HTML string directly for small presentational components.
- Use `html(...)` to keep markup, server interpolation, and a PulsePoint `<script>` inline in one Python file (single-file component) for small and medium UI. It renders through Caspian's Jinja env, so `{{ ... }}` is server-side and `{ ... }` stays for PulsePoint.
- Use `render_html(...)` with a same-name `.html` file when the component has large markup, a long script, or you want clearer separation between Python logic and UI.
- When a component needs first-party interactivity, bind events in the component template with PulsePoint-handled `on*` attributes and keep state in `pp.state(...)`; do not build id-driven `querySelector(...)` or `addEventListener(...)` wiring for normal component behavior.
- Keep page-level workflows in `src/app/`, move reusable UI into `src/components/`, and keep helpers, services, validators, and adapters in `src/lib/`.

## Framework Internals Note

When the task is about component internals rather than normal app-owned components, these runtime files are the most relevant:

- `.venv/Lib/site-packages/casp/component_decorator.py` owns `@component`, `Component`, `render_html(...)`, the inline `html(...)` helper, and component loading.
- `.venv/Lib/site-packages/casp/components_compiler.py` owns `@import` parsing, `x-*` tag resolution (including resolving a component's own tags from its Python module imports), parent-scope expansion of slot content, root validation, and `pp-component` injection.
- `.venv/Lib/site-packages/casp/html_attrs.py` owns `get_attributes(...)` and the Python-side `merge_classes(...)` contract.
Use [core-runtime-map.md](./core-runtime-map.md) when you need the broader Python runtime-module map. Use [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md) when a component task is specifically about browser-side PulsePoint state, refs, context, portals, events, RPC, or SPA behavior.

## Basic Component

This is the simplest pattern: accept props, assemble the final class value, and return HTML.

```python
from casp.component_decorator import component
from casp.html_attrs import get_attributes, merge_classes

@component
def Container(children: str = "", **props) -> str:
    incoming_class = props.pop("class", "")
    final_class = merge_classes("mx-auto max-w-7xl px-4", incoming_class)

    attributes = get_attributes({
        "class": final_class,
    }, props)

    return f'<div {attributes}>{children}</div>'
```

Notes:

- `children` receives the inner content passed between opening and closing tags.
- `**props` lets the component accept additional HTML attributes such as `id`, `data-*`, and `aria-*`.
- The runtime normalizes `class`, `className`, and `class_name` into one `class` value before the component is called.
- Pass the `merge_classes(...)` result straight into `get_attributes(...)` or the rendered `class` attribute; do not wrap it in another helper.

## Tailwind Merge Contract

When `caspian.config.json` has `tailwindcss: true`, Caspian uses a frontend-first Tailwind merge contract.

- In Python components, use `merge_classes(...)` to assemble class defaults plus incoming class props.
- `merge_classes(...)` emits a frontend-ready `{twMerge(...)}` expression instead of doing Python-side Tailwind conflict resolution.
- In authored PulsePoint markup and scripts, use global `twMerge(...)` directly for attribute expressions and script-local derived values.

Python example:

```python
from casp.component_decorator import component
from casp.html_attrs import get_attributes, merge_classes

@component
def IconButton(**props) -> str:
    incoming_class = props.pop("class", "")
    final_class = merge_classes("size-4 rounded-md", incoming_class)

    attributes = get_attributes({
        "class": final_class,
    }, props)

    return f'<button {attributes}></button>'
```

Authored PulsePoint examples:

```html
<p class="{twMerge(baseClass, inputClass)}">Merged badge preview</p>

<script>
  const merged = twMerge(baseClass, inputClass);
</script>
```

## Import And Use Components

Place component imports at the top of the HTML template, above the authored root element.

```html
<!-- @import Container from "../components" -->

<x-container class="py-10">
  <h1>Dashboard</h1>
</x-container>
```

The import comment is the bridge between the Python component export and the `x-*` tag you author in HTML.

In section-based apps, follow the same mental model as the Next.js App Router: import shared shell components such as sidebars, topbars, breadcrumbs, and section frames in the parent folder's `layout.html`, then import page-specific components in each child route's `index.html`.

Treat `<!-- @import ... -->` as a file-level directive, not as rendered markup. It does not count as the template root, and it should not be nested inside the root wrapper element.

- `<!-- @import Container from "../components" -->` resolves `Container.py` from that folder.
- The exported component name maps to the tag you render by kebab-casing it and prefixing `x-`, such as `Container` for `<x-container />` or `CommandDialog` for `<x-command-dialog />`.
- Grouped imports and aliases are also supported, for example `<!-- @import { Button, Card as UserCard } from "../components/ui" -->`.
- If one Python file exports several `@component` functions, point `from` at that exact file instead of assuming one file per tag.
- In that case, prefer one grouped import so every rendered tag resolves back to the same Python module.

Good:

```html
<!-- @import { Button, Input, Label } from "../../../lib/maddex" -->

<section class="auth-panel auth-panel-compact fade-up">
  <x-label>Email</x-label>
  <x-input type="email" />
  <x-button>Continue</x-button>
</section>
```

Bad:

```html
<section class="auth-panel auth-panel-compact fade-up">
  <!-- @import { Button, Input, Label } from "../../../lib/maddex" -->
  <Label>Email</Label>
</section>
```

### Same-File Component Exports

Caspian does not require one Python file per component. A single file can export a main component plus related subcomponents.

If `Breadcrumb.py` defines `Breadcrumb`, `BreadcrumbList`, `BreadcrumbItem`, `BreadcrumbLink`, `BreadcrumbPage`, and `BreadcrumbSeparator`, import those names from `Breadcrumb.py` itself.

```html
<!-- @import { Breadcrumb, BreadcrumbItem, BreadcrumbLink, BreadcrumbList, BreadcrumbPage, BreadcrumbSeparator } from "../maddex/Breadcrumb.py" -->

<div class="dashboard-topbar">
  <x-breadcrumb class="dashboard-breadcrumbs" aria-label="Dashboard breadcrumbs">
    <x-breadcrumb-list class="dashboard-breadcrumbs__list">
      <x-breadcrumb-item class="dashboard-breadcrumbs__item">
        <x-breadcrumb-link href="/dashboard">Dashboard</x-breadcrumb-link>
      </x-breadcrumb-item>
      <x-breadcrumb-separator class="dashboard-breadcrumbs__separator" />
      <x-breadcrumb-item class="dashboard-breadcrumbs__item">
        <x-breadcrumb-page class="dashboard-breadcrumbs__page">Reports</x-breadcrumb-page>
      </x-breadcrumb-item>
    </x-breadcrumb-list>
  </x-breadcrumb>
</div>
```

If you only need one component from that file, keep the same file path:

```html
<!-- @import Breadcrumb from "../maddex/Breadcrumb.py" -->
```

Do not invent folder-level imports for symbols that do not have their own file. If `BreadcrumbItem` lives inside `Breadcrumb.py`, import it from `Breadcrumb.py`, not from `../maddex` as if `BreadcrumbItem.py` existed.

## Template-Backed Components

When a component has large markup or a long script, keep the Python file focused on props or server-side preparation and move the markup into a same-name HTML file. For small and medium components, including ones with PulsePoint behavior, prefer the single-file `html(...)` form described in the next section instead.

`Counter.py`

```python
from casp.component_decorator import component, render_html

@component
def Counter(label: str = "Clicks") -> str:
    return render_html(__file__, {
        "label": label,
    })
```

`Counter.html`

```html
<div>
  <h3>{{ label }}</h3>
  <button onclick="setCount(count + 1)">
    {count}
  </button>

  <script>
    const [count, setCount] = pp.state(0);
  </script>
</div>
```

Use this split when:

- Python is shaping props, loading data, or exposing server-side helpers.
- The template has enough markup that returning a raw f-string becomes noisy.
- PulsePoint state, effects, refs, or event handlers belong next to the component markup.

This keeps the component easy to read: Python owns the server-side logic and template context, while the HTML file owns the rendered UI and browser behavior.

## Single-File Components With `html(...)`

When a component is small or medium, keep the markup, server-side interpolation, and the PulsePoint `<script>` inline in the Python file instead of a sibling `.html`. Import `html` from `casp.component_decorator` and return `html("""...""", **context)`.

`html(...)` renders the string through the same Jinja environment Caspian uses for template files, so three brace dialects coexist without colliding:

- `{{ value }}` is server render (Python to HTML).
- `{{ value | json }}` safely serializes a server value into a `<script>`.
- `{# comment #}` is a Jinja comment, stripped from output.
- `{ value }` is left untouched for PulsePoint client reactivity.

```python
from casp.component_decorator import component, html
from casp.html_attrs import get_attributes, merge_classes

@component
def UserCard(user, **props):
    attrs = get_attributes({
        "class": merge_classes("card", props.pop("class", "")),
    }, props)

    # html
    return html("""
      <div {{ attrs }}>
        <h3>{{ user.name }}</h3>
        <button onclick="setLikes(likes + 1)">Likes: {likes}</button>
        <script>
          const [likes, setLikes] = pp.state({{ user.likes | json }});
        </script>
      </div>
    """, attrs=attrs, user=user)
```

The returned value is `Markup`, so the normal pipeline still injects `pp-component` on the single root and `transform_scripts(...)` rewrites the plain `<script>` to `type="text/pp"`. A single-file component renders identically to the two-file `render_html(...)` form; the choice is purely about readability.

Rules for inline `html(...)`:

- The single-root rule still applies: exactly one top-level element with any `<script>` nested inside it.
- Autoescaping is ON (Jinja default), so `{{ value }}` escapes HTML automatically and user text is safe without `| e`. The flip side: trusted HTML you want rendered as-is must be `Markup(...)` or piped through `| safe`.
- A `children` value is auto-marked safe (parity with `render_html(...)`), so `{{ children }}` renders nested component markup correctly without `| safe`.
- Do not use a Python f-string for the markup. PulsePoint single braces `{ ... }` would collide with f-string interpolation. Use a plain triple-quoted string passed to `html(...)`.
- Prefer `render_html(...)` with a same-name `.html` file when the markup is large or the `<script>` carries real logic. A long client script is a smell regardless of file shape; move heavy work into `@rpc()` actions or smaller composed components.

The leading `# html` comment above the string is an optional editor hint: some editors color the HTML inside a string tagged that way (JetBrains uses `# language=HTML`). It has no runtime effect.

## Component Imports: Python Imports Or `@import`

A component can render other components with `x-*` tags in either the two-file `.html` or the inline `html(...)` form. There are two ways to tell Caspian which component a tag refers to.

1. The `<!-- @import Name from "path" -->` HTML comment, resolved by path string. This works in both `.html` templates and inline strings.
2. A real Python import at the top of the component module. A component's own `x-*` tags resolve from the components imported into that module, with no `@import` comment needed.

Prefer the Python import inside single-file components. It is the unambiguous source of truth for which component a tag refers to, it gives normal editor navigation, and it prevents same-name collisions across directories. If `variantA/Tag.py` and `variantB/Tag.py` both export `Tag`, the Python import in each consumer file decides which `Tag` its `<x-tag>` resolves to.

```python
from casp.component_decorator import component, html
from src.components.variantA.Tag import Tag  # resolves <x-tag> below

@component
def Panel(children="", **props):
    # No @import comment. <x-tag> resolves from the Python import above,
    # so it is unambiguously variantA's Tag.
    # html
    return html("""
      <div class="panel">
        <x-tag>inside-panel</x-tag>
        {{ children }}
      </div>
    """, children=children)
```

Resolution precedence for an `x-*` tag inside a component's own output, lowest to highest:

- inherited components from an ancestor template
- the component's own Python module imports
- a local `<!-- @import ... -->` in that same template

So a component's Python import wins over an inherited same-name component, and an explicit local `@import` wins over both.

### Slot Content Resolves In The Parent Scope

Component tags written as slot content (children passed between a component's opening and closing tags) resolve in the scope where they were authored, not in the child component's scope. This matches slot semantics in other component systems: a `<x-tag>` written by the parent stays bound to the parent's `Tag` even when it is slotted into a child that has its own `Tag`. The component that authors a tag in markup must import that component, the same way it would for any tag it renders directly.

## Auto-Injected `pp-component` And PulsePoint Script Type

Treat `pp-component="componentName"` and `type="text/pp"` as framework output, not authored source. The canonical authored-vs-runtime explanation lives in [pulsepoint.md](./pulsepoint.md).

- Do not manually add `pp-component="..."` to route templates, layout templates, or component HTML files.
- Do not manually add `type="text/pp"` to authored PulsePoint scripts.
- The Python render pipeline injects `pp-component` onto the single root element during render.
- `main.py` runs `transform_scripts(...)`, which rewrites authored body `<script>` tags to `<script type="text/pp">` before the browser runtime mounts.
- Add a plain `<script>` only when the route or component actually needs PulsePoint logic, and keep it inside that single root element.
- If a runtime example already shows `pp-component` and `type="text/pp"`, treat it as rendered output rather than authored source.

## Single-Root Rule

Every component HTML template must render exactly one authored top-level parent node.

The same rule applies to route and layout templates such as `src/app/**/index.html` and `src/app/**/layout.html`. See [routing.md](./routing.md) and [pulsepoint.md](./pulsepoint.md) for the non-component versions of the same authoring contract.

Top-of-file `<!-- @import ... -->` directives are allowed before that root element and do not violate the single-root rule.

In source, that parent may be a native HTML element or a single imported `x-*` component tag. After component expansion, the template must still resolve to one final native HTML root so Caspian has exactly one place to inject `pp-component`.

This is not just style guidance. The installed compiler injects `pp-component` onto the final root element, and it raises an error when the template has no root, multiple sibling roots, a sibling script after the root, or stray top-level text.

For AI-generated templates, treat this as a hard authoring rule: write the HTML the same way a React component returns one parent node. If the template needs a PulsePoint script, keep that script inside the same parent root.

Failure shape to avoid:

- one root element followed by a sibling `<script>`
- two top-level sibling elements
- top-level text outside the root
- an imported `x-*` root that expands to multiple native siblings

Good:

```html
<div class="card">
  <h2>Title</h2>

  <script>
    const [open, setOpen] = pp.state(false);
  </script>
</div>
```

Bad:

```html
<div class="card">
  <h2>Title</h2>
</div>

<script>
  const [open, setOpen] = pp.state(false);
</script>
```

Also bad:

```html
<h2>Title</h2>
<p>Body</p>
```

Also bad:

```html
<x-card>
  <h2>Title</h2>
</x-card>

<script>
  const [open, setOpen] = pp.state(false);
</script>
```

If you choose an imported `x-*` component as the authored root of the file, make sure it resolves to one native HTML root when compiled. Do not leave an unresolved component tag as the root, and do not emit multiple top-level component siblings.

## Props, Types, And Children

Props come from the attributes on the component tag.

```python
from typing import Any, Literal

from casp.component_decorator import component
from casp.html_attrs import get_attributes, merge_classes

ButtonVariant = Literal["default", "outline", "destructive"]

@component
def Button(
    children: Any = "",
    variant: ButtonVariant = "default",
    **props,
) -> str:
    classes = {
        "default": "btn btn-primary",
        "outline": "btn btn-outline",
        "destructive": "btn btn-danger",
    }

    incoming_class = props.pop("class", "")
    attrs = get_attributes({
        "class": merge_classes(classes[variant], incoming_class),
    }, props)

    return f'<button {attrs}>{children}</button>'
```

Use typed parameters when you want better editor hints and stricter component APIs. `Literal[...]` is a good fit for variants, sizes, or other closed sets of values.

## Async Components

Components can also be async. Use this only when the component render contract really depends on awaited work such as a database query, service call, or file read.

```python
from casp.component_decorator import component, render_html

@component
async def ProfileCard(user_id: str) -> str:
    user = await get_user_by_id(user_id)

    return render_html(__file__, {
        "user": user,
    })
```

Keep synchronous components as the default. Switch to `async def` only when the component itself needs awaited I/O.

## Best Practices

- Put reusable components in `src/components/` and keep route files in `src/app/`.
- For dashboards, admin areas, account sections, and route groups with child routes, put the shared shell in the parent folder's `layout.html` and compose it from reusable components there instead of repeating the same shell in every child `index.html`.
- Default to a single Python file with inline `html(...)` for most components, including ones with PulsePoint behavior. Move to a thin Python wrapper plus a same-name `.html` template only when the markup is large or the `<script>` carries real logic. Both forms render identically, so this is a readability choice, not a capability one.
- For component clicks, inputs, menus, toggles, filters, and list updates, use PulsePoint events and directives inside the component template (inline `html(...)` or the `.html` file). Avoid manual DOM selection, manual listener setup, and manual `innerHTML` rendering unless integrating a third-party imperative widget.
- Keep the component file name, exported function name, and authored tag aligned, such as `Button.py`, `def Button(...)`, and `<x-button />`.
- Accept `children` or `**props` when the component should support nested content.
- Keep page-level data loading in `page()` when the data is not intrinsic to the component itself.
- If you add `@rpc()` functions inside a component file, keep their names globally unique because component RPCs are not route-scoped.

## Related Reading

- Read [project-structure.md](./project-structure.md) for where reusable components should live.
- Read [routing.md](./routing.md) when importing components into route templates.
- Read [pulsepoint.md](./pulsepoint.md) for the runtime contract of `pp-component`, `script[type="text/pp"]`, props, and browser-side reactivity.
- Read [fetch-data.md](./fetch-data.md) when a component needs browser-triggered `pp.rpc()` calls or component-owned server actions.

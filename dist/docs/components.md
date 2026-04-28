---
title: Components
description: Use this page when the task mentions `@component`, reusable UI, HTML-first `x-*` component tags, component imports, same-name `.html` templates, `merge_classes(...)`, `twMerge(...)`, or where shared components belong in a Caspian project.
related:
  title: Related docs
  description: Use the structure guide for file placement, the routing guide for route templates, the PulsePoint guide for browser-side scripts, and the data guide for component-owned RPC flows.
  links:
    - /docs/project-structure
    - /docs/routing
    - /docs/pulsepoint
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
- Use `render_html(...)` with a same-name `.html` file when the component has more markup, PulsePoint behavior, or clearer separation between Python logic and UI.
- Keep page-level workflows in `src/app/`, move reusable UI into `src/components/`, and keep helpers, services, validators, and adapters in `src/lib/`.

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

When a component includes richer UI or PulsePoint behavior, keep the Python file focused on props or server-side preparation and move the markup into a same-name HTML file.

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
  <h3>[[ label ]]</h3>
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

## Auto-Injected `pp-component` And PulsePoint Script Type

Treat `pp-component="componentName"` and `type="text/pp"` as framework output, not authored source.

- Do not manually add `pp-component="..."` to route templates, layout templates, or component HTML files.
- Do not manually add `type="text/pp"` to authored PulsePoint scripts.
- The Python render pipeline injects `pp-component` onto the single root element during render.
- `main.py` runs `transform_scripts(...)`, which rewrites authored body `<script>` tags to `<script type="text/pp">` before the browser runtime mounts.
- Add a plain `<script>` only when the route or component actually needs PulsePoint logic.
- When you do add a PulsePoint script, keep it inside that single root element.

Examples in runtime docs often show rendered HTML that already contains `pp-component` and `type="text/pp"`. That is the final output shape, not the normal authoring pattern.

### Authored Source vs Rendered Output

Author this:

`Counter.html`

```html
<div>
  <h3>[[ label ]]</h3>
  <button onclick="setCount(count + 1)">
    {count}
  </button>

  <script>
    const [count, setCount] = pp.state(0);
  </script>
</div>
```

Do not author this by hand:

```html
<div pp-component="counter_ab12cd34">
  <h3>Clicks</h3>
  <button onclick="setCount(count + 1)">0</button>

  <script type="text/pp">
    const [count, setCount] = pp.state(0);
  </script>
</div>
```

The second example is the runtime shape after the Python side injects `pp-component` onto the component root.

## Single-Root Rule

Every component HTML template must render exactly one top-level lowercase HTML element.

The same rule applies to route templates such as `src/app/**/index.html`: one root element, no sibling roots, and no manual `pp-component` authoring.

Top-of-file `<!-- @import ... -->` directives are allowed before that root element and do not violate the single-root rule.

This is not just style guidance. The installed compiler injects `pp-component` onto the root element, and it raises an error when the template has no root, multiple sibling roots, stray top-level text, or a component tag as the root.

For AI-generated templates, treat this as a hard authoring rule: write the HTML the same way a React component returns one parent element. If the template needs a PulsePoint script, keep that script inside the same parent root.

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

Also avoid making another component tag the root of the HTML file. The root must be a normal lowercase HTML element such as `<div>`, `<section>`, or `<article>`.

Think about this rule the same way you would in a React component: one parent element per template root. In Caspian, that requirement exists so the Python renderer has exactly one place to attach `pp-component`.

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
- If the component includes PulsePoint behavior, prefer a thin Python wrapper plus a same-name `.html` template.
- Keep the component file name, exported function name, and authored tag aligned, such as `Button.py`, `def Button(...)`, and `<x-button />`.
- Accept `children` or `**props` when the component should support nested content.
- Keep page-level data loading in `page()` when the data is not intrinsic to the component itself.
- If you add `@rpc()` functions inside a component file, keep their names globally unique because component RPCs are not route-scoped.

## Related Reading

- Read [project-structure.md](./project-structure.md) for where reusable components should live.
- Read [routing.md](./routing.md) when importing components into route templates.
- Read [pulsepoint.md](./pulsepoint.md) for the runtime contract of `pp-component`, `script[type="text/pp"]`, props, and browser-side reactivity.
- Read [fetch-data.md](./fetch-data.md) when a component needs browser-triggered `pp.rpc()` calls or component-owned server actions.

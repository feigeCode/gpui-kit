# Script authoring

Contents: module names; view/state lifetime; async; permissions; declarations.

## Import the host's actual surface

| JavaScript specifier | Surface |
| --- | --- |
| `gpui-kit` | `View`, primitive elements and shell UI APIs; not the Rust facade's complete API |
| `gpui-base` | Unstyled behavior, flex helpers and retained states such as `InputState` |
| `gpui-component` | Styled catalog, only when injected by the component adapter |
| `gpui-fps` | Native frame-rate monitor |
| A host-defined name, e.g. `market` | Exactly the functions and native element constructors granted by a Rust `HostModule` |

`gpui` is a compatibility alias; prefer explicit `gpui-kit` / `gpui-base`
imports. Do not invent a runtime `import ... from "gpui-shell"` because that is
the Rust crate's name. Typing-only declarations are not proof of runtime exports.
This is QuickJS with a curated runtime, not a DOM/WebView or Node environment.
Do not assume `require`, `node:` imports or arbitrary npm packages work.

## Start with a small view

Save as `main.js` and use the **component** host:

```js
import { View, div } from "gpui-kit";
import { Spinner } from "gpui-component";

export default class LoadingPanel extends View {
  render() {
    return div().p(12).child("Loading").child(new Spinner().size("small"));
  }
}
```

This is a catalog-loading smoke test, not a complete loading/error UX. For
interaction and retained state, follow `examples/js_todolist/main.js`:

- Use `init(props, cx)` for instance state and `InputState.new(...)`, and bind
  state subscriptions there rather than recreating them during render.
- **Do not declare class fields for values assigned in `init`.** `View` calls
  `init` during `super()`; subclass field initialization then overwrites those
  values, even for a declaration with no initializer. Put JSDoc annotations
  directly on assignments in `init`, as the todo example does.
- Store model data and retained state/view handles; never store a built
  element for reuse in the next render.
- `render(cx)` returns a fresh tree. Native painting can replay its snapshot
  without executing JS again. Mutation alone does not invalidate the snapshot.
- In event handlers, use the context **passed to the handler** and call
  `cx.notify()` after changes. Do not close over the render context.
- Controlled controls need the application to update their value and notify;
  base controls supply behavior, not the product's styled appearance.

Host API names generally use `snake_case` (`set_value`, `on_click`);
application methods can use normal JS naming. Do not convert Rust constructors
or event signatures by analogy. Read the generated declaration.

## Context and async lifetime

`init` receives an `AsyncContext`. Render/event `Context` values belong to their
call scope; retaining one does not extend that scope. Use `cx.spawn` for work
that awaits, and use its callback's context after the await:

```js
// Inside an event handler or an instance method called by that handler.
// readSummary is an imported, host-granted async function.
this.loading = true;
cx.notify();
cx.spawn(async (taskCx) => {
  try {
    this.summary = await readSummary();
  } catch (error) {
    this.summary = null;
    console.error(String(error));
  } finally {
    this.loading = false;
    taskCx.notify();
  }
});
```

Use `cx.timer` / the task context's sleep APIs with their current generated
signatures, not browser `setTimeout` / `setInterval` (throwing compatibility
stubs). Respect task ownership and cancellation; do not detach work with
`owner: null` merely to suppress lifecycle errors. Layout-time/deferred item
renderers are read-only: no notify, overlays, state construction or I/O there.

## Identity, theme and large collections

- IDs describe domain identity, not current row positions. Virtualization
  supplies a visible range: construct only that range and return stable keys.
  Match a `Scrollbar` target to its list ID; see
  `examples/js_story/stories/virtual_list.js`.
- Prefer semantic values from `cx.theme()` and supported rem/spacing helpers.
  Numeric `.p(12)` / `.gap(8)` values mean pixels, not Tailwind scale indices.
  Do not assume the Rust and JS theme shapes or fluent methods are identical.
- Distinguish base window overlays from component-catalog overlays; preserve
  the host setup required by whichever surface the script uses.

## Manifest, grants and dependencies

The manifest filename is **`gpui-shell.json`**:

```json
{
  "id": "com.example.notes",
  "name": "Notes",
  "entry": "main.js",
  "capabilities": {
    "storage": true
  }
}
```

`id`, `name`, `entry` are required when supplying a manifest; `version`,
`shell-version`, `dependencies` and `capabilities` are optional. Do not copy a
historical version number without checking compatibility. Standalone directory
apps can omit the manifest and use `main.js`; plugin discovery needs identity.

Default system access is denied. The shipped CLI installs manifest grants,
whereas an embedding host must independently authorize requests. `localStorage`
is supported but gated, and persistence needs a configured storage path.
`examples/js_todolist/storage.js` demonstrates catching denial and explicitly
falling back to memory.

Use the smallest FS/HTTP/process grants. A `network.hosts` grant is broader than
HTTP alone (also TCP/WebSocket); prefer `network.http` method/path/scheme rules
when sufficient. Never “fix” a denial by granting everything.

Manifest Git dependencies are fetched by the host before entry evaluation and
before script capabilities apply. Pin commits when reproducibility matters;
moving refs are not immutable. Tooling links under `node_modules` do not turn
this into Node or mean `npm install` is required to launch a plain script app.

## Generated declarations

Generate types with the **same catalog and host modules** as the runtime.
`gpui-kit.d.ts` is generated output: do not hand-edit or commit it as the source
of truth. `jsconfig.json` is scaffolded once and then application-owned.
Custom module declarations must describe the exports actually registered.

The plain `gpui-shell types` command cannot describe an embedding application's
private HostModules. Use that host's declaration-generation path (or maintain
its explicit declaration input) and test runtime exports as well.

Sources: `crates/shell/README.md`, `crates/shell/src/plugin.rs`,
`crates/shell/src/typings.rs`, `examples/js_todolist/`,
`crates/story/js/quotes/main.js`, `examples/js_story/stories/virtual_list.js`.

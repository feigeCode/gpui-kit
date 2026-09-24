# Layer selection: base, styled components and Shell

All source paths below are relative to the gpui-component checkout. Verify the
embedding application's pinned revision before applying this checkout's API.

## Different responsibilities, not competing implementations

| Name | Rust responsibility | JavaScript surface |
| --- | --- | --- |
| `gpui-base` | Unstyled interaction/state primitives; presentation belongs to the caller | Built-in `gpui-base`: layout helpers, editable state and base elements |
| `gpui-component` | Ready-styled native components built on base behavior | `gpui-component`, **only when the host supplies its adapter catalog** |
| `gpui-shell` | JS execution, `View`/contexts, base bindings, policies, host modules, loading and registry protocol | `gpui-kit` (also `gpui` compatibility alias), `gpui-base`, and other supported runtime modules |
| `gpui-component-shell` | Registers typed component constructors and native materializers into Shell; provides a component-aware host/CLI | Registers `gpui-component`; not a JS import named `gpui-component-shell` |

The dependency relationships are:

```text
gpui-component       -> gpui-base
gpui-shell           -> gpui-base
gpui-component-shell -> gpui-shell + gpui-component
application host     -> runtime + chosen catalog + application HostModules
```

Do not reverse the arrows or move concrete component/Navop business knowledge
into the generic runtime. `gpui-base` does not execute extensions; the adapter
does not replace the runtime or create another permissions system.

## Which should I use?

| Work | Choose |
| --- | --- |
| Native Rust app with standard themed controls, no scripts | `gpui-component` / app facade; no Shell adapter needed |
| Native Rust custom design system or low-level interaction implementation | `gpui-base`; supply presentation and follow its state/focus contract |
| Script UI using only layout, text and intentionally custom/base elements | `gpui-shell` host + imports from `gpui-kit` and `gpui-base` |
| Script UI using ready-styled buttons, inputs, selectors, tables or dock bindings | Host adds `gpui-component-shell`; script imports **bound** controls from `gpui-component` |
| Styled script form with custom layout or editable text state | **Mix** base layout/state with styled component elements |
| Expose an application operation to a script | An approved `HostModule`; not a new visual adapter or a provider UI protocol |

A base-only page remains valid in a component-enabled host. A styled page cannot
assume a bare Shell CLI/host has the catalog. Conversely, importing base layout
helpers does not mean that the host should remove its component adapter.

For embedded hosts, catalog installation and global initialization are separate
requirements. Use the application's existing bootstrap/root ownership contract;
do not call a standalone initializer a second time or replace the host window
with `ShellRoot` merely to load a page. Catalog availability alone does not prove
that an overlay has a correctly mounted host.

## Same name does not mean the same constructor

For editable text in this checkout, use the public base state API:

```js
import { View, div } from "gpui-kit";
import { InputState, TextareaState, v_flex } from "gpui-base";
import { Input, Textarea } from "gpui-component";

export default class Form extends View {
  init() {
    this.topic = InputState.new({ value: "topic", placeholder: "Topic" });
    this.body = TextareaState.new({ value: "", rows: 4 });
  }
  /** @param {import("gpui-kit").Context} cx */
  render(cx) {
    return v_flex()
      .child(new Input(this.topic))
      .child(new Textarea(this.body))
      .child(div().child(this.topic.value()));
  }
}
```

This is the base-state → styled-element bridge exercised by
`crates/component-shell/tests/base_state_bridge_render_test.rs`,
`component_input_materializes_from_a_gpui_base_state`.
It is a runtime-bridge example, **not a claim that the currently checked-in
declarations accept it**: the 2026-09-16 documentation audit ran strict JS
type-checking against `examples/js_story/gpui-kit.d.ts` and reproduced `TS2345`
for both states (missing component `__gpuiComponentState` brand). The Rust
materialization test was inspected, not executed in that documentation audit.
For intentionally unstyled inputs, import `Input`/`Textarea` from `gpui-base`
instead and supply the desired presentation. Do not rebuild state in `render`.

Do **not** generalize this to “all state lives only in gpui-base”:

- The component catalog also has retained state descriptors. Its same-named
  `InputState(...)` is not the base `InputState.new(...)` API; do not describe all
  catalog state as empty or interchangeable.
- A runtime bridge test proves that particular conversion, not every
  base/component state pair, reverse conversion or every older host revision.
- Generated declarations can retain separate branded state types. If JS renders
  but TypeScript rejects a mixed constructor, compare the **same revision's**
  bridge implementation, declaration generator and generated types. Report/fix
  that mismatch at its source; do not invent a constructor or broadly cast to
  `any` to declare compatibility.
- Use matching generated types plus a materialization/interaction test.
  Merely finding a Rust component or passing a JS tree check is insufficient.

## Small examples and source anchors

- `crates/base/src/lib.rs`, `crates/base/src/input/input/mod.rs`:
  unstyled ownership and `InputState`-based input.
- `crates/component-shell/src/lib.rs`: `components()`, `init()`,
  `new_isolated_runtime()` and component-aware window opener.
- `crates/shell/src/component_registry.rs`: default module name and reserved
  runtime modules; a catalog must not shadow `gpui-base` or `gpui-kit`.
- `examples/js_story/app.js`: base `Input` + base `InputState` for search;
  `README.md` explains editable state and generating declarations.
- `crates/component-shell/tests/base_state_bridge_render_test.rs`: minimal
  mixed-input regression, including native materialization error assertions.
- `examples/js_todolist`: state/events/async lifecycle; not a template for an
  embedding application's manifest or permission grants.

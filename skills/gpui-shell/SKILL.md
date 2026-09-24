---
name: gpui-shell
description: 'Build and debug JavaScript extensions hosted by GPUI Shell. Use for gpui-base vs gpui-component-shell layer selection, gpui-shell runtime vs component adapter, embedded ScriptView and ShellRoot, HostModule Rust APIs, per-plugin Policy and capabilities, PluginManager lifecycle, ComponentRegistry adapters, gpui-shell.json, generated JavaScript typings, hot reload, or the js_todolist and js_story examples. Not a system terminal shell, a Node/browser runtime, or a WASM/IPC driver extension mechanism.'
---

# GPUI Shell Extensions

Use this skill for the JavaScript extension boundary, not as a replacement for
the Rust GPUI skills. Read the relevant Coding Guides before Rust architecture,
ownership, or API changes, and the Design Guides before visible UI changes
(`gpui-kit` / `gpui-kit-design-guides`, or the workspace `gpui-component` skill).
In this repository these are `skills/gpui-kit/references/coding-guides.md`
and `skills/gpui-kit-design-guides/references/design-guides.md`. For a narrow
hosting change, read the coding guide's architecture, agent rules and bootstrap/
root-ownership sections; include the design guide's feedback/overlays section
when adding dialogs. Follow the companion skills' full-read rules for new
features or screens.

## First choose the layer

| Layer | Owns | Does not imply |
| --- | --- | --- |
| `gpui-base` (`crates/base`) | Unstyled behavior, retained state, layout primitives; exposed to scripts through the runtime's built-in `gpui-base` module | A JS runtime, styled component catalog or plugin manager |
| `gpui-component` (`crates/component`) | Styled Rust components built on the base layer | Automatic availability in JavaScript |
| `gpui-shell` (`crates/shell`) | JavaScript engine, script views, contexts, policies, host modules, component-registry protocol, generated types, loader/watcher, base CLI | The styled `gpui-component` catalog |
| `gpui-component-shell` (`crates/component-shell`) | Concrete component descriptors/materializers, catalog initialization, component-aware window opener, CLI with this catalog | A second JS engine or an alternative plugin security model |
| Application host | Approved capabilities/modules, state, plugin lifetime, mounting and product contributions | Automatic approval of every manifest request |

Dependency direction: `gpui-component-shell` depends on both `gpui-shell` and
`gpui-component`; the runtime must not depend back on the adapter. Ordinary
Rust applications can use the `gpui-kit` facade, but Shell hosting needs separate
dependencies. Do not assume `gpui_kit::shell` exists.

**Base and adapter are not alternatives.** Script authors normally combine
`gpui-base` layout/input state with styled elements imported from
`gpui-component`; Rust hosts supply those styled bindings through
`gpui-component-shell`. Use base-only elements for deliberately custom
presentation or a base-only host. Do not import `gpui-component-shell` in JS.
See [Layer selection and mixed inputs](references/layer-selection.md).

Choose the extension seam before writing code:

1. **Compose a script UI:** a default-exported JS `View`, with the catalog the
   host actually supplies.
2. **Expose application operations:** Rust `HostModule` functions imported by
   module name. This is a plain-data service boundary, not native plugin loading.
3. **Expose a host-owned native element:** `HostModule::component` gives JS an
   opaque constructor with an explicit ID, plain-data props and children.
4. **Expose a full native component catalog:** Rust `ComponentRegistry`
   descriptors and materializers define typed constructors/fluent methods.
   Extend the adapter, not the engine's concrete component list.
5. **Manage installed plugins:** `PluginManager` and per-plugin `Policy`;
   discovery, authorization, loading, mounting and unloading are distinct steps.
   An embedding product with its own catalog/manifest should reuse the explicit
   policy-aware loader instead of adding a second manifest or plugin manager.
   Read that product's companion extension skill for its installation contract.

## Read by task

| Task | Reference |
| --- | --- |
| Choose base vs styled components, distinguish Rust crates and JS imports | [Layer selection and mixed inputs](references/layer-selection.md) |
| Write a JS app, events, async work, manifest or imports | [Script authoring](references/script-authoring.md) |
| Embed views, expose Rust functions, permissions, lifecycle, add native bindings | [Hosting and extension seams](references/hosting-and-extensions.md) |
| Find an example, choose checks, debug a mismatch | [Examples and verification](references/examples-and-verification.md) |

Paths such as `crates/shell/src/...` in these references are relative to the
**gpui-component repository root**, not the skill directory or an outer
workspace. Locate the checkout first. If working against another revision,
verify its source/API instead of mechanically applying this revision's names.

## Workflow

1. Identify the actual Rust host and its frozen catalog; distinguish the crate,
   CLI executable and JavaScript module specifier.
2. Read the smallest relevant example and the public signature being used.
   Generate `gpui-kit.d.ts` using that same host/catalog.
3. Implement one vertical slice: initialize retained state, render, handle one
   event, notify; then add host calls and narrowly scoped permissions.
4. Validate declarations, JS loading/materialization, and real interaction
   separately. A successful `check` is not proof that a dialog paints.
5. Record an observed failure as a scoped rule with a source/test pointer.
   Do not copy a complete component catalog or freeze example counts here.

## Rules that prevent the common failures

- **Source over stale prose.** Use public Rust APIs, current adapter descriptors,
  matching generated declarations and tests together. Some shell pages still
  show `gpui_kit::shell`; old state documentation also denies `localStorage`
  despite current capability-gated support. Historical design plans are not
  evidence that a feature is absent.
- **No Rust-to-JS API transliteration.** A Rust component existing does not mean
  it is bound, and a generic-looking fluent method need not be legal on every
  registered component. Verify constructors, callbacks and rejected methods.
- **State is retained, elements are not.** Create state in `init` or a permitted
  update; build fresh elements in `render`. Do not notify or start side effects
  during render/layout.
- **Contexts have lifetimes.** Events use their own `cx`; asynchronous work uses
  the `AsyncContext` supplied by `cx.spawn`, not a captured render/event context.
- **Two setup steps.** Adapter `init(cx)` initializes globals; a runtime still
  needs the adapter's catalog. Correct component overlays also need `Root` and
  the sheet/dialog/notification layers.
- **Authority belongs to the host.** The CLI's manifest-grant behavior is not an
  embedding policy. Host-module registration itself grants access; validate
  arguments and avoid exposing unrestricted file/process/network operations.
- **Native repaint is not JS invalidation.** Host changes consumed by script
  require the script refresh path, not just a GPUI `cx.notify()`.
- **Report the evidence level.** Separate source inspection, generated-type
  checks, runtime materialization, compiled tests and manual UI interaction.

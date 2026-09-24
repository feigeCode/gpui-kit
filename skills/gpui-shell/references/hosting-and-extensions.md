# Hosting and extension seams

Contents: host setup; invalidation/lifecycle; HostModule; ComponentRegistry;
plugin policy.

## Choose a host setup, not just an import

The catalog is frozen data owned by the runtime. These steps are independent:

| Host | Initialize on the GPUI app thread | Construct runtime |
| --- | --- | --- |
| Base only | `gpui_shell::init(cx)` | `ShellRuntime::new(cx)` or deliberately `new_isolated()` |
| Styled adapter | `gpui_component_shell::init(cx)` | `ShellRuntime::new_with_components(cx, gpui_component_shell::components()?)` or `gpui_component_shell::new_isolated_runtime()` |
| Generic catalog host | `gpui_shell::init_with_components(cx, &catalog)` | Pass that catalog to the appropriate `*_with_components` constructor |

`init` alone does not inject descriptors into a subsequently created bare
runtime. `components()` builds and freezes the adapter catalog, including its
initializer and window opener. The adapter's `register` function is private;
do not publish a recipe calling `gpui_component_shell::register`.

`ShellRuntime` is `Rc`-owned and belongs to the GPUI/UI thread; do not put it in a
worker thread or invent an `Arc<Mutex<_>>` workaround. `new(cx)` installs one
default runtime; additional independently owned VMs use the isolated path.
Use the checkout's matching Cargo dependencies and GPUI types; do not mix
unrelated crate versions or assume a `gpui_kit::shell` re-export.

### Mounting and window roots

Three public routes serve different needs:

- `runtime.try_load(directory, window, cx)` gives a `ShellRoot` with application
  metadata, content and base overlays, suitable for refresh/watch. `load` is the
  error-surface variant. See the Rust Shell story.
- `load_application(&path, entry)` then `mount_application(&loaded, window, cx)`
  gives an embedded `Entity<ScriptView>`. See
  `crates/component-shell/tests/public_host.rs`. A loaded
  handle belongs to its originating runtime and is **single mount**, including
  a failed attempt. **This uses the default policy**, not independent plugin
  grants, and is not a complete window/overlay setup.
- For per-plugin host APIs/authority, use `load_view(ViewLoadOptions, ...)`
  instead; see the explicit-policy recipe below.

For styled windows, retain `gpui_component::Root` as the actual window root.
Its child must render the sheet, dialog and notification layers: `Root` itself
does not draw those three. A wrong root can panic; missing layers can make a
successfully opened dialog invisible.

The adapter's `open_window_with_root` / `CatalogHost` implementation is the
reference composition: component `Root` -> catalog host -> `ShellRoot` plus
component overlay layers. The shared CLI uses the catalog's opener. An
embedding host opening its own window must satisfy the same contracts; do not
assume `mount_application` invokes that opener or installs a new root.

### Invalidation, watching and teardown

- JS event/task `cx.notify()` invalidates the script view. Rust GPUI
  `cx.notify()` alone can repaint the already-published snapshot. When host
  state read by JS changes, use `runtime.refresh(&root, cx)?` for a loaded
  `ShellRoot`. For a directly mounted `Entity<ScriptView>`, use
  `script.update(cx, |view, cx| view.refresh(cx))`; see
  `crates/shell/src/view.rs::ScriptView::refresh`. Follow the Shell story's
  observer rather than inventing a notification bridge.
- `runtime.watch(&root, window, cx)` expects a root loaded by that runtime.
  Keep the returned `Watcher` or deliberately call `.forget()`. Watchers hold
  weak references; they do not keep the runtime/view alive.
- Do not call `watch` while `WindowHandle<ShellRoot>::update` leases that same
  root. Enter through `AnyWindowHandle::update` / `App::update_window`.
  A rejected reload preserves the last good application; test both paths.
  See `crates/shell/src/watch.rs::ShellRuntime::watch`.
- The public watcher is **not** a generic `LoadedScriptView` / `PluginManager`
  watcher. Do not switch a policy-aware plugin to `try_load` just to obtain a
  watchable root: that changes its authority route. A custom-policy host needs
  an explicit reload integration that preserves policy and lifetime, with
  last-good/error behavior tested separately.
- Keep owning runtime/root/plugin handles alive for the intended UI lifetime.
  Unload policy-owned work when removing a plugin. Registered closures can
  retain host entities; release the relevant registrations at teardown, without
  clearing unrelated plugins' grants.

## Rust service extension: HostModule

Read `crates/shell/src/host_modules.rs` and
`crates/story/src/stories/shell_story.rs::market_module` before adding a bridge.

Minimal **single-application** registration, before loading script:

```rust
use gpui_shell::{HostModule, HostValue};

gpui_shell::export_module(
    HostModule::new("workspace")
        .function("project_name", |_| Ok(HostValue::from("Example"))),
)?;
```

Script: `import { project_name } from "workspace";`.
For a plugin-specific grant, use `Policy::with_host_module` and the policy-aware
loading path rather than granting the module through the default registry.
Registration is itself the grant; no separate manifest permission protects a
carelessly exposed function.

Function contract (the native element builder below has a different Rust API):

1. Inputs/outputs are `HostArguments` / `HostValue`: null, bool, number, string,
   arrays and objects. No JS callback, engine value or GPUI entity crosses.
2. Validate types, bounds and domain authorization in Rust. A typed signature
   is not an authorization check. Reserved module names are rejected.
3. `.function` runs synchronously on the UI thread. Slow I/O belongs in
   `.async_function`: its setup closure runs on the UI thread, captures owned
   data, and returns a `Send` future for the background executor. Never move an
   `App`, window or runtime into that future.
   For expensive/cancellable queries, use `.cancellable_async_function` with
   a `HostAsyncTask` cancellation action that stops the underlying operation.
   Ordinary async cancellation suppresses the JS continuation (its Promise
   stays pending); do not assume that also stops external I/O or a subprocess.
4. Do not synchronously re-enter the VM or pump the event loop during a host
   call. Schedule notifications to be processed after the call unwinds.
5. Supply truthful `.declarations(...)` for custom APIs. Export-name validation
   is not a proof that every value returned matches the declared TS type, and
   does not even enforce `Promise` on asynchronous exports.
6. Register before linking: exported names are fixed for that module import.
   Calls forward through the live registry, so revocation/replacement affects
   subsequent calls; adding a new export is not retroactive relinking.

This mechanism exposes Rust **compiled into the host**, not arbitrary native
libraries loaded by plugins. Do not introduce `dlopen` as a permission-safe
extension mechanism.

## Host-owned native element: HostModule::component

Not every native UI extension needs a catalog descriptor.
`HostModule::new("mail").component("Body", build)` exports a constructor backed
by Rust compiled into the host. Grant this module through the same default or
per-plugin policy paths as functions.

After the host registers and grants it, JS can use:

```js
import { div } from "gpui-kit";
import { Body } from "mail";

// Return from render(); "mail" is a custom host module, not a built-in import.
Body.new("message-body", { text: "Hello", zoom: 1.25 })
  .child(div().child("fallback"));
```

The builder receives `ComponentArgs`, `&mut Window`, `&mut App` and returns
`AnyElement`. `args.id()` is an explicit **non-empty string**; `args.props()`
is plain `HostValue` data to validate. Children are already materialized native
elements: inspect `args.children()` or consume `args.take_children()` and
deliberately mount them. Unlike service functions, this Rust callback can use
the window/app to build UI; it does not expose those native handles to JS.

This is a host-owned element boundary, not automatic access to a Rust
component's fluent API. Do not invent styled catalog methods on the exported
constructor. Keep factory captures bounded by module/plugin lifetime; test
revocation as well as construction. See `crates/shell/src/component.rs`,
`crates/shell/src/host_modules.rs::HostModule::component` and the
`component_is_imported_by_name_and_receives_an_explicit_id`,
`component_requires_an_explicit_string_id` and
`revoking_a_component_module_releases_its_builder_while_the_view_lives` tests
in `crates/shell/src/tests/render.rs`.

## Full native component catalog: ComponentRegistry

Use this seam for a descriptor-defined constructor/method/schema/type surface,
rather than one opaque host-owned element. Start with
`crates/component-shell/src/shell/spinner.rs`; then follow the nearest retained,
structured-child or deferred-slot binding rather than treating every component
as a leaf.

1. Declare constructors, methods, closed argument schemas and documentation
   using `ComponentDescriptor` and related descriptors.
2. Parse inputs into typed `ComponentPayload` values. Do not pass opaque JS
   handles through the generic materializer boundary.
3. Implement `ComponentMaterializer` with the real component. Apply only the
   supported styles and methods; preserve child order and state ownership.
   Unsupported fluent calls must fail, not silently become no-ops.
4. Register in `crates/component-shell/src/shell/mod.rs`, then freeze.
   `COMPONENT_REGISTRY_API_VERSION` and registry validation are real contracts.
   Put concrete `gpui-component` imports in the adapter, never in `gpui-shell`.
5. If the catalog requires app globals or a special window root, carry the
   initializer/window opener with the catalog so generic hosts can honor it.
6. Test runtime acceptance **and rejection**, generated typings (including
   negative TS fixtures), eager materialization, and real deferred/interactive
   rendering as appropriate. A Rust API is not automatically exposed to JS.

`components()` returns a frozen catalog, not a public mutable stock registry.
Do not promise an external caller can append arbitrary descriptors to it.
For a composed/custom catalog, inspect the available assembly API first or make
an explicit adapter API change with tests.

## Plugin policy is a host decision

`PluginManager::discover` reads manifests without evaluating scripts.
Authorization precedes loading/evaluation; do not load code merely to find out
what permission it wants. Use the `Policy` / plugin loading APIs for identity,
capabilities, storage path and per-plugin HostModules.

There are two distinct policy-aware paths in `crates/shell/src/plugin.rs`:

- `PluginManager::load(..., authorize, ...)` approves/rejects an inert manifest
  and builds its policy from the approved manifest. Its public load signature
  does **not** accept arbitrary custom HostModules or a caller-supplied policy.
- `ShellRuntime::load_view(ViewLoadOptions, ...)` accepts an explicit policy.
  Use it when the host needs custom per-plugin modules, narrower grants or its
  own lifecycle integration; perform discovery/authorization before this call.

Example setup inside the host's GPUI window update (after authorization; the
runtime already has the required catalog):

```rust
use std::rc::Rc;
use gpui_shell::{HostModule, HostValue, ViewLoadOptions, policy::Policy};

let policy = Rc::new(
    Policy::new()
        .with_application("com.example.panel")
        .with_host_module(
            HostModule::new("workspace")
                .declarations("export function project_name(): string;")
                .function("project_name", |_| Ok(HostValue::from("Example"))),
        )?,
);
let loaded = runtime.load_view(
    ViewLoadOptions::new(plugin_directory, "main.js", policy)
        .write_type_declarations(true),
    window,
    cx,
)?;
// Mount loaded.view().clone() in the host's UI and retain `loaded` there.
// On removal: loaded.unload(cx), then remove the displayed view.
```

This deliberately grants no system capabilities. Add only authorized
capabilities and a plugin-specific storage path when needed. Do not drop
`LoadedScriptView` immediately after cloning its view: the handle owns the
application/policy lifetime. Its `unload(&mut self, cx)` retires the view,
cancels policy tasks (including ownerless work) and revokes modules. Underlying
host operations still need the cancellation action described above.
`PluginManager` instead owns its loaded entries and unloads by plugin ID.
Do not assume a JS `deactivate` hook or that rebuilding a default policy changes
grants already captured by live views.

Capability-controlled JavaScript is not an OS process isolation guarantee.
HostModules can exercise all privileges of the native host unless deliberately
restricted. A complete commands/panels/settings marketplace contract must be
checked in the current host, not inferred from the existence of a manifest or
from historical design plans.

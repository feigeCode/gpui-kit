# Examples and verification

Contents: learning order; commands; test levels; troubleshooting.

Run commands from the gpui-component repository root. Inspect working-tree
changes first: runtime loading can regenerate ignored `gpui-kit.d.ts`, scaffold
missing editor config, fetch declared Git dependencies, or initialize storage.
Use a temporary app directory for a smoke test of new documentation snippets.

## Read examples in this order

| Example / test | Learn | Do not infer |
| --- | --- | --- |
| `examples/js_todolist/main.js`, `storage.js`, `confirm.js` | `init`, retained input state, controlled changes, notification, base dialogs, graceful storage denial | That it needs the styled adapter or that persistence is always granted |
| `examples/js_story/main.js`, `app.js`, `stories/registered.js` | Catalog-backed construction and host selection | That a gallery inventory proves every interaction works |
| `examples/js_story/stories/virtual_list.js`, `stories/dock.js` | Visible-range construction, stable keys, matching scrollbar ID, retained dock state | That old design prose saying Dock is unavailable is current |
| `crates/story/src/stories/shell_story.rs`, `crates/story/js/quotes/` | Real Rust+JS embedding, shared model, `market` HostModule, async split, refresh vs repaint, watch | That standalone shell CLI provides the private `market` module |
| `crates/component-shell/src/lib.rs` | Catalog assembly, init, component window root and overlay layers | That `init` alone injects bindings or `Root` draws all layers |
| `crates/component-shell/tests/public_host.rs` | Public load/mount boundary, errors, same-runtime and single-mount constraints | That directly drawing a ScriptView tests window overlays |
| `crates/component-shell/tests/structured_host.rs`, `tests/types/` | Structured children, positive/negative type contracts and runtime rejection | That type success proves deferred rendering or paint |
| `crates/shell/src/tests/render.rs` (`component_*`, `revoking_a_component_module_*`) | `HostModule::component`, explicit string ID, props, materialized children and factory revocation | That native elements always require a full catalog descriptor |
| `crates/shell/src/tests/host_api.rs`, `src/plugin.rs`, `src/watch.rs` | Host-call/async contract, policy and reload lifetime tests | That default registry grants isolate multiple plugins |

## Match run, types and check to the host

```sh
# Base runtime: unstyled behavior plus the example's own presentation.
cargo run --locked -p gpui-shell -- examples/js_todolist
cargo run --locked -p gpui-shell -- types examples/js_todolist
cargo run --locked -p gpui-shell -- check examples/js_todolist

# Styled catalog: use the adapter executable for all three operations.
cargo run --locked -p gpui-component-shell --bin gpui-component-shell -- examples/js_story
cargo run --locked -p gpui-component-shell --bin gpui-component-shell -- types examples/js_story
cargo run --locked -p gpui-component-shell --bin gpui-component-shell -- check examples/js_story
```

Using the bare host against `import ... from "gpui-component"` is not an
installation problem; it is a catalog mismatch. A custom host needs equivalent
checks with its own HostModules and policies. Never “fix” missing types by
inventing ambient declarations for functionality the running host lacks.

## Verification is layered

1. **Declarations/editor:** regenerate with the matching runtime, then type-check
   consumers and negative fixtures. Generation alone does not run TypeScript.
2. **Headless runtime `check`:** resolves/imports the app, constructs its root,
   executes render and eagerly materializes reachable native nodes. It does
   **not** exercise full layout/paint, deferred slots, every nested view,
   input interaction or eventual async outcomes.
3. **Compiled contracts:** use the narrowest relevant test first:

   ```sh
   cargo test --locked -p gpui-component-shell --test public_host
   cargo test --locked -p gpui-component-shell --test structured_host
   cargo test --locked -p gpui-shell --lib typings
   cargo test --locked -p gpui-shell --lib tests::render
   ```

4. **Broader adapter verification:**

   ```sh
   node examples/js_story/fixtures/verify-coverage.mjs
   npm --prefix crates/component-shell/tests/types ci
   npm --prefix crates/component-shell/tests/types test
   # Aggregate Rust/render/adapter/type contracts; potentially expensive.
   script/check-ai shell
   ```

   Coverage checks catalog/example inventory, not pixel or interaction
   correctness. `GPUI_COMPONENT_SHELL_BIN` can point the type runner at an
   existing executable; report that choice and do not equate a stale executable
   with a build of the current source.

5. **Real window:** verify click/keyboard activation, retained state after
   rerender, dialog/sheet/notification visibility and focus, async success/error,
   capability denial and unload/reload. For a hot-reload change, verify a bad
   edit keeps the last good view and a corrected edit recovers.

Before reporting success, separate checks actually executed from recommended
checks. Compilation/linker/environment failures are not passing tests. Do not
delete unrelated build artifacts to make space without user agreement.

## Symptom -> first boundary to inspect

| Symptom | First check |
| --- | --- |
| Unknown `gpui-component` import | Which executable/catalog actually loaded the app? |
| Method type-checks but throws | Are declarations generated by the same runtime/catalog? Is the method rejected on this component? |
| Initialized property becomes `undefined` | Does a subclass field overwrite `init` after `super()`? |
| “Context expired” after a click/await | Captured render/event context instead of event/task `cx`? |
| JS state changes but old content remains | Missing JS notification or host script refresh? |
| Dialog is open but invisible | Actual window `Root`, and sheet/dialog/notification layers? |
| Typings work but custom module cannot import | Did this host register that module before linking under the right policy? |
| UI freezes on host operation | Synchronous host function performing slow I/O or VM re-entry? |
| Watch does nothing / root borrow error | Watcher retained, matching runtime/root, and no typed-root update lease? |
| Works in `check`, fails in real window | Deferred callbacks, nested views, layout/paint and actual interactions not covered by `check`? |
| Storage/process/network denied | Approved policy and configured storage path; do not widen grants blindly |

When extending this skill, attach new gotchas to a concrete source or regression
test and keep historical limitations out unless they still hold in the checkout.

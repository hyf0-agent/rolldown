# Plugin Asset Module

## Summary

Asset modules (`ModuleType::Asset`) are handled by a built-in Rust plugin (`rolldown_plugin_asset_module`) instead of hardcoded logic in the core. The plugin uses existing plugin APIs (`load`, `renderChunk`, `emitFile`) following the same pattern as `CopyModulePlugin`.

**`load` hook (Post order):** Reads the file as binary, emits it via `ctx.emit_file_async()`, associates the module with the emitted file, and returns `export default "__ROLLDOWN_ASSET__#<ref_id>"` as `ModuleType::Js`.

**`renderChunk` hook (Pre order):** Scans for `__ROLLDOWN_ASSET__#` placeholders using `memchr::memmem`, resolves each ref_id to an asset filename via `ctx.get_file_name()`, and replaces placeholders with relative paths.

**`new URL()` bridge:** The `FileEmitter` has a `module_to_emitted_file` map so the `new URL('./asset', import.meta.url)` finalizer can look up the emitted asset by module ID.

## `has_lazy_export` and CJS Interop

The old built-in `ModuleType::Asset` was in the `has_lazy_export` list, making asset modules get `ExportsKind::CommonJs` treatment. The plugin version returns `export default "..."` with `ModuleType::Js`, making them ESM. This changes wrapping when CJS modules `require()` an asset:

- Old: `__commonJSMin` — `require('./asset')` returns the string directly
- New: `__esmMin` + `__toCommonJS` — `require('./asset')` returns `{ __esModule: true, default: "..." }`

**Decision:**

- **No runtime behavior change** — the resolved asset path value is the same.
- **Pure ESM code** has no output size difference, since ESM imports don't go through CJS interop wrappers.
- If we ever want to support CJS lazy-export behavior for plugins, it should be explored as a **plugin architecture feature** (e.g. a new plugin API or hook), not hardcoded per module type. `has_lazy_export` is a linker-stage mechanism (manipulates AST, ExportsKind, stmt infos) that plugins cannot replicate directly.

## Snapshot Differences from Old Built-in Approach

1. **Hash values** — `FileEmitter` hashing vs old `HashPlaceholderGenerator` pipeline
2. **`./` prefix** — `compute_relative_path()` adds `./` for consistency with `CopyModulePlugin`
3. **CJS wrapping** — `__commonJSMin` → `__esmMin` + `__toCommonJS` (ESM instead of lazy export)

## Related

- `crates/rolldown_plugin_asset_module/` — plugin implementation
- `crates/rolldown_plugin_copy_module/` — similar plugin pattern for `ModuleType::Copy`

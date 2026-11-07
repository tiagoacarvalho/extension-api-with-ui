# Zed Extension API: current state and a proposal for declarative UI extensions

Scope: this document describes the extension system in `/home/tiago/Repositories/zed` (read-only research, main at `9a8ab54db4`), and proposes how to extend it so extensions can contribute UI. Context is GitHub discussion #53403 (see `thread.md`): people want sidebar panels, status bar items, custom views (DB/SQLite/JSONL viewers), automatic theming, workspace context, streaming/async work, and an explicit permission model. The RFC author prefers restricted GPUI-based rendering over webviews.

All paths are relative to `/home/tiago/Repositories/zed` unless noted. Line numbers are as of the commit above and are approximate to the line the symbol starts on.

---

## 1. Current architecture

### 1.1 Crates

| Crate | Role |
|---|---|
| `crates/extension_api` (`zed_extension_api`, v0.8.0, `publish = false` until 0.8.0 ships: `Cargo.toml:9-10`) | Guest SDK. Rust trait `Extension`, `register_extension!` macro, `wit_bindgen::generate!` over `wit/since_v0.8.0` (`src/extension_api.rs`, `mod wit` near line 357). Also `src/http_client.rs`, `src/process.rs`, `src/settings.rs` (ergonomic wrappers). `build.rs` encodes the crate version into a `zed:api-version` wasm custom section (6 bytes: major/minor/patch as big-endian u16). |
| `crates/extension_api/wit/since_vX.Y.Z/` | One WIT package per API version. Versions present: 0.0.1, 0.0.4, 0.0.6, 0.1.0, 0.2.0, 0.3.0, 0.4.0, 0.5.0, 0.6.0, 0.8.0 (there is no 0.7.0). |
| `crates/extension` | Host-agnostic types: `ExtensionManifest` (`src/extension_manifest.rs:88`), `ExtensionCapability` (`src/capabilities.rs:14`) and the three capability structs (`src/capabilities/*.rs`), the `Extension` trait the rest of Zed calls (`src/extension.rs:52`), delegates `WorktreeDelegate`/`ProjectDelegate`/`KeyValueStoreDelegate` (`src/extension.rs:35-50`), `ExtensionHostProxy` and the per-feature proxy traits (`src/extension_host_proxy.rs`), `ExtensionBuilder` (compiles Rust to wasm, `src/extension_builder.rs`), `copy_extension_resources`. |
| `crates/extension_host` | `ExtensionStore` (install/uninstall/index/load, `src/extension_host.rs:157`), `WasmHost`/`WasmExtension`/`WasmState` (`src/wasm_host.rs`), per-version bindings (`src/wasm_host/wit/since_v*.rs`, dispatcher in `src/wasm_host/wit.rs`), `CapabilityGranter` (`src/capability_granter.rs`), `ExtensionSettings` (`src/extension_settings.rs`), `HeadlessExtensionStore` for remote servers (`src/headless_host.rs`). |
| `crates/extension_cli` | `zed-extension` binary: compiles an extension, validates the manifest (`validate_extension_manifest`, `src/main.rs:265`), tests grammars/languages/themes/snippets/debug schemas, packages the archive. |
| `crates/extensions_ui` | The Extensions page (`ExtensionCard`, install/uninstall UI). It currently has no capability display or consent UI (grep for "capabilit" only hits an empty `capabilities: Vec::new()` in a test fixture, `src/components/extension_card.rs:497`). |
| `crates/settings_content/src/extension.rs` | `ExtensionSettingsContent` and `ExtensionCapabilityContent` (settings JSON shape). |

### 1.2 Host/guest model

- An extension is a WebAssembly **component** (component model, WASI preview 2) compiled from Rust with target `wasm32-wasip2` (`crates/extension/src/extension_builder.rs:26-27`, `cargo build --target wasm32-wasip2` at `:214`). wasi-sdk is downloaded and used for C dependencies (`:449-560`).
- The guest implements the `world extension` (`wit/since_v0.8.0/extension.wit`). The host implements the world's imports. Host calls into the guest through `export`ed functions. Everything crossing the boundary is WIT data; there are no shared handles except WIT `resource`s (`worktree`, `project`, `key-value-store`, `http-response-stream`), which the host backs with `Arc<dyn Delegate>` objects (`since_v0_8_0.rs:38-60`; the `bindgen!` `with:` map).
- The guest is single-threaded and long-lived: one `Store<WasmState>` per extension, one instance. The host talks to it by queueing boxed closures on an unbounded channel; a single tokio task drains them one at a time (`wasm_host.rs:~707-722` `extension_task`, `WasmExtension::call` at `:912`). **Calls into one extension are therefore strictly serialized**; a slow call blocks all later calls to that extension.
- Guest-to-host calls that need the GPUI main thread use `WasmState::on_main_thread` (`wasm_host.rs:945`), a channel to a foreground task owned by `WasmHost` (`main_thread_message_tx`, `:54-55`, spawned in `WasmHost::new` at `:612`). Example: `get_settings` (`since_v0_8_0.rs:952`).

### 1.3 Version selection

- Guest declares its API version in the `zed:api-version` custom section; the host parses it with `wasmparser` (`wasm_host.rs:819 parse_wasm_extension_version`).
- `wit.rs:108 Extension::instantiate_async` dispatches on the version to one of 10 generated bindings (`since_v0_0_1` ... `since_v0_8_0`), each with its own `MIN_VERSION`/`MAX_VERSION` and its own `linker()` (`wit.rs:43 new_linker` adds WASI p2 async + that version's host imports). Older bindings convert to a common set of host types so the rest of Zed sees one interface.
- Supported range: `wit.rs:60 wasm_api_version_range`. Dev/Nightly allow up to `latest::MAX_VERSION` (0.8.0); Stable/Preview are capped at `since_v0_6_0::MAX_VERSION`. `authorize_access_to_unreleased_wasm_api_version` (`wit.rs:75`) rejects unreleased versions on Stable/Preview. So 0.8.0 is currently an unreleased staging version.
- `extension_host.rs:135 is_version_compatible` also checks `schema_version` (manifest schema; `SchemaVersion::CURRENT = 1`, `extension_manifest.rs:56`) and the extension's recorded `wasm_api_version` for the registry.
- Breaking-change batching policy: `crates/extension_api/PENDING_CHANGES.md` ("list of pending changes ... that require a breaking change").

### 1.4 WIT interfaces in v0.8.0 (the latest)

World `extension` (`wit/since_v0.8.0/extension.wit`) imports the interfaces `context-server`, `dap`, `github`, `http-client`, `platform`, `process`, `nodejs`, plus top-level functions. `common`, `lsp`, `slash-command` are used for types.

**Imports (guest calls host):**

| Import | Signature / notes | Gated? |
|---|---|---|
| `get-settings` | `(option<settings-location>, category: string, key: option<string>) -> result<string,string>`; JSON of `language`, `lsp`, `context_servers`, `project` settings (`since_v0_8_0.rs:952`) | No |
| `download-file` | `(url, file-path, downloaded-file-type) -> result<_,string>`; writes inside the extension work dir; types bzip2, bzip2-tar, gzip, gzip-tar, zip, uncompressed (`:1068`) | `download_file` capability (`:1076`) |
| `make-file-executable` | `(filepath) -> result` (no-op on Windows); path resolved through `writeable_path_from_extension` (`:1162`) | Path confined to work dir |
| `set-language-server-installation-status` | `(name, status)` (`:1047`) | No |
| `process.run-command` | `record command {command, args, env}` -> `output {status: option<s32>, stdout, stderr}`; blocks until exit, collects all output (`:899`) | `process:exec` (`:906`) |
| `nodejs.*` | `node-binary-path`, `npm-package-latest-version`, `npm-package-installed-version`, `npm-install-package` (`:757-807`) | Only `npm-install-package` (`:797`, `npm:install`) |
| `http-client.fetch` / `fetch-stream` | `http-request {method, url, headers, body, redirect-policy}`; response body fully buffered, or `http-response-stream.next-chunk` (`:632-690`). 4xx/5xx become errors (`:640-642`) | **No capability check** |
| `github.latest-github-release`, `github-release-by-tag-name` | (`:830-867`) | No |
| `platform.current-platform` | `tuple<os, architecture>` | No |
| `dap.resolve-tcp-template` | (`:927`) | No |
| `worktree` resource | `id`, `root-path`, `read-text-file(path)` (relative path, `RelPath`), `which(binary)`, `shell-env` (`:579-627`) | Only available as a borrow passed into guest exports |
| `project` resource | `worktree-ids` | same |
| `key-value-store` resource | `insert(key, value)` | same |

**Exports (host calls guest):**

`init-extension`; `language-server-command`, `language-server-initialization-options`, `language-server-workspace-configuration`, `language-server-initialization-options-schema`, `language-server-workspace-configuration-schema`, `language-server-additional-initialization-options`, `language-server-additional-workspace-configuration`; `labels-for-completions`, `labels-for-symbols` (return `code-label` with spans); `complete-slash-command-argument`, `run-slash-command`; `context-server-command`, `context-server-configuration`; `suggest-docs-packages`, `index-docs`; DAP: `get-dap-binary`, `dap-request-kind`, `dap-config-to-scenario`, `dap-locator-create-scenario`, `run-dap-locator`.

Observations relevant to UI:

- Output types are all *data*: commands, strings of JSON, completion labels, slash command text + sections. There is no export that returns anything renderable except `code-label` (syntax-highlighted label spans) and slash command output sections.
- JSON-in-string is already the house style for open-ended data (`get-settings`, initialization options, schemas), because WIT cannot express recursive types or arbitrary JSON.
- `language_model_providers` exists in the manifest and has a proxy (`extension_host_proxy.rs:419`, `crates/language_models/src/extension.rs`) but has no WIT surface in 0.8.0 (grep of `*.wit` finds nothing). `agent_servers` extensions are deprecated in favor of the ACP Registry (`docs/src/extensions/agent-servers.md`).

### 1.5 Manifest (`extension.toml`)

Struct: `ExtensionManifest` (`crates/extension/src/extension_manifest.rs:88`), parsed with `toml::from_str` in `ExtensionManifest::load`. Legacy `extension.json` (schema v0) is still read.

| Field | Type | Notes |
|---|---|---|
| `id`, `name`, `version`, `schema_version` | string/int | `schema_version` default 0 for legacy, current 1 |
| `description`, `repository`, `authors` | optional | CLI validation requires non-empty description longer than the name (`extension_cli/src/main.rs:265`) |
| `lib` | `{kind = "Rust", version}` | `kind` is auto-set if `Cargo.toml` exists (`populate_defaults`) |
| `themes`, `icon_themes`, `languages` | `Vec<RelPathBuf>` | auto-discovered from `themes/*.json`, `icon_themes/`, `languages/*/config.toml` |
| `grammars` | map name -> `{repository, rev|commit, path?}` | Tree-sitter grammars, built to wasm |
| `language_servers` | map name -> `{language(s), language_ids, code_action_kinds, opt_in_languages}` | |
| `context_servers` | map id -> `{}` | MCP servers |
| `slash_commands` | map name -> `{description, requires_argument}` | |
| `snippets` | path or list of paths | |
| `capabilities` | `Vec<ExtensionCapability>` (`:120`) | `[[capabilities]]` array of tables, tagged by `kind` |
| `debug_adapters` | map -> `{schema_path?}` | defaults to `debug_adapter_schemas/<name>.json` |
| `debug_locators` | map -> `{}` | |
| `language_model_providers` | map -> `{name, icon?}` | |

`manifest.provides()` derives the `ExtensionProvides` set used by the registry; `remote_load()` decides whether the extension is synced to remote servers (only if language servers/debug adapters/locators exist).

Example of a capability list as written in an extension (`extensions/test-extension/extension.toml:17`):

```toml
[[capabilities]]
kind = "process:exec"
command = "echo"
args = ["**"]
```

### 1.6 Lifecycle

1. **Build/package** (`zed-extension` CLI, `extension_cli/src/main.rs`): `ExtensionManifest::load` -> `ExtensionBuilder::compile_extension` (cargo build wasm, grammars, writes `extension.wasm`) -> validate -> copy resources -> tar.gz archive + `manifest.json` for the registry (including `wasm_api_version`, `:172`).
2. **Install**: `ExtensionStore::install_extension` / `install_or_upgrade_extension_at_endpoint` (`extension_host.rs:851, :861`) downloads the tarball from the registry API over HTTP and unpacks into `<data>/extensions/installed/<id>` via a `staging` dir (dirs initialised in `ExtensionStore::new`, `:369-430`; sibling dirs `installed`, `staging`, `work`). Dev extensions: `install_dev_extension` (`:1119`) compiles from a local dir; `rebuild_dev_extension` (`:1220`).
3. **Index**: `rebuild_extension_index` (`:1754`) builds `ExtensionIndex` of themes, icon themes, languages, grammars, manifests.
4. **Load**: `extensions_updated` (`:1263`) diffs old vs new index, unregisters removed extensions, registers themes/languages/snippets, then for each manifest with `lib.kind` calls `WasmExtension::load` (`:1667` -> `wasm_host.rs:859` -> `WasmHost::load_extension` `:643`):
   - read bytes of `extension.wasm`;
   - on the background executor: `parse_wasm_extension_version` and `Component::from_binary` (compile; incremental compilation cache `IncrementalCompilationCache`, `:1008`);
   - on tokio: `build_wasi_ctx`, `Store::new`, `Extension::instantiate_async`, `call_init_extension`, spawn the per-extension message-loop task.
5. **Register**: after load, `ExtensionHostProxy` fans out to registered handlers: `register_language_server`, `register_context_server`, `register_debug_adapter`, `register_debug_locator` (`extension_host.rs:1700-1736`). Proxy trait objects (`extension_host_proxy.rs:103-419`) are installed by other crates (language, project, debugger, language_models, theme). **This proxy pattern is exactly where a `ui` proxy would plug in.**
6. **Uninstall/unload**: unregister via proxy, drop `WasmExtension` (its `Drop` at `wasm_host.rs:72` stops the task).
7. **Remote**: `HeadlessExtensionStore` loads the subset needed on remote servers (language servers etc.) with its own `WasmHost` (`headless_host.rs:84`).

### 1.7 Wasmtime setup

`wasm_host.rs:557 wasm_engine` (one process-global engine):

- `wasm_component_model(true)`; `enable_incremental_compilation(cache_store())`.
- `epoch_interruption(true)`; a detached task bumps the epoch every 100 ms (`EPOCH_INTERVAL`, ~`:589`). Each `Store` gets `set_epoch_deadline(1)` + `epoch_deadline_async_yield_and_update(1)` (`:686-687`) so long guest computation yields to the executor every tick. This is **cooperative yielding only**: it does not trap or time out a runaway guest.
- **No fuel metering, no memory/table limits (`ResourceLimiter`), no wall-clock call timeout** are configured anywhere in `extension_host`.

WASI context (`wasm_host.rs:742 build_wasi_ctx`):

- Creates `<data>/extensions/work/<id>/`.
- `inherit_stdio()`, env vars `PWD=<work dir>` and `RUST_BACKTRACE=full` only (`:755-757`).
- Two preopens of the work dir with `FsPerms::ReadWrite`: at `"."` and at its absolute path (`:759-760`). No other directory is preopened, so direct WASI filesystem access is confined to the work dir. No `inherit_network` or socket configuration appears (wasmtime-wasi denies sockets unless enabled).
- `register_extension!` overrides `chdir` to return `NOTSUP` so guests cannot change cwd (`extension_api.rs:~293-322`).
- Host-side file writes (download, make executable) go through `WasmHost::writeable_path_from_extension` (`:765`), which lexically normalizes, canonicalizes the nearest existing ancestor (symlink resolution), and requires the result to start with `<work dir>/<id>` (tested by `test_writeable_path_rejects_escape_attempts`, `:1058`).
- HTTP, GitHub, npm, and process execution run in the *host* process via `WasmHost`'s `http_client`, `node_runtime`, and `util::command::new_command`, not through WASI.

---

## 2. Current permission / capability model (exact)

### 2.1 Data types

- `crates/extension/src/capabilities.rs:14`

```rust
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum ExtensionCapability {
    #[serde(rename = "process:exec")] ProcessExec(ProcessExecCapability),
    DownloadFile(DownloadFileCapability),            // kind = "download_file"
    #[serde(rename = "npm:install")] NpmInstallPackage(NpmInstallPackageCapability),
}
```

- `ProcessExecCapability { command: String, args: Vec<String> }` (`capabilities/process_exec_capability.rs`). `allows()` semantics: command equals or `"*"`; walk `args` pairwise: `"**"` -> allow the rest; `"*"` matches exactly one arg; literal must be equal; a pattern shorter than the request fails; so `["*"]` allows exactly one argument and `["test","**"]` allows `test` with any tail.
- `DownloadFileCapability { host: String, path: Vec<String> }`. `allows(url)`: host equal or `"*"`; path segments compared with the same `*`/`**` rules. Scheme and port are **not** checked.
- `NpmInstallPackageCapability { package: String }`: `"*"` or exact name. No version constraint.
- Settings-side mirror: `ExtensionCapabilityContent` (`crates/settings_content/src/extension.rs:45`) with JSON-schema derivation, converted in `extension_settings.rs:40-63`.

### 2.2 Two independent allowlists

| Layer | Source | Where it lives | Applies to |
|---|---|---|---|
| Manifest | `[[capabilities]]` in the extension's `extension.toml` | `ExtensionManifest.capabilities` (`extension_manifest.rs:120`) | **Only `process:exec`** is checked against the manifest (`ExtensionManifest::allow_exec`, `:168`) |
| User/host | `granted_extension_capabilities` setting | `ExtensionSettings.granted_capabilities` (`extension_settings.rs:21`), default in `assets/settings/default.json:2403` | `process:exec`, `download_file`, `npm:install` |

Default grants (all wildcard): `process:exec * **`, `download_file * **`, `npm:install *`. Docs: `docs/src/extensions/capabilities.md`, `docs/src/reference/all-settings.md:416-430`. An empty list denies everything; `None` is treated as empty (`unwrap_or_default`, `extension_settings.rs:47-49`).

### 2.3 Enforcement

`CapabilityGranter` (`crates/extension_host/src/capability_granter.rs:7`) is constructed per `WasmState` (`wasm_host.rs:678`) with a **clone of `WasmHost.granted_capabilities`**, itself a snapshot of the setting taken in `WasmHost::new` (`wasm_host.rs:629`, field `:55`). Consequences: settings changes are not observed by already-created hosts; there is no per-extension grant, only a global one.

- `grant_exec(command, args)` (`:23`): first `manifest.allow_exec` (must be declared in the extension's own manifest), then must also match at least one user grant. Error messages: `capability for process:exec ... was not listed in the extension manifest` / `... is not granted by the extension host`.
- `grant_download_file(&Url)` (`:49`): user grant only; **the manifest is not consulted** (an extension need not declare `download_file`).
- `grant_npm_install_package(name)` (`:67`): user grant only.

Call sites: `process::Host::run_command` (`since_v0_8_0.rs:906`), `ExtensionImports::download_file` (`:1076`), `nodejs::Host::npm_install_package` (`:797`). The same call sites exist in older bindings.

### 2.4 Gaps in the current model (as observed)

1. `http-client.fetch`/`fetch-stream` (`since_v0_8_0.rs:632-666`) perform arbitrary HTTP requests with **no capability check**; `download_file` is gated but plain fetch is not. Redirect policy is under guest control.
2. `github` releases, `get-settings`, `worktree.read-text-file`, `worktree.shell-env`, `worktree.which`, `npm-package-*-version`, `node-binary-path` are ungated (the worktree handle is only passed by the host on specific calls, which is the implicit scope).
3. `process:exec` runs with the *host's* privileges, no cwd control, no env filtering (`.envs(command.env)` adds to the inherited environment), no timeout, no output size limit, no stdin, no streaming (`since_v0_8_0.rs:908-914`). It matches the program string literally, not a resolved path.
4. `download_file` ignores scheme/port, has no size cap, no checksum, and decompresses tar variants fully into memory (`:1111-1155`).
5. No UI for consent: `extensions_ui` does not list requested capabilities; grants are global JSON settings.
6. No resource limits on wasm (memory, fuel, wall-time).
7. Manifest declarations are not enforced for `download_file` and `npm:install`, and the manifest is not shown to the user at install.

---

## 3. What extensions can do today / hard limitations

**Can do:** provide themes, icon themes, snippets, Tree-sitter grammars and language configs; run/launch language servers (with download, npm install, settings access); MCP context servers (`context-server-command`); slash commands for the Assistant (text + sections); `/docs` indexing; DAP adapters and locators; label formatting for completions/symbols; language model providers (manifest + registry proxy).

**Cannot do (hard limits):**

- **No UI of any kind.** Not a panel, pane item, modal, status bar item, toolbar item, context-menu entry, notification, command-palette command, or gutter decoration. The only visual influence is `code-label` spans and slash-command sections.
- No registering **actions** or **keybindings** (`Action`s are compile-time registered via `actions!`/derive in Rust, `crates/gpui/src/action.rs:117`).
- No access to editor buffers, selections, diagnostics, or project tree beyond `read-text-file`, worktree id/root, `which`, `shell-env` (and only when the host passes a worktree).
- No event subscriptions (buffer saved, active item changed, etc.); the guest is purely request/response, host-driven.
- No timers, background work, or guest-initiated callbacks; the guest runs only inside calls from the host.
- No sockets, no long-lived child processes, no streaming stdout (only `run-command`, which buffers).
- Calls are serialized per extension (§1.2), so any UI built on today's mechanism would stall behind slow calls.
- Settings: extensions cannot contribute settings schema except LSP/context-server option schemas.
- `lib.kind` is `Rust` only (`ExtensionLibraryKind` in `extension_manifest.rs`); the SDK is Rust-only in practice, though any language emitting a WIT component could work.

---

## 4. Relevant GPUI and workspace extension points

### 4.1 GPUI essentials

- `Render` (`crates/gpui/src/element.rs:164`): `fn render(&mut self, window, cx: &mut Context<Self>) -> impl IntoElement`. `Entity<T: Render>` is a "view".
- `RenderOnce` (`:180`): stateless components, `#[derive(IntoElement)]`.
- `Element` (`:52`) with `request_layout`/`prepaint`/`paint` and optional `id()`; `IntoElement` (`:146`), `AnyElement`, `ParentElement` (`:189`: `.child`, `.children`), `Styled` (`styled.rs:22`: tailwind-like), `div()` (`elements/div.rs:1844`), `uniform_list` (`elements/uniform_list.rs:22`), `list` (`elements/list.rs:24`), `img`, `svg`, `canvas` (`elements/canvas.rs:10`).
- `ui` crate components (`crates/ui/src/components`): `Button`, `IconButton`, `Label`, `Icon`, `Divider`, `Tab`/`TabBar`, `List`, `TreeViewItem`, `Disclosure`, `ContextMenu`, `PopoverMenu`, `RightClickMenu`, `Tooltip`, `Modal`, `Toggle`, `DropdownMenu`, `DataTable`, `Scrollbar`, `Banner`, `Callout`, `Chip`, `Indicator`, `Progress`, `KeybindingHint`, `h_flex()`/`v_flex()` (`stack.rs:7,13`). Text input: `ui_input::InputField` wrapping a single-line `Editor` (`crates/ui_input/src/input_field.rs:21`). `markdown` renders Markdown (`crates/markdown/src/markdown.rs:493`). `picker` is the fuzzy-list modal pattern.
- Theme: `ThemeColors` (`crates/theme/src/styles/colors.rs:15`, ~150 named `Hsla` fields: `border`, `border_variant`, `border_focused`, `text`, `element_background`, ...), `ActiveTheme::theme()`; UI density in `theme/src/ui_density.rs`.
- Actions: `trait Action` (`gpui/src/action.rs:117`) with `build(serde_json::Value)` and optional JSON schema; `App::build_action(name, params)` (`app.rs:2784`, `action.rs:351`) constructs any registered action from its **string name + JSON**, which is how the keymap works. `App::all_action_names` (`:2794`). `App::bind_keys` (`:2732`), `Window::dispatch_action`, `is_action_available` (`app.rs:2886`).
- Focus: `FocusHandle`, `Focusable`, `KeyContext`.
- Clipboard: `App::read_from_clipboard` (`app.rs:1834`), `write_to_clipboard` (`:1868`).
- Platforms: `gpui_web` compiles **GPUI itself** to `wasm32-unknown-unknown` for the browser (`crates/gpui_web/Cargo.toml`, deps `wasm-bindgen`, `gpui_wgpu`). This is host-side GPUI running in a browser, not a way to run GPUI inside a sandboxed guest. In this proposal the guest does *not* link GPUI. (No `embedded_gpui` crate exists in this tree; it is referenced only in the thread.)

### 4.2 Workspace surfaces (`crates/workspace/src`)

| Surface | Trait / API | Where registered |
|---|---|---|
| Dock panel (left/right/bottom) | `trait Panel: Focusable + EventEmitter<PanelEvent> + Render` (`dock.rs:36`): `persistent_name`, `panel_key`, `position`, `set_position`, `default_size`, `icon`, `icon_tooltip`, `toggle_action`, `activation_priority`, `starts_open`, `enabled`, `set_active`, `hide_button_setting`, ... `DockPosition` at `dock.rs:323`. `Workspace::add_panel` (`workspace.rs:2837`). `persistent_name`/`panel_key`/`icon_tooltip` return `&'static str` and `toggle_action()` returns an `Action`; dynamic panels need interned strings and a generic action (§5.7). | `crates/zed/src/zed.rs:788-845` (`add_panel_when_ready`) |
| Pane item / tab | `trait Item: Focusable + EventEmitter<Self::Event> + Render` (`item.rs:170`): `tab_content_text`, `tab_icon`, `tab_tooltip_text`, `to_item_events`, `deactivated`, `navigate`, ...; `Workspace::add_item_to_active_pane` (`workspace.rs:5055`), `add_item` (`:5074`). Persistence: `SerializableItem` (`item.rs:409`) + `register_serializable_item` (`workspace.rs:1306`). | |
| Status bar | `trait StatusItemView: Render` (`status_bar.rs:42`): `set_active_pane_item`, `hide_button_setting`; `StatusBar::add_left_item` (`:356`) / `add_right_item` (`:421`). | `zed.rs` workspace init |
| Toolbar (per pane) | `trait ToolbarItemView` (`toolbar.rs:24`) returning `ToolbarItemLocation`; `Toolbar::add_item` (`toolbar.rs:184`); used at `zed.rs:1466-1470`. | |
| Modal | `trait ModalView: ManagedView` (`modal_layer.rs:49`), `ManagedView = Focusable + EventEmitter<DismissEvent> + Render` (`gpui/src/window.rs:726`); `Workspace::toggle_modal` (`workspace.rs:8747`), `ModalLayer::toggle_modal` (`modal_layer.rs:152`). | |
| Toasts / notifications | `Workspace::show_toast` (`notifications.rs:196`), `show_notification` (`:97`). | |
| Workspace actions | `Workspace::register_action` (`workspace.rs:8698`), `register_action_renderer` (`:8712`). | |
| Context menus | `ui::ContextMenu`, `PopoverMenu`, `RightClickMenu`. Editor/project-panel menus are built imperatively in each crate (no central contribution registry). | |
| Settings | `Settings` trait + `RegisterSetting` derive (`extension_settings.rs:6-8`), schema in `settings_content`. Settings UI is page-data driven (`crates/settings_ui/src/page_data.rs`). | |

### 4.3 Existing declarative / JSON-driven pieces to reuse

- **Keymap**: JSON binding of key -> `[action_name, json_params]` built via `build_action` (`gpui/src/action.rs:351`). Context predicates are strings.
- **Theme**: JSON schema (`theme/src/schema.rs`) with named color tokens, already extension-provided.
- **Debug adapter / LSP option JSON schemas**: extensions provide JSON schemas; `crates/json_schema_store` serves them to the JSON language server. The same pattern can ship a UI-node JSON schema so authors get completion/validation.
- **Settings UI** (`settings_ui`): data-driven pages of typed controls, a miniature declarative UI compiled to native GPUI.
- **Markdown** and `html_to_markdown`: rich text from data.
- **Slash command output sections**: structured label/range data rendered by the host.
- **Component preview** (`RegisterComponent`) catalog: shows the primitive set.

---

## 5. Proposal: declarative UI extensions

### 5.1 Principles

1. **Host renders, guest describes.** The guest never touches GPUI. It returns JSON "UI trees" and receives JSON events. Validation, theming, layout, accessibility, focus, and security checks are host-side.
2. **Retained, not immediate.** GPUI `render()` is synchronous on the main thread; guest calls are async and serialized per extension (§1.2). The host keeps the **last accepted tree** per surface and renders from that cache. The guest produces new trees only in response to events/ticks, never during paint.
3. **Declare statically, instantiate dynamically.** Surfaces (panel id, icon, default position, status bar slot, actions) are declared in `extension.toml` so Zed can lay them out and show buttons before the wasm loads (lazy activation). Contents are dynamic JSON.
4. **Deny by default.** New capability kinds are not covered by the default wildcard grants; the user must consent (§5.8).
5. **Additive and versioned.** New WIT version, new optional manifest tables; extensions that do not use UI are unaffected.

### 5.2 Where it plugs in

- **WIT**: new `since_v0.9.0` directory (copy of 0.8.0 plus new interfaces `ui`, `fs`, `net`, `proc`, `clipboard`, `notify`). Since 0.8.0 is itself unreleased (Stable max is 0.6.0, `wit.rs:66`), an alternative is landing everything in 0.8.0; recommended is a new 0.9.0 so the 0.8.0 shape other work depends on stays stable. Steps mirror past versions: add `wit/since_v0.9.0/`, bump `zed_extension_api` to 0.9.0, add `wasm_host/wit/since_v0_9_0.rs` with `bindgen!`, `MIN/MAX_VERSION`, `linker()`, make it `latest` in `wit.rs:24`, add a branch in `instantiate_async` (`wit.rs:118`), extend `wasm_api_version_range` (`:60`).
- **Manifest**: new optional tables on `ExtensionManifest` (`extension_manifest.rs:88`): `[ui.*]` and richer `[[capabilities]]`. All `#[serde(default)]` so older manifests stay valid; bump `SchemaVersion::CURRENT` to 2 so old Zed refuses extensions needing the new semantics (`extension_host.rs:135` already gates this).
- **Host**: a new crate `crates/extension_ui` (library root `extension_ui.rs`) holding the JSON schema types, validator, differ, and the GPUI builder, plus entity types `ExtensionPanel`, `ExtensionPaneItem`, `ExtensionStatusItem`, `ExtensionModal`, each wrapping an `ExtensionSurfaceState`. Registration goes through a new `ExtensionUiProxy` in `extension_host_proxy.rs` (same shape as `ExtensionContextServerProxy`, `:336`), called from `extension_host.rs` next to `register_context_server` (`:1709`). The `zed` crate (workspace init, `zed.rs:788+`) installs the proxy implementation, so `extension_host` need not depend on `workspace`.
- **Guest SDK**: `zed_extension_api::ui` module (builders that emit the JSON, typed event enums), and new `Extension` trait methods with default no-op implementations (like the existing ones, `extension_api.rs:69+`).

### 5.3 Manifest additions

```toml
id = "sqlite-viewer"
name = "SQLite Viewer"
version = "0.1.0"
schema_version = 2
description = "Browse SQLite databases in a panel and pane"
lib = { kind = "Rust", version = "0.9.0" }

# --- UI surfaces: declared statically, filled dynamically ------------------

[ui.panels.sqlite]
title        = "SQLite"
icon         = "icons/db.svg"        # path in extension, or a built-in icon: icon = "builtin:database"
position     = "left"                # left | right | bottom (user may move)
valid_positions = ["left", "right", "bottom"]
default_size = 280                   # px
activation_priority = 80
starts_open  = false
toggle_action = "sqlite::TogglePanel" # namespaced; auto-generated if omitted

[ui.items.table_view]                # a pane item (tab)
title        = "Table"
icon         = "builtin:table"
singleton    = false                  # multiple tabs allowed (keyed by instance id)
serializable = true                   # host persists instance key + state blob
open_for_extensions = ["sqlite", "db"] # open files with these extensions in this item
close_confirmation = "if_dirty"      # never | if_dirty | always

[ui.status_items.sqlite_status]
alignment    = "right"               # left | right
priority     = 40
tooltip      = "Active SQLite connection"
hideable     = true                  # host adds "Hide Button" (cf. StatusItemView::hide_button_setting)

[ui.toolbar_items.sqlite_toolbar]
applies_to   = { language = ["SQL"], path_glob = ["**/*.sqlite", "**/*.db"] }
location     = "secondary_left"      # maps to ToolbarItemLocation

[ui.modals.connect]
title        = "Connect to database"
dismissable  = true
width        = "medium"              # small | medium | large

[ui.context_menus.editor_selection]  # contribute entries to an existing host menu
target       = "editor.context"       # closed set of host-defined menu ids
entries      = [
  { id = "run_selection", label = "Run selection as SQL", action = "sqlite::RunSelection", when = "editorLangId == 'sql'" },
]

[ui.decorations.gutter_status]       # gutter / inline
kind         = "gutter_icon"         # gutter_icon | inline_text | line_highlight
applies_to   = { language = ["SQL"] }

# --- Actions & keybindings -------------------------------------------------

[[ui.actions]]
name        = "sqlite::RunSelection"          # must be prefixed by "<namespace>::" declared by the extension
title       = "SQLite: Run selection"
palette     = true                             # visible in the command palette
params_schema = "schemas/run_selection.json"   # optional JSON schema for action args

[[ui.keybindings]]
key     = "cmd-alt-r"
action  = "sqlite::RunSelection"
context = "Editor && language == sql"

# --- Permissions (see 5.8 for full grammar) --------------------------------

[[capabilities]]
kind = "ui"
surfaces = ["panel", "item", "status_item", "toolbar", "modal", "context_menu", "decoration"]

[[capabilities]]
kind = "fs"
op   = ["read", "stat", "watch"]
scope = "worktree"
glob = ["**/*.sqlite", "**/*.db"]
```

Validation (extension_cli `validate_extension_manifest`, `main.rs:265`, plus host load): every `[ui.*]` surface requires a matching `ui` capability surface; action names are namespaced; icons exist; unknown fields are rejected in the `ui` tables (`deny_unknown_fields`).

### 5.4 WIT sketch

```wit
// wit/since_v0.9.0/ui.wit
interface ui {
    type surface-id = string;          // "panel:sqlite", "item:table_view", ... (manifest ids)
    type instance-id = string;         // for multi-instance items/modals
    type node-key = string;

    /// Versioned UI tree, JSON conforming to zed-ui/1 (see 5.5).
    /// A string because WIT has no recursive types; same convention as get-settings.
    type ui-tree-json = string;

    record render-reply {
        /// Monotonic per surface instance; host drops replies older than the last applied.
        revision: u64,
        /// Full tree, or...
        tree: option<ui-tree-json>,
        /// ...a JSON patch list (see 5.6).
        patches: option<string>,
    }

    variant ui-event {
        click(click-event),
        input(input-event),        // text change (debounced by host per node.debounce_ms)
        submit(submit-event),      // enter pressed
        toggle(toggle-event),      // checkbox/switch
        select(select-event),      // list/tree/tabs/dropdown selection
        expand(expand-event),      // tree node open/close
        scroll-end(scroll-end-event),  // virtualized list asked for more rows
        context-menu(menu-event),
        key(key-event),            // only for nodes with `capture_keys`
        action(action-event),      // a manifest-declared action fired (keybinding, palette, menu)
        lifecycle(lifecycle-event),
        timer(u64),
        io-ready(u64),             // a process/websocket/watch resource has data
    }

    record click-event  { node: node-key, handler: string, button: u8, modifiers: u8 }
    record input-event  { node: node-key, handler: string, value: string, cursor: u32 }
    record submit-event { node: node-key, handler: string, value: string }
    record toggle-event { node: node-key, handler: string, checked: bool }
    record select-event { node: node-key, handler: string, value: string, index: option<u32> }
    record expand-event { node: node-key, handler: string, expanded: bool }
    record scroll-end-event { node: node-key, handler: string, offset: u64 }
    record menu-event   { node: node-key, handler: string, item-id: string }
    record key-event    { node: node-key, handler: string, keystroke: string }
    record action-event { name: string, params-json: option<string> }
    record lifecycle-event { kind: lifecycle-kind, size: option<tuple<u32,u32>> }
    enum lifecycle-kind { opened, closed, shown, hidden, focused, blurred, resized, theme-changed }

    // Host commands the guest can issue (validated and permission-checked).
    variant ui-command {
        open-item(open-item-args),
        close-item(instance-id),
        show-modal(show-modal-args),
        close-modal(instance-id),
        set-panel-badge(option<string>),
        set-status-text(set-status-args),
        toast(toast-args),
        copy-to-clipboard(string),            // requires clipboard:write
        dispatch-action(dispatch-action-args),// extension-namespaced actions or an allowlisted host set
        request-render(surface-id),
        set-decorations(set-decorations-args),
    }

    record ui-response {
        render: option<render-reply>,
        commands: list<ui-command>,
    }

    // Guest exports
    export surface-open: func(surface: surface-id, instance: option<instance-id>, args-json: option<string>) -> result<render-reply, string>;
    export surface-event: func(surface: surface-id, instance: option<instance-id>, ev: ui-event) -> result<ui-response, string>;
    export surface-close: func(surface: surface-id, instance: option<instance-id>);
    export serialize-instance: func(surface: surface-id, instance: instance-id) -> option<string>;
    export deserialize-instance: func(surface: surface-id, instance: instance-id, blob: string) -> result<render-reply, string>;

    // Guest imports
    /// Ask the host to re-run the surface's render path after async work completes.
    request-render: func(surface: surface-id, instance: option<instance-id>);
    /// Schedule a `timer` event; capped per extension (min interval, max count).
    request-timer: func(id: u64, millis: u32);
    cancel-timer: func(id: u64);
    /// Push a new tree from an async continuation (e.g. after a fetch completes).
    push-render: func(surface: surface-id, instance: option<instance-id>, reply: render-reply) -> result<_, string>;

    record theme-info { name: string, appearance: appearance, ui-density: string, font-family: string, font-size: f32 }
    enum appearance { light, dark }
    current-theme: func() -> theme-info;
}
```

Why not typed recursive WIT: WIT disallows recursive variants/records. Alternatives considered: (a) JSON string (chosen: matches `get-settings`/schema conventions, forward compatible, easy to JSON-Schema-validate and fuzz); (b) a flat node table `list<node>` with child indexes (typed, but versioning is painful). The host parses with `serde_json` into a Rust enum with `#[serde(tag = "type")]` and rejects unknown node types per schema version.

World changes: `import ui; import fs; import net; import proc; import clipboard; import notify;` plus the exports above. Because guest calls are serialized (§1.2), long operations must be modeled as **guest-started operation handles** (`proc.spawn` returns a resource, `net.fetch` returns a streaming resource) whose readiness the host reports as `io-ready`/`timer` events, so no guest call blocks the queue for long. The host must also add a call timeout around every `WasmExtension::call` (today there is none) and give UI events priority over background calls (see R1).

### 5.5 JSON UI schema sketch (`zed-ui/1`)

Envelope:

```json
{
  "schema": "zed-ui/1",
  "root": { "type": "column", "key": "root", "children": [ ] }
}
```

Every node has: `type` (required), `key` (string, unique among siblings; stable identity for diffing, focus, scroll, animation), optional `visible` (bool, default true), `enabled` (bool), `tooltip` (string or node), `a11y` `{role, label, description}`, `test_id`. Unknown fields are errors, except those prefixed `x-`.

**Layout**

| type | key props |
|---|---|
| `row`, `column` (aliases `h_flex`, `v_flex`) | `gap` (token), `padding`, `align` (start/center/end/stretch/baseline), `justify` (start/center/end/between/around), `wrap`, `grow`, `shrink`, `basis`, `children` |
| `stack` | overlay children (z-order by index) |
| `grid` | `columns` (count or tracks), `gap`, `children` with optional `col_span`/`row_span` |
| `scroll` | `axis` (x/y/both), `child`, `scrollbar` (auto/always/never), `scroll_to` `{key, align}` |
| `spacer`, `divider` | `divider`: `orientation`, `variant` (default/muted) |
| `split` | resizable panes: `axis`, `children`, `sizes`, `min_sizes` |
| `container` | box with `background` token, `border` token/width, `radius` token, `elevation` (0-3) |

**Content**

| type | key props |
|---|---|
| `text` | `value`, `style` (`body`/`small`/`muted`/`heading1..3`/`code`/`label`), `color` token, `weight`, `italic`, `underline`, `strikethrough`, `truncate` (none/end/middle), `max_lines`, `selectable`, `wrap` |
| `rich_text` | `spans` `[{text, style?, color?, link?}]` |
| `markdown` | `source` (rendered by the host `markdown` crate; links go through `open_url` permission) |
| `code` | `value`, `language` (host Tree-sitter highlights), `line_numbers`, `wrap`, `selectable`, `highlights` ranges |
| `icon` | `name` (built-in `IconName` allowlist) or `path` (extension-relative SVG), `size` token, `color` token |
| `image` | `path` (extension-relative raster/SVG) or `data_uri` (size capped), `fit`, `width`, `height`, `alt` |
| `badge`, `chip`, `indicator`, `progress` (`value`, `max`, indeterminate), `spinner` | |
| `empty_state` | `icon`, `title`, `description`, `actions` |

**Input**

| type | key props / events |
|---|---|
| `button` | `label`, `icon`, `variant` (primary/subtle/ghost/danger), `size`, `disabled`, `loading`, `toggled`, `on_click` |
| `icon_button` | `icon`, `tooltip` (required), `on_click` |
| `input` | `value`, `placeholder`, `multiline`, `password`, `max_len`, `prefix_icon`, `validation` `{regex, message}`, `on_input` (`debounce_ms`), `on_submit`, `autofocus`. Backed by `ui_input::InputField`; `value` is *controlled*: the host keeps local edit state and only overwrites on a newer revision that changes `value`. |
| `checkbox`, `switch` | `checked`, `label`, `on_toggle` |
| `radio_group`, `dropdown`, `segmented` | `options` `[{value, label, icon?, description?}]`, `selected`, `on_select` |
| `slider` | `min`, `max`, `step`, `value`, `on_change` |
| `search_box` | `input` with fuzzy-filter affordances, `on_input` |
| `link` | `label`, `url` (`http(s)`/`zed://`, requires `open_url`) |

**Collections**

| type | key props / events |
|---|---|
| `list` | `items` (nodes) or `virtual: {count, row_height, window}` for lazily requested rows (`scroll-end` events); `selection` (none/single/multi), `selected_keys`, `on_select`, `on_activate`, `empty` node |
| `tree` | `nodes` `[{key, label, icon?, children?, expandable, expanded, badge?, actions?}]`; lazy-load via `on_expand` returning children in a later tree/patch; `selected`, `on_select`, `on_activate`, `on_context_menu`, `indent_guides` |
| `tabs` | `tabs` `[{key, label, icon?, closable, content}]`, `active`, `on_select`, `on_close` |
| `table` | `columns` `[{key, title, width, align, sortable}]`, `rows` or `virtual`, `sort`, `on_sort`, `on_select`, `resizable_columns` (maps to `ui::DataTable`) |
| `accordion` / `disclosure` | `title`, `expanded`, `on_toggle`, `child` |
| `menu`, `context_menu` | `items` `[{id, label, icon?, shortcut?, action?, disabled?, children?, separator?}]`; `context_menu` wraps a child and attaches to right-click (`ui::RightClickMenu`) |
| `popover`, `tooltip` | `anchor` child, `content` node, `placement` |

**Visualization (later phases; the thread asks for graph canvas and dashboards)**

`chart` (line/bar/area/pie with series data, host-drawn on `canvas`), `graph` (nodes/edges with auto layout, pan/zoom, `on_node_click`, bounded node count), `canvas2d` (a bounded retained display list of `rect/line/path/text`, not an immediate-mode API).

**Styling tokens** (no raw colors or pixels by default; everything maps to the active theme so theming is automatic):

- Color tokens map to `ThemeColors` fields (`crates/theme/src/styles/colors.rs:15`): `text`, `text.muted`, `text.placeholder`, `text.disabled`, `text.accent`, `border`, `border.variant`, `border.focused`, `border.selected`, `surface`, `panel`, `elevated_surface`, `element`, `element.hover`, `element.active`, `element.selected`, `ghost_element.*`, `status.error|warning|success|info|hint` (plus `.background`/`.border` variants), syntax tokens (`syntax.keyword`, ...), and a small `palette` of 8 accent hues resolved per theme. A `color` field is a token string or `{token, alpha}`.
- Spacing tokens: `none, xxs, xs, sm, md, lg, xl` mapped to `ui::DynamicSpacing` (respects UI density).
- Size tokens: `xs, sm, md, lg` for icon/button/text. Radius tokens `none, sm, md, lg, full`. Elevation 0-3.
- Fonts: `ui`, `buffer`, `mono` only; no arbitrary font families.
- Fixed sizes: `width`/`height`/`min_*`/`max_*` accept `{px: n}`, `{pct: n}`, `{fr: n}`, `"fill"`, `"hug"`, clamped by the host (max 4096 px, no negatives). No CSS, no raw RGB unless the `ui.raw_colors` option is granted (default off; host clamps contrast).

**Events -> guest callbacks.** An event prop is a small object: `"on_click": {"id": "run", "payload": {"table": "users"}}`. The guest receives `ui-event::click` with the node key and handler `id`; `payload` is echoed (size-capped). Handlers are *data*, never closures. The host can also run **local effects** without a guest round trip to stay responsive: `"on_click": {"local": "copy", "value": "..."}`, `{"local": "toggle", "target": "section1"}`, `{"local": "open_url", ...}` (permissioned), `{"local": "dispatch_action", "action": "..."}`. Local effects are a closed enum.

**Example tree**

```json
{
  "schema": "zed-ui/1",
  "root": {
    "type": "column", "key": "root", "gap": "sm", "padding": "md",
    "children": [
      { "type": "row", "key": "toolbar", "gap": "xs", "align": "center", "children": [
        { "type": "input", "key": "filter", "placeholder": "Filter tables",
          "value": "", "grow": 1, "on_input": { "id": "filter", "debounce_ms": 150 } },
        { "type": "icon_button", "key": "refresh", "icon": "builtin:rotate_cw",
          "tooltip": "Refresh", "on_click": { "id": "refresh" } }
      ]},
      { "type": "divider", "key": "d1" },
      { "type": "tree", "key": "tables", "grow": 1, "selection": "single",
        "nodes": [
          { "key": "main", "label": "main.sqlite", "icon": "builtin:database", "expanded": true,
            "children": [
              { "key": "t:users", "label": "users", "icon": "builtin:table", "badge": "1.2k" },
              { "key": "t:orders", "label": "orders", "icon": "builtin:table", "badge": "34k" }
            ]}
        ],
        "on_activate": { "id": "open_table" },
        "on_context_menu": { "id": "table_menu" } }
    ]
  }
}
```

### 5.6 State, updates, and diffing

- **Model**: the guest owns application state; the host owns *view state* that is not meaningful to the guest (scroll offsets, text cursor, focus, hover, and expanded state when the node is `uncontrolled`).
- **Revision numbers**: every `render-reply` carries a per-surface-instance `revision`. The host applies strictly increasing revisions and drops stale ones.
- **Full tree or patches**: the first reply is a full tree. Later replies may be a full tree or a patch list keyed by `key` paths: `replace(key, node)`, `insert(parent_key, index, node)`, `remove(key)`, `move(key, parent_key, index)`, `set(key, prop, value)`, `append(key, items)`. The host validates and, on any patch error, requests a full tree (a `resync` lifecycle event).
- **Diffing on the host**: reconcile new vs old by `key`. Unchanged subtrees are reused (keeping the `Entity` backing `input` editors, scroll handles, list state); changed leaves become property updates; one `cx.notify()` per applied batch. Elements get `ElementId` derived from the key path so GPUI's element-state tracking (focus, hover, scroll) survives re-renders.
- **Controlled inputs**: React-style. The host owns edit state while typing, sends debounced `input` events, and applies guest-provided `value` only if its revision is newer and it differs from the host's value, avoiding cursor jumps.
- **Limits**: max N trees per second per surface (e.g. 30, coalesced to the latest); max nodes (e.g. 20k), bytes (e.g. 4 MiB), depth (e.g. 64), string length. Over-limit replies are rejected, logged to the extension log, and surfaced once as a toast.
- **Streaming**: `push-render` from async continuations, `list.virtual`, and `append` patches with a host-side ring-buffer cap cover log tails and streaming output without resending full trees.
- **Persistence**: `serialize-instance`/`deserialize-instance` give a small opaque blob stored in the workspace DB (like `SerializableItem`, `item.rs:409`).

### 5.7 Surfaces, host-side mapping

| Surface | Host entity | Implements | Notes |
|---|---|---|---|
| Panel | `ExtensionPanel` | `Panel` (`dock.rs:36`) + `Render` | `panel_key`/`persistent_name`/`icon_tooltip` need `&'static str`: intern once per extension id (bounded by installed extensions) or relax the trait to `SharedString` (small upstream change preferred). `toggle_action` returns a generic `ToggleExtensionPanel { extension_id, panel_id }` action registered once. Registered through `Workspace::add_panel` (`workspace.rs:2837`) when the proxy fires. |
| Pane item / tab | `ExtensionPaneItem` | `Item` (`item.rs:170`), optionally `SerializableItem` | `tab_content_text` from the latest `title`; `tab_icon`; `to_item_events` for dirty/close; opened via `OpenExtensionItem { extension_id, item_id, instance, args }` or by file association. |
| Status bar item | `ExtensionStatusItem` | `StatusItemView` (`status_bar.rs:42`) | Content constrained to a `row` of `icon`/`text`/`badge`/`progress`; click opens a `popover` or dispatches an action. `hide_button_setting` returns `Some` so users can hide it. |
| Toolbar | `ExtensionToolbarItem` | `ToolbarItemView` (`toolbar.rs:24`) | `set_active_pane_item` evaluates the manifest `applies_to` filter on the host (no guest call) to return `ToolbarItemLocation::Hidden` or a location. |
| Modal | `ExtensionModal` | `ModalView` + `ManagedView` (`modal_layer.rs:49`, `window.rs:726`) | Opened via `ui-command::show-modal`; one at a time per `ModalLayer`; focus trap and Esc handled by host. |
| Context menu | contribution registry | new `MenuContributionRegistry` global | Closed set of host menu ids (`editor.context`, `project_panel.context`, `tab.context`, `git_panel.context`). Entries are manifest-declared actions with a `when` expression evaluated by the host against `KeyContext`. Menu code in each crate consults `registry.entries_for(menu_id, &context)`. |
| Gutter / inline decorations | `DecorationProvider` | new editor-side hook | Guest returns `set-decorations` lists: `{path, line, kind, icon, tooltip_text, action}`; host maps to editor highlight/gutter APIs. Phase 3; the guest never sees buffer text unless `editor:read` is granted. |
| Notifications | `Workspace::show_toast` / `show_notification` | | Rate-limited per extension; actions limited to extension actions. |
| Command palette | actions from `[[ui.actions]]` | `Action` | See below. |

**Actions and keybindings.** Rust actions are monomorphic types registered at startup. For extensions, add one host-defined action type:

```rust
#[derive(Clone, PartialEq, Deserialize, JsonSchema, Action)]
#[action(namespace = extension)]
pub struct InvokeExtensionAction { pub extension_id: Arc<str>, pub name: Arc<str>, pub params: Option<serde_json::Value> }
```

Manifest-declared `[[ui.actions]]` entries become command-palette items and keymap entries expressed as `["extension::InvokeExtensionAction", {extension_id, name, params}]`. This reuses `Action::build(serde_json::Value)` (`gpui/src/action.rs:117`) and the keymap loader, so no new dispatch machinery. A workspace handler forwards to `surface-event(action(...))`. `[[ui.keybindings]]` are loaded as an extension keymap layer with lower precedence than the user keymap, and may only reference the extension's own actions plus an allowlist of host actions (e.g. `workspace::ToggleLeftDock`); conflicts are shown in the keymap editor with provenance.

### 5.8 Permission model

#### 5.8.1 Structure

Same mechanism as today, generalized: capabilities are `[[capabilities]]` tagged by `kind` (compatible with `ExtensionCapability`, `capabilities.rs:14`), evaluated by `CapabilityGranter` (`capability_granter.rs:7`) as **manifest ∩ user grants**. Existing kinds keep their exact meaning and wire format. New kinds: `fs`, `net`, `proc`, `ui`, `clipboard`, `notify`, `editor`, `open_url`, `settings`. Unknown kinds in a manifest are a hard error on a Zed that does not know them (the `schema_version` bump), so older Zed never silently ignores a restriction.

Two behavior changes relative to today, for extensions with `schema_version >= 2`: (1) the manifest is always checked, not only for `process:exec`; (2) `http-client.fetch` is gated by `net`. Extensions on `schema_version = 1` keep current semantics (legacy mode) with the existing wildcard defaults so nothing breaks; the `fetch` gap is closed for them by deriving an implicit `net` allowlist from their `download_file` hosts and GitHub usage, with a deprecation warning (see §6).

Matching is *deny-wins*: grants may carry deny entries that override any allow.

#### 5.8.2 Filesystem: `kind = "fs"`

```toml
[[capabilities]]
kind   = "fs"
op     = ["read", "stat", "list"]          # read | stat | list | write | create | delete | rename | copy | watch | chmod_exec
scope  = "worktree"                        # see scopes below
glob   = ["src/**", "!**/.git/**", "**/*.sqlite"]   # relative to scope root; leading "!" = deny
follow_symlinks = "within_scope"           # never | within_scope | any
max_file_size = "64MiB"                    # per read/write
max_total_write = "512MiB"                 # per extension lifetime
max_watchers = 8
reason = "Open SQLite databases that live in the project"   # shown in consent UI
optional = false                           # optional capabilities can be declined without blocking install
```

Scopes (resolved by the host to canonical absolute roots at call time):

| `scope` | Root | Default ops allowed to request | Notes |
|---|---|---|---|
| `extension_work` | `<data>/extensions/work/<id>/` | all | Always readable/writable (today's WASI preopens, `wasm_host.rs:759-760`); an explicit entry is only needed to set quotas. |
| `extension_install` | `<data>/extensions/installed/<id>/` | read, stat, list | Extension's own shipped assets. Never writable. |
| `worktree` | each open worktree root of the project in which the surface is open | any | Resolved through the `worktree` delegate; honors `file_scan_exclusions`/`private_files` unless `include_private = true` is separately granted (default false); path is a `RelPath`, as `read-text-file` does today (`since_v0_8_0.rs:593`). |
| `project` | union of all worktrees of the project | same | |
| `open_buffers` | files currently open in editors | read | Unsaved buffer text is an `editor:read` concern, not `fs`. |
| `home` | user home | read/list by default; writes need `allow_write_home = true` and a non-empty glob | Credential locations are on a built-in deny list that cannot be granted (`~/.ssh`, `~/.gnupg`, `~/.aws`, `~/.config/gcloud`, browser profile dirs, keychains). |
| `temp` | per-extension subdirectory of the OS temp dir | all | Host creates and cleans it up. |
| `path` | explicit absolute directory, e.g. `path = "/var/log"` or `"$XDG_DATA_HOME/foo"` | per `op` | High-severity consent prompt; globs required; no write on system roots. |
| `user_selected` | paths chosen through a host file dialog (`prompt_for_open_path`, `workspace.rs:3471`) | read/write on that path only | Powerbox pattern: no up-front permission; grant is per path, lasts for the session, optionally remembered. |

Rules:

- Resolution: join (scope root, requested relative path), `normalize_lexically` (rejecting `..` escapes; the helper already used in `writeable_path_from_extension`, `wasm_host.rs:765`), canonicalize the nearest existing ancestor (existing pattern, `:783-805`), then check `starts_with(canonical scope root)` **after** symlink resolution unless `follow_symlinks = "any"`. `never` rejects any path traversing a symlink (`symlink_metadata` per component). TOCTOU mitigation: open with no-follow semantics and `openat`-relative handles where the platform allows; on Windows reject reparse points. Hardlinks into scope are out of the model and documented.
- Globs: `globset` syntax evaluated on the scope-relative, `/`-normalized path; `!` entries are denies. Case sensitivity follows the filesystem.
- `op` semantics: `read` bytes; `stat` metadata; `list` directory listing; `write` modify existing file; `create` new file/dir (no overwrite); `delete`; `rename`/`copy` need `create` (+`write`) on the destination and `delete` (rename) or `read` (copy) on the source; `watch` registers a watcher (events delivered as `io-ready`, only for paths matching the glob); `chmod_exec` replaces `make-file-executable` (still confined to `extension_work`).
- Quotas: per-call size limits, aggregate write quota, watcher count, open-handle count. Large I/O streams through a WIT `resource file-handle`.
- Remote projects: `worktree` scope resolves on the remote host via the existing remote delegates; `home`/`path` scopes are refused for remote worktrees unless the extension is loaded on that remote (`headless_host.rs`).
- Auditing: every denial is logged with extension id, op, resolved path, and the capability that would have been needed; the first denial surfaces a non-modal toast.

WIT sketch:

```wit
interface fs {
    enum scope { extension-work, extension-install, worktree, project, home, temp, user-selected }
    record path-ref { scope: scope, worktree-id: option<u64>, path: string }   // relative to scope root, '/'-separated
    record file-stat { size: u64, is-dir: bool, is-symlink: bool, modified-ms: option<u64>, readonly: bool }
    record dir-entry { name: string, is-dir: bool, is-symlink: bool }
    variant fs-error { not-granted(string), not-found, already-exists, is-dir, not-dir, too-large, outside-scope, io(string) }
    enum open-mode { read, write, read-write, create-new, append }
    enum pick-kind { file, directory, save-file }
    type watch-id = u64;

    resource file-handle {
        read: func(offset: u64, len: u32) -> result<list<u8>, fs-error>;
        write: func(offset: u64, data: list<u8>) -> result<u32, fs-error>;
        set-len: func(len: u64) -> result<_, fs-error>;
        close: func();
    }
    open: func(p: path-ref, mode: open-mode) -> result<file-handle, fs-error>;
    read-file: func(p: path-ref) -> result<list<u8>, fs-error>;
    read-text-file: func(p: path-ref) -> result<string, fs-error>;
    write-file: func(p: path-ref, data: list<u8>, create-only: bool) -> result<_, fs-error>;
    stat: func(p: path-ref) -> result<file-stat, fs-error>;
    list: func(p: path-ref) -> result<list<dir-entry>, fs-error>;
    create-dir: func(p: path-ref, recursive: bool) -> result<_, fs-error>;
    remove: func(p: path-ref, recursive: bool) -> result<_, fs-error>;
    rename: func(from: path-ref, to: path-ref) -> result<_, fs-error>;
    watch: func(p: path-ref, recursive: bool) -> result<watch-id, fs-error>;
    unwatch: func(id: watch-id);
    request-path: func(kind: pick-kind, title: string) -> result<option<path-ref>, fs-error>;
}
```

#### 5.8.3 Network: `kind = "net"`

```toml
[[capabilities]]
kind     = "net"
schemes  = ["https"]                       # http | https | ws | wss | tcp | tls (tcp/tls only with raw = true)
hosts    = ["api.github.com", "*.githubusercontent.com"]   # exact, "*.suffix", literal IP/CIDR; "*" allowed but flagged critical
ports    = [443]                           # default per scheme; ranges as "8000-8100"
methods  = ["GET", "HEAD"]                 # http(s) only
paths    = ["/repos/*/releases/**"]        # same */** grammar as DownloadFileCapability.path
websockets = false                         # ws/wss requires true
max_request_body   = "1MiB"
max_response_body  = "64MiB"
max_concurrent     = 4
rate_limit         = "60/min"
redirects          = "same_host"           # none | same_host | allowlisted | any ; every hop is re-checked
allow_credentials  = false                 # whether Authorization/Cookie headers may be set
forbidden_headers  = ["Host", "Proxy-*"]   # always enforced in addition
resolve            = "public_only"         # public_only | loopback_ok | private_ok (SSRF / DNS-rebinding defense)
dns                = false                 # raw dns-lookup permission, separate from connecting
reason = "Query the GitHub releases API"
```

Rules:

- Check order per request: scheme -> host (match on the **parsed, IDNA-normalized URL host**, before DNS) -> port -> method -> path -> body size -> rate/concurrency. Then resolve DNS **in the host** and connect to the pinned resolved IP, rejecting loopback/link-local/private/multicast/metadata addresses (e.g. `169.254.169.254`) per `resolve` unless explicitly allowed (`loopback_ok` for local dev servers with `hosts = ["localhost"]`). Redirects re-run the whole check on each hop.
- IP-literal hosts and `localhost` need an explicit entry; `"*"` never matches IP literals or private ranges.
- TLS: uses the host's client (`WasmHost.http_client`, `wasm_host.rs:51`); guests cannot disable certificate validation or add roots.
- WebSockets and raw sockets exist only as host resources checked at connect; messages stream through resources, never WASI sockets (WASI networking stays denied, as today).
- `download_file` stays valid as its own kind and is also the only path to the archive-extraction code; semantically it is `net` GET + `fs` write to `extension_work`.
- Honors Zed's configured proxy; offline mode returns `network-disabled`.
- Auditing per request: method, URL (query redacted by default), status, bytes.

```wit
interface net {
    record header { name: string, value: string }
    record request {
        method: http-method, url: string, headers: list<header>, body: option<list<u8>>,
        timeout-ms: option<u32>, redirects: option<redirect-mode>,
    }
    variant net-error { not-granted(string), dns-blocked(string), tls(string), timeout, too-large, cancelled, network-disabled, io(string) }
    resource response {
        status: func() -> u16;
        headers: func() -> list<header>;
        next-chunk: func() -> result<option<list<u8>>, net-error>;
        cancel: func();
    }
    fetch: func(req: request) -> result<response, net-error>;
    resource websocket {
        send-text: func(s: string) -> result<_, net-error>;
        send-binary: func(b: list<u8>) -> result<_, net-error>;
        next-message: func() -> result<option<ws-message>, net-error>;
        close: func(code: u16, reason: string);
    }
    connect-websocket: func(url: string, protocols: list<string>) -> result<websocket, net-error>;
    dns-lookup: func(host: string) -> result<list<string>, net-error>;      // requires dns = true
}
```

(`http-method`, `redirect-mode`, `ws-message` reuse/extend the existing `http-client` types.)

#### 5.8.4 Process: `kind = "proc"` (superset of today's `process:exec`)

```toml
[[capabilities]]
kind        = "proc"
command     = "git"                       # exact name via PATH, or absolute path; "*" flagged critical
resolve     = "path_lookup"               # path_lookup | absolute_only | extension_binary (a binary under extension_work)
args        = ["status", "--porcelain", "**"]   # same grammar as ProcessExecCapability.args
arg_patterns = ["^--format=[a-z]+$"]      # optional anchored regex for "*" slots
deny_args   = ["--exec-path=*", "-c"]     # always-deny, overrides allows
cwd         = ["worktree", "extension_work"]   # allowed working dirs (fs scopes); default: extension_work
env         = { allow = ["LANG", "LC_*", "PATH"], set = { GIT_TERMINAL_PROMPT = "0" }, inherit = "none" }  # inherit = none | allowlisted | shell_env
stdin       = "pipe"                      # none | pipe
stdout      = "pipe"                      # pipe | null ; stderr likewise
max_output  = "16MiB"                     # per stream; truncation is reported
timeout     = "30s"                       # wall-clock; host kills the process group
long_running = false                      # true permits spawn() beyond timeout
max_concurrent = 2
kill        = "own_children_only"         # the guest may only signal handles it spawned
sandbox     = "none"                      # none | no_network | readonly_fs (best effort per OS)
reason      = "Show git status in the panel"
```

Rules:

- Evaluation extends `ProcessExecCapability::allows` (`process_exec_capability.rs`): same `*`/`**` semantics, then `arg_patterns`, then `deny_args`. The existing `process:exec` kind is a compatible alias for `proc` in legacy mode.
- **Compatibility caveat**: today commands inherit the host's cwd and **full environment** (`since_v0_8_0.rs:908-914`). Under `proc` (schema >= 2) defaults tighten: `env.inherit = "none"` with a minimal env, `cwd = extension_work`. Legacy `process:exec` keeps legacy behavior.
- No shell: `command` + `args` go to `execve` directly; the host refuses known shells/interpreters (`sh`, `bash`, `zsh`, `cmd`, `powershell`, `python -c`, `node -e`) with `*`/`**` args unless the args pattern pins a script path under `extension_work`/`extension_install`.
- Path binding: with `resolve = path_lookup` the host resolves the binary using the same logic as the worktree delegate's `which`/`shell-env` (`since_v0_8_0.rs:607-622`) and records the resolved absolute path in the audit log; the consent UI shows it. `absolute_only` binds the capability to a specific path.
- Long-running processes are host resources tied to the extension's lifetime:

```wit
interface proc {
    enum stdio-mode { null, pipe }
    variant cwd-ref { extension-work, worktree(u64), temp }
    record spawn-options { command: string, args: list<string>, env: list<tuple<string,string>>, cwd: option<cwd-ref>, stdin: stdio-mode, stdout: stdio-mode, stderr: stdio-mode }
    variant proc-error { not-granted(string), not-found, timeout, output-limit, killed, io(string) }
    variant exit { code(s32), signal(s32), killed-by-host(string) }

    resource process {
        id: func() -> u64;
        write-stdin: func(data: list<u8>) -> result<_, proc-error>;
        close-stdin: func();
        read-stdout: func(max: u32) -> result<option<list<u8>>, proc-error>;   // none = EOF
        read-stderr: func(max: u32) -> result<option<list<u8>>, proc-error>;
        wait: func() -> result<exit, proc-error>;
        try-wait: func() -> result<option<exit>, proc-error>;
        kill: func(signal: option<s32>) -> result<_, proc-error>;               // this handle only
    }
    run: func(opts: spawn-options) -> result<output, proc-error>;               // today's run-command, with limits
    spawn: func(opts: spawn-options) -> result<process, proc-error>;             // requires long_running = true
}
```

- Teardown: all spawned processes are killed (process group / job object) when the extension unloads, the window closes, or the grant is revoked; pids tracked in `WasmState`.
- Interaction with the UI: stdout readiness surfaces as `io-ready` events so the guest never blocks.

#### 5.8.5 UI, clipboard, notifications, editor, and others

```toml
[[capabilities]]
kind     = "ui"
surfaces = ["panel", "item", "status_item", "toolbar", "modal", "context_menu", "decoration"]
max_panels = 1
max_open_items = 20
max_tree_nodes = 20000
allow_images = true
allow_markdown_links = true
raw_colors = false
reason = "Show database tables in a sidebar panel"

[[capabilities]]
kind = "clipboard"
op   = ["write"]             # write | read ; read is only honored inside a call triggered by a user-gesture event
reason = "Copy query results"

[[capabilities]]
kind = "notify"
levels = ["info", "warning", "error"]
rate_limit = "5/min"
os_notifications = false     # system-level notifications are a separate, higher grant

[[capabilities]]
kind = "open_url"
schemes = ["https"]
hosts   = ["docs.example.com"]    # non-allowlisted hosts always require confirmation

[[capabilities]]
kind = "editor"
op   = ["read_selection", "read_buffer", "edit", "decorate", "diagnostics_read"]
scope = "active_buffer"      # active_buffer | open_buffers | project
glob = ["**/*.sql"]

[[capabilities]]
kind = "settings"
op   = ["read"]
categories = ["language", "lsp"]      # gates get-settings, which is ungated today
```

- `ui` surfaces are individually grantable; a read-only dashboard might request only `status_item`. The host enforces surface use per command (`show-modal` without `modal` -> error).
- Clipboard read is the most sensitive: only accepted inside a call triggered by a user gesture (`App::read_from_clipboard`, `app.rs:1834`).
- Notifications are rate-limited, attributed ("From extension X"), with a "mute this extension" action.
- `editor` is Phase 3; the UI-only MVP never exposes buffer contents.

#### 5.8.6 Consent and granting flow

Modeled on, and compatible with, the existing path (`ExtensionSettings.granted_capabilities` -> `WasmHost` -> `CapabilityGranter`):

1. **Declaration**: capabilities live in `extension.toml`; the registry exposes them; the `zed-extension` CLI prints a human-readable summary and fails if a UI surface/action is used without the matching capability (extend `validate_extension_manifest`, `extension_cli/src/main.rs:265`).
2. **Install-time review** (new, in `extensions_ui`): the extension card and install confirmation show a grouped permission list with severity (`low`: `ui:status_item`; `medium`: network to named hosts; `high`: `proc *`, `fs home`, `net *`), each with the extension's `reason`. `optional = true` entries can be toggled off. Dev extensions (`install_dev_extension`, `extension_host.rs:1119`) are auto-approved with a one-time notice.
3. **Grant storage**: a per-extension record, stored in settings so it is user-visible and editable (consistent with `granted_extension_capabilities`):

```jsonc
"extension_permissions": {
  "sqlite-viewer": {
    "approved_version": "0.1.0",          // re-prompt when the capability set grows on update
    "granted": [ { "kind": "ui", "surfaces": ["panel","item"] },
                 { "kind": "fs", "op": ["read","stat"], "scope": "worktree", "glob": ["**/*.sqlite"] } ],
    "denied":  [ { "kind": "net", "hosts": ["*"] } ],
    "prompt": "on_first_use"              // always | on_first_use | never
  }
}
```

   The existing `granted_extension_capabilities` stays as the **global ceiling** applied to every extension (wildcard defaults for legacy kinds; the default for new kinds is *no grant*, so power users and enterprises keep it as policy). Effective permission = manifest ∩ global ceiling ∩ per-extension grant. Project-level `.zed/settings.json` must never widen grants; only the user layer can grant `fs`/`net`/`proc`.
4. **Runtime prompts** (just-in-time): when the guest calls an API whose capability is in the manifest but ungranted and `prompt = "on_first_use"`, the call suspends (async) while the host shows a prompt ("Extension X wants to run `git status` in this project"): Allow once / Allow for this session / Always allow (writes the grant) / Deny. A denial returns `not-granted` to the guest.
5. **Update flow**: `upgrade_extension` (`extension_host.rs:1009`) compares the new manifest's capabilities to `approved_version`'s; additions need re-consent, removals apply automatically. Silent auto-updates (`auto_update_extensions`) stage but do not activate widened capabilities until approved.
6. **Live updates**: `WasmHost.granted_capabilities` is a construction-time snapshot (`wasm_host.rs:629`). Change `CapabilityGranter` to read through a shared `Arc<RwLock<GrantTable>>` updated by a settings observer so revocation takes effect on the next call without reload; revocation also tears down resources (processes, watchers, websockets) for that capability.
7. **Visibility**: the Extensions page shows active permissions with per-capability "Revoke"; a log view lists denied and prompted accesses (also written to the extension's log channel).
8. **Enforcement point**: host-side only, in a single `CapabilityGranter` extended with `grant_fs`, `grant_net`, `grant_proc`, `grant_ui_surface`, `grant_clipboard`, ... Every WIT host function begins with a `grant_*` call returning a structured error like the current `bail!("capability for ... is not granted by the extension host")` (`capability_granter.rs:39-42`).

#### 5.8.7 Resource limits (non-capability safety, needed before UI)

Add to `wasm_engine` / `WasmHost::load_extension`: `Store::limiter` with memory/table/instance limits (e.g. 512 MiB), a per-call epoch deadline that traps (not just yields) after N ticks for UI-handling calls, a CPU budget (fuel or time) per interval, and a per-call wall-clock timeout in `WasmExtension::call` (`wasm_host.rs:912`) that marks the extension unhealthy and offers "Restart extension". A hung guest must never freeze the UI, because the host renders from cache (§5.1 principle 2).

### 5.9 Render path (sketch)

```
extension.toml [ui.*] -> ExtensionStore::extensions_updated -> ExtensionHostProxy::register_ui_surfaces(extension, manifest.ui, cx)
                                                                         |
   zed crate: ExtensionUiProxy impl <------------------------------------+
     creates ExtensionPanel / Item / StatusItem entities (lazy; wasm already loaded, guest only called on first open)
     workspace.add_panel / status_bar.add_right_item / toolbar.add_item

user opens panel -> ExtensionPanel::set_active -> wasm_extension.call(surface_open) --(async, serialized)--> render-reply
  apply_reply: validate (size/depth/schema/permission) -> reconcile into ExtensionSurfaceState -> cx.notify()
GPUI paint: ExtensionPanel::render -> UiBuilder::build(&state.tree, theme, cx) -> AnyElement
  event closures enqueue ui-event -> wasm_extension.call(surface_event) -> ui-response -> apply_reply + commands
```

The builder is a `match node.type` over a closed Rust enum producing `ui` components (`column` -> `v_flex()`, `button` -> `ui::Button`, `tree` -> `TreeViewItem` inside a `uniform_list`, `input` -> `ui_input::InputField`, `table` -> `ui::DataTable`, `markdown` -> `markdown::Markdown`). Color/spacing tokens resolve against `cx.theme().colors()` at paint time, so a theme switch re-skins cached trees without a guest round trip. JSON parsing and validation run on the background executor; only reconciliation runs on the main thread.

### 5.10 Phasing (aligned with the RFC's "status bar first")

0. **Safety and plumbing**: wasm resource limits, call timeouts, gating of `fetch`, `CapabilityGranter` refactor, consent UI skeleton, `extension_permissions` settings.
1. **Status bar item, modal, toast, actions/keybindings**: smallest node set (`row`, `text`, `icon`, `badge`, `button`, `progress`, `divider`, `menu`). Proves the `ui` WIT, proxy, validator, and diffing.
2. **Panel and pane item**: `column`, `scroll`, `list`, `tree`, `tabs`, `input`, `table`, `markdown`, `code`, virtualization, serialization.
3. **Toolbar, context menus, decorations, `editor` capability, charts/graph.**
4. **Streaming resources** (`proc.spawn`, `net.websocket`, `fs.watch`) wired to `io-ready`.

---

## 6. Risks, open questions, compatibility and versioning

### Risks

- **R1, serialized guest execution**: one `Store` and one queue per extension (§1.2) means a slow handler blocks all later UI events and background calls. Mitigations: per-call timeouts and trapping deadlines; non-blocking-by-contract UI calls; optionally two instances (UI and background) per extension; render from host-side cache so paint never waits.
- **R2, main-thread cost**: reconciliation and element construction run on the foreground thread; large trees can drop frames. Enforce size/rate caps, virtualize lists/tables/trees, parse and validate off-thread.
- **R3, API freezing**: a declarative schema becomes a long-lived contract. Version the schema string (`zed-ui/1`), keep the node set small and orthogonal, reserve `x-` properties, return structured validation errors with node paths.
- **R4, security**: untrusted JSON (giant or deeply nested trees, huge strings), spoofing (fake dialogs or permission prompts, lookalike system chrome), clickjacking via overlay `stack`, focus stealing, keybinding hijack, notification spam, SSRF/DNS rebinding, path traversal and symlink races, process argument injection, malicious archives (zip-slip; the existing `extract_zip` and tar paths should be audited for traversal), exfiltration via `net` after `fs read` (so consent UI surfaces the combination as "can read your files and send them to X"). Host chrome (title bars, permission prompts) must never be drawable by extensions; extension surfaces always carry attribution (extension icon and name in tab/tooltip).
- **R5, accessibility**: the declarative tree needs roles and labels from day one; GPUI's accessibility layer (`crates/gpui/src/_accessibility.rs`) should receive node roles, otherwise extension UI regresses a11y relative to built-ins.
- **R6, static-string traits**: `Panel::persistent_name/panel_key/icon_tooltip` are `&'static str` (`dock.rs:37-38,72`); needs a trait relaxation or interning.
- **R7, remote development**: UI runs on the client but extension wasm may need worktree access on the remote. Today `remote_load` only syncs language-server/debug extensions. Options: guest on the client with fs/proc calls proxied to remote worktrees (mirrors how `WorktreeDelegate` already abstracts local vs remote; recommended), or guest on the remote with UI trees/events marshaled over the existing proto channel.
- **R8, upstream direction**: the maintainer said (thread) this is team-driven and "on our radar"; the team may prefer a different shape. Coordinate through the RFC before implementing; Phase 0 items are independently valuable and uncontroversial.
- **R9, guest-side GPUI alternative**: `gpui_web` shows GPUI can target `wasm32-unknown-unknown`, which could allow guest-side GPUI, but there is no cross-module GPU/window sharing, it couples guests to GPUI internals, and it enlarges the attack surface. The declarative route trades expressiveness for sandboxing and stability. Rich custom drawing (graphs, charts) is handled by bounded host-side node types.

### Open questions

1. Should `fetch` gating (closing the §2.4 gap) land in 0.8.0 retroactively (it is unreleased) so Stable never ships an ungated `fetch` in a new API version? Versions <= 0.6.0 would need a compatibility shim with an implicit allowlist.
2. Where do per-extension grants live: settings JSON (visible and editable, but trust-bearing data in a file that may sync across machines) vs. the local database/keychain? Consider storing grants locally and only policy ceilings in settings.
3. Project-local settings must not be able to widen grants (user layer only); needs enforcement in the settings merge.
4. Expose any buffer/editor API in v1, or keep UI extensions "outside the editor"? The thread's use cases (DB viewers, trackers, dashboards, package managers) mostly do not need buffer access.
5. Multiple windows/workspaces: one surface instance per workspace or per window? Proposal: per-workspace instances keyed by `(workspace_id, surface_id)`, with the guest told the instance id.
6. Language support: the SDK is Rust-only; a JS/TS or Python story (componentize-js) becomes feasible because the UI contract is JSON.
7. Registry policy: should the registry review or flag extensions requesting critical capabilities?
8. Should hosts also accept a *static* UI file (`ui/*.json` shipped in the extension, no wasm) for read-only panels and status items? That gives UI extensions with zero code execution and could be a cheap, very safe Phase 1.
9. Ordering: panels must register before workspace deserialization or their persisted size/position is lost (`add_panel` reads `persisted_panel_size_state`, `workspace.rs:2850`).

### Compatibility and versioning notes

- **WIT**: add `since_v0.9.0`; keep all older bindings. Old extensions never see the new imports; the new linker (`wit.rs:43`) only adds them for >= 0.9.0 instances. Guests built with the new SDK declare `0.9.0` in `zed:api-version`, which already makes older Zed reject them (`is_supported_wasm_api_version`, `wit.rs:54`) rather than fail at link time. Bundle any breaking cleanups taken in the same bump per `PENDING_CHANGES.md` (e.g. the `SlashCommand.tooltip_text` rename already listed there).
- **Release channels**: gate the new version via `wasm_api_version_range` (`wit.rs:60`); Dev/Nightly first. `authorize_access_to_unreleased_wasm_api_version` (`wit.rs:75`) covers the dev workflow.
- **Manifest**: all new tables optional with `serde(default)`; extensions using `[ui.*]` or new capability kinds set `schema_version = 2`. Zed with `SchemaVersion::CURRENT = 1` refuses them via `is_version_compatible` (`extension_host.rs:135`), so there is no silent permission downgrade.
- **Capabilities**: legacy kinds (`process:exec`, `download_file`, `npm:install`) keep serde names (`capabilities.rs:14-22`) and semantics for `schema_version <= 1`. Add new variants to `ExtensionCapability`, mirror them in `ExtensionCapabilityContent` (`settings_content/src/extension.rs:45`) and the conversion in `extension_settings.rs:40-63`; the `default.json:2403` default stays unchanged, and new kinds default to ungranted.
- **Registry/CLI**: `zed-extension` should embed the capability and `ui` surface lists into the registry manifest so the website and Extensions page show permissions before install; `ExtensionProvides` (`cloud_api_types`) gains `Panels`, `StatusItems`, etc. for filtering.
- **Tests**: patterns to copy: `CapabilityGranter` unit tests (`capability_granter.rs:~110`), `*_capability.rs` `allows` tests, `test_writeable_path_rejects_escape_attempts` (`wasm_host.rs:1058`), `extension_store_test.rs` for load flows, `extensions/test-extension` as fixture, GPUI `VisualTestContext` for rendered trees. Add fuzz targets for the UI JSON validator and patch applier.
- **Docs**: extend `docs/src/extensions/capabilities.md`, add `docs/src/extensions/ui.md`, and add `extension_permissions` to `docs/src/reference/all-settings.md`.

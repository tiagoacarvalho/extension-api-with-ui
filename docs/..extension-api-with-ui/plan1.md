# Plan 1: re-implementing Mentenaz Forge Super as Zed extensions

Inputs: `thread.md` (discussion #53403), `API.md` (current extension API plus the `since_v0.9.0` / `zed-ui/1` proposal), `applications.md` (the 26 fork features).

## Revision notes

Revision 2 (2026-11-06). The inputs changed: `thread.md` is now a full-fidelity rewrite (294 lines), and `applications.md` gained an identification section, theme-bridge details, DB engine notes and `gpui_flow` zoom/pan details. What changed in this plan:

1. **Identity.**
   - The local repo is `mentenaz/zed_Source`, continued and renamed to `mentenaz/mentenaz.Forge.Super` (apps §0 "Identification").
   - It is a Zed fork with no Tauri code.
   - The Tauri app "Forge", with its schema-driven panels and about 30 engines, is a separate codebase and is not available locally.
   - Fixed in the path conventions below. The earlier text never attributed Tauri code to the repo, so nothing else needed correcting.
2. **New §7, "Reconciliation with discussion #53403".** It covers:
   - **§7.1** The RFC body (WIT `panel`/`components`/`status-bar`, phases 1–4, security, alternatives) compared with this plan.
   - **§7.2** nakajimayoshi's `UIExtension: Extension` trait with a GPUI `render`, and why R2 rejects it.
   - **§7.3** mentenaz's Forge (Tauri) schema-driven panel design as prior art for R2: what to adopt, and why WASM rather than a native binary.
   - **§7.4** `embedded_gpui` as an alternative or a risk.
   - **§7.5** A use-case coverage table for the use cases raised in the thread.
3. **G22 sidecars.** Adds a Forge-compatible `oneshot-json` / `jsonl-stdio` protocol, and ties the escape hatch to §7.3.
4. **G3 zoom.**
   - Corrected: zoom, pan and fit view live in `gpui_flow` (`FlowState::request_fit`, `canvas_size`), built on the core `.scale()` patch (apps §19).
   - The host-builder zoom only replaces `.scale()` for the restricted node-body subset.
   - Upstreaming `.scale()` is now the preferred path, and builder zoom is the fallback.
5. **F9 Oracle.**
   - Oracle is present in the code (`oracledb = "26.0.0-beta.4"`, `fork/Cargo.toml:995` [V]), even though the README lists four engines.
   - Whether it needs native client libraries is unverified.
   - The sidecar is now a fallback, not the plan of record.
6. **F20 theme bridge.**
   - The fork maps Zed's theme onto gpui-component's theme in `sync_gpui_component_theme` (`fork/crates/zed/src/main.rs:235` [V]).
   - The `zed-ui/1` token model removes the need for such a bridge.
   - The fork's `SemanticThemeTokens` is noted as naming prior art.
7. **§6.2 risk 4 (upstream acceptance) rewritten.**
   - macraig is CONTRIBUTOR and is the only team voice ("closely tied to our extensions roadmap", "driven by our team", "not ... in the near future").
   - The strategy now shifts to a community fork track (`tiagoacarvalho/extension-api-with-ui`) that tracks upstream.
8. **§6.1 milestones** now put the RFC's phase 1, a status-bar item, first (M1), matching `tiagoacarvalho/extension-api-with-ui`.
9. **§6.3 open question 11** replaced: the collapsed replies are now known. Open question 12 was added (contributor coordination).
10. **New gaps found by the §7.5 coverage check:**
    - host request signers (AWS SigV4);
    - PKCE loopback OAuth;
    - editable `data_grid` cells;
    - a `color_input` node;
    - `image.fs_path`;
    - a possible git lane-graph widget.
11. **§1 executive summary** gained a short "Relation to the discussion (§7)" paragraph.

Path conventions used below:

- `zed/...` is `/home/tiago/Repositories/zed` at `9a8ab54db4`.
- `fork/...` is `/home/tiago/Repositories/mentenaz.Forge.Super` at `HEAD` of `main`.
  - This is mentenaz's Zed fork: originally `mentenaz/zed_Source`, renamed in 2026-11 (apps §0).
  - It contains only Rust/GPUI code built into the binary, with no Tauri code and no schema-driven panel system.
  - "Forge (Tauri)" in §7.3 refers to mentenaz's separate app, which is described only in `thread.md`.
- `API.md §x` and `apps §n` point into the input documents.
- **[A]** marks an assumption that I did not verify against source. **[V]** marks a claim I re-checked in source while writing this plan. Line numbers are approximate to where a symbol starts.

The three hard rules, referred to as R1, R2 and R3 throughout:

- **R1.** UI is GPUI, delivered through the WASM extension pipeline, inside Zed's existing structure: `extension_host`, WIT versioning, `extension.toml`, and the registry.
- **R2.** Extensions add UI only declaratively, as JSON. The host turns the JSON into GPUI elements. Extensions never run GPUI code.
- **R3.** Every permission is declared in the manifest. Filesystem, network and process permissions are modelled in maximal detail.

---

## 1. Executive summary

**Result.** 18 of the 26 features can become extensions:

- **9 Full**: features 1, 3, 4, 7, 8, 11, 12, 14 and 21.
- **9 Partial**: features 2, 5, 6, 9, 10, 13, 15, 16 and 17. Each of these is mostly an extension, but depends on a small, generic host hook that should go upstream.
- **8 Core-only**: features 18, 19, 20, 22, 23, 24, 25 and 26.

The 18 features carry almost all of the fork's original panel and backend code, roughly 85k lines (apps §0). The 8 core-only items are of three kinds:

- **Infrastructure the API itself needs**: GPUI `scale`, the graph canvas, panel ordering, and extension-system fixes.
- **Code that is deleted, not ported**: the vendored `gpui_component` stack (about 160k lines), which the host's JSON node catalog replaces.
- **Product and branding policy**: telemetry, rebrand, updater and packaging.

**API additions needed beyond what API.md proposes.** API.md's `since_v0.9.0` proposal (`ui`, `fs`, `net`, `proc`, `clipboard`, `notify`, plus `zed-ui/1`) is necessary but covers only about features 1, 3, 7, 8 and 12. The rest also need:

1. **Host-owned data channels** (`ui-data`): row stores, series and log buffers. Grids, charts and streaming output then scroll and update without a guest round trip or a tree re-send.
2. **New node types**: `data_grid`, `chart`/`gauge`/`stat`, `graph` (node editor), `editor` (an embedded real Zed `Editor`), `diff`, `log`, `terminal_view`, and `schema_form`. Also `draggable`/`drop_target` props.
3. **`sys` interface**: host-side `sysinfo` for metrics, the process list, and signals to processes the extension did not spawn.
4. **`terminal` interface**: run a task in Zed's terminal through `Workspace::spawn_in_terminal` (`zed/crates/workspace/src/tasks.rs:222` [V]), and contribute terminal env/PATH through a new provider hook.
5. **`secrets` and `auth` interfaces**: keychain through `CredentialsProvider` (`zed/crates/credentials_provider/src/credentials_provider.rs:14` [V]), a host-run OAuth device flow, and **secret handles** bound to network hosts, so tokens never enter guest memory.
6. **`git` interface**: read API over `project::git_store`, `before_remote_op` hooks (push gating), a credential provider, scoped config writes, and **host-surface slots** such as the git panel tab and footer.
7. **`editor` interface** plus dynamic context-menu entries, a completion provider for embedded editors, and decorations.
8. **File handlers**: claim `**/*.flow.json` as a project item. Also JSON-schema contributions for file globs.
9. **`ipc`**: manifest-declared services and topics between extensions, routed by the host.
10. **Activation and background**: `[activation]` events (startup, `workspace_contains`, action, surface), workspace lifecycle events, and timers.
11. **`kv`** persistence scoped per extension, workspace or project, backed by `db::kvp` (`zed/crates/db/src/kvp.rs:12` [V]). Also extension settings schemas, rendered by `settings_ui`.
12. **Raw TCP/TLS sockets**, needed by database drivers. They are host-checked, either through WASI `socket_addr_check` or a host resource.
13. **Sidecars**: native helper binaries with a sha256 pin, as an explicit, high-severity escape hatch. Examples are the Oracle driver and any backend that cannot reasonably target `wasm32-wasip2`.

**What stays in core.**

- **Generic hooks** that any extension could use, proposed as upstream contributions:
  - the `TerminalEnvProvider` registry in `project/src/terminals.rs`;
  - git hook and credential routing, with Zed acting as a git credential helper;
  - host-surface slots in `git_ui`;
  - a `ProjectItem` dispatcher for extension file handlers;
  - `Panel` trait strings as `SharedString`;
  - panel ordering;
  - GPUI `.scale()`;
  - extension-install error reporting and the README tab.
- **Fork-only policy**, never extensions: the rebrand, telemetry hard-off, keychain namespacing per app scheme, the notify-only updater, data import, and packaging.

**Corrections to API.md that the rules require** (details in §5):

- **(a) Manifest declaration for the `user_selected` powerbox.** API.md §5.8.2 lets the `fs` `user_selected` powerbox work "with no up-front permission". That violates R3. The scope must be declared, and only the specific path is granted at runtime.
- **(b) Static UI files with no wasm.** API.md open question 8 proposes these, which conflicts with R1. They are allowed only as a placeholder tree of a wasm extension, and are loaded through `extension_host`.
- **(c) `canvas2d`.** It stays a bounded, retained display list (data only). It is not an escape hatch for custom drawing logic (R2).
- **(d) Ungated legacy calls.** For `schema_version >= 2`, every legacy ungated call must also be covered by a manifest declaration: `get-settings`, `worktree.read-text-file`, `which`, `shell-env`, `github.*`, `http-client.fetch` and `download_file`.
- **(e) The proposed process kill.** `proc.kill` with `kill = "own_children_only"` cannot express feature 3. External PIDs need the separate `sys` signal capability, with a host-drawn confirmation.
- **(f) Shells.** API.md bans shells. That makes features 4 and 10 impossible unless a **user-attested shell** mode is added. In that mode, only text the user typed into a host-owned input, or saw in a host confirmation, can reach a shell.

**Relation to the discussion (§7).**

- The plan is a superset of the RFC's WIT, surfaces and phases, and its first milestone is the RFC's phase 1, a status-bar item (§7.1).
- It rejects guest-side GPUI `render` for the reasons nakajimayoshi gave himself (§7.2).
- It adopts the useful parts of mentenaz's Forge schema-driven panels (manifest-level discovery, bindings, JSON-lines streaming, host theme tokens). It replaces Forge's native backend with a WASM guest, so that R3 permissions are enforceable, and keeps native code only as the G22 sidecar exception (§7.3).
- `embedded_gpui` is the main strategic risk (§7.4).
- The thread's use cases are covered, with a few small gaps such as AWS SigV4 signing (§7.5).

---

## 2. Gap analysis

### 2.1 Need versus offer matrix

Legend: **T** = available today (v0.8.0), **P** = in API.md's proposal, **–** = missing (proposed in §2.2).

| Need (from applications.md) | Features | Today | API.md | Gap ID |
|---|---|---|---|---|
| Dock panel, pane item, status item, modal, toast | all panels | – | P (`[ui.*]`, §5.7) | – |
| Actions, keybindings, command palette | all | – | P (`InvokeExtensionAction`) | – |
| Layout, text, buttons, input, list, tree, tabs, split, markdown | all | – | P (`zed-ui/1`, §5.5) | – |
| Virtualized data grid with sort, filter, resize, copy and export, holding 100k+ rows | 3, 9, 11 | – | partial (`table`, rows in tree) | G1 |
| Charts, gauges, time series, dashboard cards | 1, 2, 15a | – | mention only (`chart`, "later") | G2 |
| Graph/canvas node editor (drag, connect, ports, LOD, minimap) | 9, 10, 13 | – | mention only (`graph`, auto-layout) | G3 |
| Embedded real editor (SQL with completions, raw JSON, read-only file view, diff) | 9, 10, 14 | – | `code` node is display-only | G4 |
| Schema-generated form (Designer inspector) | 10 | – | – | G5 |
| Drag and drop (catalog to canvas) | 10 | – | – | G6 |
| Long-running processes with streaming output, stop, exit status | 4, 5, 6, 10 | buffered `run-command` | P (`proc.spawn`, `io-ready`) | G7 (perf and shells) |
| Terminal and shell console integration, run in Zed terminal | 4, 5–8, 11 | – | – | G8 |
| Terminal env/PATH injection | 5, 6, 16, 17 | – | – | G9 |
| System metrics, OS process list, kill external PID | 1, 3, 5, 6 | – | – | G10 |
| OAuth device flow, keychain/secret storage | 9, 13, 14 | – | – | G11 |
| Git read (log stats, status, remotes), push gating, credential helper, repo-config writes, git panel tabs/footer | 2, 13, 15, 16 | – | – | G12 |
| Editor context-menu entries with selection or word under cursor | 11 | – | static `[ui.context_menus]` entries only | G13 |
| File-type claims for custom editors (`*.flow.json`) | 10 | – | `open_for_extensions` (single extension, no buffer integration) | G14 |
| JSON schema for file globs | 10 | LSP option schemas only | – | G15 |
| Inter-extension communication | 2, 4, 5, 10, 13, 15 | – | – | G16 |
| Background services, activation events, workspace lifecycle events | 5, 6, 12, 13, 16 | – | timers only | G17 |
| Persistence: KV, per-workspace state, settings | most | `key-value-store.insert` only (docs) | `serialize-instance` | G18 |
| Raw TCP/TLS (DB drivers), localhost port probing | 9, 10 | – | `tcp`/`tls` listed in schemes, no WIT | G19 |
| Remote images (avatars, README images) | 11, 13, 14 | – | `image` path or `data_uri` only | G20 |
| Downloads with checksum, runtime toolchains under the extension dir | 5, 6 | `download-file` (no checksum) | unchanged | G21 |
| Native code that cannot target wasm (Oracle client, heavy drivers) | 9 | LSP-style binaries via `process:exec` | – | G22 |
| Open or replace workspace, open path, reveal, open URL | 13, 14 | – | `open_url` | G23 |
| Confirmation dialogs drawn by the host | 3, 6, 11, 13 | – | – | G24 |
| Panel ordering when many extension panels exist | all | – | – | core (feature 25) |

### 2.2 Missing primitives and proposed additions

Each subsection gives the WIT sketch, the JSON UI additions, the manifest fields and the permissions. WIT goes into `zed/crates/extension_api/wit/since_v0.9.0/`, and every host function starts with a `CapabilityGranter::grant_*` call (`zed/crates/extension_host/src/capability_granter.rs:23-67` [V], to be extended).

#### G1. Data grids, and host-owned data channels in general

**Problem.** A `table` whose rows sit inside the JSON tree means re-serializing the tree on every page or sort. A query result of 100k rows cannot work like that (API.md R2 limits). The guest is serialized (API.md §1.2), so scrolling cannot wait on a guest call either.

**Design.** The guest writes rows into a host-owned **row store**. The tree only references it with `"source": "rows:<id>"`. The host virtualizes rendering with `uniform_list` and `ui::Table` (`zed/crates/ui/src/components/data_table.rs:356` [V]). It sorts and filters locally when `sort: "local"`, and asks the guest otherwise.

```wit
interface ui-data {
    type store-id = string;
    variant data-error { not-found, too-large, schema(string), quota }

    record column { key: string, title: string, kind: cell-kind, width: option<u32>, align: option<string> }
    enum cell-kind { text, number, bool, datetime, bytes, json, link, badge, null-aware }

    /// Rows are JSON arrays aligned to columns; JSON-lines keeps the boundary cheap.
    create-row-store: func(id: store-id, columns: list<column>, max-rows: u32) -> result<_, data-error>;
    append-rows: func(id: store-id, rows-jsonl: string) -> result<u64, data-error>;   // returns new length
    replace-rows: func(id: store-id, start: u64, rows-jsonl: string) -> result<_, data-error>;
    clear-rows: func(id: store-id) -> result<_, data-error>;
    /// Selected rows, read back for copy or export: JSON, CSV or SQL inserts.
    export-rows: func(id: store-id, selection: option<list<u64>>, format: export-format) -> result<string, data-error>;
    enum export-format { json, csv, sql-insert, tsv }

    create-series: func(id: store-id, names: list<string>, capacity: u32) -> result<_, data-error>;  // ring buffer
    push-points: func(id: store-id, t-ms: u64, values: list<f64>) -> result<_, data-error>;

    create-log: func(id: store-id, capacity-lines: u32) -> result<_, data-error>;
    append-log: func(id: store-id, lines: list<log-line>) -> result<_, data-error>;
    record log-line { text: string, stream: log-stream }
    enum log-stream { stdout, stderr, info, command }

    drop-store: func(id: store-id);
}
```

**JSON node.**

```json
{ "type": "data_grid", "key": "results", "source": "rows:q1",
  "sort": "local", "filter": "local", "resizable_columns": true,
  "selection": "multi", "row_height": "dense", "sticky_header": true,
  "on_activate": { "id": "open_row" }, "on_context_menu": { "id": "row_menu" },
  "on_sort": { "id": "sort" }, "copy": ["json", "csv", "sql-insert"] }
```

**Limits.** Row stores live in host memory and count against a per-extension quota (default 64 MiB, `ui.max_store_bytes`). Cells are truncated at 4 KiB for display, and the full value is fetched on demand through `export-rows`.

**Manifest and permission.** A `ui` capability field: `data_stores = { max_bytes = "64MiB", max_rows = 1_000_000 }`. Export to a file goes through `fs user_selected` with `op = ["create","write"]`.

#### G2. Charts, gauges and dashboards

**Nodes.**

- `chart`: `{kind: line|area|bar|stacked_bar|pie|donut, source: "series:<id>" | inline points, x: {kind: time|category}, y: {min, max, unit}, legend, colors: palette tokens}`.
- `gauge`: `{value, max, label, thresholds: [{at, color_token}]}`.
- `stat`: `{label, value, delta, trend: up|down|flat, status}`.
- `sparkline`: `{source}`.
- `card`: a container with `title`, `actions` and a `status` (`healthy|unknown|failed`, the Dashboard's three-state rule, apps §2).
- `grid` gains `min_column_width` for responsive dashboards.

**Rendering.** The host draws these with GPUI `canvas` (`zed/crates/gpui/src/elements/canvas.rs`) and `ThemeColors`, and re-skins them on theme change. Series updates from `push-points` only trigger `cx.notify()` on the surface. No tree diff happens.

**Permission.** Charts need no new capability. They are covered by `ui`.

#### G3. Graph and canvas node editors

Needed by the Flows Designer, the DB schema graph and the Helm Actions job graph. The host implements one `graph` element. It is a port of `gpui_flow` (fork/crates/gpui_flow, MIT) re-themed on Zed's `ui`. The guest provides data only: nodes, ports, edges, and a small node-body subtree drawn from a **restricted node subset** (`row`, `column`, `text`, `icon`, `badge`, `chip`, `progress`).

```json
{ "type": "graph", "key": "designer", "source": "graph:flow1",
  "layout": "manual" , "auto_layout": { "algorithm": "layered", "direction": "LR" },
  "interaction": { "pan": true, "zoom": [0.2, 2.0], "drag_nodes": true, "connect": true,
                   "select": "multi", "delete": true, "insert_on_edge": true },
  "minimap": true, "controls": ["zoom_in", "zoom_out", "fit_view"],
  "lod": { "compact_below": 0.6, "minimal_below": 0.35 },
  "drop_target": { "accepts": ["flow-action"], "on_drop": { "id": "add_action" } },
  "on_node_activate": { "id": "inspect" }, "on_connect": { "id": "connect" },
  "on_edge_insert": { "id": "insert_between" }, "on_change": { "id": "graph_changed", "debounce_ms": 250 } }
```

Graph data lives in a host-owned **graph store**, so dragging a node never calls the guest:

```wit
// in ui-data
record graph-node { id: string, x: f32, y: f32, kind: string, title: string, status: option<string>,
                    ports-in: list<string>, ports-out: list<string>, body-json: option<string> }
record graph-edge { id: string, from-node: string, from-port: string, to-node: string, to-port: string, label: option<string>, kind: option<string> }
create-graph: func(id: store-id, max-nodes: u32) -> result<_, data-error>;
set-graph: func(id: store-id, nodes: list<graph-node>, edges: list<graph-edge>) -> result<_, data-error>;
patch-graph: func(id: store-id, upsert-nodes: list<graph-node>, remove-nodes: list<string>,
                  upsert-edges: list<graph-edge>, remove-edges: list<string>) -> result<_, data-error>;
set-node-status: func(id: store-id, node: string, status: option<string>) -> result<_, data-error>; // run colouring
read-graph: func(id: store-id) -> result<tuple<list<graph-node>, list<graph-edge>>, data-error>;   // positions after user drags
```

**Zoom.** In the fork, zoom, pan and fit view are features of `gpui_flow`, not of GPUI.

- The graph is laid out once, in flow units, inside a world container scaled with `.scale(zoom)` (apps §18, §19).
- Pan is an offset.
- Fit view is `FlowState::request_fit(padding)` (`fork/crates/gpui_flow/src/store.rs:145` [V]).
- The viewport is measured with `FlowState::canvas_size()` (`:122` [V]).

The host `graph` element ports this state machine as-is.

For scaling, there are two options:

- **Preferred: upstream `.scale()`** (§4). The port then keeps `gpui_flow`'s single scaled world container unchanged.
- **Fallback: builder-resolved zoom.** The host builder owns every style, because all values come from tokens and px props. It can multiply font size, padding, borders and positions for the restricted node-body subset before GPUI sees them. This works only because node bodies are limited to that small subset. It is not a general replacement for `.scale()` [A: text re-layout at scaled font sizes is assumed acceptable for performance at 500 nodes].

**Limits.** At most 2,000 nodes and 5,000 edges per graph. Node bodies are limited to 30 subtree nodes.

#### G4. Embedded editor and diff nodes

The `code` node in API.md is display-only. Database workbench, Flows Raw and companions, and Helm Workspace need a real `Editor`: syntax highlighting, undo, multi-cursor, and completions. Under R2 the guest *declares* an editor. The host instantiates `editor::Editor` and the guest never touches it.

```json
{ "type": "editor", "key": "sql", "buffer": { "kind": "scratch", "id": "wb-1", "initial": "select 1" },
  "language": "SQL", "mode": "full", "min_lines": 6, "max_lines": 40,
  "completions": { "provider": "schema" },
  "on_change": { "id": "sql_changed", "debounce_ms": 300 },
  "on_submit": { "id": "run", "keystroke": "cmd-enter" } }
```

`buffer.kind` takes one of these values:

- `scratch`: owned by the host and kept for the surface lifetime.
- `project_path`: `{worktree, path}`. Opens the real project buffer, so save, dirty and undo are Zed's. Requires `editor` `edit`/`read_buffer` on that glob.
- `readonly_text`: display text, syntax-highlighted.

The `diff` node is `{patch: "<unified diff>", language_from_path: true}`, rendered as a read-only multibuffer.

```wit
interface editor {
    type buffer-ref = string;   // "scratch:wb-1" or "path:<worktree-id>:<rel-path>"
    record selection-info { path: option<string>, language: option<string>, text: string,
                            word-at-cursor: option<string>, line-text: option<string>, line: u32, column: u32 }
    record completion { label: string, insert-text: string, kind: string, detail: option<string>, documentation-md: option<string> }
    record text-edit { start: u32, end: u32, text: string }      // UTF-8 byte offsets into current text
    variant editor-error { not-granted(string), not-found, conflict, io(string) }

    read-buffer: func(b: buffer-ref) -> result<string, editor-error>;
    apply-edits: func(b: buffer-ref, edits: list<text-edit>, version: option<u64>) -> result<u64, editor-error>;
    active-selection: func() -> result<option<selection-info>, editor-error>;   // editor:read_selection
    set-decorations: func(b: buffer-ref, decorations-json: string) -> result<_, editor-error>;
}
// Export: called by the host for `completions.provider`, with a 150 ms budget (results dropped after that).
export provide-completions: func(surface: string, node: string, provider: string, text: string, offset: u32) -> result<list<completion>, string>;
```

#### G5. Schema-generated forms

The Designer inspector generates a form from each action's schema. Node:

```json
{ "type": "schema_form", "key": "inspector", "schema": { "$ref": "schemas/http-action.json" },
  "value": { "method": "GET", "url": "" }, "ui_hints": { "url": { "widget": "input", "placeholder": "https://" } },
  "on_change": { "id": "action_changed", "debounce_ms": 200 } }
```

- **Renderer.** The host renders it with the same widget mapping as `settings_ui` (`zed/crates/settings_ui/src/page_data.rs`). That mapping is a closed set: string, number, bool, enum, array of primitives, nested object, and a JSON-editor fallback through the `editor` node.
- **Schemas.** They are extension-relative files or inline JSON, validated host-side with the same JSON Schema crate `json_schema_store` uses [A].

#### G6. Drag and drop

Any node may carry `"draggable": {"kind": "flow-action", "payload": {...}}`. Targets declare `"drop_target": {"accepts": [kinds], "on_drop": {...}}`. Payloads are JSON (capped at 4 KiB), and kinds are namespaced to the extension. Dropping files from the OS is a separate prop, `accepts_files: true`. The drop then yields `user_selected` fs grants for the dropped paths, but only if the manifest declares `fs scope = "user_selected"`.

#### G7. Long-running processes with streaming output

API.md's `proc.spawn` delivers readiness through `io-ready`. Every chunk still costs a guest call, and calls are serialized. Additions:

- **Host-side piping.** `spawn-options.stdout-to: option<store-id>` writes output straight into a `log` store. No guest call happens per chunk.
- **Watch patterns.** `watch-patterns: list<string>` (anchored regexes, at most 8) produce `proc-match` events. The SPFx debug URL detection in feature 4 uses this.
- **Exit events.** `exit` and `proc-match` events go to the guest. The guest only reads raw output when it needs to parse it.

```wit
// additions to proc
record spawn-options-v2 { base: spawn-options, stdout-to: option<string>, stderr-to: option<string>,
                          watch-patterns: list<string>, process-group: bool, env-from-secret: list<tuple<string, secret-ref>> }
spawn-v2: func(opts: spawn-options-v2) -> result<process, proc-error>;
/// User-attested shell: runs exactly the text the user typed in a host-owned `input` node marked `user_command`
/// (or approved in a host confirmation); the guest cannot supply or alter the string.
spawn-user-command: func(node: string, surface: string, cwd: option<cwd-ref>, stdout-to: option<string>) -> result<process, proc-error>;
```

**User-attested shell (fixes the conflict between API.md's shell ban and feature 4).**

1. An `input` node with `"user_command": true` is host-owned.
2. The host tracks provenance: the value counts as "user-authored" only if the last change came from keystrokes or paste, not from a guest `value` write.
3. On submit, the host runs `sh -c` / `cmd /C` with that exact text.
4. A guest-proposed command, such as `npm install` from a package manager button, instead goes through `terminal.run-task` (G8). There the host shows the command line in a confirmation, or in the task's terminal tab, before it runs.

Manifest: `kind = "proc"`, `mode = "user_attested_shell"` (§5.4).

#### G8. Terminal and task integration

Most fork commands are "run X in a console and show output". Zed already has the right sink: `SpawnInTerminal` (`zed/crates/task/src/task.rs:42` [V]) via `Workspace::spawn_in_terminal`, which returns `Task<Option<Result<ExitStatus>>>` (`zed/crates/workspace/src/tasks.rs:222-233` [V]). This route reuses the terminal panel, stop, rerun and hyperlinks. It also removes the need for the Script Runner as a shared console, and with it most of G16.

```wit
interface terminal {
    record task-spec { label: string, command: string, args: list<string>, cwd: option<cwd-ref>,
                       env: list<tuple<string,string>>, reveal: reveal, use-new-terminal: bool,
                       allow-concurrent: bool, confirm: confirm-mode }
    enum reveal { always, no-focus, never }
    enum confirm-mode { never, first-time, always }      // host prompt showing the exact command line
    type task-handle = u64;
    variant terminal-error { not-granted(string), cancelled-by-user, no-terminal, io(string) }
    run-task: func(spec: task-spec) -> result<task-handle, terminal-error>;   // completion arrives as task-exit event
    stop-task: func(h: task-handle) -> result<_, terminal-error>;
}
// Export the host calls when building any terminal or task environment (G9):
export terminal-env: func(ctx: terminal-env-context) -> result<env-contribution, string>;
record terminal-env-context { worktree-id: option<u64>, cwd: option<string>, kind: terminal-kind }
enum terminal-kind { shell, task, extension-process }
record env-contribution { path-prepend: list<path-ref>, set: list<tuple<string,string>>, unset: list<string> }
```

**JSON node.** `{"type": "terminal_view", "key": "out", "task": <task-handle>}` embeds the live `TerminalView` of an extension-started task in a panel. This mirrors the fork's in-panel console, while the terminal itself stays host code.

#### G9. Terminal env/PATH injection

**Core hook (upstream).** Add a `TerminalEnvProviderRegistry` global. It is consulted in `zed/crates/project/src/terminals.rs` where the env is assembled (`env.extend(settings.env)` at `:137` and `:406` [V]), next to where the fork calls `add_ide_runtimes` (`fork/crates/project/src/terminals.rs:39` [V]).

- **Guest calls.** The extension provider calls the `terminal-env` export above. The host caches the result per `(worktree, cwd)` and invalidates it when the guest calls `terminal.invalidate-env()`. So "an already-open terminal follows a version switch" works for new shells immediately. Existing shells pick it up through a `PATH` re-export prompt only [A: live env mutation of a running shell is not possible; the fork's behaviour likely applies on next command via a wrapper; not verified].
- **Rules.**
  - `path-prepend` entries must resolve inside `extension_work` or `extension_install`. The guest cannot put `/tmp` or worktree paths first on PATH.
  - `set` keys must match the manifest `vars` allowlist.
  - A fixed denylist applies: `LD_PRELOAD`, `LD_LIBRARY_PATH`, `DYLD_*`, `GIT_ASKPASS`, `SSH_ASKPASS`, `GIT_CONFIG_*`, `NODE_OPTIONS`, `PYTHONSTARTUP`, `BASH_ENV`, `ENV`, `PROMPT_COMMAND`, `ZDOTDIR`.
  - Local terminals only. Remote terminals are covered in §6 risks.

#### G10. System metrics and OS processes

WASM has no access to the OS, and the fork uses `sysinfo` directly (`fork/crates/cockpit_panel/Cargo.toml:19` [V], `fork/crates/processes_panel/src/processes_panel.rs:142` [V]). Zed already depends on `sysinfo` 0.39 with `features = ["system"]` (`zed/Cargo.toml:925` [V]). It would need `disk` and `network` added.

```wit
interface sys {
    record cpu { usage-pct: f32, frequency-mhz: u64 }
    record memory { total: u64, used: u64, swap-total: u64, swap-used: u64 }
    record disk { mount: string, total: u64, available: u64, fs: string, removable: bool }
    record net-iface { name: string, rx-bytes: u64, tx-bytes: u64 }
    record metrics { cpus: list<cpu>, memory: memory, disks: list<disk>, networks: list<net-iface>, uptime-s: u64, load: option<tuple<f32,f32,f32>> }
    record proc-info { pid: u32, parent: option<u32>, name: string, exe: option<string>, cmd: option<list<string>>,
                       cpu-pct: f32, memory: u64, status: string, user-is-self: bool, started-by-extension: bool }
    variant sys-error { not-granted(string), not-found, denied-by-user, os(string) }
    enum sig { term, kill, int, hup }

    metrics: func() -> result<metrics, sys-error>;                // sys:metrics; host samples at most every 1 s
    processes: func(filter: option<string>) -> result<list<proc-info>, sys-error>;   // sys:processes
    signal: func(pid: u32, s: sig) -> result<_, sys-error>;       // sys:signal; host-drawn confirmation unless pid is own child
}
```

- **Command lines.** `cmd` and `exe` are only populated with `include_cmdline = true`. The host applies the same secret redaction as the fork's `runtime_processes`, for example `--password=***` [A: the redaction rules are ported from `fork/crates/runtime_processes`, not reviewed].
- **Signals.** `signal` is limited to processes owned by the current user. It always shows a host modal naming the process, unless the PID is a child the extension spawned.

#### G11. OAuth device flow, keychain and secret handles

```wit
interface secrets {
    resource secret-ref { label: func() -> string; }           // opaque; bytes never cross into the guest unless `reveal` is granted
    variant secret-error { not-granted(string), not-found, denied-by-user, keychain(string) }
    store: func(key: string, username: string, value: list<u8>) -> result<secret-ref, secret-error>;
    open: func(key: string) -> result<option<secret-ref>, secret-error>;
    delete: func(key: string) -> result<_, secret-error>;
    list-keys: func(prefix: string) -> result<list<string>, secret-error>;
    reveal: func(s: borrow<secret-ref>) -> result<list<u8>, secret-error>;   // only with secrets.reveal = true (high severity)
    /// Host-drawn masked input (trusted chrome). Value goes straight to keychain.
    prompt: func(key: string, title: string, label: string) -> result<option<secret-ref>, secret-error>;
}

interface auth {
    record device-flow-config { provider-id: string, device-code-url: string, token-url: string,
                                client-id: string, scopes: list<string>, verification-hint: option<string> }
    variant auth-error { not-granted(string), cancelled, expired, denied, http(string) }
    /// Host performs RFC 8628: shows user code + verification URL in a host modal (copy button, open browser),
    /// polls token-url, stores the token in the keychain under `key`, returns a handle.
    device-flow: func(cfg: device-flow-config, key: string) -> result<secret-ref, auth-error>;
}
```

- **Key namespacing.** Keys are namespaced by the host as `zed-ext://<extension-id>/<key>` in `CredentialsProvider`. One extension can never read another's keys.
- **Secrets in requests.** `net.request` headers become `variant header-value { plain(string), secret(tuple<secret-ref, string>) }`. The string is a template such as `"Bearer {}"`. The host substitutes the secret at send time, and only if the request host matches the secret's `bound_hosts`. This is an exfiltration defense: a GitHub token can only go to `api.github.com`.
- **Secrets in processes.** `proc` `env-from-secret` (G7) works the same way. An example is the git clone that receives `GIT_CONFIG_*` auth headers, as in fork feature 15c.
- **Importing from `gh`.** The `gh auth token` import (fork `gh_cli.rs:61`) uses `proc.run-to-secret(opts, key)`. Stdout goes to the keychain, and the guest receives a handle.

#### G12. Git integration hooks

```wit
interface git {
    record repo { id: u64, worktree-id: option<u64>, work-dir: string, remotes: list<remote>, branch: option<string>, is-remote-project: bool }
    record remote { name: string, url: string, host: option<string> }
    record commit-stat { sha: string, author-name: string, author-email: string, time-s: i64, files: option<list<string>> }
    record status-entry { path: string, status: string }
    variant git-error { not-granted(string), not-found, unsupported-remote, io(string) }

    repositories: func() -> result<list<repo>, git-error>;                                  // git:read
    status: func(repo: u64) -> result<list<status-entry>, git-error>;                       // git:read
    ahead-behind: func(repo: u64) -> result<option<tuple<u32,u32>>, git-error>;
    recent-commits: func(repo: u64, max: u32, with-files: bool) -> result<list<commit-stat>, git-error>;  // replaces fork's recent_commit_stats
    get-config: func(repo: u64, key: string) -> result<option<string>, git-error>;          // git:config_read, allowlisted keys
    set-config: func(repo: u64, key: string, value: string) -> result<_, git-error>;        // git:config_write, local scope only
    clone: func(url: string, dest: path-ref, auth: option<secret-ref>) -> result<_, git-error>;  // git:clone; dest must be user_selected
}

// Exports (host -> guest), registered via manifest [git.hooks]
record remote-op { repo: u64, op: remote-op-kind, remote: remote, branch: option<string> }
enum remote-op-kind { push, pull, fetch }
variant hook-verdict { allow, deny(hook-denial) }
record hook-denial { message: string, actions: list<hook-action> }      // rendered as host toast with buttons
record hook-action { label: string, action: string }                     // extension-namespaced action
export before-remote-op: func(op: remote-op) -> result<hook-verdict, string>;   // 300 ms budget; on timeout -> manifest `on_timeout`
export after-remote-op: func(op: remote-op, success: bool, stderr-tail: string) -> result<option<hook-denial>, string>;  // retry offers
record credential-request { protocol: string, host: string, path: option<string>, repo: option<u64> }
variant credential-answer { pass, token(tuple<string, secret-ref>), quit }   // username + secret handle
export git-credential: func(req: credential-request) -> result<credential-answer, string>;  // 500 ms budget
```

**Core hooks (upstream).**

1. **Remote-op hooks.** `GitStore` calls the `before_remote_op` and `after_remote_op` registries around `Repository::push`/`pull`/`fetch` (`zed/crates/project/src/git_store.rs:8808` [V] for `push`).
2. **Credential routing.** A credential provider registry is consulted by Zed's existing askpass path for panel-initiated git (`zed/crates/askpass/src/askpass.rs` [V]). For terminals, Zed registers itself as a git credential helper, generalizing the fork's `git_credential` crate (`fork/crates/git_credential/src/git_credential.rs:87,131` [V]) into an upstream `zed --git-credential` mode. Its socket server routes `get` requests to the extension providers whose `hosts` match. The token goes to git through the helper protocol. It never enters the terminal env (the fork's rule, apps §16).
3. **Slots** (G-slots below) for the git panel's tab strip and footer.

#### G-slots. Contributions into host-owned surfaces

Zed's git panel has a closed `GitPanelTab { Changes, History }` (`zed/crates/git_ui/src/git_panel.rs:648` [V]). Instead of extensions patching host panels, the host exposes a **closed set of named slots**. Each slot renders an extension JSON subtree inside host chrome, with a size budget:

| Slot id | Host location | Content constraints |
|---|---|---|
| `git_panel.tab` | adds a tab next to Changes/History (`render_tab_bar`, `git_panel.rs:7714` [V]) | full subtree, lazy |
| `git_panel.footer` | inside `render_footer` (`git_panel.rs:7300` [V]) | one `row` of `avatar`/`text`/`button`/`menu`, height 1 line |
| `project_panel.header` | above the project tree | one `row` |
| `extensions.card_actions` | not exposed | (reserved, avoids spoofing of install UI) |
| `status_bar.*` | existing status items | already a surface |

Manifest: `[ui.slots.details] slot = "git_panel.tab", title = "Details", action = "helm::ActivateDetailsTab"`. Permission: `ui.surfaces` includes `"slot:git_panel.tab"`.

#### G13. Editor context-menu contributions

API.md's static entries with `when` cannot produce `Search npm for "<pkg>"`. Two-step model:

1. **Static declaration** in the manifest (menu id, `when` predicate, max entries). Zed only consults the extension when the predicate matches the `KeyContext`, for example `Editor && (extension == ts || extension == tsx || extension == js)`.
2. **Optional dynamic labels** via the `menu-items` export. It receives `selection-info` (G4) only if `editor:read_selection` is granted, and has a 30 ms budget. If the budget is missed, the static fallback label is shown. Clicking dispatches `InvokeExtensionAction` with `params` the guest returned.

```wit
record menu-request { menu-id: string, context: option<selection-info> }
record menu-entry { id: string, label: string, icon: option<string>, action: string, params-json: option<string> }
export menu-items: func(req: menu-request) -> result<list<menu-entry>, string>;
```

**Host side.** A `MenuContributionRegistry` global, consulted from `deploy_context_menu` (`zed/crates/editor/src/mouse_context_menu.rs:158`, menu built at `:246` [V]). The fork's direct edit (`fork/crates/editor/src/mouse_context_menu.rs:248-321` [V]) then becomes extension data.

#### G14. File handlers (custom editors for file types)

```toml
[ui.file_handlers.flow_designer]
path_glob      = ["**/*.flow.json"]          # globset on worktree-relative path; double extensions supported
priority       = "default"                    # default | option (appears under "Open With…") 
buffer_mode    = "project_buffer"             # project_buffer: host opens the real Buffer; guest edits via editor.apply-edits
open_as_text_action = true                    # host adds "Open as Text" (fork's Raw button) automatically
item           = "designer"                   # refers to [ui.items.designer]
```

- **Host side.** One generic `ExtensionFileItem` implementing `workspace::ProjectItem` (`zed/crates/workspace/src/item.rs:1220` [V]). It is registered once through `register_project_item` (`zed/crates/workspace/src/workspace.rs:1171` [V]). Its `project::ProjectItem::try_open` (`zed/crates/project/src/project.rs:195` [V]) consults the handler registry by glob.
- **Undo and save.** Dirty state, save and undo are the Buffer's, so the guest never writes the file directly. This replaces `fork/crates/designer_panel/src/designer_panel.rs:313` [V].
- **Permission.** `editor` `op = ["read_buffer","edit"]` with `glob` matching the handler's globs. This is checked at load: a handler whose glob is not covered fails validation.

#### G15. JSON schema contributions

```toml
[json_schemas.flow]
schema    = "schemas/flow.schema.json"
file_globs = ["**/*.flow.json", "**/*.flow-http/**/*.body.json"]
```

The host registers the schema with `json_schema_store` for the JSON language server, replacing the fork's patch to `json_schema_store.rs` and `languages/src/json.rs` (apps §10). It needs no capability, since it is static data. The registry shows it under "Provides".

#### G16. Inter-extension communication

Bundling removes most cross-panel links (§3, packaging). Two links remain across separate extensions: Helm to git hooks (same bundle) and Node to the Flows Task Chain. Host-routed IPC with manifest contracts:

```toml
[ipc.provides.health]                     # a service other extensions may call
schema   = "ipc/health.schema.json"       # JSON schema for request/response
accept_from = ["forge.dashboard"]         # explicit allowlist of caller extension ids ("*" = any, flagged)

[[ipc.consumes]]
extension = "forge.flows"
service   = "task-chain"
```

```wit
interface ipc {
    variant ipc-error { not-granted(string), not-installed, timeout, schema(string), remote(string) }
    call: func(extension-id: string, service: string, request-json: string, timeout-ms: u32) -> result<string, ipc-error>;
    publish: func(topic: string, payload-json: string) -> result<_, ipc-error>;   // topics declared in [ipc.topics.*]
    subscribe: func(extension-id: string, topic: string) -> result<_, ipc-error>; // delivered as ipc-message events
}
export ipc-request: func(caller: string, service: string, request-json: string) -> result<string, string>;
```

- **Validation and limits.** The host validates payloads against both sides' schemas, caps them at 256 KiB, and enforces timeouts.
- **No shared state.** Calls are queued on the callee's serialized queue. Nothing is shared except JSON.
- **Opening surfaces.** Opening another extension's surface is just `dispatch-action` of its namespaced action, which `InvokeExtensionAction` already covers.

#### G17. Activation, background work and workspace events

```toml
[activation]
events = ["on_startup", "workspace_contains:package.json", "workspace_contains:**/*.csproj",
          "on_action:node::*", "on_surface:panel:node", "on_file_handler:flow_designer"]
background = true          # receive workspace events without an open surface
```

```wit
interface workspace {
    record worktree-info { id: u64, root: string, is-local: bool, visible: bool }
    record workspace-info { id: u64, worktrees: list<worktree-info>, active-path: option<string> }
    current: func() -> result<workspace-info, string>;
    open-path: func(p: path-ref) -> result<_, string>;                     // workspace:open_path
    open-folder: func(p: path-ref, mode: open-mode) -> result<_, string>;  // workspace:open_folder (Helm clone-and-switch)
    enum open-mode { new-window, replace-current, add-to-current }
    reveal-in-os: func(p: path-ref) -> result<_, string>;                  // workspace:reveal
}
// new ui-event / background-event variants
variant background-event {
    workspace-opened(u64), workspace-closed(u64), worktree-added(worktree-info), worktree-removed(u64),
    active-item-changed(option<string>), file-saved(string), fs-changed(tuple<u64, list<string>>),  // fs.watch
    timer(u64), task-exit(tuple<u64, option<s32>>), proc-exit(tuple<u64, option<s32>>), proc-match(tuple<u64, string>),
    ipc-message(tuple<string, string, string>), settings-changed, grant-changed(string),
}
export on-background-event: func(ev: background-event) -> result<list<ui-command>, string>;
```

The guest is still event-driven, with no free-running threads. "Background services" means activation without UI, plus event delivery and timers, plus long-lived host resources such as processes, watchers and sockets.

#### G18. Persistence

```wit
interface kv {
    enum kv-scope { extension, workspace, project }   // workspace = per workspace id; project = per worktree root path hash
    variant kv-error { not-granted(string), quota, io(string) }
    get: func(scope: kv-scope, key: string) -> result<option<string>, kv-error>;
    set: func(scope: kv-scope, key: string, value: string) -> result<_, kv-error>;
    delete: func(scope: kv-scope, key: string) -> result<_, kv-error>;
    list: func(scope: kv-scope, prefix: string) -> result<list<string>, kv-error>;
}
```

- **Storage.** Backed by `db::kvp::KeyValueStore` (`zed/crates/db/src/kvp.rs:12`, `write_kvp` `:72` [V]) with keys `ext:<id>:<scope>:<key>`. Quota defaults to 4 MiB per extension. Data is removed on uninstall, after a prompt.
- **Settings.** `[settings] schema = "settings.schema.json"` adds an `extensions.<id>` settings namespace. It is readable through `get-settings` (category `extension`), and `settings_ui` renders it from the schema automatically. This replaces the fork's `settings_content/src/panels.rs` and `page_data.rs` edits (apps §26). Writing requires `settings` `op = ["write_own"]`.

#### G19. Raw TCP/TLS sockets

DB drivers need TCP, and Flows `WaitForPort` needs a localhost connect. Two options, both checked against `net` capabilities:

- **(a) WASI sockets** with `WasiCtxBuilder::socket_addr_check` and `allow_ip_name_lookup` [A: present in wasmtime-wasi 48, which Zed pins at `zed/Cargo.toml:1000` [V]]. Today's `build_wasi_ctx` enables no network (`zed/crates/extension_host/src/wasm_host.rs:742-763` [V]). The check callback evaluates the resolved address against `net` `hosts`, `ports` and `resolve`. Pros: guest drivers can use std-like sockets. Cons: the host matches IPs, not hostnames, after the guest's own DNS lookup. Host-side `ip_name_lookup` must therefore record name-to-IP pairs, so the check can map an IP back to the host the guest asked for.
- **(b) Host `tcp-stream`/`tls-stream` resources** in `net` (`connect-tcp(host, port, tls: option<tls-options>)`). The host connects by hostname, verifies TLS with the native roots, and enforces the policy on the name. The data path is `read`/`write` chunks.

**Recommendation.** Use (b) for `tls`, because certificate validation stays on the host, and allow (a) for plain `tcp` only to listed hosts. Drivers must be sans-io or `futures-io` adapters [A: `tokio-postgres` and `mysql_async` assume tokio; porting effort counted in feature 9].

#### G20. Remote images

`image` nodes accept `url`, but only `https`, and only hosts in a `net` capability whose `purpose` includes `"images"`. The host fetches through its HTTP client with size caps (2 MiB) and decodes images through GPUI's asset pipeline. Markdown images follow the same rule. This matches the fork's README hardening (apps §22: GitHub-only images, https only).

#### G21. Toolchain downloads

Extend `download-file` with `sha256: option<string>`, `max-bytes`, `preserve-mode: bool` (keep the executable bits from tar), and `.tar.xz` support. Also add `fs.remove` for `extension_work` cleanup. Node and Python toolchains then live in `<data>/extensions/work/<id>/runtimes/...` (API.md §1.7). A `download_file` capability is required and checked against the manifest (correction d).

#### G22. Sidecars (escape hatch)

```toml
[sidecars.oracle_bridge]
download = { url = "https://github.com/org/oracle-bridge/releases/download/v1.2.0/oracle-bridge-{os}-{arch}.tar.gz",
             sha256 = { "linux-x86_64" = "…", "macos-aarch64" = "…", "windows-x86_64" = "…" } }
binary   = "oracle-bridge"
protocol = "jsonrpc-stdio"         # jsonrpc-stdio | lsp-stdio | jsonl-stdio | oneshot-json | raw
timeout  = "30s"                     # per call for oneshot-json; per request for jsonrpc
```

- **Running a sidecar.** It is spawned only via `proc` with `resolve = "extension_binary"`. The sha256 is checked on every spawn.
- **Consent.** Consent shows "runs native code outside the sandbox (critical)".
- **Precedent.** This is the same trust posture as today's language servers, but made explicit.
- **Protocols.** The host speaks these on behalf of the guest, which only sees parsed JSON:
  - **`oneshot-json`** is the Forge (Tauri) backend contract (thread.md, mentenaz 2026-11-04; §7.3):
    - a call is `binary --function <name> --Arguments <raw>`;
    - stdout carries exactly one JSON object;
    - on failure, the binary exits non-zero and writes the error to stderr;
    - the host kills the process after `timeout`;
    - `binary --introspect` lists the available functions. The host runs it once at install and shows the result in the consent UI.
  - **`jsonl-stdio`** is the persistent mode mentenaz says a system monitor needs: newline-delimited JSON objects in both directions. Output can be piped straight into a `ui-data` store (G1, G7), like `stdout-to`.
- **Limits.** A sidecar is the Forge backend model, admitted under R3 only as an explicit, critical-severity exception. The host constrains it only at spawn time: args, env, cwd, timeout, and best-effort `sandbox` per OS. Once running, it has the user's full permissions, which is exactly the limit mentenaz states. It is therefore never the default path (§7.3).

#### G23 and G24. Workspace navigation and host confirmations

Navigation is covered by the `workspace` interface (G17) and `open_url` (API.md). Confirmations:

```wit
// in ui
record confirm-args { title: string, message: string, detail: option<string>, confirm-label: string, destructive: bool }
confirm: func(args: confirm-args) -> result<bool, string>;   // host-drawn (window.prompt); async; never spoofable by a tree
```

---

## 3. Per-feature migration plan

**Packaging recommendation [A].** Ship five extensions, so in-process entity coupling becomes in-extension code instead of IPC:

| Bundle (extension id) | Features |
|---|---|
| `forge.system` | 1 Cockpit, 3 Processes |
| `forge.runtimes` | 2 Dashboard, 4 Script Runner, 5 Node, 6 Python, 7 .NET, 8 Rust, 11 Package managers, 12 npm bootstrap, 17 PATH providers |
| `forge.database` | 9 Database workbench (+ coordinates with 21 `sqls`) |
| `forge.flows` | 10 Flows |
| `forge.helm` | 13 Helm, 14 Helm Workspace, 15 git panel features, 16 credential answers |

Each feature below lists its permission block as if it were standalone. A bundle's manifest is the union of its features' blocks.

**Effort scale [A]:** S < 1 week, M 1–3 weeks, L 1–2 months, XL > 2 months, for one engineer, once the required API phase exists. Effort covers porting the backend to host resources (no tokio, reqwest or sysinfo inside the guest [A: those crates do not work as-is on `wasm32-wasip2` under Zed's WASI context]) and rewriting the UI as JSON.

**Common header** for every manifest below, omitted after F1:

```toml
schema_version = 2
lib = { kind = "Rust", version = "0.9.0" }
```

### F1. Cockpit (system metrics sidebar)

- **Surfaces.** A left dock panel `cockpit`. The dock's status-bar button comes automatically. Actions `cockpit::ToggleFocus` and `cockpit::OpenDashboard`. The second is handled in the same bundle, which removes the fork's cycle workaround.
- **JSON sketch.**

```json
{ "schema": "zed-ui/1", "root": { "type": "column", "key": "root", "gap": "sm", "padding": "sm", "children": [
  { "type": "row", "key": "hdr", "justify": "between", "children": [
    { "type": "text", "key": "cores", "value": "16 cores · up 3d 4h", "style": "muted" },
    { "type": "button", "key": "dash", "label": "Dashboard", "size": "sm", "on_click": { "local": "dispatch_action", "action": "cockpit::OpenDashboard" } } ] },
  { "type": "row", "key": "gauges", "gap": "md", "children": [
    { "type": "gauge", "key": "cpu", "label": "CPU", "value": 37, "max": 100, "thresholds": [{ "at": 85, "color": "status.error" }] },
    { "type": "gauge", "key": "ram", "label": "RAM", "value": 61, "max": 100 } ] },
  { "type": "accordion", "key": "cores_s", "title": "Per core", "expanded": true, "child":
    { "type": "chart", "key": "cores_c", "kind": "bar", "source": "series:cores" } },
  { "type": "accordion", "key": "hist", "title": "History", "child":
    { "type": "chart", "key": "hist_c", "kind": "area", "source": "series:cpu_ram", "x": { "kind": "time" }, "y": { "min": 0, "max": 100, "unit": "%" } } },
  { "type": "accordion", "key": "net", "title": "Network", "child": { "type": "chart", "key": "net_c", "kind": "area", "source": "series:net" } },
  { "type": "accordion", "key": "disks", "title": "Disks", "child": { "type": "data_grid", "key": "disks_g", "source": "rows:disks", "sort": "local" } } ] } }
```

- **WIT.**
  - `sys.metrics` on `ui.request-timer(2000)`.
  - `ui-data.create-series`/`push-points` with a capacity of 30 samples. The ring buffer lives on the host, replacing the panel's own ring buffer.
  - `ui-data.replace-rows` for disks.
  - Timers stop on `lifecycle hidden`, so a closed panel costs nothing.
- **Permissions.**

```toml
[ui.panels.cockpit]
title = "Cockpit"
icon = "icons/cockpit.svg"
position = "left"
activation_priority = 20

[[capabilities]]
kind = "ui"
surfaces = ["panel"]
max_panels = 1
reason = "Show live system metrics in a sidebar"

[[capabilities]]
kind = "sys"
op = ["metrics"]
min_interval = "1s"
include_cmdline = false
reason = "Read CPU, memory, disk and network usage"
```

- **Stays in core.** Nothing beyond the API. Enabling `sysinfo` features `disk` and `network` in the host is a one-line change.
- **Feasibility.** **Full**. Effort **S**. Phase 2.

### F2. Dashboard (project health tab)

- **Surfaces.** A pane item `dashboard`, singleton and serializable, opened by `cockpit::OpenDashboard`.
- **JSON sketch.**

```json
{ "type": "grid", "key": "cards", "min_column_width": { "px": 320 }, "gap": "md", "children": [
  { "type": "card", "key": "node", "title": "Node.js", "status": "unknown",
    "actions": [{ "type": "button", "key": "scan_node", "label": "Scan", "on_click": { "id": "scan", "payload": { "eco": "node" } } }],
    "child": { "type": "column", "key": "node_b", "children": [
      { "type": "stat", "key": "n_out", "label": "Outdated", "value": "—" },
      { "type": "stat", "key": "n_vul", "label": "Vulnerable", "value": "—", "status": "unknown" },
      { "type": "data_grid", "key": "n_find", "source": "rows:node_findings", "max_height": { "px": 200 } } ] } },
  { "type": "card", "key": "git", "title": "Git", "child": { "type": "column", "key": "git_b", "children": [
      { "type": "text", "key": "branch", "value": "main · 3 changed · ↑1 ↓0" } ] } },
  { "type": "card", "key": "sys", "title": "System", "child": { "type": "chart", "key": "sys_c", "kind": "area", "source": "series:cpu_ram" } } ] }
```

- **WIT.**
  - Inside `forge.runtimes`: each ecosystem's scanner is plain guest code, called directly.
  - Across bundles: the Cockpit series comes from `ipc.subscribe("forge.system","metrics")`, or the Dashboard calls `sys.metrics` itself, which is simpler and recommended.
  - Git: `git.repositories`, `git.status`, `git.ahead-behind`.
  - Restore: `serialize-instance`.
- **Permissions.**

```toml
[ui.items.dashboard]
title = "Dashboard"
icon = "builtin:layout"
singleton = true
serializable = true

[[capabilities]]
kind = "ui"
surfaces = ["item"]
max_open_items = 1

[[capabilities]]
kind = "git"
op = ["read"]
reason = "Show branch, changes and ahead/behind"

[[capabilities]]
kind = "sys"
op = ["metrics"]
min_interval = "2s"
# plus the union of F5–F8 scan permissions (same bundle), see those features
```

- **Stays in core.** Nothing. The fork's in-process `workspace.panel::<T>()` graph (apps §2) is replaced by code inside one bundle, plus optional IPC.
- **Feasibility.** **Partial**: Full inside the bundle, and IPC is needed only if split across bundles. Effort **M**. Phase 2 (cards and charts), with git in Phase 4 (until then, `proc git status` is a fallback).

### F3. Processes panel

- **Surfaces.** A bottom dock panel `processes`. Keybinding `ctrl-k x`. Actions `ToggleFocus`, `KillSelected`, `ToggleAdoptSelected`. A row context menu.
- **JSON sketch.**

```json
{ "type": "column", "key": "root", "children": [
  { "type": "search_box", "key": "filter", "placeholder": "Filter processes", "on_input": { "id": "filter", "debounce_ms": 150 } },
  { "type": "data_grid", "key": "procs", "source": "rows:procs", "sort": "local", "filter": { "local_from": "filter" },
    "selection": "single", "capture_keys": ["a", "delete", "backspace"],
    "on_key": { "id": "row_key" }, "on_context_menu": { "id": "row_menu" } } ] }
```

- **WIT.**
  - `sys.processes` every 2 s into `ui-data.replace-rows`. Sort and filter run on the host, so a refresh never fights the user's sort.
  - Killing calls `sys.signal(pid, term)`. The host draws the confirmation (G24), replacing the fork's inline banner, which must not be spoofable.
  - The adopted set is held in guest memory, plus `kv` (workspace scope) if it should persist (the fork keeps it in memory only).
- **Permissions.**

```toml
[ui.panels.processes]
title = "Processes"
icon = "builtin:list_tree"
position = "bottom"
activation_priority = 24

[[ui.actions]]
name = "processes::KillSelected"
title = "Processes: Kill Selected"
palette = true

[[ui.keybindings]]
key = "ctrl-k x"
action = "processes::ToggleFocus"

[[capabilities]]
kind = "ui"
surfaces = ["panel", "context_menu"]

[[capabilities]]
kind = "sys"
op = ["processes", "signal"]
include_cmdline = true            # redacted by host
signal_scope = "same_user"        # never other users / root
signals = ["term", "kill"]
confirm = "each"                  # host modal per signal (cannot be lowered by the extension)
reason = "List processes and stop the ones you choose"
```

- **Stays in core.** Nothing.
- **Feasibility.** **Full**. Effort **S**. Phase 2.

### F4. Script Runner (shared shell console)

- **Surfaces.** A bottom dock panel `script_runner`. Keybinding `ctrl-k k`.
- **Recommended re-architecture.** Other features must not route into this panel. They use `terminal.run-task` (G8), so their output lands in Zed's terminal panel with stop and rerun. The Script Runner remains as the "type a command, see streaming output, copy the SPFx URL" console.
- **JSON sketch.**

```json
{ "type": "column", "key": "root", "children": [
  { "type": "row", "key": "bar", "gap": "xs", "children": [
    { "type": "input", "key": "cmd", "user_command": true, "placeholder": "Command", "grow": 1, "on_submit": { "id": "run" } },
    { "type": "button", "key": "stop", "label": "Stop", "variant": "danger", "enabled": false, "on_click": { "id": "stop" } } ] },
  { "type": "banner", "key": "spfx", "visible": false, "kind": "info",
    "child": { "type": "row", "key": "u", "children": [
      { "type": "text", "key": "frag", "value": "?debug=true&noredir=true&debugManifestsFile=…", "style": "code", "selectable": true },
      { "type": "icon_button", "key": "copy", "icon": "builtin:copy", "tooltip": "Copy", "on_click": { "local": "copy", "from_node": "frag" } } ] } },
  { "type": "log", "key": "out", "source": "log:run", "follow": true, "link_detection": ["url", "path"] },
  { "type": "text", "key": "exit", "value": "exit 0", "style": "muted" } ] }
```

- **WIT.**
  - `proc.spawn-user-command("cmd", "panel:script_runner", cwd=worktree(id), stdout-to="log:run")`. The host pipes output directly into the log store.
  - `watch-patterns = ["https://[^ ]+/_layouts/workbench\\.aspx\\?[^ ]+", "debugManifestsFile=[^ ]+"]` fire `proc-match`.
  - `process.kill()` kills the process group, so the fork's `Command::new("kill")` at `fork/crates/script_runner_panel/src/script_runner_panel.rs:495` [V] is gone.
  - `clipboard` write.
- **Permissions.**

```toml
[ui.panels.script_runner]
title = "Script Runner"
icon = "builtin:terminal"
position = "bottom"
valid_positions = ["left", "right", "bottom"]
activation_priority = 23

[[capabilities]]
kind = "ui"
surfaces = ["panel"]

[[capabilities]]
kind = "proc"
mode = "user_attested_shell"     # only text typed by the user into a user_command input
shells = { unix = "/bin/sh", windows = "cmd.exe" }
cwd = ["worktree"]
env = { inherit = "shell_env", deny = ["*_TOKEN", "*_SECRET"] }   # the user's own shell env; deny-pattern hides obvious secrets
stdout = "pipe"
stderr = "pipe"
max_output = "64MiB"
long_running = true
timeout = "none"
max_concurrent = 1
kill = "own_children_only"
process_group = true
reason = "Run commands you type and stream their output"

[[capabilities]]
kind = "clipboard"
op = ["write"]
```

- **Stays in core.** User-attested input provenance, in the host `input` node.
- **Feasibility.** **Full**. Effort **S**. Phase 2.

### F5. Node panel

- **Surfaces.** A left dock panel `node` with collapsible sections. Keybinding `ctrl-k o`. A status item showing the active version (optional). A modal for "Create .nvmrc".
- **JSON sketch** (sections abbreviated).

```json
{ "type": "scroll", "key": "s", "child": { "type": "column", "key": "root", "children": [
  { "type": "row", "key": "proj", "children": [ { "type": "dropdown", "key": "project", "options": [{ "value": "web", "label": "web/package.json" }], "selected": "web", "on_select": { "id": "project" } } ] },
  { "type": "accordion", "key": "versions", "title": "Node Versions · v20.11.1 (Auto: .nvmrc)", "expanded": true, "child":
    { "type": "column", "key": "v", "children": [
      { "type": "segmented", "key": "mode", "options": [{ "value": "auto", "label": "Auto" }, { "value": "system", "label": "System" }, { "value": "pinned", "label": "Pinned" }], "selected": "auto", "on_select": { "id": "mode" } },
      { "type": "list", "key": "installed", "items": [ { "type": "row", "key": "v20", "children": [ { "type": "text", "key": "t", "value": "v20.11.1" }, { "type": "icon_button", "key": "rm", "icon": "builtin:trash", "tooltip": "Uninstall", "on_click": { "id": "uninstall", "payload": { "v": "20.11.1" } } } ] } ] },
      { "type": "button", "key": "install", "label": "Install version…", "on_click": { "id": "install_modal" } } ] } },
  { "type": "accordion", "key": "scripts", "title": "Scripts", "child": { "type": "data_grid", "key": "scripts_g", "source": "rows:scripts", "on_activate": { "id": "run_script" }, "on_context_menu": { "id": "script_menu" } } },
  { "type": "accordion", "key": "outdated", "title": "Outdated (4)", "child": { "type": "data_grid", "key": "out_g", "source": "rows:outdated" } },
  { "type": "accordion", "key": "vulns", "title": "Vulnerabilities", "child": { "type": "data_grid", "key": "vul_g", "source": "rows:vulns" } },
  { "type": "accordion", "key": "procs", "title": "Processes", "child": { "type": "data_grid", "key": "np_g", "source": "rows:node_procs", "on_context_menu": { "id": "proc_menu" } } },
  { "type": "accordion", "key": "spfx", "title": "SPFx", "visible": false, "child": { "type": "list", "key": "sppkg", "items": [] } } ] } }
```

- **WIT.**
  - **Project scan.** `fs.list`/`read-text-file` on the worktree, for `**/package.json`, `.nvmrc`, `.node-version` and `.yo-rc.json`.
  - **Watching.** `fs.watch` on manifests and lockfiles, delivering `fs-changed` events.
  - **Edits.** `editor.apply-edits` on `package.json` when the file is open in a buffer, otherwise `fs.write-file`. The script edits keep JSON formatting through a guest JSON-edit library.
  - **Versions.**
    1. `net.fetch https://nodejs.org/dist/index.json`.
    2. `download-file(... sha256 from SHASUMS256.txt, preserve-mode)` into `extension_work/runtimes/node/<v>`.
    3. Remove a version with `fs.remove`.
    4. The `fnm` library dependency is replaced by guest logic [A: the fnm fork's features beyond version resolution and download are not needed].
  - **Package commands.** `terminal.run-task {npm|yarn|pnpm install/update/audit}`, or `proc.run` with `--json` for `npm outdated`/`npm audit` parsing.
  - **Processes.** `sys.processes(filter="node")` and `sys.signal`.
  - **PATH injection.** The `terminal-env` export returns `path-prepend = [extension_work/runtimes/node/<v>/bin]` for the workspace (G9).
  - **State.** `kv` (workspace scope) for open sections and version choice, replacing the `node_panel.rs:89-93` KVP keys.
  - **Task Chain button.** `dispatch-action("flows::AddTaskChain", {root})`. Across bundles this is resolved through `InvokeExtensionAction`.
- **Permissions.**

```toml
[ui.panels.node]
title = "Node"
icon = "icons/node.svg"
position = "left"
activation_priority = 26

[activation]
events = ["workspace_contains:**/package.json", "on_surface:panel:node"]
background = true

[[capabilities]]
kind = "ui"
surfaces = ["panel", "modal", "status_item"]

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list", "watch"]
glob = ["**/package.json", "**/.nvmrc", "**/.node-version", "**/.yo-rc.json", "**/package-lock.json", "**/yarn.lock", "**/pnpm-lock.yaml", "**/sharepoint/solution/*.sppkg", "!**/node_modules/**"]
follow_symlinks = "within_scope"
max_file_size = "8MiB"
max_watchers = 16
reason = "Find Node projects and react when manifests change"

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["write", "create"]
glob = ["**/package.json", "**/.nvmrc", "!**/node_modules/**"]
max_total_write = "4MiB"
reason = "Edit package.json scripts and create .nvmrc"

[[capabilities]]
kind = "fs"
scope = "extension_work"
op = ["read", "stat", "list", "write", "create", "delete", "rename", "chmod_exec"]
glob = ["runtimes/node/**"]
max_total_write = "4GiB"
reason = "Store built-in Node.js versions"

[[capabilities]]
kind = "download_file"
host = "nodejs.org"
path = ["dist", "**"]
sha256 = "required"
max_bytes = "200MiB"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["nodejs.org", "registry.npmjs.org"]
ports = [443]
methods = ["GET"]
paths = ["/dist/**", "/*", "/@*/*", "/-/npm/v1/security/**"]
redirects = "same_host"
allow_credentials = false
resolve = "public_only"
max_response_body = "32MiB"
rate_limit = "120/min"
reason = "Resolve Node versions, package metadata and advisories"

[[capabilities]]
kind = "proc"
command = "npm"
resolve = "path_lookup"
args = ["outdated", "--json", "**"]
cwd = ["worktree"]
env = { inherit = "allowlisted", allow = ["PATH", "HOME", "USERPROFILE", "APPDATA", "LANG", "npm_config_*"] }
timeout = "120s"
max_output = "16MiB"
reason = "Read outdated packages"

[[capabilities]]
kind = "proc"
command = "npm"
resolve = "path_lookup"
args = ["audit", "--json", "**"]
deny_args = ["--fix", "fix"]
cwd = ["worktree"]
timeout = "120s"
max_output = "32MiB"

[[capabilities]]
kind = "terminal"
op = ["run_task"]
commands = ["npm", "yarn", "pnpm", "node", "npx"]       # the program of each task; args shown to user
confirm = "first_time"
cwd = ["worktree"]
reason = "Run installs and scripts in Zed's terminal"

[[capabilities]]
kind = "terminal"
op = ["env"]
path_prepend = ["extension_work:runtimes/node/*/bin"]
vars = []
reason = "Put the selected Node.js version first on PATH in terminals"

[[capabilities]]
kind = "sys"
op = ["processes", "signal"]
name_filter = ["node", "node.exe"]
signal_scope = "same_user"
confirm = "each"

[[capabilities]]
kind = "kv"
scopes = ["workspace", "extension"]
max_bytes = "1MiB"

[[capabilities]]
kind = "workspace"
op = ["reveal"]          # "Open folder" (fork used explorer/opener at node_backend.rs:78,97)
```

- **Stays in core.** The `TerminalEnvProvider` hook (G9, upstream).
- **Feasibility.** **Partial**: Full once G9 lands. Effort **L**. Phases 2 and 3, with PATH in Phase 4.

### F6. Python panel

- **Surfaces.** A left dock panel `python`. Keybinding `ctrl-k i`. A modal for the install confirmation naming version, size and source, as the fork does.
- **JSON sketch.** The same structure as F5. Sections are Versions, Environment (`stat` rows: interpreter, pip, venv), Frameworks (`list` of `{label:"Django", detail:"python manage.py runserver · :8000"}` with a `button` Run), Packages, Outdated, Vulnerabilities and Processes.
- **WIT.**
  - **Versions.** `net.fetch raw.githubusercontent.com/.../latest-release.json` (`fork/crates/python_versions/src/python_versions.rs:39` per apps §6), then `ui.confirm` (G24), then `download-file(sha256)` into `extension_work/runtimes/python/<v>`.
  - **Packages.** `proc.run python -m pip list --format=json`, `pip list --outdated --format=json`, and `python -m venv` via `terminal.run-task`.
  - **Metadata and advisories.** `net` to `pypi.org/pypi/<pkg>/json` and `api.osv.dev/v1/querybatch` (POST).
  - **Frameworks.** Framework run commands go through `terminal.run-task`.
  - **PATH, processes, state.** `terminal-env` export (PATH), `sys.processes` (flask, uvicorn, manage.py), and `kv` for the per-workspace active version.
- **Permissions.**

```toml
[ui.panels.python]
title = "Python"
icon = "icons/python.svg"
position = "left"
activation_priority = 25

[activation]
events = ["workspace_contains:**/pyproject.toml", "workspace_contains:**/requirements*.txt", "workspace_contains:**/manage.py", "on_surface:panel:python"]

[[capabilities]]
kind = "ui"
surfaces = ["panel", "modal"]

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list", "watch"]
glob = ["**/pyproject.toml", "**/requirements*.txt", "**/setup.py", "**/setup.cfg", "**/Pipfile*", "**/manage.py", "**/.python-version", "**/pyvenv.cfg", "**/.venv/bin/python*", "**/venv/Scripts/python.exe", "!**/node_modules/**", "!**/site-packages/**"]
max_file_size = "4MiB"
max_watchers = 16

[[capabilities]]
kind = "fs"
scope = "extension_work"
op = ["read", "stat", "list", "write", "create", "delete", "rename", "chmod_exec"]
glob = ["runtimes/python/**"]
max_total_write = "6GiB"

[[capabilities]]
kind = "download_file"
host = "github.com"
path = ["astral-sh", "python-build-standalone", "releases", "download", "**"]
sha256 = "required"
max_bytes = "150MiB"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["raw.githubusercontent.com", "pypi.org", "api.osv.dev"]
ports = [443]
methods = ["GET", "POST"]
paths = ["/astral-sh/python-build-standalone/latest-release/latest-release.json", "/pypi/*/json", "/v1/querybatch", "/v1/vulns/*"]
redirects = "same_host"
resolve = "public_only"
max_request_body = "1MiB"
max_response_body = "16MiB"
rate_limit = "120/min"

[[capabilities]]
kind = "proc"
command = "python"                 # also python3 / py: one entry each, same shape
resolve = "path_lookup"
alt_binaries = ["extension_work:runtimes/python/*/bin/python3", "worktree:**/.venv/bin/python"]   # explicit allowed interpreters
args = ["-m", "pip", "list", "**"]
deny_args = ["-c", "-"]
cwd = ["worktree"]
env = { inherit = "allowlisted", allow = ["PATH", "HOME", "LANG", "VIRTUAL_ENV", "PIP_*"] }
timeout = "60s"
max_output = "8MiB"

[[capabilities]]
kind = "terminal"
op = ["run_task", "env"]
commands = ["python", "python3", "py", "pip", "flask", "uvicorn", "pytest", "gradio"]
confirm = "first_time"
path_prepend = ["extension_work:runtimes/python/*/bin"]
vars = ["VIRTUAL_ENV"]

[[capabilities]]
kind = "sys"
op = ["processes", "signal"]
name_filter = ["python*", "flask", "uvicorn", "gunicorn"]
signal_scope = "same_user"
confirm = "each"

[[capabilities]]
kind = "kv"
scopes = ["workspace"]
```

- **Stays in core.** G9 (PATH).
- **Feasibility.** **Partial**. Effort **L**. Phases 2, 3 and 4.

### F7. .NET panel

- **Surfaces.** A left dock panel `dotnet` with tooltip "Dotnet". Sections: SDK, Projects, Quick actions, Packages, Outdated, Vulnerabilities and Processes.
- **JSON sketch.** `accordion` sections with `data_grid` sources `rows:projects`, `rows:packages`, `rows:outdated` and `rows:vulns`. Quick actions are a `row` of `button`s (`Run`, `Restore`, `Build`, `Test`) that emit `on_click {id:"task", payload:{verb:"restore", project:"…"}}`.
- **WIT.**
  - `fs` reads of `**/*.csproj` and `**/*.sln`.
  - `proc.run dotnet --list-sdks`, `dotnet list <proj> package --outdated --format json`, and `--vulnerable`.
  - `terminal.run-task dotnet restore|build|run|test`.
  - `net` to `api.nuget.org` for metadata.
  - The fork's `cmd /C dotnet` fallback (`dotnet_backend.rs:478`, apps §7) is dropped. The host resolves `dotnet.exe` itself.
- **Permissions.**

```toml
[ui.panels.dotnet]
title = "Dotnet"
icon = "icons/dotnet.svg"
position = "left"
activation_priority = 27

[activation]
events = ["workspace_contains:**/*.csproj", "workspace_contains:**/*.sln"]

[[capabilities]]
kind = "ui"
surfaces = ["panel"]

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list", "watch"]
glob = ["**/*.csproj", "**/*.sln", "**/Directory.Packages.props", "**/global.json", "!**/bin/**", "!**/obj/**"]

[[capabilities]]
kind = "proc"
command = "dotnet"
resolve = "path_lookup"
args = ["--list-sdks"]
timeout = "15s"

[[capabilities]]
kind = "proc"
command = "dotnet"
resolve = "path_lookup"
args = ["list", "*", "package", "**"]
arg_patterns = ["^[^;&|`$]+\\.(csproj|sln)$"]
cwd = ["worktree"]
env = { inherit = "allowlisted", allow = ["PATH", "HOME", "USERPROFILE", "DOTNET_*", "NUGET_*"] }
timeout = "120s"
max_output = "16MiB"

[[capabilities]]
kind = "terminal"
op = ["run_task"]
commands = ["dotnet"]
confirm = "first_time"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["api.nuget.org"]
methods = ["GET"]
paths = ["/v3/index.json", "/v3/registration5-semver1/**"]
resolve = "public_only"

[[capabilities]]
kind = "sys"
op = ["processes", "signal"]
name_filter = ["dotnet", "dotnet.exe"]
signal_scope = "same_user"
confirm = "each"
```

- **Stays in core.** Nothing.
- **Feasibility.** **Full**. Effort **M**. Phase 2.

### F8. Rust panel

- **Surfaces.** A left dock panel `rust`. Keybinding `ctrl-k c`.
- **JSON sketch.**
  - A header: `stat` rows for `rustc` and `cargo`.
  - A crates `list`, plus a `search_box` that is `visible` when there are more than 8 crates.
  - Quick actions as a `row` of buttons.
  - `data_grid`s for `rows:deps`, `rows:outdated` (a `badge` column with values `patch|minor|major|needs Rust 1.80|yanked`) and `rows:advisories`, grouped through a `tabs` node (Vulnerabilities, Unsound, Unmaintained, Notices).
- **WIT.**
  - `proc.run cargo metadata --format-version 1 --no-deps` and `rustc --version`.
  - `fs` reads of `Cargo.toml` and `Cargo.lock`.
  - `net` to `crates.io/api/v1/crates*` and `api.osv.dev/v1/querybatch`, `/v1/vulns/{id}` (apps §8).
  - `terminal.run-task cargo check|build|run|test`.
  - `fs.watch` on `Cargo.toml`/`Cargo.lock`.
- **Permissions.**

```toml
[ui.panels.rust]
title = "Rust"
icon = "icons/rust.svg"
position = "left"
activation_priority = 29

[activation]
events = ["workspace_contains:**/Cargo.toml"]

[[capabilities]]
kind = "ui"
surfaces = ["panel"]

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list", "watch"]
glob = ["**/Cargo.toml", "Cargo.lock", "**/Cargo.lock", "rust-toolchain*", "!target/**"]
max_file_size = "16MiB"

[[capabilities]]
kind = "proc"
command = "cargo"
resolve = "path_lookup"
args = ["metadata", "--format-version", "1", "**"]
deny_args = ["--config", "-Z"]
cwd = ["worktree"]
env = { inherit = "allowlisted", allow = ["PATH", "HOME", "CARGO_HOME", "RUSTUP_HOME", "RUSTUP_TOOLCHAIN"] }
timeout = "120s"
max_output = "64MiB"

[[capabilities]]
kind = "proc"
command = "rustc"
resolve = "path_lookup"
args = ["--version"]
timeout = "10s"

[[capabilities]]
kind = "terminal"
op = ["run_task"]
commands = ["cargo"]
confirm = "first_time"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["crates.io", "api.osv.dev"]
methods = ["GET", "POST"]
paths = ["/api/v1/crates*", "/api/v1/crates/**", "/v1/querybatch", "/v1/vulns/*"]
resolve = "public_only"
rate_limit = "60/min"      # crates.io crawler policy
reason = "Check for newer crate versions and security advisories"
```

- **Stays in core.** Nothing.
- **Feasibility.** **Full**. Effort **M**. Phase 2.

### F9. Database workbench

- **Surfaces.**
  - A bottom dock panel `database`, a three-pane `split`.
  - A pane item `workbench`, serializable and multi-instance.
  - A pane item `schema_graph`, serializable.
  - A modal `connect`.
  - Keybinding `ctrl-k d`.
- **JSON sketch** (workbench item).

```json
{ "type": "split", "key": "wb", "axis": "y", "sizes": [0.4, 0.6], "children": [
  { "type": "column", "key": "top", "children": [
    { "type": "row", "key": "tb", "gap": "xs", "children": [
      { "type": "dropdown", "key": "conn", "options": [{ "value": "pg-local", "label": "postgres@localhost/app" }], "selected": "pg-local", "on_select": { "id": "conn" } },
      { "type": "button", "key": "run", "label": "Run", "icon": "builtin:play", "variant": "primary", "on_click": { "id": "run" } },
      { "type": "menu", "key": "export", "label": "Export", "items": [{ "id": "copy_json", "label": "Copy JSON" }, { "id": "copy_sql", "label": "Copy SQL" }, { "id": "save_csv", "label": "Save as CSV…" }] } ] },
    { "type": "editor", "key": "sql", "buffer": { "kind": "scratch", "id": "wb-1" }, "language": "SQL", "completions": { "provider": "schema" }, "on_submit": { "id": "run", "keystroke": "cmd-enter" } } ] },
  { "type": "column", "key": "bottom", "children": [
    { "type": "text", "key": "status", "value": "1,204 rows · 38 ms", "style": "muted" },
    { "type": "data_grid", "key": "results", "source": "rows:wb-1", "sort": "local", "resizable_columns": true, "selection": "multi", "copy": ["json", "csv", "sql-insert"] } ] } ] }
```

Schema graph item: `{ "type": "graph", "key": "g", "source": "graph:schema", "layout": "auto", "auto_layout": {"algorithm":"layered","direction":"LR"}, "interaction": {"pan":true,"zoom":[0.2,2],"drag_nodes":true,"connect":false}, "minimap": true }`. Each node body is a `column` of column `text` rows with a key `icon`.

- **WIT.**
  - **SQLite.** `rusqlite` with bundled C compiled through wasi-sdk (already supported, API.md §1.2) and a custom VFS over `fs.open` → `file-handle` [A: a SQLite VFS over the WIT file-handle has acceptable performance; WAL needs shared memory, so use `journal_mode=DELETE` for wasm]. The file is chosen through `fs.request-path(file)` (the powerbox), and may be remembered.
  - **Postgres, MySQL/MariaDB, SQL Server.** Guest drivers over `net.connect-tcp(host, port, tls)` (G19). Candidate crates: `postgres-protocol` (sans-io), `mysql_common`, and `tiberius` via a futures-io adapter [A].
  - **Oracle.**
    - Oracle is present in the code: `oracledb = "26.0.0-beta.4"` (`fork/Cargo.toml:995`, used by `fork/crates/database_backend/Cargo.toml:19` [V]), added in `12dea53620`. The README still lists four engines (apps §9).
    - Whether this crate is a pure-Rust thin client or needs Oracle client libraries is unverified [A]. If it is pure Rust over TCP, it follows the Postgres route above (guest driver over G19).
    - Otherwise, a sidecar (G22) is the fallback. The manifest keeps the sidecar `optional = true`, so installs without Oracle need no critical grant.
  - **Passwords.** `secrets.prompt("database://pg/localhost:5432", …)` produces a handle. The driver needs the bytes for authentication, so this feature requires `secrets.reveal = true`, scoped to the `database://*` keys. TCP auth cannot use header templating.
  - **Results.** `ui-data.append-rows` in pages of 1,000 rows, and `export-rows`.
  - **Completions.** The `provide-completions` export answers from the cached schema.
  - **State.** `kv` for connections (no secrets) and the schema cache. `serialize-instance` for tab restore and history.
  - **Clipboard.** Copy JSON/SQL.
- **Permissions.**

```toml
[ui.panels.database]
title = "Database"
icon = "builtin:database"
position = "bottom"
activation_priority = 22

[ui.items.workbench]
title = "SQL Workbench"
icon = "builtin:database"
singleton = false
serializable = true

[ui.items.schema_graph]
title = "Schema Graph"
icon = "builtin:share"
singleton = false
serializable = true

[ui.modals.connect]
title = "Connect to Database"
width = "medium"

[[ui.keybindings]]
key = "ctrl-k d"
action = "database::ToggleFocus"

[[capabilities]]
kind = "ui"
surfaces = ["panel", "item", "modal", "context_menu"]
max_open_items = 32
data_stores = { max_bytes = "512MiB", max_rows = 2_000_000 }
reason = "Browse schemas and show query results"

[[capabilities]]
kind = "fs"
scope = "user_selected"
op = ["read", "write", "stat"]
pick = ["file"]
extensions = ["sqlite", "sqlite3", "db", "db3"]
remember = "per_workspace"
reason = "Open the SQLite files you choose"

[[capabilities]]
kind = "fs"
scope = "user_selected"
op = ["create", "write"]
pick = ["save_file"]
extensions = ["json", "sql", "csv"]
reason = "Save exported results where you choose"

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list"]
glob = ["**/*.sqlite", "**/*.sqlite3", "**/*.db"]
reason = "Offer SQLite files found in the project"

[[capabilities]]
kind = "net"
schemes = ["tcp", "tls"]
raw = true
hosts = ["user_configured"]      # special token: only hosts the user typed into the host-owned connect modal (attested), or confirmed at runtime
ports = [5432, 3306, 1433, 1521, "1024-65535"]
resolve = "private_ok"          # DB servers commonly live on LAN/loopback
prompt = "per_host"             # first connection to a new host:port → host prompt
max_concurrent = 16
reason = "Connect to the database servers you configure"

[[capabilities]]
kind = "secrets"
keys = ["database://*"]
op = ["store", "open", "delete", "prompt", "reveal"]
bound_hosts = "same_as_key"     # reveal only while a socket to that key's host:port is open
reason = "Keep database passwords in the system keychain"

[[capabilities]]
kind = "kv"
scopes = ["extension", "workspace"]
max_bytes = "8MiB"

[[capabilities]]
kind = "clipboard"
op = ["write"]

[[capabilities]]
kind = "proc"                    # only for the Oracle sidecar; optional
optional = true
command = "oracle-bridge"
resolve = "extension_binary"
sha256 = "pinned_by_sidecar"
args = ["--stdio"]
long_running = true
env = { inherit = "none", set = {} }
sandbox = "none"
reason = "Oracle support needs a native helper"
```

- **Stays in core.** Nothing feature-specific. The `editor` node, the `graph` node and TCP sockets are generic host API.
- **Feasibility.** **Partial**. SQLite, PostgreSQL and MySQL are Full, SQL Server is likely, and Oracle is either Full (pure-Rust driver) or sidecar-only [A]. Effort **XL**: mentenaz puts each engine backend at about 1,000 lines (thread.md, 2026-11-04). Phase 3.

### F10. Flows (engine, Flows panel, Designer)

- **Surfaces.**
  - A left dock panel `flows` (an index of `<workspace>/.zed/workflows`).
  - A file handler `flow_designer` for `**/*.flow.json`, with the item `designer`.
  - A pane item `run_results`.
  - A modal `task_chain` (the wizard).
  - A JSON schema contribution.
  - Keybinding `ctrl-k f`.
- **JSON sketch** (Designer).

```json
{ "type": "split", "key": "d", "axis": "x", "sizes": [0.18, 0.57, 0.25], "children": [
  { "type": "column", "key": "catalog", "children": [
    { "type": "search_box", "key": "q", "placeholder": "Actions" },
    { "type": "list", "key": "acts", "items": [
      { "type": "row", "key": "a_http", "draggable": { "kind": "flow-action", "payload": { "type": "http" } }, "children": [{ "type": "icon", "key": "i", "name": "builtin:globe" }, { "type": "text", "key": "t", "value": "HTTP" }] },
      { "type": "row", "key": "a_proc", "draggable": { "kind": "flow-action", "payload": { "type": "StartProcess" } }, "children": [{ "type": "icon", "key": "i", "name": "builtin:play" }, { "type": "text", "key": "t", "value": "Start Process" }] } ] } ] },
  { "type": "column", "key": "center", "children": [
    { "type": "row", "key": "tb", "gap": "xs", "children": [
      { "type": "button", "key": "run", "label": "Run", "variant": "primary", "on_click": { "id": "run" } },
      { "type": "button", "key": "fit", "label": "Fit View", "on_click": { "local": "graph_fit", "target": "canvas" } },
      { "type": "button", "key": "raw", "label": "Raw", "on_click": { "local": "dispatch_action", "action": "extension::OpenAsText" } },
      { "type": "button", "key": "res", "label": "Results", "on_click": { "id": "results" } } ] },
    { "type": "banner", "key": "parse_err", "visible": false, "kind": "error", "child": { "type": "text", "key": "m", "value": "" } },
    { "type": "graph", "key": "canvas", "source": "graph:flow", "grow": 1,
      "interaction": { "pan": true, "zoom": [0.2, 2], "drag_nodes": true, "connect": true, "insert_on_edge": true, "delete": true },
      "drop_target": { "accepts": ["flow-action"], "on_drop": { "id": "add_action" } }, "minimap": true,
      "on_node_activate": { "id": "inspect" }, "on_connect": { "id": "connect" }, "on_change": { "id": "graph_changed", "debounce_ms": 250 } } ] },
  { "type": "schema_form", "key": "inspector", "schema": { "$ref": "schemas/actions/http.json" }, "value": {}, "on_change": { "id": "action_changed", "debounce_ms": 200 } } ] }
```

- **WIT.**
  - **Buffer integration.** The file handler opens the project Buffer. `editor.read-buffer` parses it and `editor.apply-edits` writes changes back as edits. Zed then handles undo, dirty state and save, and the guest never writes the `.flow.json` itself. Layout sidecars (`.flow.layout.json`) use `fs.write-file` under the scoped glob.
  - **Graph.** `ui-data.set-graph`/`patch-graph`/`read-graph`, and `set-node-status` during runs.
  - **Engine.** `workflow_engine` is ported into the guest as an event-driven scheduler (no tokio). Action mapping:
    - `http` → `net.fetch`;
    - `notify` → `net.fetch POST`;
    - `WaitForPort` → `net.connect-tcp("127.0.0.1", port)` retried on timers;
    - `script` → `proc.spawn-v2` (JSON on stdin);
    - `StartProcess` → `proc.spawn-v2`;
    - `delay` → `request-timer`;
    - `SetEnv` → the guest-side env map passed in `spawn-options.env`;
    - containers and JSONLogic → pure guest code.
  - **`FORGE_CONTEXT` temp file.** `fs` `temp` scope.
  - **Run history.** `kv` (project scope).
  - **Toasts.** `notify`.
  - **Task Chain.** Exposes `ipc.provides.task-chain` and the action `flows::AddTaskChain`.
- **Permissions.** User-authored flows can contain arbitrary URLs and programs. Those must not become blanket `*` grants. The model uses **runtime-attested scopes**: `prompt = "per_host"` / `"per_program"`. The first run of a flow lists every distinct host and program it uses in a host-drawn confirmation, and the approval is stored per `(flow path, content hash)`.

```toml
[ui.panels.flows]
title = "Flows"
icon = "icons/flows.svg"
position = "left"
activation_priority = 28

[ui.items.designer]
title = "Designer"
icon = "icons/flows.svg"
serializable = true
close_confirmation = "if_dirty"

[ui.items.run_results]
title = "Run Results"
icon = "builtin:list"
serializable = false

[ui.file_handlers.flow_designer]
path_glob = ["**/*.flow.json"]
priority = "default"
buffer_mode = "project_buffer"
open_as_text_action = true
item = "designer"

[ui.modals.task_chain]
title = "Add Task Chain"
width = "large"

[json_schemas.flow]
schema = "schemas/flow.schema.json"
file_globs = ["**/*.flow.json"]

[json_schemas.flow_http_body]
schema = "schemas/flow-http-body.schema.json"
file_globs = ["**/*.flow-http/**/*.body.json"]

[[capabilities]]
kind = "ui"
surfaces = ["panel", "item", "file_handler", "modal", "toast"]

[[capabilities]]
kind = "editor"
op = ["read_buffer", "edit"]
scope = "project"
glob = ["**/*.flow.json", "**/*.flow-http/**/*.body.json", "**/*.flow-scripts/**"]

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list", "watch", "write", "create", "rename", "delete"]
glob = [".zed/workflows/**", "**/*.flow.json", "**/*.flow.layout.json", "**/*.flow-http/**", "**/*.flow-scripts/**"]
follow_symlinks = "never"
max_total_write = "64MiB"

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list"]
glob = ["**/package.json", "**/*.csproj", "**/pyproject.toml", "**/Cargo.toml", "**/docker-compose*.yml", "!**/node_modules/**"]
reason = "Task Chain wizard detects services to start"

[[capabilities]]
kind = "fs"
scope = "temp"
op = ["create", "write", "read", "delete"]
max_total_write = "16MiB"
reason = "Pass run context to script actions"

[[capabilities]]
kind = "net"
schemes = ["https", "http"]
hosts = ["flow_declared"]        # hosts appearing in the flow file being run
prompt = "per_host"
methods = ["GET", "POST", "PUT", "PATCH", "DELETE", "HEAD"]
redirects = "same_host"
resolve = "public_only"
allow_credentials = true          # flows may set Authorization headers the user wrote
max_request_body = "8MiB"
max_response_body = "64MiB"
max_concurrent = 8
reason = "Execute HTTP and webhook actions you put in your workflows"

[[capabilities]]
kind = "net"
schemes = ["tcp"]
raw = true
hosts = ["localhost", "127.0.0.1", "::1"]
ports = ["1-65535"]
resolve = "loopback_ok"
connect_only = true                # WaitForPort: no data sent
reason = "Wait until local services are listening"

[[capabilities]]
kind = "proc"
command = "flow_declared"         # programs named in the flow's script/StartProcess actions
prompt = "per_program"
resolve = "path_lookup"
cwd = ["worktree"]
env = { inherit = "shell_env", deny = ["*_TOKEN", "*_SECRET"] }
stdin = "pipe"
long_running = true
max_concurrent = 16
process_group = true
kill = "own_children_only"
reason = "Run the programs your workflow starts"

[[capabilities]]
kind = "kv"
scopes = ["project"]
max_bytes = "16MiB"

[[capabilities]]
kind = "notify"
levels = ["info", "warning", "error"]
rate_limit = "20/min"

[ipc.provides.task-chain]
schema = "ipc/task-chain.schema.json"
accept_from = ["forge.runtimes"]
```

- **Stays in core.** Generic API only: the file handler `ProjectItem` dispatcher, JSON schema registration and the graph element.
- **Feasibility.** **Partial**: Full functionally, but the engine port is large. Effort **XL**. Phase 3.

### F11. Package managers (npm, NuGet, PyPI, Cargo)

- **Surfaces.**
  - Four pane items: `npm_manager`, `nuget_manager`, `pypi_manager`, `cargo_manager`. Cargo's is serializable.
  - An editor context-menu contribution: "Open in Package Manager" and `Search npm for "<pkg>"`.
  - Keybindings `ctrl-k j/g/y`.
- **JSON sketch** (one manager).

```json
{ "type": "split", "key": "pm", "axis": "x", "sizes": [0.2, 0.4, 0.4], "children": [
  { "type": "list", "key": "nav", "selection": "single", "selected_keys": ["search"], "on_select": { "id": "page" },
    "items": [ { "type": "text", "key": "general", "value": "General" }, { "type": "text", "key": "search", "value": "Search" }, { "type": "text", "key": "installed", "value": "Installed" }, { "type": "text", "key": "updates", "value": "Updates · 4" }, { "type": "text", "key": "vulns", "value": "Vulnerabilities · 1" } ] },
  { "type": "column", "key": "mid", "children": [
    { "type": "search_box", "key": "q", "placeholder": "Search npm", "on_input": { "id": "search", "debounce_ms": 250 } },
    { "type": "data_grid", "key": "pkgs", "source": "rows:search", "selection": "single", "on_select": { "id": "pick" }, "on_activate": { "id": "install" } } ] },
  { "type": "scroll", "key": "details", "child": { "type": "column", "key": "dc", "children": [
    { "type": "row", "key": "h", "children": [ { "type": "text", "key": "n", "value": "lodash", "style": "heading2" }, { "type": "dropdown", "key": "ver", "options": [{ "value": "4.17.21", "label": "4.17.21" }], "selected": "4.17.21" },
      { "type": "button", "key": "install", "label": "Install", "variant": "primary", "on_click": { "id": "install" } } ] },
    { "type": "markdown", "key": "readme", "source": "# lodash\n…", "images": "net" } ] } } ] }
```

- **WIT.**
  - `net.fetch` to the registries (apps §11 table), and `markdown` with remote images (G20).
  - Install, Update, Remove and Fix all go through `terminal.run-task` with `confirm=first_time`. The host shows the exact command line, which replaces the fork's argument validator as the safety net. The validator is still kept in the guest.
  - Cargo dry-run: `proc.run cargo update --dry-run -p <pkg>`.
  - The context menu uses the `menu-items` export with `selection-info`, then `npm_backend::package_from_specifier` (pure Rust, portable).
  - Cargo restore uses `serialize-instance`.
- **Permissions.**

```toml
[ui.items.npm_manager]
title = "npm"
icon = "icons/npm.svg"
singleton = true

[ui.items.cargo_manager]
title = "Cargo"
icon = "icons/rust.svg"
singleton = true
serializable = true
# (nuget_manager, pypi_manager analogous)

[ui.context_menus.npm_editor]
target = "editor.context"
when = "Editor && (extension == ts || extension == tsx || extension == js || extension == jsx || extension == mjs || extension == cjs)"
dynamic = true
max_entries = 2
entries = [{ id = "open_pm", label = "Open in Package Manager", action = "npm_manager::Open" }]

[[capabilities]]
kind = "ui"
surfaces = ["item", "context_menu", "modal"]
allow_images = true
allow_markdown_links = true

[[capabilities]]
kind = "editor"
op = ["read_selection"]
scope = "active_buffer"
glob = ["**/*.{ts,tsx,js,jsx,mjs,cjs}", "**/package.json"]
max_chars = 512
reason = "Read the import under the cursor to look up the package"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["registry.npmjs.org", "api.npmjs.org", "api.nuget.org", "azuresearch-usnc.nuget.org", "pypi.org", "crates.io", "static.crates.io", "api.osv.dev"]
methods = ["GET", "POST"]
paths = ["/**"]
resolve = "public_only"
rate_limit = "120/min"
max_response_body = "32MiB"
reason = "Search registries, read package details and advisories"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["raw.githubusercontent.com", "user-images.githubusercontent.com", "camo.githubusercontent.com", "*.githubusercontent.com"]
methods = ["GET"]
purpose = ["images"]
max_response_body = "2MiB"
reason = "Show images in package READMEs"

[[capabilities]]
kind = "open_url"
schemes = ["https", "mailto"]
hosts = ["*"]
confirm = "non_allowlisted"

[[capabilities]]
kind = "terminal"
op = ["run_task"]
commands = ["npm", "yarn", "pnpm", "dotnet", "python", "python3", "cargo"]
confirm = "first_time"
cwd = ["worktree"]

[[capabilities]]
kind = "proc"
command = "cargo"
resolve = "path_lookup"
args = ["update", "--dry-run", "**"]
cwd = ["worktree"]
timeout = "120s"

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["read", "stat", "list"]
glob = ["**/package.json", "**/*.csproj", "**/pyproject.toml", "**/requirements*.txt", "**/Cargo.toml", "**/Cargo.lock", "!**/node_modules/**", "!target/**"]
```

- **Stays in core.** The `MenuContributionRegistry` (G13, upstream).
- **Feasibility.** **Full**. Effort **L**, since four tabs share one guest module. Phase 2 for the tabs, and Phase 3 for the context menu.

### F12. npm bootstrap

- **Surfaces.** A toast with an action only.
- **JSON / commands.** `ui-command::toast { message: "This project has package.json but no node_modules.", actions: [{label: "Run npm install", action: "node::BootstrapInstall"}] }`. The result is a second toast on `task-exit`.
- **WIT.**
  - `activation workspace_contains:package.json` → `on-background-event(workspace-opened)`.
  - Then `fs.stat` of `node_modules` and `kv.get(project, "bootstrap_offered")`.
  - Then a toast, then `kv.set`, then `terminal.run-task npm install` (the fork's `Command::new(npm_bin)` at `npm_bootstrap.rs:125`).
- **Permissions.**

```toml
[activation]
events = ["workspace_contains:package.json"]
background = true

[[capabilities]]
kind = "ui"
surfaces = ["toast"]

[[capabilities]]
kind = "fs"
scope = "worktree"
op = ["stat"]
glob = ["package.json", "node_modules"]

[[capabilities]]
kind = "kv"
scopes = ["project"]
max_bytes = "4KiB"

[[capabilities]]
kind = "terminal"
op = ["run_task"]
commands = ["npm"]
args = [["install"]]
confirm = "never"          # the toast click is the confirmation; command line still shown in task tab
cwd = ["worktree_root"]

[[capabilities]]
kind = "notify"
levels = ["info", "error"]
rate_limit = "2/min"
```

- **Stays in core.** Nothing.
- **Feasibility.** **Full**. Effort **S**. Phase 1 needs activation and toasts. `terminal.run-task` lands in Phase 2, and until then `proc.run npm install` is a fallback.

### F13. Helm (GitHub client)

- **Surfaces.**
  - A left dock panel `helm`, with screen navigation driven by guest state.
  - Modals `repo_edit`, `repo_create` and `clone`.
  - A pane item `actions_run` (with a graph).
  - An account menu in the panel header.
  - Toasts.
  - Keybinding `ctrl-k h`.
- **JSON sketch** (panel header and list).

```json
{ "type": "column", "key": "root", "children": [
  { "type": "row", "key": "hdr", "justify": "between", "children": [
    { "type": "menu", "key": "acct", "trigger": { "type": "row", "key": "a", "children": [{ "type": "image", "key": "av", "url": "https://avatars.githubusercontent.com/u/1?s=40", "width": { "px": 20 }, "height": { "px": 20 }, "radius": "full" }, { "type": "text", "key": "l", "value": "octocat" }] },
      "items": [{ "id": "switch:work", "label": "Switch to work-account" }, { "id": "add", "label": "Add account…" }, { "id": "signout", "label": "Sign out" }] },
    { "type": "progress", "key": "rate", "value": 4820, "max": 5000, "tooltip": "API rate limit" } ] },
  { "type": "tabs", "key": "screen", "active": "repos", "on_select": { "id": "screen" }, "tabs": [
    { "key": "repos", "label": "Repositories", "content": { "type": "list", "key": "repos_l", "virtual": { "count": 120, "row_height": "md" }, "on_activate": { "id": "open_repo" } } },
    { "key": "orgs", "label": "Organizations", "content": { "type": "list", "key": "orgs_l", "items": [] } } ] } ] }
```

The Actions run item uses a `graph` with `layout: auto` (jobs as nodes, `needs` as edges) and `set-node-status` for running and failed states.

- **WIT.**
  - **Sign-in.** `auth.device-flow({device-code-url: "https://github.com/login/device/code", token-url: "https://github.com/login/oauth/access_token", client-id: HELM_OAUTH_CLIENT_ID, scopes: ["repo","read:org","user"]}, key="github.com/<login>")` (`oauth.rs:33-34` per apps §13). The host shows the code and opens the browser. A PAT goes through `secrets.prompt`. The `gh` import uses `proc.run-to-secret gh auth token`.
  - **API.** `net.fetch` with `header-value::secret(handle, "Bearer {}")` to `api.github.com`. The token never enters the guest.
  - **Clone.** `fs.request-path(directory)` → `git.clone(url, dest, auth=handle)` → `workspace.open-folder(dest, replace-current)`.
  - **Identity.** `git.set-config(repo, "user.email", …)` (local scope).
  - **Accounts.** `kv`/settings for account logins (no secrets), and `settings` for `helm.accounts`.
  - **Repo modals.** `show-modal`, plus `ui.confirm` for destructive edits (visibility change, collaborator removal).
- **Permissions.**

```toml
[ui.panels.helm]
title = "Helm"
icon = "icons/helm.svg"
position = "left"
activation_priority = 21

[ui.items.actions_run]
title = "Actions Run"
icon = "builtin:play"
singleton = false

[ui.modals.repo_edit]
title = "Repository Settings"
width = "large"
# (repo_create, clone analogous)

[settings]
schema = "settings.schema.json"     # helm.accounts, helm.default_account (no secrets)

[[capabilities]]
kind = "ui"
surfaces = ["panel", "item", "modal", "toast", "status_item"]
allow_images = true

[[capabilities]]
kind = "auth"
flow = "oauth_device"
providers = [{ id = "github", device_code_url = "https://github.com/login/device/code", token_url = "https://github.com/login/oauth/access_token", scopes = ["repo", "read:org", "user"] }]
reason = "Sign in to GitHub"

[[capabilities]]
kind = "secrets"
keys = ["github.com/*"]
op = ["store", "open", "delete", "prompt", "list"]
reveal = false
bound_hosts = ["api.github.com", "github.com", "uploads.github.com"]
reason = "Keep GitHub tokens in the system keychain, usable only with GitHub"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["api.github.com", "uploads.github.com"]
methods = ["GET", "POST", "PATCH", "PUT", "DELETE"]
paths = ["/**"]
allow_credentials = true          # only via secret handles bound to these hosts
redirects = "allowlisted"
redirect_hosts = ["codeload.github.com", "objects.githubusercontent.com"]
resolve = "public_only"
max_response_body = "64MiB"
rate_limit = "300/min"
reason = "Use the GitHub REST API"

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["avatars.githubusercontent.com"]
methods = ["GET"]
purpose = ["images"]

[[capabilities]]
kind = "git"
op = ["read", "clone", "config_write"]
config_keys = ["user.name", "user.email"]
config_scope = "local"           # never --global
clone_hosts = ["github.com"]
reason = "Clone repositories and set the per-repo commit identity"

[[capabilities]]
kind = "fs"
scope = "user_selected"
op = ["create", "list", "stat"]
pick = ["directory"]
reason = "Choose where to clone"

[[capabilities]]
kind = "workspace"
op = ["open_folder", "open_path"]

[[capabilities]]
kind = "proc"
optional = true
command = "gh"
resolve = "path_lookup"
args = ["auth", "token", "**"]
output = "to_secret"             # stdout goes to keychain, never to the guest
timeout = "20s"
env = { inherit = "allowlisted", allow = ["PATH", "HOME", "GH_CONFIG_DIR", "XDG_CONFIG_HOME"] }
reason = "Import an existing gh CLI login (optional)"

[[capabilities]]
kind = "open_url"
schemes = ["https"]
hosts = ["github.com"]

[[capabilities]]
kind = "clipboard"
op = ["write"]

[[capabilities]]
kind = "kv"
scopes = ["workspace", "extension"]
```

- **Stays in core.** The host-run device flow and the secret-handle substitution (generic). Per-workspace "which account" state is shared with F15/F16 inside the same bundle, so no core global is needed (the fork's `ide_env` global, apps §13).
- **Feasibility.** **Partial**: the panel, tabs and modals are Full in Phase 3, and the cross-cutting identity depends on F15/F16 hooks. Effort **XL**.

### F14. Helm Workspace (read-only remote repo tab)

- **Surfaces.** A pane item `repo_workspace`, multi-instance, opened by the `helm::OpenRepositoryWorkspace` action.
- **JSON sketch.**

```json
{ "type": "tabs", "key": "pages", "active": "code", "tabs": [
  { "key": "code", "label": "Code", "content": { "type": "split", "key": "cs", "axis": "x", "sizes": [0.3, 0.7], "children": [
    { "type": "column", "key": "l", "children": [
      { "type": "dropdown", "key": "ref", "options": [{ "value": "main", "label": "main" }], "selected": "main", "on_select": { "id": "ref" } },
      { "type": "search_box", "key": "f", "placeholder": "Filter files", "filter": { "local_target": "tree" } },
      { "type": "tree", "key": "tree", "nodes": [], "on_expand": { "id": "expand" }, "on_activate": { "id": "open_file" } } ] },
    { "type": "editor", "key": "file", "buffer": { "kind": "readonly_text", "text": "", "path_hint": "src/lib.rs" }, "mode": "full" } ] } },
  { "key": "commits", "label": "Commits", "content": { "type": "data_grid", "key": "cg", "source": "rows:commits", "on_activate": { "id": "open_commit" } } },
  { "key": "compare", "label": "Compare", "content": { "type": "diff", "key": "dv", "patch": "", "language_from_path": true } },
  { "key": "prs", "label": "Pull requests", "content": { "type": "list", "key": "pl", "virtual": { "count": 0, "row_height": "lg" } } } ] }
```

- **WIT.**
  - `net.fetch` with the secret handle to `api.github.com` (contents, commits, compare, search/code, pulls, issues, runs).
  - `ui` `editor` and `diff` nodes.
  - **Open locally.** `git.repositories` to find a checkout whose remote matches, then `ui.confirm` on a commit mismatch, then `workspace.open-path`.
  - **Remember ref.** `kv` per repository.
  - **Permalinks.** `open_url`.
- **Permissions.**

```toml
[ui.items.repo_workspace]
title = "Repository"
icon = "icons/helm.svg"
singleton = false
serializable = true

[[capabilities]]
kind = "ui"
surfaces = ["item"]
max_open_items = 16
allow_images = true

[[capabilities]]
kind = "net"
schemes = ["https"]
hosts = ["api.github.com", "raw.githubusercontent.com"]
methods = ["GET"]
paths = ["/repos/**", "/search/code", "/*/*/*/**"]
allow_credentials = true
resolve = "public_only"
max_response_body = "32MiB"

[[capabilities]]
kind = "secrets"
keys = ["github.com/*"]
op = ["open"]
reveal = false
bound_hosts = ["api.github.com", "raw.githubusercontent.com"]

[[capabilities]]
kind = "git"
op = ["read"]

[[capabilities]]
kind = "workspace"
op = ["open_path"]

[[capabilities]]
kind = "open_url"
schemes = ["https"]
hosts = ["github.com"]

[[capabilities]]
kind = "kv"
scopes = ["extension"]
```

- **Stays in core.** Nothing.
- **Feasibility.** **Full**. Effort **L**. Phase 3.

### F15. Git panel extensions (Details tab, GitHub account, retry, push gating)

- **Surfaces.** The slots `git_panel.tab` (Details) and `git_panel.footer` (account and email), toasts, and the hooks `before_remote_op`/`after_remote_op`.
- **JSON sketch.**
  - Details slot: a `column` with `chart` (commit activity by day, from `series:activity`), `data_grid` (most-changed files, `rows:hot_files`) and `data_grid` (collaborators).
  - Footer slot:

```json
{ "type": "row", "key": "acct", "gap": "xs", "children": [
  { "type": "image", "key": "av", "url": "https://avatars.githubusercontent.com/u/1?s=32", "width": { "px": 16 }, "height": { "px": 16 }, "radius": "full" },
  { "type": "menu", "key": "sw", "label": "octocat · me@users.noreply.github.com", "items": [{ "id": "switch:work", "label": "Use work-account" }] },
  { "type": "badge", "key": "warn", "visible": true, "value": "cannot push", "color": "status.warning" } ] }
```

- **WIT.**
  - **15a.** `git.recent-commits(repo, DETAILS_COMMIT_WINDOW, with-files=true)`. The host implements it over `GitRepository`, which upstreams the fork's `recent_commit_stats`/`file_change_frequency` as a generic git API.
  - **15b.** `before-remote-op` → `deny({message: "work-account cannot push to org/repo", actions:[{label:"Switch to owner-account", action:"helm::SwitchAccount"}]})`. The host renders the refusal toast and disables the button while the verdict is cached. `after-remote-op` detects the private-email rejection in `stderr-tail` and offers Retry.
  - **15c.** Authenticated git for panel operations goes through `git-credential` (G12) instead of `GIT_CONFIG_*` env injection. This removes the fork's `GitBinary` merge patch (apps §15c).
- **Permissions.**

```toml
[ui.slots.details]
slot = "git_panel.tab"
title = "Details"
action = "helm::ActivateDetailsTab"

[ui.slots.account]
slot = "git_panel.footer"

[[ui.keybindings]]
key = "ctrl-3"
action = "helm::ActivateDetailsTab"
context = "GitPanel"

[git.hooks]
before_remote_op = { ops = ["push", "pull", "fetch"], remotes = ["https://github.com/**", "git@github.com:**"], on_timeout = "allow" }
after_remote_op = { ops = ["push"], remotes = ["https://github.com/**"] }

[[capabilities]]
kind = "ui"
surfaces = ["slot:git_panel.tab", "slot:git_panel.footer", "toast"]

[[capabilities]]
kind = "git"
op = ["read", "hook_before_remote_op", "hook_after_remote_op", "config_read", "config_write"]
config_keys = ["user.email", "user.name"]
config_scope = "local"
remotes = ["github.com"]
stderr_tail_bytes = 4096
reason = "Show the GitHub account for this repo and stop pushes that would fail"
```

- **Stays in core (upstream).** The git panel slot API, remote-op hooks in `GitStore`, and the `recent_commits` git API.
- **Feasibility.** **Partial**. Effort **L**, mostly the host hooks. Phase 4.

### F16. Git credential helper

- **Surfaces.** None.
- **WIT.** The `git-credential` export answers `credential-answer::token("x-access-token", handle)` for repositories whose remote is github.com and that map to a Helm account. Otherwise it answers `pass`, so `gh` or GCM answer as before, or `quit` when signed out. The host writes the token to git's stdin through the helper protocol, and the guest never sees the bytes.
- **Permissions.**

```toml
[git.credential_provider]
hosts = ["github.com"]
protocols = ["https"]
ops = ["get"]              # store/erase are never forwarded
cache_ttl = "60s"
on_timeout = "pass"

[[capabilities]]
kind = "git_credentials"
hosts = ["github.com"]
secret_keys = ["github.com/*"]
reason = "Let git in Zed's terminals sign in as this workspace's GitHub account"
```

- **Stays in core (upstream).**
  - A generic `zed --git-credential` helper mode, plus the socket server (generalizing `fork/crates/git_credential/src/git_credential.rs:87,131` [V] and `fork/crates/zed/src/main.rs:315` per apps §16).
  - `credential.helper` injection for local terminals, through the G9 env hook. It is host-owned and not extension-settable, because `GIT_CONFIG_*` is on the denylist.
  - A provider registry also used by `askpass` for panel git.
- **Feasibility.** **Partial**. Effort **M** for the core and **S** for the extension. Phase 4.

### F17. Built-in runtimes and PATH injection (`ide_env`)

- **Migration.** `ide_env` disappears. Its role splits into:
  - **(a)** the host `TerminalEnvProvider` registry (G9), consulted by terminals, tasks and `terminal.run-task`;
  - **(b)** each runtime extension's `terminal-env` export (F5, F6).
  - Extension-spawned processes (`proc.spawn`) receive the same contributions when `env.inherit = "shell_env"`. The Flows `script` actions therefore pick up built-in Node and Python.
- **Permissions.** The `kind = "terminal"`, `op = ["env"]` blocks shown in F5 and F6.
- **Stays in core.** The provider registry and merge order: user settings `terminal.env` win over extensions; extensions are applied in a deterministic order by extension id, with conflicts logged.
- **Feasibility.** **Partial**. Effort **M** (core). Phase 4.

### F18. GPUI layout `scale`

**Core-only.** Upstream as a GPUI contribution (§4). This is the preferred base for the host `graph` element, because `gpui_flow`'s zoom is built on it (apps §19). Builder-resolved zoom (G3) is the fallback if upstream declines.

### F19. gpui_flow canvas

**Core-only.** It becomes the host implementation of the `graph` node in `crates/extension_ui`, ported to Zed `ui` theming (`ThemeColors` instead of raw `u32` colours, apps §19). The MIT license allows it.

- The port keeps `FlowState`: pan offset, `request_fit`, `canvas_size`, and the LOD thresholds.
- Zoom uses either upstream `.scale()` or the G3 fallback.
- It is not an extension, because R2 forbids guest-side GPUI.
- It is also what mentenaz means by "GPUI had no zoom/pan/fit-view, so I added it and plan to offer it upstream" (thread.md). Coordinating with that upstream offer avoids two competing graph widgets.

### F20. gpui_component stack

**Core-only, and deleted.** It is a parallel UI stack with a second theme system (apps §20).

- **Theme bridge.** The fork needs the bridge function `sync_gpui_component_theme` (`fork/crates/zed/src/main.rs:235` [V]). It maps Zed's active theme onto `gpui_component::Theme` and then rebuilds `ThemeTokens`, because some widgets read the token snapshot directly (apps §0).
- **Why no bridge is needed here.**
  - In `zed-ui/1`, nodes reference only Zed theme tokens (`ThemeColors` fields), and the host resolves them at paint time.
  - No second theme exists, so no bridge is needed. A theme switch re-skins cached trees for free (API.md §5.9).
  - This satisfies mentenaz's lesson "theme inheritance must be automatic" (thread.md).
- **Naming prior art.** The fork's role-based `SemanticThemeTokens` (`fork/crates/gpui_base/src/theme_tokens.rs`, per apps §0: colors, radius, spacing, typography, shadow) is useful prior art for the `zed-ui/1` spacing, radius and typography token names.
- **Replacement widgets.** The JSON node catalog maps onto Zed's own `ui` components (API.md §4.1), so no extension needs `gpui_component`. The few genuinely missing widgets are added to `ui` as host implementations of the new nodes: data grid improvements over `ui::Table`, charts and gauges, schema form, and graph.

### F21. sqls extension

- Already an extension. It stays on its current API version.
- Optional upgrade to 0.9.0: subscribe to `ipc` topic `forge.database/connections` to generate `workspace_configuration` (closing `crates/extension/SQLS_EXTENSION_TODO.md` per apps §21).
- **Feasibility.** **Full**. Effort **S**. Phase 3, optional.

```toml
[[ipc.consumes]]
extension = "forge.database"
topic = "connections"            # host DSNs without passwords

[[capabilities]]
kind = "proc"
command = "sqls"
resolve = "path_lookup"
args = ["**"]
long_running = true
```

### F22. Extension system changes

**Core-only, upstream.**

- **Registry URL.** A setting `extensions.registry_url`, with the same API shape, instead of `build_forge_extensions_url`.
- **Install failure notification.** `Event::ExtensionFailedToInstall`.
- **README tab.** With the same untrusted-content rules, the host can reuse the `markdown` node policy from G20.
- **Razor suggestion.** A data entry in `extension_suggest`.
- **`--lenient`.** Fork-local only. It defeats validation, which matters more once manifests carry permissions.

### F23. Identity, telemetry, keychain namespacing, rebrand

**Core-only / fork policy.**

- **Telemetry.** Upstream already has telemetry settings. The compile-time hard-off is a distribution policy.
- **Keychain namespacing by app scheme.** An upstream candidate, because Zed's channels (Stable, Preview, Nightly, Dev) share keys today [A].
- **Rebrand and hidden hosted features.** Fork-only.
- Extensions must not be able to change any of this, by design: product identity and telemetry are not extension surfaces.

### F24. Updater, data import, packaging

**Core-only / fork.**

- **Notify-only update check.** An upstream candidate as an `auto_update: "notify"` mode.
- **Import previous data and packaging.** Fork-only.

### F25. Panel order and Move Left/Right

**Core, upstream.** It is required for usability once many extension panels exist. `panel_order` must accept extension panel keys of the form `ext:<id>:<panel>`, which means relaxing `Panel::panel_key() -> &'static str` (`zed/crates/workspace/src/dock.rs:37-38` [V]).

### F26. Smaller patches

**Core, upstream** individually: the debugger ignore-breakpoints toggle, the new-process modal rework, the JS adapter chained-command debugging, and shell/terminal hyperlinks.

- The per-panel settings pages are replaced by the G18 `[settings]` schema rendering.
- The CLI rename and Windows explorer integration are fork-only.

### 3.1 Feasibility summary

| # | Feature | Feasibility | Effort | Phase | Host/core dependency |
|---|---|---|---|---|---|
| 1 | Cockpit | Full | S | 2 | `sys` |
| 2 | Dashboard | Partial (Full in bundle) | M | 2 / 4 | `git` read |
| 3 | Processes | Full | S | 2 | `sys` signal + confirm |
| 4 | Script Runner | Full | S | 2 | user-attested shell |
| 5 | Node | Partial | L | 2–4 | G9 env hook |
| 6 | Python | Partial | L | 2–4 | G9 env hook |
| 7 | .NET | Full | M | 2 | – |
| 8 | Rust | Full | M | 2 | – |
| 9 | Database | Partial | XL | 3 | TCP/TLS, editor node, graph; Oracle sidecar |
| 10 | Flows | Partial | XL | 3 | file handler, graph, schema form |
| 11 | Package managers | Full | L | 2–3 | menu registry |
| 12 | npm bootstrap | Full | S | 1–2 | activation |
| 13 | Helm | Partial | XL | 3–4 | auth/secrets, git clone |
| 14 | Helm Workspace | Full | L | 3 | editor/diff nodes |
| 15 | Git panel | Partial | L | 4 | git slots + hooks (upstream) |
| 16 | Credential helper | Partial | M+S | 4 | credential routing (upstream) |
| 17 | PATH injection | Partial | M | 4 | env provider registry (upstream) |
| 18 | GPUI scale | Core | – | U | GPUI |
| 19 | gpui_flow | Core (host `graph`) | L | 3 | – |
| 20 | gpui_component | Core (deleted) | – | – | – |
| 21 | sqls | Full | S | 3 | ipc (optional) |
| 22 | Extension system | Core / upstream | M | U | – |
| 23 | Identity, telemetry | Core / fork | – | – | – |
| 24 | Updater, packaging | Core / fork | – | – | – |
| 25 | Panel order | Core / upstream | S | U (before 2) | – |
| 26 | Small patches | Core / upstream / fork | S each | U | – |

---

## 4. Features that remain core patches or become upstream contributions

| Item | Recommendation | Reason |
|---|---|---|
| GPUI `.scale()` (F18) | Upstream GPUI PR (release note `[GPUI] Added Styled::scale …`). Preferred base for the host `graph` node; builder zoom (G3) is the fallback. | It changes `ToTaffy::to_taffy` and `Style::overflow_mask` (apps §18). As a fork it is the hardest merge, and as an upstream feature it benefits zoomable views generally. `gpui_flow`'s zoom is built on it (apps §19). mentenaz already plans to offer zoom/pan upstream (thread.md), so coordinate with that. |
| gpui_flow (F19) | Host component in `crates/extension_ui` (or `ui`), not an extension. | R2: guests cannot run GPUI. One host implementation serves the Designer, the schema graph and the Actions graph. |
| gpui_component (F20) | Delete. | A second theme system and keymap re-registration hacks (apps §0). The node catalog covers the needs. |
| `Panel` static strings | Upstream: `persistent_name`, `panel_key`, `icon_tooltip` as `SharedString` (`dock.rs:37-38,72` [V]). | Needed for any dynamic panel. A small and uncontroversial change. |
| Panel ordering (F25) | Upstream. | It is more necessary with extension panels than in the fork. |
| Terminal env provider (F17) | Upstream generic hook. | Useful for any toolchain-manager extension (nvm, pyenv, asdf). |
| Git remote-op hooks, credential routing, `zed --git-credential` (F15, F16) | Upstream, in coordination with the git team. | Security-sensitive. It must be one well-reviewed host path, not per-extension env tricks. |
| Git panel slots (F15) | Upstream, small closed slot set. | Prevents per-feature edits to `git_panel.rs` (the fork added about 430 lines, apps §15). |
| Editor context-menu registry (G13) | Upstream. | Replaces direct `editor` edits. |
| File handler `ProjectItem` dispatcher (G14) | Upstream as part of the UI API. | Generic. |
| Telemetry hard-off (F23) | Fork policy. | Upstream already offers settings, and a compile-time assertion is a distribution choice. |
| Keychain namespacing (F23) | Upstream candidate. | Channel and edition isolation of credentials is generally useful. It also underpins the `zed-ext://<id>/` namespace of G11. |
| Updater notify-only (F24) | Upstream candidate (`auto_update: "notify"`). The rest is fork-only. | |
| Packaging, rebrand, data import (F23, F24) | Fork-only. | Product identity. |
| Registry URL, install errors, README tab (F22) | Upstream. | Generic improvements. A registry setting also helps self-hosted or enterprise mirrors. |
| `extension_cli --lenient` (F22) | Drop. | It undermines manifest permission validation. |
| `sysinfo` features `disk`, `network` | Host dependency change. | Needed by `sys`. |
| Debugger and terminal patches (F26) | Upstream individually. | Unrelated to the extension API. |

---

## 5. Consolidated final API (`since_v0.9.0`, manifest `schema_version = 2`, UI schema `zed-ui/1`)

### 5.1 Corrections to API.md under the three rules

1. **R3: powerbox.** `fs` `scope = "user_selected"` must be declared, with `op`, `pick` kinds and optional `extensions`. Runtime selection only narrows it to concrete paths. The same applies to OS drag-and-drop (G6).
2. **R3: legacy calls.** For schema 2, every import is gated by a declared capability:
   - `get-settings` → `settings`;
   - `worktree.read-text-file` → `fs worktree read`;
   - `which`/`shell-env` → `proc` (`which` needs a matching command entry; `shell-env` needs `env.inherit = "shell_env"` on some entry);
   - `github.*` → `net api.github.com`;
   - `http-client.fetch` → `net`;
   - `download-file` → `download_file` in the manifest.
3. **R3: runtime grants.** "Runtime-attested" scopes (`flow_declared`, `user_configured`, `user_attested_shell`) are themselves manifest declarations. The manifest declares the *kind* of dynamic scope, and the host collects the concrete values with host-drawn prompts. Nothing is grantable that is not declared.
4. **R1: static UI.** Static UI files (API.md OQ 8) are allowed only as `initial_tree` assets of a wasm extension, loaded by `extension_host`. They show instantly while the guest loads. There are no wasm-less UI extensions.
5. **R2: drawing nodes.** `canvas2d` is limited to a validated display list of at most 5,000 primitives with theme tokens. `graph` node bodies use a restricted node subset. No node accepts code, scripts, HTML, CSS or shaders. `markdown` is CommonMark without raw HTML.
6. **R2: embedded widgets.** Embedded host widgets (`editor`, `terminal_view`, `diff`) are declared by reference. The guest receives events and data, never handles to GPUI entities.
7. **Process API.** `proc` gains the `user_attested_shell` mode, `extension_binary` with sha256, `output = "to_secret"`, `process_group`, `stdout-to` stores, and `watch-patterns`. Killing non-children moves to `sys`.
8. **`ui` capability.** It gains `slot:*` surfaces, `file_handler`, `toast`, and `data_stores` quotas.

### 5.2 WIT package list

`zed/crates/extension_api/wit/since_v0.9.0/`. Host bindings go in `zed/crates/extension_host/src/wasm_host/wit/since_v0_9_0.rs`, added to the `instantiate_async` dispatch and to `wasm_api_version_range` (`zed/crates/extension_host/src/wasm_host/wit.rs:60` [V]). Dev/Nightly come first.

| Interface | Status | Contents | Gated by |
|---|---|---|---|
| `common`, `lsp`, `slash-command`, `context-server`, `dap`, `platform`, `nodejs`, `github`, `http-client`, `process` | carried from 0.8.0 | unchanged signatures; schema 2 adds gating (5.1 #2) | as 5.1 |
| `settings` (existing `get-settings`) | extended | `get-settings`, plus `update-own-settings(json-patch)` | `settings` |
| `ui` | new (API.md §5.4, extended) | surface exports, `ui-event`/`ui-command`, `push-render`, `request-render`, timers, `current-theme`, `confirm`, `host-capabilities()` | `ui` |
| `ui-data` | new (G1–G3) | row stores, series, logs, graph stores, export | `ui` (`data_stores`) |
| `fs` | new (API.md §5.8.2) | as proposed, plus `request-path` declared kinds | `fs` |
| `net` | new (API.md §5.8.3, extended) | `fetch` with secret-templated headers, websockets, `connect-tcp`/`connect-tls`, `dns-lookup` | `net`, `download_file` |
| `proc` | new (API.md §5.8.4, extended) | `run`, `spawn`, `spawn-v2`, `spawn-user-command`, `run-to-secret` | `proc` |
| `sys` | new (G10) | `metrics`, `processes`, `signal` | `sys` |
| `terminal` | new (G8, G9) | `run-task`, `stop-task`, `invalidate-env`; export `terminal-env` | `terminal` |
| `secrets` | new (G11) | `store`, `open`, `delete`, `list-keys`, `reveal`, `prompt` | `secrets` |
| `auth` | new (G11) | `device-flow` (later: `pkce-loopback`) | `auth` |
| `git` | new (G12) | read, `clone`, config; exports `before-remote-op`, `after-remote-op`, `git-credential` | `git`, `git_credentials` |
| `editor` | new (G4, G13) | `read-buffer`, `apply-edits`, `active-selection`, `set-decorations`; exports `provide-completions`, `menu-items` | `editor` |
| `workspace` | new (G17, G23) | `current`, `open-path`, `open-folder`, `reveal-in-os`; export `on-background-event` | `workspace` |
| `kv` | new (G18) | `get`, `set`, `delete`, `list` | `kv` |
| `ipc` | new (G16) | `call`, `publish`, `subscribe`; export `ipc-request` | `[ipc.*]` manifest |
| `clipboard` | new (API.md) | `write-text`, `read-text` (gesture-bound) | `clipboard` |
| `notify` | new (API.md) | toasts and notifications with extension actions | `notify` / `ui toast` |

**World `extension` v0.9.0.**

- **Imports**: all of the above.
- **Exports**: everything from 0.8.0, plus:
  - `surface-open`, `surface-event`, `surface-close`, `serialize-instance`, `deserialize-instance`;
  - `on-background-event`;
  - `provide-completions`, `menu-items`;
  - `terminal-env`;
  - `before-remote-op`, `after-remote-op`, `git-credential`;
  - `ipc-request`.
- **Default implementations.** The SDK gives every export a default no-op or `pass` implementation, as `zed/crates/extension_api/src/extension_api.rs:69+` does today (API.md §5.2).

**Call classes and budgets.** The host tags each export with a class. The per-extension queue becomes a priority queue (§6, risk 1):

| Class | Exports | Budget (trap after) |
|---|---|---|
| interactive | `surface-event`, `menu-items`, `provide-completions` | 50 ms soft, 500 ms hard |
| hook | `before-remote-op`, `git-credential`, `terminal-env` | 300–500 ms, then manifest `on_timeout` |
| render | `surface-open`, `deserialize-instance` | 2 s |
| background | `on-background-event`, `ipc-request`, LSP/DAP exports | 10 s |

### 5.3 JSON UI node catalog (`zed-ui/1`)

**Base props.** Every node has: `type`, `key`, `visible`, `enabled`, `tooltip`, `a11y {role, label, description}`, `test_id`, `draggable`, `drop_target`, sizing (`width`/`height`/`min_*`/`max_*`/`grow`/`shrink`/`basis`), and `x-*`. Event props are `{id, payload?, debounce_ms?}` or `{local: <closed set>}`.

**Local effects** (closed set): `copy` (`value` | `from_node`), `toggle` (`target`), `open_url` (permissioned), `dispatch_action`, `graph_fit`/`graph_zoom` (`target`), `focus` (`target`), `scroll_to` (`target`, `key`).

| Group | Nodes | Origin |
|---|---|---|
| Layout | `row`/`h_flex`, `column`/`v_flex`, `stack`, `grid` (+`min_column_width`), `scroll`, `split`, `spacer`, `divider`, `container`, `card` | API.md, plus `card` (G2) |
| Content | `text`, `rich_text`, `markdown` (+`images: "none"\|"extension"\|"net"`), `code`, `icon`, `image` (+`url`, G20), `badge`, `chip`, `indicator`, `progress`, `spinner`, `empty_state`, `banner` | API.md, plus `banner`, image `url` |
| Input | `button`, `icon_button`, `input` (+`user_command`), `checkbox`, `switch`, `radio_group`, `dropdown`, `segmented`, `slider`, `search_box` (+`filter.local_target`), `link`, `schema_form` | API.md, plus G5, G7 |
| Collections | `list` (+`virtual`), `tree`, `tabs`, `table` (small, inline rows), `data_grid` (store-backed), `accordion`/`disclosure`, `menu`, `context_menu`, `popover`, `tooltip` | API.md, plus G1 |
| Visualization | `chart`, `gauge`, `stat`, `sparkline`, `graph`, `canvas2d` (bounded) | G2, G3 |
| Embedded host widgets | `editor`, `diff`, `log`, `terminal_view` | G4, G7, G8 |

**Data references.** Nodes reference stores as `"source": "rows:<id>" | "series:<id>" | "log:<id>" | "graph:<id>"`. Unknown store IDs render an `empty_state` with an error, and are logged.

**Limits** (host-enforced, per surface; overridable downward by the `ui` capability):

- 20k nodes, 4 MiB JSON, depth 64, 30 renders per second;
- strings up to 64 KiB (`markdown` up to 1 MiB);
- 2,000 graph nodes;
- row stores counted against `data_stores.max_bytes`.

**Diffing and state.** As in API.md §5.6: keys, revisions, patches, controlled inputs, and host-owned view state. Additionally, store-backed nodes never re-render the tree when data changes. Only the element is notified.

### 5.4 Manifest schema (`schema_version = 2`)

New or extended top-level tables (all `#[serde(default, deny_unknown_fields)]` on `ExtensionManifest`, `zed/crates/extension/src/extension_manifest.rs:88`):

```toml
schema_version = 2
lib = { kind = "Rust", version = "0.9.0" }

[activation]                      # G17
events = ["on_startup" | "workspace_contains:<glob>" | "on_action:<ns>::<name|*>" | "on_surface:<kind>:<id>" | "on_file_handler:<id>"]
background = false

[ui.panels.<id>]                  # API.md §5.3, plus:
initial_tree = "ui/panel.json"    # optional placeholder (5.1 #4)
[ui.items.<id>]                   # API.md
[ui.status_items.<id>]            # API.md
[ui.toolbar_items.<id>]           # API.md
[ui.modals.<id>]                  # API.md
[ui.context_menus.<id>]           # API.md, plus: when, dynamic, max_entries (G13)
[ui.decorations.<id>]             # API.md
[ui.slots.<id>]                   # G-slots: slot = "git_panel.tab" | "git_panel.footer" | "project_panel.header"
[ui.file_handlers.<id>]           # G14: path_glob, priority, buffer_mode, open_as_text_action, item
[[ui.actions]]                    # API.md
[[ui.keybindings]]                # API.md

[settings]                        # G18
schema = "settings.schema.json"

[json_schemas.<id>]               # G15
schema = "schemas/x.json"
file_globs = ["**/*.x.json"]

[ipc.provides.<service>]          # G16
schema = "ipc/x.schema.json"
accept_from = ["ext.id"]
[ipc.topics.<topic>]
schema = "ipc/topic.schema.json"
subscribers = ["ext.id"]
[[ipc.consumes]]
extension = "ext.id"
service = "x"                     # or topic = "y"

[git.hooks]                       # G12
before_remote_op = { ops = [], remotes = [], on_timeout = "allow" | "deny" }
after_remote_op  = { ops = [], remotes = [] }
[git.credential_provider]
hosts = []
protocols = ["https"]
ops = ["get"]
cache_ttl = "60s"
on_timeout = "pass"

[sidecars.<id>]                   # G22
download = { url = "", sha256 = { "<os>-<arch>" = "" } }
binary = ""
protocol = "jsonrpc-stdio" | "lsp-stdio" | "raw"

[[capabilities]]                  # §5.5
kind = "..."
```

**Validation** (`zed-extension` CLI `validate_extension_manifest`, `zed/crates/extension_cli/src/main.rs:265`, and again at host load):

- every surface, slot and file handler has a matching `ui` surface capability;
- every file handler glob is covered by `editor`/`fs` capabilities;
- every `terminal.commands` entry is a bare program name;
- `sidecars` require a `proc` capability with `resolve = "extension_binary"`;
- `secrets.bound_hosts` ⊆ declared `net.hosts`;
- `git.hooks.remotes` are covered by the `git` capability;
- every capability has a `reason`, which is required for severity ≥ medium;
- `kind` values unknown to the host are a hard error (API.md §5.8.1).

**Registry.**

- `ExtensionProvides` (`zed/crates/cloud_api_types/src/extension.rs:38` [V]) gains `Panels`, `Items`, `StatusItems`, `FileHandlers`, `GitIntegrations` and `TerminalEnv`.
- The registry `manifest.json` embeds the capability list, so permissions show before install.

### 5.5 Permission taxonomy

All kinds are `[[capabilities]]` tables. Effective permission is **manifest ∩ global ceiling (`granted_extension_capabilities`) ∩ per-extension grant**, with deny-wins (API.md §5.8.1 and §5.8.6). The global ceiling's default for every new kind is *no grant*. Severity drives the consent UI.

#### Filesystem: `fs`

| Field | Values | Default |
|---|---|---|
| `scope` | `extension_work`, `extension_install`, `worktree`, `worktree_root`, `project`, `open_buffers`, `home`, `temp`, `path`, `user_selected` | required |
| `op` | `read`, `stat`, `list`, `write`, `create`, `delete`, `rename`, `copy`, `watch`, `chmod_exec` | required |
| `glob` | globset patterns relative to the scope root; `!` = deny | required except `extension_work`/`temp` |
| `path` | absolute dir for `scope = "path"` (env vars `$HOME`, `$XDG_*` expanded by host) | – |
| `pick`, `extensions`, `remember` | for `user_selected`: `file`, `directory`, `save_file`; extension filter; `session`, `per_workspace`, `always` | `session` |
| `follow_symlinks` | `never`, `within_scope`, `any` | `within_scope` |
| `include_private` | honour `file_scan_exclusions`/`private_files` unless true | false |
| `max_file_size`, `max_total_write`, `max_watchers`, `max_open_handles` | sizes and counts | 64MiB / 512MiB / 8 / 32 |
| `remote` | `allow`, `deny` (worktree scope on remote projects) | allow for worktree, deny otherwise |

Built-in non-grantable denies: `~/.ssh`, `~/.gnupg`, `~/.aws`, `~/.azure`, `~/.config/gcloud`, `~/.kube`, `~/.docker/config.json`, `~/.netrc`, `~/.git-credentials`, browser profiles, OS keychains, Zed's own `extensions/installed/*` (other extensions), Zed's settings and DB files. Write is denied on `/`, `/usr`, `/etc`, `C:\Windows` and `C:\Program Files`. Path resolution follows API.md §5.8.2: lexical normalize, canonicalize the nearest ancestor, prefix check, and no-follow opens.

Severity:

- low: `extension_*`, `temp`;
- medium: `worktree` read, `user_selected`;
- high: `worktree` write/delete, `project`, `home` read;
- critical: `home` write, `path`.

#### Network: `net` (plus legacy `download_file`)

| Field | Values | Default |
|---|---|---|
| `schemes` | `https`, `http`, `wss`, `ws`, `tcp`, `tls` (`tcp`/`tls` need `raw = true`) | `["https"]` |
| `hosts` | exact, `*.suffix`, IP, CIDR, `localhost`; dynamic tokens `user_configured`, `flow_declared`; `*` (critical) | required |
| `ports` | numbers or ranges | scheme default |
| `methods` | HTTP methods | `["GET"]` |
| `paths` | `*`/`**` segment grammar | `["/**"]` |
| `purpose` | `api`, `images`, `download` | `api` |
| `websockets`, `raw`, `connect_only` | bool | false |
| `redirects`, `redirect_hosts` | `none`, `same_host`, `allowlisted`, `any`; list | `same_host` |
| `allow_credentials` | Authorization/Cookie headers (plain or via secret handle) | false |
| `resolve` | `public_only`, `loopback_ok`, `private_ok` | `public_only` |
| `dns` | raw lookup permission | false |
| `prompt` | `never`, `per_host` (runtime approval per host:port, stored with context hash) | `never` |
| `max_request_body`, `max_response_body`, `max_concurrent`, `rate_limit`, `timeout` | limits | 1MiB / 64MiB / 4 / 60/min / 30s |

`download_file` gains `sha256 = "required" | "optional"`, `max_bytes`, `preserve_mode` and archive types. It is always checked against the manifest under schema 2. Every hop of a redirect is re-checked. The host pins the resolved IP. WASI sockets are only enabled with `socket_addr_check` bound to these rules (G19).

Severity:

- medium: named public hosts;
- high: `allow_credentials`, `private_ok`, raw TCP to named hosts, `per_host` prompts;
- critical: `*`, `user_configured` together with `private_ok` and wide port ranges.

#### Process: `proc` (plus legacy `process:exec`, `npm:install`)

| Field | Values | Default |
|---|---|---|
| `mode` | `exec` (default), `user_attested_shell` | `exec` |
| `command` | program name, absolute path, dynamic `flow_declared`, or `*` (critical) | required for `exec` |
| `resolve` | `path_lookup`, `absolute_only`, `extension_binary` | `path_lookup` |
| `alt_binaries` | explicit extra interpreter paths in fs scopes | [] |
| `sha256` | for `extension_binary`: literal or `pinned_by_sidecar` | required for `extension_binary` |
| `args`, `arg_patterns`, `deny_args` | `*`/`**` grammar, anchored regex, always-deny | required |
| `shells` | for `user_attested_shell`: `{unix, windows}` | `/bin/sh`, `cmd.exe` |
| `cwd` | `extension_work`, `worktree`, `worktree_root`, `temp`, `user_selected` | `extension_work` |
| `env` | `{inherit = none \| allowlisted \| shell_env, allow = [...], deny = [...], set = {...}}` | `none` |
| `stdin`, `stdout`, `stderr` | `none`/`pipe`, `pipe`/`null` | `none`, `pipe`, `pipe` |
| `output` | `to_guest`, `to_secret`, `to_store` | `to_guest` |
| `max_output`, `timeout`, `long_running`, `max_concurrent` | limits | 16MiB / 30s / false / 2 |
| `process_group`, `kill` | bool; `own_children_only` | true; fixed |
| `prompt` | `never`, `per_program` (for dynamic `command`) | `never` |
| `sandbox` | `none`, `no_network`, `readonly_fs` (best effort per OS, [A]) | `none` |

Always refused, regardless of grants:

- known shells and interpreters with `-c`/`-e`-style args under `exec`;
- `sudo`, `doas`, `runas`, `pkexec`;
- writing the guest's own `extension_install` dir.

Severity:

- medium: named binaries with pinned args;
- high: `**` args, `shell_env` inheritance, `long_running`;
- critical: `user_attested_shell`, `flow_declared`, `*`, `extension_binary` (native code).

#### Other kinds

| Kind | Fields | Severity |
|---|---|---|
| `ui` | `surfaces` (`panel`, `item`, `status_item`, `toolbar`, `modal`, `toast`, `context_menu`, `decoration`, `file_handler`, `slot:<id>`), `max_panels`, `max_open_items`, `max_tree_nodes`, `data_stores`, `allow_images`, `allow_markdown_links`, `raw_colors` | low (`slot:*` medium) |
| `sys` | `op` (`metrics`, `processes`, `signal`), `min_interval`, `include_cmdline`, `name_filter`, `signal_scope = "same_user"`, `signals`, `confirm = "each"` (fixed for non-children) | metrics low, processes medium, signal high |
| `terminal` | `op` (`run_task`, `env`), `commands`, `args`, `confirm` (`never`, `first_time`, `always`), `cwd`, `path_prepend` (fs-scoped), `vars` | run_task medium, env high |
| `secrets` | `keys` (glob under the extension namespace), `op` (`store`, `open`, `delete`, `list`, `prompt`, `reveal`), `bound_hosts` (or `same_as_key`) | high (`reveal` critical) |
| `auth` | `flow` (`oauth_device`, later `oauth_pkce_loopback`), `providers` with exact URLs and scopes | high |
| `git` | `op` (`read`, `clone`, `config_read`, `config_write`, `hook_before_remote_op`, `hook_after_remote_op`), `config_keys`, `config_scope = "local"`, `clone_hosts`, `remotes`, `stderr_tail_bytes` | read medium, hooks/config high |
| `git_credentials` | `hosts`, `secret_keys` | critical |
| `editor` | `op` (`read_selection`, `read_buffer`, `edit`, `decorate`, `diagnostics_read`), `scope` (`active_buffer`, `open_buffers`, `project`), `glob`, `max_chars` | read medium, edit high |
| `workspace` | `op` (`open_path`, `open_folder`, `reveal`) | medium |
| `kv` | `scopes` (`extension`, `workspace`, `project`), `max_bytes` | low |
| `settings` | `op` (`read`, `write_own`), `categories` | low |
| `clipboard` | `op` (`write`, `read` gesture-bound) | write low, read high |
| `notify` | `levels`, `rate_limit`, `os_notifications` | low |
| `open_url` | `schemes`, `hosts`, `confirm` (`never`, `non_allowlisted`, `always`) | low |
| IPC (manifest `[ipc.*]`) | `accept_from`, `consumes`, schemas | medium (`accept_from = ["*"]` high) |

#### Combination rules (shown as a single warning in consent)

- `fs` read on `worktree`/`home`, plus `net` to any non-registry host: "can read your files and send them to X".
- `secrets` without `bound_hosts`: refused at validation.
- `proc` `user_attested_shell`, or `flow_declared`, plus `net` `*`: "can run commands and contact any host".
- `terminal env` plus `path_prepend`: "can change which programs run in your terminals".

### 5.6 Consent and grant flow

This extends API.md §5.8.6. Per-extension grants are stored as API.md proposes (`extension_permissions`). Following API.md OQ 2, they move to the local DB, with only the ceilings in settings [A: the recommended choice, not decided upstream].

1. **Install.**
   - The Extensions page shows the grouped permission list with severity, `reason`, combination warnings and the sidecar notice.
   - Optional capabilities can be toggled off.
   - Critical items need an explicit per-item checkbox.
   - Dev extensions are auto-approved with a banner, but are still validated against their manifest.
2. **First use.**
   - Declared-but-ungranted capabilities with `prompt = "on_first_use"` suspend the call and show a host prompt: Allow once, session, or always / Deny.
   - Dynamic scopes (`per_host`, `per_program`, `user_configured`, `flow_declared`) always prompt for each new concrete value. The prompt shows the exact host:port or program plus args. Approvals are keyed by `(extension, kind, value, context hash)`, so editing a flow file re-prompts for new hosts.
3. **Host-drawn confirmations.**
   - Mandatory: `sys.signal` of non-children, `terminal.run-task` with `confirm != never`, and the `auth` device code display.
   - Destructive guest-requested `ui.confirm`.
   - These are trusted chrome, and extension trees can never draw over them (API.md R4).
4. **Update.** A widened capability set, or new combination warnings, keeps the new version staged until it is re-approved (API.md §5.8.6 #5).
5. **Revocation.**
   - The grant table is live (`Arc<RwLock<GrantTable>>` replacing the snapshot at `zed/crates/extension_host/src/wasm_host.rs:637` [V]).
   - Revoking tears down processes, watchers, sockets, row stores and terminal-env contributions (and invalidates the env cache).
   - It revokes secret-handle usability immediately.
6. **Project settings.** `.zed/settings.json` can only narrow. It can never widen a grant or a ceiling.
7. **Audit.** Every denial and every runtime-prompted access is logged per extension and visible on the extension card. The first denial in a session shows one non-modal toast.

---

## 6. Phased roadmap, risks and open questions

### 6.1 Phases

| Phase | Contents | Unlocks | Depends on |
|---|---|---|---|
| **U (upstream track, parallel)** | `Panel` `SharedString` keys; panel ordering (F25); `sysinfo` disk/network features; registry URL setting and install errors (F22); optional GPUI `.scale()` | usability of many panels | upstream review |
| **0. Safety and permissions** | wasm `ResourceLimiter`, trapping deadlines, per-call timeouts (API.md §5.8.7); priority call queue with classes (§5.2); `CapabilityGranter` live table; schema 2 manifest parsing and validation; gating of legacy ungated imports; consent UI skeleton; per-extension grants | nothing visible; prerequisite for all | – |
| **1. Surfaces MVP** | WIT 0.9.0 skeleton; `crates/extension_ui` (validator, differ, builder); panel, item, status item, modal, toast; actions and keybindings; layout/content/input nodes, `list`, `tree`, `tabs`, `accordion`, `split`, `markdown`; `kv`; timers; `[activation]` and `workspace` basic events; `fs` (worktree, work, temp), `net` HTTP, `proc.run`; `[settings]` schema | **F12**; skeletons of F5–F8 | 0 |
| **2. Data and processes** | `ui-data` row stores, series, logs; `data_grid`, `chart`, `gauge`, `stat`, `card`, `log`, `banner`; `sys`; `proc.spawn`/`spawn-v2`/`spawn-user-command`; `terminal.run-task`, `terminal_view`; `fs.watch`; `download_file` sha256; remote images; `ui.confirm`; `open_url` | **F1, F3, F4, F7, F8, F11** (except menu), F2 (without git), F5/F6 (without PATH) | 1, U (panel keys) |
| **3. Rich editing and identity** | `editor`/`diff` nodes plus `provide-completions`; file handlers; JSON schemas; `schema_form`; drag and drop; `graph` (gpui_flow port); `secrets`, `auth` device flow, secret-templated headers; `net` TCP/TLS; `ipc`; `workspace.open-*`; editor context-menu registry; sidecars | **F9, F10, F13** (panel), **F14, F21** (ipc), F11 menu | 2 |
| **4. Core hooks** | `TerminalEnvProvider` (F17); git read API; git remote-op hooks; credential provider routing plus `zed --git-credential`; git panel slots | **F5, F6** complete, **F15, F16, F17**, F2 git card, F13 identity | 3, upstream acceptance of hooks |

**Milestones.**

Milestones M0 to M1b follow the RFC's own phase order: status bar first, then read-only panels (§7.1). That is also the order `tiagoacarvalho/extension-api-with-ui` announced (thread.md, 2026-11-06). That fork is still at upstream `9a8ab54db4`, with no UI work committed yet [V: `git log` of `/home/tiago/Repositories/extension-api-with-ui`]. These milestones can therefore be its first PR-sized steps.

- **M0** (end of phase 0): an extension with schema 2 installs on Nightly with a consent dialog. Ungated `fetch` is closed for schema 2.
  - Can be done in parallel with M1a.
  - Only the `ui` capability kind and the consent list are strictly required before M1a.
- **M1a, the RFC's phase 1: status-bar item.** This is the smallest end-to-end slice:
  - `[ui.status_items.*]` in the manifest;
  - the `ui` WIT with `surface-open`, `surface-event` and `push-render`;
  - an `ExtensionStatusItem` implementing `StatusItemView` (`zed/crates/workspace/src/status_bar.rs:42`, `add_right_item` at `:421` [V]);
  - nodes `row`, `text`, `icon`, `badge`, `progress` and `button`;
  - a click that opens a `menu` or dispatches `InvokeExtensionAction`.
  - It proves the validator, the differ, theming and the call budget.
  - Demo extension: a CI status item polling GitHub Actions (§7.5), or jmtrs's Nyan Cat progress (thread.md).
- **M1b, the RFC's phase 2: read-only panel.** "Hello panel" plus F12 npm bootstrap (toast), shipped as dev extensions.
  - Requires the `Panel` `SharedString` keys from the U track (§4).
- **M2**: `forge.system` (F1, F3) and the Rust and .NET parts of `forge.runtimes` running without fork patches. The fork drops `cockpit_panel`, `processes_panel`, `rust_panel` and `dotnet_panel`.
- **M3**: `forge.database` with SQLite and Postgres, and `forge.flows` Designer editing a `.flow.json` with undo and save through the buffer.
- **M4**: `forge.helm` signs in through the device flow, and git push gating works through hooks. The fork then carries only the F23 and F24 product patches.

### 6.2 Risks

1. **Serialized guest execution** (API.md §1.2, `WasmExtension::call`, `zed/crates/extension_host/src/wasm_host.rs:912` [V]). A bundle like `forge.runtimes` runs five panels plus hooks on one queue.
   - Mitigations:
     - priority classes and hard budgets (§5.2);
     - host-owned stores, so scroll, sort, filter, graph drag and log streaming never call the guest;
     - streaming via `stdout-to`;
     - watch patterns instead of per-chunk calls;
     - hook verdict caching.
   - Option: a second instance per extension for `background` class calls, which would require the guest to keep state in `kv` [A: memory cost acceptable].
   - Long term: component-model async (WASI p3) [A: not yet usable in wasmtime 48 for this].
   - Bundling trades IPC complexity for queue contention. Split bundles if profiling shows contention.
2. **Performance of large JSON trees and grids.**
   - Parse and validate off the main thread.
   - Reconcile by key on the main thread, with one `notify` per batch.
   - Keep data in stores rather than trees.
   - `uniform_list` virtualization for `list`/`tree`/`data_grid`.
   - Size and rate caps.
   - Measure with GPUI benchmarks.
   - Targets [A]: a 2k-node tree reconciled in under 2 ms; a 1M-row grid scrolling at frame rate.
3. **Security.**
   - **Token exfiltration**: mitigated by secret handles bound to hosts. DB passwords need `reveal`, which is critical and clearly flagged.
   - **Shell injection**: user-attested provenance and host-shown task command lines.
   - **Native sidecars**: sha256 pins, critical severity.
   - **Hook abuse**: a `before_remote_op` denial is only advisory UI. A credential provider can redirect auth for its declared hosts only.
   - **Spoofing**: trusted chrome for prompts and confirmations, attribution on every surface, slots limited to a closed set.
   - **Terminal env**: denylist of dangerous variables, and PATH entries only inside extension dirs.
   - **Dynamic scopes**: approvals keyed to content hash.
   - **Supply chain**: auto-updates with widened permissions are staged.
4. **Upstream acceptance (high).**
   - **Who has spoken.** The only Zed-team voice in the thread is macraig. macraig's GitHub association is CONTRIBUTOR, not MEMBER, but macraig speaks as "our team" (thread.md §1 note).
   - **What macraig said** (2026-05-21): the feature is "closely tied to our extensions roadmap", "needs to be driven by our team to make sure it's done in a coherent way", and is "not something we're likely to get to in the near future". macraig promised updates, but none followed, and vitalics' ETA request went unanswered (thread.md §6).
   - **Competing direction.** The team may prefer `embedded_gpui` (§7.4).
   - **Consequence.** A community-built UI API is unlikely to be merged as a whole, and certainly not before the team's design exists. mentenaz reached the same conclusion: "I doubt Zed Industries would accept this as a feature".
   - **Strategy:**
     - **(a) Community fork track.** Develop `extension_ui` in `tiagoacarvalho/extension-api-with-ui`, gated to Dev/Nightly via `wasm_api_version_range`. Rebase regularly, since mentenaz reports regular syncs with about 20-minute full builds (thread.md). Treat it as a reference implementation and an RFC appendix, not a PR.
     - **(b) Upstream only small, standalone pieces**, each useful without UI extensions: phase 0 safety, gating of ungated `fetch`, `Panel` `SharedString` keys, panel ordering, the registry URL setting and install errors, and `.scale()`.
     - **(c) Post findings back to discussion #53403.** Include the permission taxonomy, the call-budget data, and the §7.5 coverage. The team then has a tested design to reuse when it picks the work up "in a coherent way".
     - **(d) Keep the WIT surface additive and versioned** (`since_v0.9.0` or later), so it can be renumbered to whatever version the team chooses.
   - Mitigation: land the uncontroversial pieces first: phase 0, the U track, `Panel` keys, gated `fetch`.
   - Keep `crates/extension_ui` self-contained, behind `wasm_api_version_range` gating (Dev/Nightly).
   - Keep the hooks (phase 4) as small generic registries that are useful without extensions.
   - If rejected, the fork carries only `extension_ui` plus hooks, which is far smaller than today's 388-file diff (apps §0).
5. **API freeze.** A node catalog this large is a long-term contract. Version the node set via `host-capabilities()`, and keep embedded-widget nodes minimal.
6. **Toolchain constraints in the guest.** Backends that use tokio, reqwest, sysinfo or native DB drivers must be rewritten against host resources [A]. This is the main effort driver of features 9, 10 and 13.
7. **Remote development** (API.md R7). Terminals, git hooks and `sys` are local-only in the fork (apps §16, §17). For remote projects:
   - `fs worktree` is proxied;
   - `sys` reports the client machine (labelled);
   - `terminal env` and git hooks are refused unless the extension also loads on the remote (`zed/crates/extension_host/src/headless_host.rs`).
8. **Accessibility.** The new visualization nodes (`chart`, `graph`, `data_grid`) need a11y roles and keyboard navigation in the host implementations from day one.

### 6.3 Open questions

1. Where should per-extension grants and dynamic-scope approvals live: settings JSON, the local DB, or the keychain for critical items?
2. Should `secrets.reveal` exist at all? The alternative is host-side "secret-aware" TCP auth helpers for Postgres and MySQL. That would be cleaner, but it puts protocol knowledge in the host.
3. Is component-model async (p3) close enough to replace the priority queue and second-instance design?
4. Instance model per workspace or per window (API.md OQ 5). Proposed: per workspace, with `workspace-info.id` passed to every export.
5. Should `terminal.run-task` support "reuse the same terminal" so that a bundle behaves like the fork's single shared console?
6. Should slots be extended to the project panel context menu and the tab context menu in phase 3 or 4?
7. Should the Flows engine stay inside the extension, or become a headless sidecar so flows can run outside Zed (CI)? The current plan keeps it in the guest for sandboxing. A sidecar would be faster to port but critical-severity.
8. Should the registry flag or review extensions that request `critical` kinds (API.md OQ 7)?
9. Should the git credential helper also serve `ssh` (via `SSH_ASKPASS`)? That interacts with `zed/crates/askpass/src/askpass.rs`.
10. Is the bundling of five extensions what the fork author wants, or should each feature be an independently installable extension? That would raise the IPC requirements (G16).
11. The collapsed thread replies are now known (thread.md is a full GraphQL capture). mentenaz's 2026-11-04 panel model is compared in §7.3. Still open: should `zed-ui/1` offer a compatibility import for Forge panel schemas (`TextField`/`Button`/`List` with bindings and actions)? Forge packages could then run in Zed with a sidecar backend. Today such schemas exist only in the Tauri app, not locally.
12. jeremycastanza's question of how a contributor group should organise went unanswered. Should the community fork (risk 4a) set up a coordination channel and a shared requirements list, seeded by applications.md and §7.5?

---

## 7. Reconciliation with discussion #53403

Source for every claim in this section: `thread.md` (full capture, 2026-11-06). Section numbers like "thread §1" refer to that file.

### 7.1 The RFC body compared with this plan

The RFC (nguyenphutrong, thread §1) and this plan agree on four principles:

- GPUI-only rendering;
- no webviews;
- predefined, validated components;
- automatic theming.

The table compares the details.

| Topic | RFC | This plan | Difference and reason |
|---|---|---|---|
| Boundary | WASM host with a "Visual Extension API (WIT)" | Same: WASM guest, WIT `since_v0.9.0` (§5.2) | None in substance. |
| Element tree | `components.element` variant (container, text, button, text-input, table, list, icon), with `container-props` holding children | `zed-ui/1` JSON string (§5.3) | WIT forbids recursive variants, so a typed `element` containing children cannot be expressed directly (API.md §5.4). The RFC's set would need a flat node table or JSON. JSON matches house style (`get-settings`, schemas) and is easier to version and fuzz. |
| Panels | `panel` interface: `register-panel(panel-info)` at runtime; `panel-handle` resource with `update(element-tree)`, `set-visible`, `is-visible`; positions left-sidebar, right-sidebar, bottom-panel | `[ui.panels.*]` declared in the manifest; contents via `surface-open`/`render-reply` and `push-render` | Static declaration lets Zed lay out dock buttons and restore persisted size and position before the wasm loads (API.md §5.1 #3, OQ 9), and lets consent show surfaces before install (R3). `push-render` covers the RFC's `panel-handle.update` push model. Visibility maps to `lifecycle shown/hidden`. |
| Status bar | `status-bar.add-item(status-item {id, content, priority (lower = further left), tooltip, on-click})`; `status-content` = text, icon or text-with-icon; handle with `update-content` and `remove` | `[ui.status_items.*]` with `alignment` and `priority`; content is a constrained `row` of `icon`/`text`/`badge`/`progress` (API.md §5.7) | A superset. The RFC's three content kinds are a subset of the allowed row. `remove` becomes `visible: false`, or uninstall. |
| Interactivity | `callback-registry` resource and `callback-id` strings; `handle-event(ui-event) -> option<element-tree>` | Event props `{id, payload}` plus `surface-event` returning `ui-response {render, commands}`, plus closed local effects | Equivalent. Handler ids are data, as in the RFC. The plan adds host-local effects (copy, toggle, `graph_fit`), so trivial interactions skip the guest queue (risk 1). It also adds commands such as toast, modal and open-item. |
| Extension trait | `build-ui() -> ui-definition` and `handle-event` added to `Extension` | New exports in a new world version, with SDK default no-ops | nakajimayoshi objects that `build_ui` breaks existing extensions (§7.2). It does not here: older guests bind to older worlds (`zed/crates/extension_host/src/wasm_host/wit.rs:60` [V]), and the SDK gives every new export a default. The manifest replaces `build-ui` for static parts. |
| Table | `table-props`: columns, rows, selection-mode, on-selection-change | `table` (small, inline) plus store-backed `data_grid` (G1) | Rows inside the tree do not scale to query results (risk 2). |
| Phases | 1 status bar (1–2 wk) · 2 read-only panels (3–4 wk) · 3 interactive panels: forms, tables, i18n editor, DB browser (6–8 wk) · 4 advanced: charts, image preview, custom renderers (2–3 mo) | 0 safety · 1 surfaces · 2 data and processes · 3 rich editing and identity · 4 core hooks (§6.1); milestones M1a/M1b follow RFC 1/2 | The plan adds phase 0, because ungated `fetch`, missing wasm limits and missing consent UI (API.md §2.4) must be fixed before UI raises the stakes. RFC phases 3 and 4 map onto plan phases 2 and 3. Plan phase 4 (git, terminal and env hooks) has no RFC counterpart: the RFC excludes workspace integration, but applications.md shows the fork needs it (mentenaz's lesson "an API that only offers an isolated widget would not have been enough"). |
| Non-goals | No webviews, no custom editor implementations, no direct windowing access | No webviews and no windowing access. Custom editors are allowed only as file handlers over a host Buffer (G14). | Flows (F10) needs a `.flow.json` designer. G14 stays within the RFC's spirit: the guest gets no editor internals, only buffer text and edits. |
| Security | WASM sandbox; no direct fs (worktree API); network via the existing `http-client`; no HTML/CSS; validated components; resource limits (panel size, component count); frame-time budget; auto-disable of slow extensions; lazy loading | All of that, plus the R3 taxonomy (§5.5), consent flow (§5.6), call classes and budgets (§5.2) | The RFC assumes `http-client` is an acceptable network path, but it is ungated today (API.md §2.4 #1). The plan gates it. "Frame-time budget" cannot apply directly, because guests never run during paint (render from cache). The plan's budget applies per call instead, and repeated overruns mark the extension unhealthy, matching the RFC's auto-disable. |
| Accessibility | ARIA metadata in WIT | `a11y {role, label, description}` on every node (§5.3) | Same. |
| Alternatives | Webview rejected; GPUI chosen; React-like DSL possible later | Same choice. A DSL can be a guest-SDK layer that emits `zed-ui/1` (no host change). | The JSON contract also makes non-Rust guests feasible (API.md OQ 6). |
| Impact estimate | New `extension-ui` crate of about 2–3k lines; changes to `extension_host` and `workspace` | `crates/extension_ui` plus host nodes, `ui-data`, the new interfaces, and hooks in `project`, `git_ui`, `editor` | Considerably larger [A: well above 2–3k lines]. That is the price of covering the fork's features rather than only the RFC MVP, and it is why phases 0–2 are separable. |

### 7.2 nakajimayoshi's `UIExtension: Extension` trait

**The proposal** (thread §2, 2026-05-11):

- a separate trait `UIExtension: Extension`, with a GPUI `render(...) -> impl IntoElement`;
- an API like `cx.create_pane(PaneOptions { position: PanePosition::Left, width: px(500) }, |_, cx| cx.new(|_| HelloWorld {...}))`;
- the argument that "there's no better abstraction for Zed UI extensions than GPUI".

**What the plan keeps from it.**

- **Separate trait.** The SDK exposes a separate `UiExtension` trait (or optional methods with defaults) on top of `Extension`, so non-UI extensions are unaffected. The world split in §7.1 delivers the same backwards compatibility.
- **GPUI vocabulary.** `row`/`column` alias `h_flex`/`v_flex`, and styles are Tailwind-like tokens, so the authoring model feels like GPUI. A guest-side Rust builder crate can mirror GPUI's builder API (`div().gap_2().child(...)`) while emitting JSON. This is how the "familiar to GPUI users" benefit survives R2 [A: the SDK design is not specified further here].

**Why R2 rejects guest-side GPUI `render`.**

1. **Security.** nakajimayoshi's own point: running GPUI code in the main process "eliminates Zed's entire security model and allows any UI extension developer to run native code on your machine". Native in-process code bypasses every R3 permission.
2. **No GPUI in the guest.** Compiling GPUI into a WASM guest is not a workable fix. A guest has no window, GPU or platform handle. Element trees hold closures and entity handles, and those cannot cross the WIT boundary. `gpui_web` targets the browser, not a sandboxed guest (API.md §4.1, R9).
3. **Stability.** A GPUI-typed API would freeze GPUI internals as an extension ABI. Every GPUI refactor (for example the `.scale()` signature change in apps §18) would break extensions. A declarative schema can stay stable while GPUI evolves underneath.
4. **Calling discipline.** GPUI `render` is synchronous on the main thread every frame, while guest calls are async and serialized (API.md §1.2). A guest `render` would put the slowest extension inside the frame loop. Retained JSON trees keep paint independent of the guest.

**Their IPC/WIT security points, and how the plan answers them.**

| Point | Answer in the plan |
|---|---|
| "A serde implementation of GPUI so subprocesses make GPUI calls over IPC, with the main process sanitizing calls and validating permissions" | Same principle: the host is the only GPUI caller, and every guest request is validated. What crosses is a declarative description (validated against `zed-ui/1`, with size, depth and rate limits; §5.3), not serialized GPUI calls. Serialized calls would still expose GPUI's whole surface, and that surface is too large to sanitize. |
| "WIT is probably better than IPC because it is already supported by the extension API and is faster" | Adopted (R1). The guest stays inside `extension_host`'s Wasmtime instance, with no extra process. |
| "How GPUI components can be translated across WIT has yet to be determined" | Answered: JSON-in-string for trees (the WIT recursion limit), typed WIT for events, commands and data stores, and host-built GPUI elements (API.md §5.9). Rich widgets are host nodes (`data_grid`, `graph`, `editor`), not guest code. |
| Permission validation in the main process | `CapabilityGranter` at every host import (§5.5), manifest ∩ ceiling ∩ grant, live revocation (§5.6). |

### 7.3 Prior art: mentenaz's Forge (Tauri) schema-driven panels

**Provenance.** Described only in thread §2 (mentenaz, 2026-11-04). The Forge (Tauri) app is a separate codebase. It is not in `fork/`, which has no schema system, no `--introspect`/`--function` and no Tauri code (apps §0 "Schema-driven panel system: does not exist locally" [V per apps]). Everything below is therefore as described by mentenaz and unverified against code [A].

**The design.**

- **Panel declaration:** direction (left/right/bottom), a registered icon, a stable ID, a title, and a UI body.
- **Packaging:** a panel is a JSON schema plus an optional backend binary. The host renders the schema without running extension UI code.
- **Components:** sections, status/metric rows, inputs, buttons, toggles, selects and templated lists. Components read named bindings, and buttons trigger named actions with the current input values.
- **Backend calls** are one-shot:
  - `binary --function <name> --Arguments <raw>`;
  - one JSON object on stdout;
  - on error, a message on stderr and a non-zero exit;
  - a timeout, after which the process is killed;
  - `--introspect` lists the available functions.
- **Theming:** the host passes a small set of theme tokens.
- **Portability:** the same package renders in both the Tauri build and the GPUI build.
- **Example:** panel `weather` with a `TextField` `city`, a `Button` action `search`, a `List` binding `results`, and backend `./logic.exe`.

**mentenaz's stated limits, and how the plan addresses each.**

| Limit (thread §2) | Plan |
|---|---|
| Rich components: tables, charts, markdown and node-graph canvases "needed native components" | A fixed set of rich host built-ins: `data_grid`, `chart`/`gauge`, `markdown`, `graph`, `editor` (G1–G4). This matches the "small set of rich built-ins" conclusion. |
| Streaming: one-shot calls do not fit a system monitor; "JSON lines over stdout" is needed | Host-owned stores fed by `append-rows` (JSON-lines), `push-points` and `append-log` (G1). `proc` `stdout-to` pipes output with no guest call (G7). A `jsonl-stdio` sidecar protocol (G22). Timers and `push-render` for polling (API.md §5.4). |
| Permissions: the process boundary gives crash isolation, but "backends run with full user permissions" | The core reason for the WASM guest (below), plus the §5.5 taxonomy. |
| Workspace context: the active project, git state and settings | `workspace`, `git` read, `settings`, `fs worktree`, and activation events (G12, G17, G18). |

**What to adopt.**

1. **Discovery without running code.** Forge's `--introspect` lets the host learn a backend's functions. The equivalent here:
   - everything security-relevant is in `extension.toml`, so the host and registry can show it before any code runs (surfaces, `[[ui.actions]]`, `[ipc.provides.*]` with schemas, capabilities);
   - the host advertises its side through `ui.host-capabilities()` (supported node types and limits, §5.3);
   - for sidecars, the host runs `--introspect` once at install (G22).
2. **Named bindings and actions.** Forge's "components read from named bindings; buttons trigger named actions with current input values" maps to `source: "rows:<id>"` store references and event handler ids.
   - Small addition: `on_click.include_values: ["city"]` makes the host attach the current values of the named `input` nodes to the event. Forms then need no `on_input` round trips.
3. **Theme tokens only.** This matches `zed-ui/1` exactly. Nodes name tokens, and the host resolves them. The guest never receives colour values, only `current-theme` metadata (appearance, density).
4. **Timeout and kill.** Adopted as call budgets (§5.2) and `proc.timeout` (§5.5).
5. **Toolkit-neutral schema.** `zed-ui/1` does not name GPUI types either, so a second renderer is possible, but it is not a goal. A Forge-schema import (§6.3 OQ 11) would be a guest-SDK translator, not a host feature.

**Why the guest is WASM, not a native backend binary.**

- **R1.** The rule requires the WASM extension pipeline: one `extension.wasm`, versioned by `zed:api-version`, loaded by `extension_host`, distributed by the registry. A per-OS native binary is outside that pipeline. It needs a build matrix and per-platform downloads, and the registry cannot verify it beyond a hash.
- **R3.** With a native backend the manifest can only describe intent: once running, the binary can open any file, socket or process. This is mentenaz's "full user permissions" limit. In WASM, every effect goes through a host import, so `CapabilityGranter` sees every call and can deny or prompt. That is the only way the detailed fs, net and proc model is actually enforced, not just declared.
- **Cost.** Forge's one-shot calls spawn a process per call. A resident WASM instance answers events in-process, with no spawn cost. Persistent state lives in guest memory or `kv`, and no ad-hoc files are needed.
- **Crash isolation**, Forge's one gain from the process boundary, is kept. A trapping guest is caught by Wasmtime, marked unhealthy and restartable (API.md §5.8.7). The host keeps rendering the last tree.

**Relation to the native-helper escape hatch (G22).** The sidecar is the Forge backend model, admitted deliberately:

- It exists for code that cannot target `wasm32-wasip2` (Oracle, if its driver is native; possibly heavy engines).
- It is optional per capability, carries a sha256 pin and is consent-flagged critical.
- `oneshot-json` and `jsonl-stdio` make existing Forge-style backends reusable as-is.
- The plan's position: sidecars are the exception for code that has to be native, and never the default backend model.

### 7.4 `embedded_gpui`

**Facts.**

- pleahmacaka (CONTRIBUTOR) linked `https://github.com/zed-industries/embedded_gpui` on 2026-09-11 with "Looks like they're experimenting with something".
- tiagoacarvalho (2026-11-04) shared a screenshot suggesting the branch had not moved in two months.
- mentenaz read it as the team "first trying that before incorporating it into zed".
- It is not in the local Zed tree (API.md §4.1).

**Unverified.** Commenters describe it as a "two-stage embedding" with "javascript plugins" (tiagoacarvalho, thread §2). The team has not confirmed this, and I did not inspect the repo [A].

**As an alternative.** If it embeds GPUI views driven by sandboxed JS, it would be a third boundary model, beside guest-side GPUI (§7.2) and declarative JSON (this plan).

- **Strength:** JS engines sandbox well, which tiagoacarvalho grants.
- **Weaknesses** (as judged in the thread, not measured):
  - a second runtime in an editor focused on speed (tiagoacarvalho);
  - a separate extension pipeline beside WIT/Wasmtime, unless the JS runs as a WASM component (for example through componentize-js) [A].

**As a risk.**

- If it is the team's direction (macraig: "driven by our team"), a WIT-plus-JSON API built outside could be superseded.
- The thread also warns that rebasing onto it would be "one hell of a merge conflict waiting to happen".

**Alignment strategy.**

1. **Before investing past M1a, inspect `embedded_gpui`.** Determine the boundary type, whether WASM is involved, and whether it has a permission model. Record findings in a plan revision.
2. **Keep the pieces that hold under any front end.** These are independent of how UI is rendered, and the team will need them whichever renderer wins:
   - phase 0 (limits, gated `fetch`, consent);
   - the R3 taxonomy;
   - `ui-data` stores;
   - the host widgets (`data_grid`, `graph`, `chart`);
   - the hooks of phase 4.
3. **If `embedded_gpui` uses a declarative or JS-component model,** `zed-ui/1` can be a compile target for it, or the reverse.
4. **If it runs native or in-process code,** it violates R2 and R3 as set for this project, and this plan remains the separate approach (Project.md: "independent approach purposeful").

### 7.5 Use-case coverage from the thread

Use cases from clementGilardy (Linear/Jira, AWS explorer, live DB viewer, CI/CD), AirboZH (SQLite viewer, JSONL preview), the RFC table, and jmtrs.

| Use case | Surfaces | API used | Permissions (summary) | Covered? |
|---|---|---|---|---|
| **Linear** tickets for current branch or file (clementGilardy) | Panel; status item with ticket id | `git.repositories` (branch), `workspace` `active-item-changed` events; `net.fetch` GraphQL `api.linear.app` with a secret-templated header; `auth` or `secrets.prompt` for the API key; `list`, `markdown`, `button` | `git read`; `net https api.linear.app POST /graphql allow_credentials`; `secrets keys=["linear/*"] bound_hosts=["api.linear.app"]`; `ui panel,status_item` | **Yes.** Linear's OAuth uses an authorization-code flow, so phase 3 needs `auth` `oauth_pkce_loopback` (listed as "later" in §5.2) or a personal API key via `secrets.prompt`. |
| **Jira** tickets (clementGilardy) | Panel, status item | Same as Linear; REST `https://<site>.atlassian.net/rest/api/3/**`; API token via `secrets.prompt` (Basic auth template `"Basic {}"` over `email:token`) | `net hosts=["*.atlassian.net"]` (or the user's site via `user_configured` + `per_host`); `secrets bound_hosts=["*.atlassian.net"]` | **Yes.** Self-hosted Jira needs `user_configured` hosts with a prompt. |
| **AWS resource explorer** (clementGilardy) | Panel with `tree` and `data_grid` | `net.fetch` to `*.amazonaws.com` needs **SigV4 request signing**. A header template cannot express it. Two options: (a) `proc` `aws` CLI with `--output json` (credentials stay with the CLI); (b) a new host helper `net` `sign = "aws_sigv4"` that uses a secret handle | (a) `proc command="aws" args=["*","describe-*","--output","json","**"] env.allow=["AWS_PROFILE","AWS_REGION","HOME","PATH"]` (`~/.aws` stays unreadable to the guest; the CLI reads it). (b) `secrets keys=["aws/*"] bound_hosts=["*.amazonaws.com"]` | **Partial.** (a) works with the phase 2 API. (b) is a **new gap**: host-side request signers (SigV4, and later others) as `net.request.sign` [A: not yet specified]. |
| **CI/CD pipeline status** (clementGilardy) | Status item (M1a demo) plus panel; Actions run graph as in F13 | Timers, `net.fetch` GitHub/GitLab APIs, `git.repositories` for the remote, `notify` on failure, `graph` for job DAGs | `net hosts=["api.github.com"]` (or `gitlab.com`), `secrets` bound, `notify rate_limit`, `git read` | **Yes.** Status item in phase 1, graph in phase 3. |
| **SQLite viewer** (AirboZH, orangethewell, nakajimayoshi) | File handler for `**/*.{sqlite,db}` opening an item; panel optional | F9's SQLite subset: guest `rusqlite` over `fs.file-handle`, `data_grid`, `editor` SQL, `export-rows` | `fs worktree read,stat glob=["**/*.sqlite","**/*.db"]` + `fs user_selected`; `ui item,file_handler`; no network | **Yes.** Phase 3 (file handler). With the powerbox only, phase 2. Row editing (orangethewell's question) needs a `data_grid` `editable` cell mode, a small addition [A]. |
| JSONL preview (AirboZH) | File handler `**/*.jsonl` | `editor.read-buffer` (or `fs` read), `data_grid` with columns inferred in the guest, `append-rows` streaming | `editor read_buffer glob=["**/*.jsonl"]` | **Yes.** Phase 3. |
| i18n manager, task runner, plots (RFC table) | Item, panel | `data_grid` (editable, as above), `terminal.run-task`, `chart` | fs worktree read/write for locale files; `terminal`; none | Yes / Yes / Yes. |
| Git GUI: commit graph, branches (RFC table) | Item | `git` read API; commit-lane rendering via `graph` with `layout: "manual"` | `git read` | **Partial.** Upstream already has `git_graph` (`zed/crates/git_ui/src/git_graph.rs` [V]); a lane-graph node may be better as a host widget. |
| Color picker, image gallery (RFC table) | Item, popover | Needs a `color_input` node, and `image` from worktree paths (today `image.path` is extension-relative only) | `fs worktree read` for images | **Gap (small).** Add `color_input` and `image.fs_path` (gated by `fs read`) to the phase 3 node list [A]. |
| Nyan Cat status bar (jmtrs) | Status item | `image` (extension asset) plus `progress`, animated by timer frames (≤ 30 renders/s) | `ui status_item` | **Yes.** Animated images would be cheaper as a host `image.animated` prop [A]. |

New gaps from this table: request signers (SigV4), PKCE loopback OAuth, editable grid cells, `color_input`, `image.fs_path`, and possibly a git lane-graph widget. All are additive, and none conflicts with R1–R3.

# Features that mentenaz.Forge.Super adds on top of Zed

Source: `/home/tiago/Repositories/mentenaz.Forge.Super`, read-only. Origin is `https://github.com/mentenaz/mentenaz.Forge.Super`. Comparison base is upstream Zed `145bcfc1bc`, the last upstream commit merged on `main` (`fbb662635f`, "Merge upstream zed-industries/zed (41 commits)"). The fork is repeatedly merged with upstream, so upstream's own recent features are not counted here.

Paths below are relative to the fork root. Line numbers refer to `HEAD` of `main`.

## 0. What the project is

### Identification (verified against the local repo)

- `git remote -v` shows only `origin = https://github.com/mentenaz/mentenaz.Forge.Super.git`. The history is the full upstream Zed history (40,991 commits, from Nathan Sobo's 2021 commits). It is not a small Tauri repo.
- The fork's own work starts on 2026-09-23 with the commit "Port gpui_component-based panels and add a database panel" (`3760132745`), authored by "Mentenaz-Server (Francois)". About 256 commits are by the fork's authors.
- It is the same lineage as `mentenaz/zed_Source`, not a different project. Merge commit `f9014e4dbf` (2026-10-04) is "Merge branch 'main' of https://github.com/mentenaz/zed_Source". It only touches `README.md`, and it shows that zed_Source was the working repo and was merged in.
- The app was then renamed "Mentenaz Forge Super" (`46129fc68b`, 2026-10-23). The repo itself was renamed to `mentenaz/mentenaz.Forge.Super` (`22fd6b2812`, 2026-11-02). So this repo is zed_Source continued and renamed, not a Tauri app.
- No Tauri code exists locally. There is no `src-tauri`, no `tauri.conf.json`, and no app-level `package.json` or `.tsx` (only Zed's `script/danger/package.json` and test fixtures). The Tauri app, which the thread calls "Forge" with about 30 engines, is not in this repo.
- The only trace of the older app is in the docs. `CLAUDE.md` and the panel READMEs say the panels were "ported from the Forge app (`E:\\Forge_GPUI`)" and rewritten against gpui-component. That predecessor is a separate codebase that is not vendored here.
- Inference: the thread's six built-in panels and its `panel_ui`/`panel_backend` split probably describe an earlier zed_Source stage or a separate design. The local code has more than six panels, and it does not use that crate split.
- `thread.md` as it stood on disk when I checked had 86 lines and did not mention zed_Source, the schema model or the Tauri build. It still labelled macraig a Maintainer. I could not verify those claims against it. The statements about the thread in this section come from the coordinator's message.

### Schema-driven panel system: does not exist locally

- I searched all of `crates`, `docs`, `assets` and `extensions` for `--introspect`, `--function`, `--Arguments`, `panel_ui`, `panel_backend`, panel schema files (`*.panel.json`), engine manifests, and a JSON-to-GPUI renderer. There are no matches, and there is no such directory in any branch's history.
- The `introspect` hits are all unrelated. They refer to database schema introspection (`database_backend`) and to a TODO in the sqls notes.
- `--function` appears only in an upstream clangd docs example.
- Every panel here is hand-written Rust/GPUI. The backends (`*_backend` crates) are in-process Rust libraries called as functions, not one-shot binaries with stdout JSON. The only JSON-declared artifacts are the `.flow.json` workflow files (feature 10), and those describe automation, not UI.
- The closest relatives are:
  - the `workflow_engine` action registry (`crates/workflow_engine/src/registry.rs`), whose `ActionDef` and `InputField` descriptors drive generated inspector forms in the Designer, and a JSON schema generated from it (`registry::body_json_schema`);
  - the `*_backend` / `*_panel` split.
- Neither is a general UI schema.
- The same package rendering in both a Tauri build and a GPUI build is not implemented locally.
- Timeout, theme tokens passed to a backend, and permissions for a panel backend do not exist here.
- The theme-token mechanism that does exist is described under "Theme bridge" below.


- It is an in-tree fork of the Zed editor. It is not an extension and not an external process.
- It is branded "Mentenaz Forge Super" in `editions.json`. The URL scheme is `forgesuper://`, and the command is `forge-super`.
- Diff against `145bcfc1bc` (excluding Cargo.lock, icons and screenshots): 388 files changed, about 86k lines added and 1.3k removed.
- There are 40 new crates, roughly 245k lines. Of those, about 160k are vendored UI libraries (`gpui_component`, `gpui_base`, `gpui_component_story`, `gpui_flow`). The remaining ~85k lines of original code are the panels and backends below.
- About 25 existing upstream crates are modified. The central ones are `zed` (wiring), `workspace` (dock ordering), `gpui` (a layout `scale`), `git`/`project`/`git_ui`, `editor` (context menu), `extension_host`/`extensions_ui`, `auto_update`, `client` (telemetry), `zed_credentials_provider`, `settings_content`/`settings_ui`, `json_schema_store`/`languages`, and `terminal`/`util`.
- Every feature below is compiled into the binary. Every new panel implements `workspace::dock::Panel` or `workspace::Item` directly. None of it goes through the WASM extension API.
- The fork adds no UI API for extensions. `crates/extension_api` only has the upstream `PENDING_CHANGES.md` in its diff.
- The author's README states the thesis: "Rather than building the experiment as a separate application, I chose to implement it directly inside Zed's Rust/GPUI architecture" (`README.md`).
- Wiring pattern shared by all features:
  - Each crate exposes `init(cx)`, called from `fn main()` in `crates/zed/src/main.rs`.
  - Each dock panel exposes `Panel::load(workspace, cx)`. It is awaited in `initialize_panels` in `crates/zed/src/zed.rs` and added with `add_panel_when_ready` or `workspace.add_panel`.
  - Keymaps are added to `assets/keymaps/default-{linux,macos,windows}.json`.
  - Per-panel settings are added to `assets/settings/default.json`, `crates/settings_content/src/panels.rs` and `crates/settings_ui/src/page_data.rs`.

Branches on origin: `main`, `Linux_Script`, `flow_zoom_detail`, `helm_hardening`, `python-hardening`, `feat/npm-search-from-editor`, `upstream-trial-2026-11-03` and `debugger-ignore-breakpoints-toggle`. All are fully merged into `main` except the debugger branch, which is 1 commit ahead.

### Shared infrastructure used by nearly every feature

- **gpui_component.** A vendored widget library (longbridge/gpui-component 0.5.2) at `crates/gpui_component/src/`. It provides `setting::{SettingGroup, SettingItem, SettingPage, Settings}`, `resizable`, `DataTable`, `chart`, `Input`/`InputState` (an editor), `tree`, `markdown`, `dialog`, `sidebar`, `notification` and others. `CLAUDE.md` makes it a rule that every ported panel be built from these widgets rather than Zed's `ui` crate.
- **Theme bridge.** `gpui_component::init(cx)` and `gpui_component::rebind_default_keys(cx)` are called in `crates/zed/src/zed.rs` and `main.rs`. Zed's `clear_key_bindings` wipes gpui-component's own shortcuts, so they are re-registered on each keymap reload. `cx.theme()` panics if the global is missing.
  - The bridge itself is `sync_gpui_component_theme` in `crates/zed/src/main.rs` (~:221-296). It maps Zed's active theme (`theme::Appearance`, Zed color tokens) onto `gpui_component::Theme` colours via `Theme::change` and `Theme::global_mut`, then rebuilds `theme.tokens = ThemeTokens::from(&theme.colors)`, because some widgets such as `Sidebar` read the token snapshot directly.
  - `crates/gpui_base/src/theme_tokens.rs` defines `SemanticThemeTokens` (colors, radius, spacing, typography, shadow), serialisable and with a JSON schema. These are role-based tokens without component names.
  - `gpui_flow` takes raw `u32` colours and callers convert them from the theme.
- **Script Runner as a shared console.** Node, Python, .NET and the package managers send commands to `script_runner_panel` (`ScriptRunnerPanel::run_external`). The target is a `WeakEntity<ScriptRunnerPanel>` wired in `crates/zed/src/zed.rs` (`script_runner_dependent_panels`), or found via `workspace.open_panel`.
- **`ide_env`** (`crates/ide_env/src/ide_env.rs`). A dependency-free registry of PATH providers, plus per-repository git config and credential state. It is consumed by terminals, the Script Runner and workflow actions.
- **Dock panel ordering and "Move Left/Right"** (see feature 25).

---

## 1. Cockpit panel (system metrics sidebar)

- **What it does.** A left-dock panel with live CPU and RAM circular gauges, a per-core usage bar for each logical core, disk usage per mount, an area chart of CPU/RAM history and an area chart of network throughput. The header shows core count and uptime, plus a **Dashboard** button that opens feature 2. Readings refresh every 2 s. History keeps 30 samples (about 1 minute), built by the panel itself because `sysinfo` only gives instantaneous values.
- **Where.** `crates/cockpit_panel/src/cockpit_panel.rs` (`actions!` at :85, `init` at :101, `impl Panel` at :389, icon at :428, `activation_priority` 20 at :441). Also `system_metrics.rs`, `header.rs`, `cockpit_panel_settings.rs` and `README.md`.
- **UI surfaces.**
  - Dock panel (left or right, set by `cockpit_panel.dock`).
  - Status-bar toggle button (`cockpit_panel.button`).
  - Collapsible sections, gauges and charts from `gpui_component::chart`.
  - Actions `cockpit_panel::ToggleFocus` and `cockpit_panel::OpenDashboard`. The second is declared here and handled in `dashboard_panel` to avoid a dependency cycle.
  - No default keybinding.
- **Zed internals.** `workspace::dock::Panel`, workspace status bar, `SettingsStore`, action registration, and `gpui_component` theming.
- **Resources.**
  - Filesystem: none directly.
  - Network: none.
  - Processes: none spawned. It reads the OS via the `sysinfo` crate (CPU, memory, disks, networks).
  - No clipboard or notifications.
- **State and persistence.** In-memory ring buffers only. Settings keys are `cockpit_panel.{button,dock}`.
- **Coupling.** Medium. It uses only the public `Panel` trait, but it needs the `gpui_component` theme global and is wired by hand in `zed.rs` and `main.rs`.

## 2. Dashboard (cross-ecosystem project health tab)

- **What it does.** A workspace tab summarising the active project: detected Node, Python, .NET and Rust runtime versions with project counts, outdated and vulnerable dependency counts, and security findings per ecosystem (severity, package, fixed-in version, advisory link). It also shows git status (repo, branch, changed files, ahead/behind) and embeds the Cockpit system charts. Its design rule is three states, Healthy, Unknown and Failed, so an unscanned project never looks clean. "Scan all" re-runs each panel's own vulnerability scan.
- **Where.** `crates/dashboard_panel/src/dashboard_panel.rs` (`init` :108, `impl Item` :229, `impl SerializableItem` :241).
- **UI surfaces.** A pane item (tab) that implements `workspace::Item` and `SerializableItem`. Its content is cards and charts built from gpui-component. It is opened by the Cockpit header button or the `cockpit panel: open dashboard` action. If a Dashboard tab is already open in the pane, it is re-activated rather than duplicated.
- **Zed internals.**
  - `workspace.panel::<T>(cx)` to fetch the other dock panels as entities and `cx.observe` them.
  - `workspace::register_serializable_item`.
  - The git store: git status is read from `project::git_store` directly.
- **Resources.** It makes no network or process calls of its own. It triggers the node, python, dotnet and rust panels' scans (see their entries for hosts and binaries).
- **State and persistence.** Tab restoration through the workspace DB (`SerializableItem`).
- **Coupling.** High at the application level: it depends on `cockpit_panel`, `node_panel`, `python_panel`, `dotnet_panel` and `rust_panel` entities, and on the `*_backend` finding types. It relies on the in-process entity graph, which a sandboxed extension could not do.

## 3. Processes panel (live process table)

- **What it does.** A bottom-dock panel with a sortable, filterable table of every OS process: PID, name, CPU %, memory, source (Managed or External) and status. The user can "adopt" processes as Managed, and can kill a process after a confirmation. It refreshes every 2 s.
- **Where.** `crates/processes_panel/src/processes_panel.rs` (`actions!` :504, `init` :523, `impl Panel` :870, `kill_process` :142, priority 24 at :908). Also `processes_panel_settings.rs`.
- **UI surfaces.**
  - Bottom-dock panel.
  - `gpui_component::DataTable` with header-click sort.
  - Filter `Input`, a right-click context menu per row, and an inline confirmation banner.
  - Key context `DataTable`: `a` adopts or releases, `delete`/`backspace` kills.
  - Keymap `ctrl-k x`.
  - Actions `ToggleFocus`, `KillSelected` and `ToggleAdoptSelected`.
- **Zed internals.** `Panel`, status bar button, key contexts, and settings (`processes_panel.button`).
- **Resources.** It reads all processes through `sysinfo`. It can kill any PID the user can signal. No network.
- **State and persistence.** The managed set is kept in memory on the panel only.
- **Coupling.** Low to medium. It is nearly stand-alone, apart from `Panel`.

## 4. Script Runner panel (shared shell console)

- **What it does.** A bottom-dock panel that runs a shell command string and streams stdout and stderr line by line. It can stop a running command. It echoes the command and shows the real exit status. It detects SPFx `gulp serve` or `heft start` workbench debug URLs and shows the query fragment in a bar for copying. It is also the output console for the Node, Python, .NET and package-manager features. `command.rs` has a validator that rejects shell-unsafe package names and versions (only letters, digits and `@ / . _ + -`).
- **Where.** `crates/script_runner_panel/src/script_runner_panel.rs` (`actions!` :231, `init` :241, `run_shell_blocking` around :448-506, `impl Panel` :692, priority 23 at :737). Also `command.rs` and `script_runner_panel_settings.rs`.
- **UI surfaces.** A dock panel (bottom by default; left, right or bottom allowed). It uses a command `Input`, a streaming output list, link detection with copy and navigate. The keymap is `ctrl-k k`.
- **Zed internals.**
  - `Panel`.
  - `ide_env` (PATH prefixing for the built-in Node and Python).
  - The default working directory logic follows the terminal's `default_working_directory`.
- **Resources.**
  - Processes: the user's string is run through a shell (`sh -c` on unix, `cmd /C` on Windows). On unix the shell is the leader of its own process group. Stopping runs `Command::new("kill")` against the group (:495 and :502).
  - Filesystem: whatever the command touches, rooted at the chosen cwd.
  - Clipboard: copy of the debug URL.
  - No direct network.
- **State and persistence.** Output is in memory. Settings are `script_runner_panel.{button,dock}`.
- **Coupling.** Medium. It reimplements a terminal-like runner instead of using Zed's `terminal` crate. Other panels call it by entity handle.

## 5. Node panel (Node.js versions, projects, scripts, packages, SPFx)

- **What it does.** A left-dock panel with collapsible sections:
  - Node Versions: built-in install, uninstall and select, per workspace, with Auto, System or Pinned modes.
  - Projects: every `package.json` under the workspace.
  - Quick Actions: install with the project's package manager, Task Chain, Package Manager.
  - Packages, Outdated (with update-all patch or minor), and Vulnerabilities (`npm audit`).
  - Scripts: run, add, edit and delete `package.json` scripts.
  - Processes: running Node processes, what each is, and Stop.
  - SPFx: SharePoint Framework actions, and `.sppkg` packages re-read when the file watcher fires.
  - Auto mode picks the version from `.nvmrc`, `.node-version`, `engines.node` or SPFx `.yo-rc.json`. It can create `.nvmrc`.
- **Where.**
  - Panel: `crates/node_panel/src/node_panel.rs` (`actions!` :177, `init` :194, `impl Panel` :2284, priority 26 at :2347).
  - Backends: `crates/node_backend` (project scan, `package.json` editing), `crates/node_versions` (built-in version manager on the `fnm` library), `crates/npm_backend` (npm CLI wrapper), `crates/runtime_processes` (process list and secret redaction).
- **UI surfaces.** A left-dock panel with collapsible sections, buttons, a project selector and the `ProcessList` view from `runtime_processes`. It has a status-bar button, keymap `ctrl-k o`, and a "Task Chain" jump to the Flows panel wizard.
- **Zed internals.**
  - `Panel` and project worktrees.
  - A file watcher on manifests and lockfiles. Panels rescan "when a manifest or lockfile changes".
  - `ide_env::register` for the Node `bin` dir.
  - The KVP store (`node_panel_open_sections`, `node_panel_choice:*`, `node_panel.rs:89-93`).
- **Resources.**
  - Filesystem:
    - Reads `package.json`, `.nvmrc`, `.node-version`, `.yo-rc.json` and `sharepoint/solution/*.sppkg`.
    - Writes `package.json` scripts and `.nvmrc`.
    - Built-in versions live at `paths::data_dir()/node_versions` (`crates/node_versions/src/node_versions.rs:56`).
  - Network: Node distribution downloads via the `fnm` library (fork `github.com/mentenaz/fnm`, pinned rev in root `Cargo.toml:628`), plus npm registry calls (see feature 11).
  - Processes: `npm`/`yarn`/`pnpm` CLI (`npm_backend.rs:144`), `explorer` and an opener (`node_backend.rs:78`, `:97`), and arbitrary scripts via the Script Runner. Process stop uses sysinfo.
- **State and persistence.** The KVP store (section open or closed, per-workspace version choice). Settings are `node_panel.{button,dock}`.
- **Coupling.** Medium-high. It is wired by hand in `zed.rs` to the Script Runner and Flows panels, and it affects every terminal's PATH through `ide_env`.

## 6. Python panel (interpreters, venvs, packages, frameworks)

- **What it does.** A left-dock panel with:
  - Built-in Python versions: self-contained CPython builds from python-build-standalone, installed only after a confirmation that names the version, size and source, and verified against a published checksum.
  - Project discovery, environment details (interpreter, pip, venv), and framework detection (Django, Flask, Gradio, pytest and others, with a suggested run command and port).
  - Quick actions.
  - Packages, Outdated (with update-all patch or minor) and Vulnerabilities (PyPI or OSV).
  - Processes (flask, uvicorn, manage.py and so on).
- **Where.**
  - Panel: `crates/python_panel/src/python_panel.rs` (`actions!` :158, `init` :168, `impl Panel` :1156, priority 25 at :1213).
  - Backends: `crates/python_backend/src/python_backend.rs`, `crates/python_versions/src/python_versions.rs`.
  - Shares `runtime_processes` and `ide_env` with the Node panel.
- **UI surfaces.** A dock panel with collapsible sections and the status-bar button. The keymap is `ctrl-k i`.
- **Zed internals.** `Panel`, `http_client` (downloads go through Zed's `HttpClient`), `ide_env`, project worktrees, and the settings store.
- **Resources.**
  - Network:
    - `pypi.org/pypi/<pkg>/json`.
    - `osv.dev`.
    - `raw.githubusercontent.com/astral-sh/python-build-standalone/latest-release/latest-release.json` (`python_versions.rs:39`) and the release assets it points to.
  - Processes: `python -m pip ...` and `python -m venv`. Commands are run via the Script Runner.
  - Filesystem: writes a `python_versions` folder in the data directory, and creates and reads venvs and requirements files.
- **State and persistence.** The active built-in version is stored per workspace. Settings are `python_panel.{button,dock}`.
- **Coupling.** Medium-high. It is the same pattern as the Node panel, plus PATH injection into every terminal.

## 7. .NET panel (SDKs, projects, NuGet status)

- **What it does.**
  - A left-dock panel showing the .NET SDK or runtime, `.csproj` projects, and quick actions (run, restore and so on).
  - Package references, Outdated (with update-all patch or minor), and Vulnerabilities (`dotnet list package --vulnerable`).
  - Running `dotnet` processes.
- **Where.** `crates/dotnet_panel/src/dotnet_panel.rs` (`actions!` :109, `init` :119, `impl Panel` :934, priority 27 at :986). Backend: `crates/dotnet_backend/src/dotnet_backend.rs`.
- **UI surfaces.** A dock panel with collapsible sections. The status-bar button tooltip is "Dotnet". It has no default keybinding.
- **Zed internals.** `Panel`, project worktrees, and workspace lookup of the Script Runner.
- **Resources.**
  - Processes: `dotnet` (`gpui_util::new_std_command("dotnet")` at `dotnet_backend.rs:463`, plus a `cmd /C dotnet` fallback at :478).
  - Network: nuget.org (see feature 11).
  - Filesystem: reads `.csproj` and `.sln`.
- **State.** Settings only: `dotnet_panel.{button,dock}`.
- **Coupling.** Medium.

## 8. Rust panel (toolchain, crates, outdated, advisories)

- **What it does.**
  - A left-dock panel with:
    - the `rustc` and `cargo` versions;
    - workspace crates (with a filter box when there are more than 8);
    - quick actions `check`, `build`, `run` and `test`, plus **Package Manager** (feature 11);
    - dependencies with their exact locked versions;
    - Outdated, tagged patch/minor/major, "needs Rust X" and yanked;
    - Advisories grouped as vulnerabilities, unsound, unmaintained and notices, via OSV.dev on demand.
  - It deliberately does not use rust-analyzer. It reads `Cargo.toml`, `Cargo.lock` and `cargo metadata` directly.
- **Where.**
  - `crates/rust_panel/src/rust_panel.rs` (`actions!` :76, `init` :86, `impl Panel` :1381, priority 29 at :1433).
  - `crates/cargo_backend/src/` (`cargo_backend.rs`, `lockfile.rs`, `search.rs`, `advisories.rs`, `actions.rs`).
  - Design note: `docs/Rust_Manager_Design_Note.md`.
- **UI surfaces.** A dock panel with sections. The keymap is `ctrl-k c`.
- **Zed internals.** `Panel`, file watching of manifests, the Script Runner, and settings.
- **Resources.**
  - Processes: `cargo` and `rustc` (`cargo_backend.rs:101`, `:172`, `actions.rs:191`).
  - Network: `crates.io/api/v1/crates*` and `api.osv.dev/v1/querybatch` and `/v1/vulns/{id}` (`advisories.rs:28`, `:38`; `search.rs:19`).
  - Filesystem: reads `Cargo.toml` and `Cargo.lock`. Writes happen via cargo.
- **State.** Settings only: `rust_panel.{button,dock}`.
- **Coupling.** Medium. It is independent of the language-server stack.

## 9. Database workbench (panel, SQL workbench tab, schema graph tab)

- **What it does.** A bottom-dock database client for SQLite, PostgreSQL, MySQL/MariaDB, SQL Server and Oracle (Oracle added in commit `12dea53620`). The README lists only four engines (SQLite, PostgreSQL, MySQL, MSSQL), so the Oracle driver is newer than the README.
  - **Connections and explorer.** The user manages multiple connections and picks databases on a server (create or attach). The panel has a searchable schema tree (tables, columns, indexes), Overview, Tables, Views and Relationships pages, and an Inspector.
  - **Workbench tab.** A SQL editor with Run, schema-driven autocomplete, hand-written SQL syntax highlighting, a results table, and Copy or Export of results as JSON or SQL.
  - **Schema Graph tab.** Tables are nodes and foreign keys are edges, with automatic layout and pan and zoom, built on `gpui_flow` (feature 19).
- **Where.**
  - Panel: `crates/database_panel/src/database_panel.rs` (`actions!` :317, `init` :327, `impl Panel` :2379, priority 22 at :2424).
  - Panel modules: `workbench.rs` (`impl Item for WorkbenchTab` :1272), `graph.rs` (`impl Item for SchemaGraphTab` :254), `explorer.rs`, `content.rs`, `inspector.rs`, `header.rs`, `sql_highlight.rs`, `ir.rs`, `layout.rs`, `persistence.rs`.
  - Backend: `crates/database_backend/src/` (`connection.rs`, `drivers/*`).
  - Design docs: `DB_SCHEMA_GRAPH_ARCHITECTURE.md`, `PHASE1_PANEL_PLAN.md`.
- **UI surfaces.** A bottom-dock panel with three resizable panes. It also opens two pane-item types, `WorkbenchTab` and `SchemaGraphTab`, which are serialized and restored. A connection form has a file picker for SQLite. The editor is gpui-component `Input` with a custom `CompletionProvider` (`SqlSchemaCompletionProvider`). Keymap: `ctrl-k d`.
- **Zed internals.**
  - `Panel`, `Item` and `SerializableItem`.
  - The `db` KVP store for saved connections.
  - `zed_credentials_provider` for passwords.
  - A native file dialog.
  - Clipboard.
- **Resources.**
  - Network: TCP to arbitrary user-configured DB hosts. Drivers are `tokio-postgres` with rustls, `mysql_async`, `tiberius` and `oracledb`. TLS roots come from `rustls-native-certs`.
  - Filesystem: opens user-chosen SQLite files read and write via `rusqlite`. Exports are written where the user chooses.
  - Credentials: `database://<type>/<host>:<port>` in the system keychain. Passwords are never stored on `ConnectionConfig`.
  - Clipboard: Copy JSON and Copy SQL.
  - No spawned processes.
- **State and persistence.**
  - The KVP keys `database-panel-connections` and `database-panel-schema-cache` (`database_panel.rs:67-68`).
  - Workbench and graph tab restore state (`persistence.rs`).
  - Workbench history (`workbench.rs:1134`).
  - Settings: `database_panel.button`.
- **Coupling.** High on the UI side (the editor `CompletionProvider`, `Item`, KVP). The backend is Zed-independent apart from credentials.

## 10. Flows (workflow engine, Flows panel and Designer tab)

This is the largest original feature: three crates and about 21k lines.

- **What it does.** Executable visual workflows stored as `*.flow.json`.
  - **Engine.** The file format has typed actions with `runAfter` dependencies and these action types: `http`, `transform`, `InitVariable`, `SetVariable`, `script`, `StartProcess`, `WaitForPort`, `notify`, `delay`, `log`, `SetEnv`, and the containers `Foreach`, `Until`, `If` and `Try`. Expressions use JSONLogic. A level scheduler runs independent actions in parallel. Validation runs before a run. A process tracker handles child processes.
  - **Flows panel.** A left-dock index of the workflows in `<workspace>/.zed/workflows` (configurable with `flows.workflows_dir`). It has Refresh, New Workflow and **Add Task Chain**, a wizard that scans the workspace for services and generates a flow that starts them. Each row has Run, Open and Graph, and shows the last result.
  - **Designer tab.** A node-graph editor. It has an action catalog, a canvas, a generated-form inspector, drag from catalog to canvas, hover "+" insertion on edges, and Run, Save, Fit View, Raw (opens the JSON as a text editor) and Results. It has per-node status colours during a run, and a Run Results tab. It claims every `.flow.json` file via `workspace::register_project_item`.
  - **Script and HTTP companions.** Script and HTTP body companion files can be edited inline with JSON schema validation.
- **Where.**
  - Engine: `crates/workflow_engine/src/` (`engine.rs`, `scheduler.rs`, `executors.rs`, `registry.rs`, `schema.rs`, `validation.rs`, `process_tracker.rs`, `task_chain.rs`, `project_scan.rs`, `history.rs`, `layout.rs`).
  - Flows panel: `crates/flows_panel/src/flows_panel.rs` (`actions!` :163, `init` :173, `impl Panel` :771, priority 28 at :823), `persistence.rs`, `flows_settings.rs`.
  - Designer: `crates/designer_panel/src/designer_panel.rs` (`init` :224, `impl Item` :4993, about 5k lines), `run_results.rs` (`impl Item` :341), `project_item.rs` (`FlowFile`), `workflow_json.rs`, `script_companion.rs`, `script_helpers.rs`, `http_companion.rs`, `run_state.rs`, `catalog.rs`, `persistence.rs`.
  - Settings: `crates/settings_content/src/flows.rs`.
  - Schema: `crates/json_schema_store/src/schemas/flow.schema.json`, and `json_schema_store.rs` / `languages/src/json.rs` for `*.flow.json` and `**/*.flow-http/**/*.body.json` associations.
  - Helm also opens a `WorkflowRunItem` tab at `crates/helm_panel/src/workflow_run_tab.rs:397`.
- **UI surfaces.**
  - A dock panel (Flows) and pane items (Designer, Run Results). The Designer uses a `gpui_flow` canvas.
  - Catalog, canvas and inspector are three panes using gpui-component `resizable`. There is a context menu, toolbar buttons, and dirty marker and save handling.
  - Banners for reload-from-disk and parse errors.
  - Actions in the `designer_panel` namespace (`ViewRaw`, `ViewScriptCompanion`, `ViewHttpBody`, `ViewRunResults`, `ShowInvalidFlowToast`). They carry a path and are `no_json`.
  - A `Raw` button opens a plain `Editor` on the buffer, which makes this a real editor integration.
  - Keymap `ctrl-k f`.
- **Zed internals.**
  - `workspace::Item`, `SerializableItem`, `ProjectItem` registration, project buffers and worktrees.
  - The JSON language server and its schema store, plus the editor for Raw and companion views.
  - The KVP store, `ide_env` PATH, and notifications and toasts.
- **Resources.**
  - Filesystem:
    - Reads and writes `<name>.flow.json`, `<name>.flow.layout.json`, `<name>.flow-http/<action>.body.json` (and `.body.schema.json`), and `<name>.flow-scripts/`.
    - Writes a per-launch temp JSON file, pointed to by `FORGE_CONTEXT`, for `script` actions. It is deleted on exit.
  - Network:
    - `http` action: any URL, with six allowed methods (`reqwest`, `executors.rs:249`).
    - `notify` action: a POST to a user URL (webhook), 15 s timeout (`executors.rs:623`).
    - `WaitForPort`: TCP connect to `127.0.0.1:<port>` (`executors.rs:722`).
  - Processes:
    - `script` (a user program, JSON on stdin).
    - `StartProcess` (`Command::new(program)` at `executors.rs:540`, `cmd.exe` at :811).
    - Kill via `taskkill.exe` or `kill` (`process_tracker.rs:240`, `:255`).
  - Env: `SetEnv` changes the environment of subsequent actions.
- **State and persistence.** Flow files in the repository, sidecar layout files, and the KVP store for run history (`flows_panel.rs:329`, `:664`; `designer_panel.rs:2579`). Settings are `flows_panel.{button,dock}` and `flows.workflows_dir`. Three action types (`sendEmail`, `DbMigration`, `AiCheck`) are parked in `registry.rs`.
- **Coupling.** High for the Designer (project items, editor, JSON schema, KVP). `workflow_engine` itself is headless (tokio and reqwest, no gpui).

## 11. Package managers (npm, NuGet, PyPI, Cargo)

Four full-width workspace tabs share one design. Each has a left sidebar with General, Search, Installed, Updates and Vulnerabilities pages, a resizable details pane with versions, dependencies and a rendered README, keyboard navigation (up, down, enter, space), and Install, Update, Remove or Fix buttons. Commands run in the Script Runner.

| Tab | Code | Opened by | Registry hosts | CLI used |
|---|---|---|---|---|
| npm | `crates/npm_manager_panel/src/npm_manager_panel.rs` (`actions!` :63, `init` :109, `impl Item` :927), `search.rs`; backend `npm_backend` | `ctrl-k j`, Node panel button, editor right-click, `npm_manager::OpenNpmManager` | `registry.npmjs.org/-/v1/search`, `registry.npmjs.org/{pkg}`, `api.npmjs.org/downloads/point/last-week/{pkg}` | `npm`, `yarn` or `pnpm`; `npm audit` |
| NuGet | `crates/nuget_manager_panel/src/` (`nuget_manager_panel.rs` :62, :109, :732; `pages.rs`, `details.rs`, `search.rs`); backend `dotnet_backend` | `ctrl-k g`, .NET panel, `nuget_manager::OpenNuGetManager` | `api.nuget.org/v3/index.json`, `/v3/registration5-semver1/...`, `/v3-flatcontainer/{id}/{ver}/readme`, `azuresearch-usnc.nuget.org/query` | `dotnet add/remove package`, `dotnet list package` |
| PyPI | `crates/python_manager_panel/src/python_manager_panel.rs` (`actions!` :66, `init` :116, `impl Item` :922); backend `python_backend` | `ctrl-k y`, Python panel, `python_manager::OpenPythonManager` | `pypi.org/pypi/{name}/json` (exact-name lookup only), `osv.dev` | `python -m pip` |
| Cargo | `crates/cargo_manager_panel/src/cargo_manager_panel.rs` (`actions!` :74, `init` :129, `impl Item` :998, `SerializableItem` :1013); backend `cargo_backend` | Rust panel, `cargo_manager::OpenCargoManager` | `crates.io/api/v1/crates*` (including `/readme`), `api.osv.dev` | `cargo add/remove/update` |

- **Cargo specifics.**
  - It shows Cargo's own dry-run before an update, says when the root manifest will change, and refuses to re-version `[workspace.dependencies]`.
  - Each version carries its minimum supported Rust version.
- **UI surfaces.** Pane items (`workspace::Item`), a settings-style sidebar from `gpui_component::setting`, a resizable split, a Markdown renderer for READMEs, and confirmation dialogs.
  - README image loading goes through GPUI assets. `crates/gpui/src/asset_cache.rs` demotes load errors to debug logs for this reason.
  - Tests in the panels use `http://evil.test`, which indicates link-scheme filtering.
- **Zed internals.** `Item` and `SerializableItem`, workspace actions in `zed_actions::npm_manager` (`crates/zed_actions/src/lib.rs`), and `http_client`.
- **Editor integration.** A context-menu hook in `crates/editor/src/mouse_context_menu.rs` shows "Open in Package Manager" and `Search npm for "<pkg>"` for TypeScript, TSX and JavaScript buffers. It parses the selection or the quoted string under the cursor through `npm_backend::package_from_specifier` and `package_at`.
- **Resources.** Network to public registries (the table above). Filesystem writes happen through the package manager CLIs (`package.json`, `Cargo.toml`, `.csproj`). READMEs and images are fetched over https.
- **State.** Mostly in-memory per tab. Cargo tabs are serialized.
- **Coupling.** Medium to high. The editor context menu patch is a direct edit to the `editor` crate.

## 12. npm bootstrap (offer `npm install`)

- **What it does.** When a workspace opens whose first worktree has `package.json` but no `node_modules`, a toast offers "Run npm install". The offer is made once per project directory, and the marker is written as soon as the toast is shown. The result is reported in a second toast.
- **Where.** `crates/npm_bootstrap/src/npm_bootstrap.rs` (`init` :17, `Command::new(npm_bin)` :125).
- **UI surfaces.** A workspace notification with an action button.
- **Zed internals.** Hooks the generic `Workspace` lifecycle (open or new workspace) and the notifications API. The README says the marker persists across sessions.
- **Resources.** Spawns `npm install` in the root. This uses the network through npm and writes `node_modules` and the lockfile.
- **State.** A per-directory marker.
- **Coupling.** Low to medium.

## 13. Helm: native GitHub client (panel, accounts, repo tabs)

- **What it does.** A left-dock GitHub client, about 10.6k lines in the panel crate plus 6.7k in the backend. It is a Zed-native replacement for browser GitHub chores:
  - **Sign-in.** GitHub OAuth device flow with Helm's own OAuth App, personal access tokens, or a one-time token import from the `gh` CLI. Multi-account is supported, with an account switcher, a per-workspace account, and per-account colours. Sign-out can also open the GitHub revoke page.
  - **Browsing.** Profile (view and edit), organizations and their repos, and repository search (global search opens a repository in the detail view).
  - **Per-repo views.** Branches, collaborators (add and remove), issues (with comments), pull requests (list, detail, create), releases (list, create), packages, commits, Actions runs, deployments, tags, traffic, and Dependabot or secret-scanning alerts.
  - **Editing.** Repository settings, topics and visibility, and Create Repository (with owner and attach-to-workspace mode).
  - **Actions run tab.** An Actions run opens in its own tab (`WorkflowRunItem`) with the jobs drawn as a `gpui_flow` graph.
  - **Clone flow.** A folder picker, then clone with git and the account token, and a workspace switch.
  - **Account and identity features.** These are covered in features 15 and 16.
- **Where.**
  - Panel: `crates/helm_panel/src/helm_panel.rs` (`actions!` :106, `init` :127, `impl Panel` :702, priority 21 at :756). Modules are `auth.rs`, `accounts.rs`, `account_switcher.rs`, `gh_cli_import.rs`, `clone.rs`, `attach.rs`, `commit_identity.rs`, `repo_access.rs`, `repos.rs`, `orgs.rs`, `profile.rs`, `issues.rs`, `pulls.rs`, `branches.rs`, `collaborators.rs`, `releases_packages.rs`, `activity.rs`, `insights.rs`, `invitations.rs`, `repository_modal.rs`, `workflow_run_tab.rs`.
  - Backend: `crates/helm_backend/src/` (`github/oauth.rs`, `github/clone.rs`, `github/gh_cli.rs`, `github/paging.rs`, `github/remote.rs`, `github/types.rs`).
  - Shared components: `crates/helm_ui` (`widgets.rs`, `rows.rs`, `list_view.rs`, `section.rs`).
  - Docs: `crates/helm_panel/README.md` ("Security at a glance" near :325), `Helm_*.md`, `REGRESSION_MATRIX.md`, `crates/helm_backend/octocrab-helm-coverage.md`.
  - Settings: `helm_panel_settings.rs` and `crates/settings_content/src/panels.rs`.
- **UI surfaces.**
  - A dock panel with screen-state navigation (`HelmScreen`), modals (repository edit, create, clone), and an account menu in the header.
  - Toasts, including "Switch to ..." suggestions. Avatars and an API-rate-limit bar.
  - Tabs: Actions run and the Workspace (feature 14).
  - Keymap `ctrl-k h`.
  - Heavy use of gpui-component widgets.
- **Zed internals.**
  - `Panel`, workspace notifications, `zed_credentials_provider`, `http_client` / `reqwest_client`, and `paths`.
  - `Workspace` open and replace (clone then switch).
  - The settings store (`helm_panel.accounts`, `default_account`).
  - Per-workspace account state is exposed to `git_ui` and `ide_env`.
- **Resources.**
  - Network:
    - `api.github.com` REST.
    - `github.com/login/device/code` and `github.com/login/oauth/access_token` (`oauth.rs:33-34`).
    - `avatars.githubusercontent.com`.
    - git over HTTPS to `github.com`.
    - OAuth client id `HELM_OAUTH_CLIENT_ID` in `auth.rs`. Scopes asked are `repo`, `read:org` and `user`.
  - Processes:
    - `git` for clone (`helm_backend/src/github/clone.rs:177`) and the repo-config identity writes (`helm_panel/src/attach.rs:495`, `commit_identity.rs:60`).
    - `gh` for the optional import (`gh_cli.rs:61`, including `gh auth login --web`).
  - Credentials: OS keychain under `helm://github.com/<login>`. A dev-build plaintext file is opt-in through `ZED_DEVELOPMENT_USE_FILE_CREDENTIALS`.
  - Filesystem:
    - Clone target directories chosen by the user.
    - Per-repo `.git/config` user.name and user.email writes. The global git config is never changed.
  - Clipboard: copy of the device code and clone URLs.
  - Browser: opens GitHub pages (revoke, settings) through the OS.
- **State and persistence.** `settings.json` stores only account logins, labels and the default account. Tokens are in the keychain. Per-window state is in memory.
- **Coupling.** Very high at the application level: it holds app-global state about which account a workspace uses, which `git_ui`, `ide_env` and `git_credential` read. This is the feature most unlike what a sandboxed extension could do.

## 14. Helm Workspace (read-only GitHub repository tab)

- **What it does.** Opens a remote GitHub repository as a full-width, read-only tab, without cloning. Pages are:
  - Overview.
  - Code (keyboard-navigable file tree with a local filter, README rendering for the repo and selected folder, syntax-highlighted preview in a read-only editor, image previews, and permalinks at the exact commit).
  - Commits (paged, per-file history, and commit diffs).
  - Compare (two refs).
  - Search (ref-scoped GitHub code search).
  - Pull requests and Issues (with comment threads).
  - Actions.
  - Files can be opened from a matching local checkout, with a commit-mismatch confirmation, or the repo can be cloned first.
- **Where.** `crates/helm_workspace/src/tab.rs` (`gpui::actions!` :49, `impl Item for WorkspaceTab` :3730, 4.3k lines in total with `ref_list.rs` and `lib.rs`). Opened by the `OpenRepositoryWorkspace` action from `helm_panel`.
- **UI surfaces.** A pane item. It uses gpui-component `tree`, `resizable` and Markdown, a read-only `Editor` buffer for file text and unified patches, and shared `helm_ui` rows.
- **Zed internals.** `Item`, `Editor` and `MultiBuffer` (read-only), the language registry (syntax highlighting), `Workspace` open-path, and a Helm account.
- **Resources.** GitHub REST (contents, commits, compare, search/code, pulls, issues and runs) with the signed-in account's token. The data is transient and held in memory. The browser opens permalinks.
- **State.** In memory. The last-used branch or tag is remembered per repository (commit `272dea983e`).
- **Coupling.** High.

## 15. Git panel extensions (Details tab, GitHub account, retry, push gating)

Three sub-features in `crates/git_ui` and `crates/git`.

- **15a. Git Details tab.**
  - A third tab in the git panel next to Changes and History, showing repo-wide insights: commit activity, most-changed files, and collaborators, computed over a window of recent commits (`DETAILS_COMMIT_WINDOW`).
  - Action `git_panel::ActivateDetailsTab`, keymap `ctrl-3`.
  - Code: `crates/git_ui/src/git_panel.rs` (`GitPanelTab::Details` :657, action :178, `load_details_data` ~:8300, render at :10434).
  - Plumbing:
    - New `GitRepository::recent_commit_stats` and `file_change_frequency` in `crates/git/src/repository.rs` (about 145 lines). They shell out to `git log --max-count=N --format=%x1e%H%x1f%an%x1f%ae%x1f%at` and `git log --full-diff --name-only -z`.
    - Mirrored in `crates/fs/src/fake_git_repo.rs`.
    - Repository jobs `Repository::recent_commit_stats` and `file_change_frequency` in `crates/project/src/git_store.rs`. They are local only and bail for remote repositories.
- **15b. GitHub account in the git panel** (`crates/git_ui/src/git_panel/github_account.rs`, 833 lines).
  - The footer shows the Helm account used for the workspace's github.com repo, with a switch menu. It shows the commit email, with a one-time choice and a question when the repo already names someone else.
  - Over HTTPS it warns when the account cannot push, with a button to switch to one that can.
  - Push, pull and fetch are gated: the button is disabled and the action refused with a toast (`RefusalToast`) naming the problem. Toasts go away when no longer applicable.
  - A private-email rejection is detected, and a temporary push failure offers Retry.
  - The empty state offers "Using zed / Using Helm" (commit `d9d5629805`), and Helm attach-mode create wires the repo to the workspace.
- **15c. Authenticated git for the panel.** Helm hands the token to git through `GIT_CONFIG_*` environment variables (an `Authorization` header) via `ide_env::with_git_remote_config`. `GitBinary` in `crates/git/src/repository.rs` merges these with the gpg wrapper config instead of skipping it.
- **UI surfaces.** The git panel's tab strip, footer and toasts.
- **Zed internals.** `git`, `project::git_store`, `git_ui` and the Helm panel entity (`HelmPanel`, `CommitIdentity`).
- **Resources.**
  - Processes: `git log` and the existing git push, pull and fetch.
  - Network: through git over HTTPS to github.com.
  - Filesystem: per-repo `.git/config` identity.
- **Coupling.** Very high. These are edits inside upstream's largest panel (`git_panel.rs` +430 lines), with a compile-time dependency from `git_ui` onto `helm_panel`.

## 16. Git credential helper (git in terminals logs in as the workspace's account)

- **What it does.** Git run in Zed's integrated terminals asks Zed's own credential helper for github.com. The helper answers with the token for the workspace's Helm account, looked up at the time git asks, so an open terminal follows an account switch. It answers nothing outside Helm-known repos, so `gh` or Git Credential Manager answer as before. If the account is signed out it replies `quit=1`. Only `get` is answered. The token is never in the terminal environment, and never logged.
- **Where.**
  - `crates/git_credential/src/git_credential.rs`: `HELPER_ARG` = `--git-credential`, `helper_command` :87, `run_helper` :95, `start_server` :131.
  - `crates/zed/src/main.rs:315-316` makes the binary act as the helper. Server start is at `main.rs:503`.
  - `crates/ide_env/src/ide_env.rs`: `set_git_credential` :227, `git_credential_for` :240, `git_credential_terminal_env` :274, `apply_git_credential_env` :305.
  - `crates/project/src/terminals.rs`: `add_ide_runtimes`, which only applies to local terminals.
- **UI surfaces.** None, apart from account toasts.
- **Zed internals.** `project::terminals` environment construction, the app binary's entry point (argv handling), and `paths`.
- **Resources.**
  - Processes: git invokes `<zed> --git-credential get`.
  - IPC: `start_server` binds a `net::UnixListener` on a `credential.sock` inside a per-process temporary folder. I did not check whether a Windows variant exists.
  - No direct network.
- **State.** An in-memory map from repository path to credential, with an owner id so a window's settings leave with it.
- **Coupling.** Very high. It edits the binary's `main()` and terminal spawning.

## 17. Built-in runtime management and PATH injection (`ide_env`)

- **What it does.** Terminals, the Script Runner and workflow actions all start with the built-in Node and Python version of the workspace first on `PATH`, without touching the machine's PATH. A terminal already open follows a version switch. Providers register at startup (`ide_env::register`). Consumers call `bin_dirs` or `path_env`.
- **Where.** `crates/ide_env/src/ide_env.rs` (`register` :44, `bin_dirs` :53, `path_env` :66, `apply_to` :91). Providers are in `node_versions` and `python_versions`. The hook is in `crates/project/src/terminals.rs`.
- **UI surfaces.** None. It is visible as version headers in the Node and Python panels.
- **Zed internals.** The project terminal env builder, and `paths::data_dir`.
- **Resources.** Reads version stores on disk, so it is called off the main thread. Environment of spawned shells. No network.
- **Coupling.** Medium. It is an invisible global hook.

## 18. gpui layout scale (`Style::scale` / `.scale()`)

- **What it does.** A CSS-zoom-like layout scale on any GPUI element. It scales sizes, insets, margins, padding, borders, gaps, text, corner radii, shadows and images, accumulated down the tree. Hitboxes and text stay correct. It is what makes `gpui_flow` zoom work and be crisp. A default of `1.0` is a no-op.
- **Where.** `crates/gpui/src/style.rs` (`Style::scale`), `styled.rs` (`fn scale`), `elements/div.rs`, `elements/text.rs`, `taffy.rs` (`element_scale` threaded through `to_taffy` and `NodeContext`), and `window.rs` (`with_element_scale`). Also `Window::render_size` was made public.
- **UI surfaces.** All GPUI element rendering.
- **Coupling.** Highest: this is a patch to the GPUI core. It changes the signature of `ToTaffy::to_taffy` and `Style::overflow_mask`, so it will conflict with upstream edits.

## 19. gpui_flow (node-graph canvas)

- **Zoom, pan and fit view.** These are part of `gpui_flow`, not of GPUI itself. Zoom works by laying the graph out once in flow units inside a world container scaled with `.scale(zoom)` (feature 18). Pan is an offset, fit view is `FlowState::request_fit(padding)`, and the viewport size is measured on layout (`FlowState::canvas_size()`). So the thread's "GPUI zoom/pan/fit-view" is this canvas plus the core `scale` patch, and the patch is what the zoom relies on.
- **What it does.** A React-Flow-style editor canvas: nodes, edges, handles, a minimap, zoom and pan controls, fit view, and zoom-to-cursor. It has level-of-detail rendering (Full from zoom 0.6, Compact down to 0.35, Minimal below). It is vendored from `pacifio/gpui-flow` (MIT). It backs the Designer, the DB schema graph and the Helm Actions job graph.
- **Where.** `crates/gpui_flow/src/` (`graph.rs`, `store.rs`, `controls.rs`, `minimap.rs`, `types.rs`, `edges/`). Tests are in `tests/zoom_geometry.rs`.
- **UI surfaces.** A custom GPUI element tree with a single scaled "world" container (uses feature 18), plus hit-testing and drag. Colours are raw `u32` (it is theme-agnostic).
- **Zed internals.** It depends only on `gpui`.
- **Resources.** None.
- **Coupling.** Low on Zed internals, but it requires the patched gpui for `.scale()`.

## 20. gpui_component, gpui_base, gpui_component_assets/fps/macros/story (vendored UI toolkit)

- **What it does.** An 84k-line widget library (vendored from longbridge/gpui-component) and its behaviour layer `gpui_base` (46k lines). Together they cover inputs and editors with LSP-like hooks, tables, charts, resizable panes, trees, settings pages, Markdown, menus, dialogs, notifications and sidebars. `gpui_base`'s LSP code was adapted to Zed's forked `lsp_types` (see `crates/gpui_component/PORTING.md`).
  - `gpui_component_story` is a gallery.
  - `gpui_component_fps` is an FPS, CPU, GPU and memory HUD. The editor itself does not render it.
- **Where.** `crates/gpui_component/src/`, `crates/gpui_base/`, `crates/gpui_component_assets`, `crates/gpui_component_fps`, `crates/gpui_component_macros`, `crates/gpui_component_story`.
- **UI surfaces.** All of the new panels. Its theme comes from a global initialised in `gpui_component::init`. Its keymaps conflict with `clear_key_bindings`, so they are re-registered.
- **Resources.** None of its own.
- **Coupling.** This is a parallel UI stack to Zed's `ui` crate: two theme systems and two component sets inside one binary.

## 21. SQL language support (sqls extension)

- **What it does.** A real Zed WASM extension, `extensions/sqls`, that launches the `sqls` language server for `*.sql`. It resolves the binary from `lsp.sqls.binary.path` or `worktree.which("sqls")`. It passes `workspace_configuration` through from the `lsp.sqls.settings` setting. The plan to share connections with the Database workbench is open (`crates/extension/SQLS_EXTENSION_TODO.md`).
- **Where.** `extensions/sqls/src/sqls.rs`, `extension.toml`, `languages/sql/config.toml`.
- **UI surfaces.** The standard LSP features in the editor: completion, hover and diagnostics.
- **Zed internals.** The extension API (`zed_extension_api`, `LspSettings`, `worktree.which`).
- **Resources.** Spawns the `sqls` binary and talks LSP over stdio. It downloads nothing.
- **Coupling.** Low. This is the only feature that goes through the public extension API.

## 22. Extension system changes (Forge registry, install errors, README tab, Razor suggestion)

- **Own registry.** `build_forge_extensions_url` in `crates/http_client/src/http_client.rs` (+18 lines, base `https://forge.mentenaz-server.com/api/registry`) reroutes the list, search and download endpoints in `crates/extension_host/src/extension_host.rs` from `api.zed.dev` to Forge's registry. Other callers of `build_zed_api_url` are untouched. The registry's read endpoints are unauthenticated.
- **Install failure reporting.** The download now checks the HTTP status (404 and non-success). It emits `Event::ExtensionFailedToInstall`, and `extensions_ui` shows a notification ("Couldn't install the '...' extension: ..."). Only failures the user asked for are reported. Tests are in `extension_store_test.rs`.
- **README viewer.** Each extension card gets a document button and a "View README" menu entry that opens the README in a tab (`crates/extensions_ui/src/extension_readme.rs`, 363 lines). It is rendered with the editor's Markdown renderer, fetched from `/extensions/{id}/readme` (`ExtensionStore::fetch_extension_readme`). The README is treated as untrusted: images only load from GitHub over https, links open only when they are web, mail or phone addresses, and relative links resolve against the registry's stored bases.
- **Razor suggestion.** `.cshtml` and `.razor` files suggest the `razor` extension (`crates/extension_suggest/src/extension_suggest.rs`). The in-tree Razor extension was removed.
- **extension_cli.** `--lenient` packages extensions that fail validation (`crates/extension_cli/src/main.rs`).
- **Network.** The Forge registry host. README images from GitHub.
- **Coupling.** Medium. All are patches to upstream crates.

## 23. Identity, credentials and account-independence changes

- **No Zed account or hosted services.**
  - Zed-hosted edit prediction defaults to `none`, and the provider option is hidden.
  - There is no default agent model (`agent_settings` no longer unwraps it).
  - The Zed AI trial upsell and announcement are removed, and the built-in agent is labelled "Forge Agent".
  - The collaboration panel is not loaded and has no menu entry. Sign-in is hidden (`show_sign_in: false`).
  - Code: `crates/agent_ui/src/agent_panel.rs`, `crates/edit_prediction/src/edit_prediction.rs`, `crates/collab_ui/src/collab_ui.rs`, `crates/zed/src/zed.rs`, `crates/zed/src/zed/app_menus.rs`, `assets/settings/default.json`.
- **Telemetry hard-off.** `USAGE_DATA_UPLOAD_ENABLED = cfg!(test)` in `crates/client/src/telemetry.rs`, with a compile-time assertion that stops a non-test build shipping it on.
- **Keychain namespacing.** `crates/zed_credentials_provider/src/zed_credentials_provider.rs` prefixes keys with the app's scheme (`forgesuper:https://...`), and migrates old entries. Dev builds use the keychain by default, and the plaintext file is opt-in through `ZED_DEVELOPMENT_USE_FILE_CREDENTIALS`.
- **Rebrand and editions.** `editions.json` and `crates/app_names` drive the product name, directory names, IPC name, scheme and app id. The URL scheme is `forgesuper://` (`client::APP_URL_SCHEME`). The command palette shows `zed::` actions as `forge:` (display only, `crates/command_palette/src/command_palette.rs`). Icons, installer assets, the welcome and onboarding logos, and `release_channel` were also changed.
- **Resources.** Keychain reads and writes. No new network hosts.
- **Coupling.** Cross-cutting edits in many upstream crates.

## 24. Release, update and distribution infrastructure

- **Notify-only update check.** `crates/auto_update/src/release_check.rs` (298 lines). It asks `https://api.github.com/repos/mentenaz/mentenaz.Forge.Super/releases/latest`, at most daily, only after the user consents on first launch. The user can skip a release. It never downloads. Releases are tagged `dev-vN`, with N from the compile-time `FORGE_BUILD_NUMBER`. It shows an "Updated to dev-vN" notice on the first start after an update, and a manual Check for Updates. The UI is in `crates/auto_update_ui/src/auto_update_ui.rs`, which was largely rewritten (+320 lines). State uses KVP keys `forge-update-check-*` and `forge-last-seen-build`.
- **Import previous data.** `crates/zed/src/zed/import_previous_data.rs` (337 lines). At startup it copies settings and data from the earlier `ZedDev` folders into the app's own folders, skipping logs and rebuildable caches. It has the action "Import Data from Previous Version", and never touches upstream Zed's folders.
- **Packaging.** `.github/workflows/build_installers.yml`, `promote-apt-stable.yml`, `build_windows_dev.yml`, `fork_tests.yml`. `script/bundle-deb`, `script/publish-apt-repo` (an aptly repo published to downloads.mentenaz-server.com), `script/install.sh`, `inno/x86_64/zed.iss` (Windows installer showing the GPL), `script/windows-register-context-menu-fallback.ps1`, and the explorer command injector manifests.
- **Docs.** `crates/release_channel` carries `FORGE_DOCS_URL`, and troubleshooting messages point at `forge.mentenaz-server.com/docs`.
- **Coupling.** Build and release layer. It does not affect the extension question.

## 25. Panel ordering and "Move Left/Right" (workspace dock changes)

- **What it does.** A new `panel_order` setting (a list of snake_case panel keys) orders panel buttons per dock. Unlisted panels follow by activation priority. The status-bar button context menu gets **Move Left** and **Move Right**, which swap two panels in the setting by writing `settings.json` (`swap_panel_order`). Panel sorting observes `SettingsStore` and keeps the active panel. Defaults also add `resize_all_panels_in_dock: ["left","bottom"]`.
- **Where.** `crates/workspace/src/dock.rs` (+154 lines: `panel_order_name`, `panel_sort_key`, `swap_panel_order`, `Dock::sort_panel_entries`). Setting in `crates/settings_content/src/workspace.rs` and `crates/workspace/src/workspace_settings.rs`.
- **Why it exists.** The fork adds about 10 dock panels, so ordering and position management became necessary. It touches the core `Dock` and `Panel` plumbing that extension UI would also rely on.
- **Resources.** Writes `settings.json` through `update_settings_file`.
- **Coupling.** Very high, since it is a change to the workspace core.

## 26. Smaller upstream-crate patches

- **Debugger.**
  - A "Toggle Ignore Breakpoints" button and greyed breakpoints in the breakpoint list (`crates/debugger_ui/src/session/running/breakpoint_list.rs`). The `debugger-ignore-breakpoints-toggle` branch is 1 commit ahead of `main`.
  - A rework of the new-process modal (`new_process_modal.rs`, +285 lines).
  - The attach modal.
  - The JavaScript adapter treats chained commands in a JS debug terminal as debugging every step (`crates/dap_adapters/src/javascript.rs`).
  - `crates/debugger_ui/DAP_DEBUGGING_GAPS_NOTES.md` is a design note.
- **Shell and terminal.** About 149 lines added to `crates/util/src/shell.rs`, and `crates/terminal/src/alacritty/hyperlinks.rs`.
- **Settings.** `settings_content/src/panels.rs` (114 lines of panel setting types), `settings_ui` pages for each panel, and `vscode_import`.
- **CLI.** `crates/cli/src/main.rs` (+148 lines) for the renamed command and the scheme.
- **Windows.** The explorer context menu, the `windows_resources` file metadata, and `install_cli`.
- **Misc.** `workspace/src/welcome.rs` and `onboarding` logos, `ui/src/components/image.rs`, `git_ui` commit tooltip and `git_graph`, and `remote_identity`.

---

## Summary table

| # | Feature | Kind | Main location | UI surface | Network | Processes | Persistence | Coupling to Zed internals |
|---|---|---|---|---|---|---|---|---|
| 1 | Cockpit | Dock panel | `crates/cockpit_panel` | Left dock, charts | none | none (sysinfo) | settings | Medium |
| 2 | Dashboard | Pane item | `crates/dashboard_panel` | Tab, cards | none (triggers others) | none | workspace DB | High (entities of other panels) |
| 3 | Processes | Dock panel | `crates/processes_panel` | Bottom dock, DataTable | none | kill PIDs (sysinfo) | in memory | Low-medium |
| 4 | Script Runner | Dock panel | `crates/script_runner_panel` | Bottom dock, streaming list | none | `sh -c` / `cmd /C`, `kill` | in memory | Medium |
| 5 | Node | Dock panel + libs | `crates/node_panel`, `node_versions`, `node_backend`, `npm_backend` | Left dock | nodejs dist (via fnm), npm | npm/yarn/pnpm, opener | KVP, data dir | Medium-high |
| 6 | Python | Dock panel + libs | `crates/python_panel`, `python_versions`, `python_backend` | Left dock | pypi, osv.dev, GitHub (python-build-standalone) | python, pip, venv | data dir, KVP | Medium-high |
| 7 | .NET | Dock panel + lib | `crates/dotnet_panel`, `dotnet_backend` | Left dock | nuget.org | `dotnet` | settings | Medium |
| 8 | Rust | Dock panel + lib | `crates/rust_panel`, `cargo_backend` | Left dock | crates.io, OSV | `cargo`, `rustc` | settings | Medium |
| 9 | Database workbench | Dock panel + 2 tabs | `crates/database_panel`, `database_backend` | Bottom dock, SQL editor, graph | user DB hosts (TCP, TLS) | none | KVP, keychain, tab restore | High (editor completion, Item) |
| 10 | Flows (engine, panel, Designer) | Panel + tabs + engine | `crates/workflow_engine`, `flows_panel`, `designer_panel` | Left dock, graph canvas, editor | arbitrary HTTP, webhooks, TCP localhost | user scripts, `StartProcess` | `.flow.json` files, KVP | High |
| 11 | Package managers (npm, NuGet, PyPI, Cargo) | 4 pane items | `crates/*_manager_panel` | Tabs, settings sidebar, README | npmjs, nuget, pypi, crates.io, OSV | package CLIs | partly serialized | Medium-high (editor menu patch) |
| 12 | npm bootstrap | Workspace hook | `crates/npm_bootstrap` | Toast | npm | `npm install` | marker | Low-medium |
| 13 | Helm (GitHub client) | Dock panel + libs | `crates/helm_panel`, `helm_backend`, `helm_ui` | Left dock, modals, Actions tab | api.github.com, github.com OAuth, git HTTPS | `git`, `gh` | keychain, settings | Very high |
| 14 | Helm Workspace | Pane item | `crates/helm_workspace` | Tab, tree, read-only editor | api.github.com | none | memory | High |
| 15 | Git panel: Details tab, GitHub account, push gating | Edits to git UI | `crates/git_ui`, `git`, `project/git_store` | Git panel tab, footer, toasts | via git | `git log` etc. | none | Very high |
| 16 | Git credential helper | Binary mode + IPC | `crates/git_credential`, `ide_env`, `zed/main.rs` | None | none | git calls `<zed> --git-credential` | memory | Very high |
| 17 | IDE runtimes and PATH injection | Global hook | `crates/ide_env`, `project/terminals.rs` | None | none | env of all spawned shells | data dir | Medium |
| 18 | GPUI layout `scale` | Core patch | `crates/gpui` | All elements | none | none | none | Highest (core) |
| 19 | gpui_flow canvas | Library | `crates/gpui_flow` | Custom element | none | none | none | Low (needs `scale`) |
| 20 | gpui_component stack | Vendored libs | `crates/gpui_component`, `gpui_base`, others | All new panels | none | none | none | Parallel UI stack |
| 21 | sqls extension | WASM extension | `extensions/sqls` | LSP in editor | none | `sqls` | none | Low (public extension API) |
| 22 | Extension registry, install errors, README tab | Edits to extension crates | `extension_host`, `extensions_ui`, `http_client` | Notification, tab | Forge registry | none | none | Medium |
| 23 | Account-independence, telemetry off, keychain namespacing, rebrand | Cross-cutting | many | Settings, menus | none new | none | keychain | Cross-cutting |
| 24 | Update check, data import, packaging | App infrastructure | `auto_update`, `zed/import_previous_data.rs`, `.github`, `script` | Prompts, notices | api.github.com, apt host | installers | KVP, data dirs | Medium |
| 25 | Panel order and Move Left/Right | Core patch | `crates/workspace/src/dock.rs` | Status-bar menu | none | none | `settings.json` | Very high (core) |
| 26 | Small patches (debugger, shell, CLI, settings UI) | Edits | various | Debugger UI | none | none | none | Medium |

## Cross-cutting patterns

- **Everything is in-tree and in-process.** None of the features uses a plugin boundary. The only WASM extension is `sqls`.
  - Panels hold `Entity<OtherPanel>` handles and read each other's state (Dashboard, Node to Flows, Node and Python to the Script Runner, `git_ui` to Helm).
  - They are wired in `crates/zed/src/zed.rs` and `main.rs`. There is no registration API.
- **Panels use three Zed integration points.**
  - `workspace::dock::Panel` (11 panels) for sidebars.
  - `workspace::Item` plus `SerializableItem` for tabs.
  - `register_project_item` to claim a file type (the Designer for `.flow.json`).
  - Status-bar buttons, key contexts and `actions!` come with these.
- **Panel/backend split.** Most features have a `*_backend` crate with no UI and no app state. Examples are `cargo_backend`, `npm_backend`, `dotnet_backend`, `python_backend`, `node_backend`, `database_backend`, `helm_backend` and `workflow_engine`. This keeps them testable and Zed-independent, and it is the only part that would port easily to an extension.
- **Package-manager pattern.** Registry search over HTTPS plus CLI execution in the Script Runner, with strict argument validation (`command.rs`) because the commands go through a shell.
- **Processes.**
  - Spawned binaries: `npm`/`yarn`/`pnpm`, `dotnet`, `python`/`pip`, `cargo`/`rustc`, `git`, `gh`, `sqls`, user scripts, and `kill`/`taskkill`.
  - Several panels block on CLIs and must be run on background tasks.
- **Network.**
  - Public registries: npmjs.org, nuget.org, pypi.org, crates.io, OSV.dev.
  - GitHub: `api.github.com`, `github.com` (OAuth device flow and git over HTTPS), `raw.githubusercontent.com`, `avatars.githubusercontent.com`.
  - The Forge registry and apt host.
  - User-defined hosts: DB servers and workflow HTTP and webhook targets.
  - Calls go through either Zed's `http_client` or raw `reqwest`, `tokio` and DB drivers.
  - The fork also removes Zed-hosted calls: telemetry, edit prediction, collab and the update server.
- **Secrets.** Always the OS keychain through `zed_credentials_provider` (`helm://github.com/<login>` and `database://<type>/<host>:<port>`). `settings.json` stores only non-secret identifiers. The git token is passed by environment variable to a single git process, or through the credential helper socket.
- **Persistence.** There are four mechanisms:
  - `settings.json` (per-panel `button` and `dock`, `panel_order`, Helm accounts, `flows.workflows_dir`).
  - The `db::kvp` store (DB connections, Node choices, update-check state, flow history).
  - The workspace DB via `SerializableItem` for tabs.
  - Files in the user's repository (`.flow.json` and sidecars, `.nvmrc`, manifests) or in `paths::data_dir()` (`node_versions`, `python_versions`).
- **Filesystem watching.** Panels rescan when a manifest or lockfile changes.
- **Theming.** A second theme layer, `gpui_component`'s `cx.theme()`, must be initialised, and `gpui_flow` takes raw `u32` colours converted by callers.
- **Keymap.** A `ctrl-k <letter>` chord family in the default keymaps: `h` Helm, `x` Processes, `d` Database, `j` npm, `y` Python manager, `g` NuGet, `o` Node, `i` Python, `f` Flows, `k` Script Runner, `c` Rust, plus `ctrl-3` for git Details.
- **Core patches (hardest to carry as a fork).**
  - The GPUI `scale` change.
  - The `Dock` ordering.
  - The `GitRepository` trait additions.
  - The `git_ui` to `helm_panel` dependency.
  - The `main()` credential-helper mode.
  - The `editor` context menu.
  - The `extension_host` registry routing.
  - These conflict most with upstream merges.
- **Relevance to the discussion #53403 API needs** (inferred from the thread summary and this code):
  - UI primitives: dock panels, pane tabs, status-bar buttons, toasts, modals, context-menu entries, tables, trees, charts, a graph canvas, resizable splits, Markdown, an editor widget, and a file picker.
  - Host-resource needs: workspace and project context, file watching, spawning processes with a PATH override, keychain access, HTTP from the host, settings and KVP persistence, and cross-panel messaging.
  - Core hook needs: terminal environment injection and git credential injection.
  - Claimed file types: the Designer for `.flow.json`.
- **Caveat.** The repo is the same lineage as mentenaz's `zed_Source` fork (merge `f9014e4dbf`), so it matches the fork described in the thread: GPL-3.0, with database browser, workflow editor, package managers, system monitor, dashboard and GitHub client. It is not the Tauri "Forge" app, and the schema-driven panel system is not implemented here (see section 0).
- **Verification limits.** Line numbers come from grep of `HEAD`. I read the READMEs and key sources, not every file. Per-panel network and process lists come from grep of string literals and `Command::new` calls, so dynamically built hosts or commands may be missing.

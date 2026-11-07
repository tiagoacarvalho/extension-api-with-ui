# Discussion 53403: RFC: Visual Extension API - Allow Extensions to Render GUI

- URL: https://github.com/zed-industries/zed/discussions/53403
- Author: nguyenphutrong (Trong Nguyen)
- Created: 2026-04-08
- Category: Feature Requests
- Upvotes on the post: 74
- Comments: 14 top-level (one spam-minimized). Fetched in full via GraphQL with no pagination left.
- Last activity seen: 2026-11-06

Staff identification: GitHub `authorAssociation` is used below. No commenter is MEMBER or OWNER. macraig is CONTRIBUTOR but speaks as the Zed team ("our team", "our extensions roadmap"), so treat macraig as the Zed-team voice. RemiKalbe and pleahmacaka are also CONTRIBUTOR. Everyone else is NONE.

## 1. Original proposal (nguyenphutrong, status Draft, "Willing to implement: Yes")

Related issues: #21400 (closed, unactionable), #48947 (discussion, open, 8 votes).

**Goals**
- Extensions register and display custom panels.
- Extensions add items to the status bar.
- Extensions render visual content, not just text.
- The API is safe, sandboxed and high-performance.

**Non-goals for the MVP**
- Full webview support (GPUI rendering only).
- Custom editor implementations.
- Direct access to the windowing system.

**Motivation**
- Today's extension features are "headless": language servers, slash commands, context servers (MCP), snippets, themes and debug adapters.
- Use cases in the RFC table:
  - i18n manager (table view)
  - database browser (tree view, query results)
  - API client (request builder, response viewer)
  - Git GUI (commit graph, branch view)
  - task runner (progress bars, output panels)
  - color picker
  - image preview (thumbnail gallery)
  - plot/chart

**Design principles ("learning from VS Code")**
- A restricted API with only safe, optimized components.
- GPUI-based, no webviews.
- A performance budget: extensions must meet frame-time requirements.
- Automatic theming.

**Architecture and API (WIT)**
- Extensions run in the WASM extension host and talk to GPUI through a "Visual Extension API (WIT)" with three parts: a panel API, a status bar API and view components.
- `panel` interface:
  - `panel-position`: left-sidebar, right-sidebar, bottom-panel.
  - `panel-info`: id, name, optional icon name, position, default-visible.
  - `panel-handle` resource: `update(element-tree)`, `set-visible`, `is-visible`.
  - `register-panel`.
- `components` interface: an `element` variant with container, text, button, text-input, table, list and icon.
  - `container-props`: direction, gap, padding, background.
  - `button-props`: label, variant, on-click callback-id, disabled.
  - `table-props`: columns, rows, selection-mode, on-selection-change.
  - A `callback-registry` resource and a `callback-id` string for interactivity.
- `status-bar` interface:
  - `add-item` takes a `status-item` (id, content, priority where lower means further left, tooltip, on-click).
  - `status-content` is text, icon or text-with-icon.
  - `status-item-handle` has `update-content` and `remove`.
- Extension trait update:
  - `build-ui() -> ui-definition` (panels, status-items).
  - `handle-event(ui-event) -> result<option<element-tree>, string>`.
  - A Rust example uses `build_ui` and `render_panel(panel_id) -> ElementTree`.

**Phased plan**
1. Status bar items (text/icon): low complexity, 1-2 weeks.
2. Simple read-only panels (text, buttons, lists): 3-4 weeks.
3. Interactive panels (forms, tables; i18n editor, DB browser): 6-8 weeks.
4. Advanced components (charts, image preview, custom renderers): 2-3 months.

**Technical challenges**
- WASM-to-GPUI bridge: async message passing via WIT.
- Event latency: batch events, async callbacks.
- Performance isolation: frame budget and yield points (high risk).
- Theming: auto-convert extension colors.
- Accessibility: ARIA metadata in WIT.

**Security and safety**
- WASM sandbox, with no direct filesystem access (worktree API) and network via the existing http-client API.
- No arbitrary HTML/CSS.
- Predefined, validated components only.
- Resource limits (panel size, component count).
- Frame-time budget, auto-disable of consistently slow extensions, lazy loading of inactive panels.

**Alternatives considered**
- WebView (VS Code approach): rejected. Easy for authors but heavy and insecure.
- GPUI-based: chosen. Native performance and secure, but harder for authors.
- React-like DSL: possible in phase 2 or later.

**Impact**
- A new crate, `extension-ui`, of about 2000-3000 lines.
- Changes to `extension_host` and `workspace`.
- A new API surface needing stable versioning.
- The author projects about 30% of extensions having UI after one year.

**Commitment**: the author offers a proof-of-concept PR for Phase 1 (status bar). RFC references: VS Code Webview API, JetBrains Plugin API (Swing), Sublime Quick Panel API.

## 2. Thread, chronological, by commenter

### KiidxAtlas (NONE), 2026-04-11
- Moved away from Zed because of this gap: "this is something that actually caused me to move away from zed some time ago".
- Says extension support should be "even more capable than what other ide alternatives have".
- Offers to help build the feature.
- On 2026-06-18 replied "Thanks for saying this man" to RemiKalbe.

### nakajimayoshi (NONE), 2026-05-11
- Wants a DB viewer "to fully replace my workflow in VSCode" and offers to help.
- Objection to the RFC's `build_ui` design: it would break every existing extension. UI semantics differ from LSP or debugger semantics, so propose a separate trait that inherits `Extension`, a `UIExtension: Extension` with a GPUI `render` that returns `impl IntoElement`. Sketch: `cx.create_pane(PaneOptions { position: PanePosition::Left, width: px(500) }, |_, cx| cx.new(|_| HelloWorld {...}))`.
- Argues "there's no better abstraction for Zed UI extensions than GPUI". It has web-like semantics, is well understood by the Zed team, has docs (https://www.gpui.rs/) and a component library (https://github.com/longbridge/gpui-component). A new DSL would be a big ask given the team's AI focus.
- Security: running GPUI code in the main process "eliminates Zed's entire security model and allows any UI extension developer to run native code on your machine". Proposes a serde implementation of GPUI so subprocesses make GPUI calls over IPC, with the main process sanitizing calls and validating permissions.
- Addendum: WIT is probably better than IPC because it is already supported by the extension API and is faster. "How GPUI components can be translated across WIT has yet to be determined."
- On 2026-06-18: "Your LLM features are only as useful as the tools they can use, and Zed is missing many found in other editors." Says the team is making "a huge mistake by not focusing on editor features".

### clementGilardy (NONE), 2026-05-19
- Wants a native sidebar panel with real-time data from external services (Linear, Jira, AWS, database views).
- MCP works for AI workflows but does not give "a persistent, always-visible panel".
- Concrete cases: a Linear/Jira panel for tickets linked to the current branch or file; an AWS resource explorer docked in the sidebar; a live DB table viewer; a CI/CD pipeline status panel.
- Tired of JetBrains resource overhead and wants to switch to Zed.
- Offers help with WIT interface prototyping and a reference extension.

### jeremycastanza (NONE), 2026-05-19 (two comments)
- Extensions are the one feature "keeping myself and other developers tethered to other IDEs like Cursor or VS Code". Wants high priority.
- UI concern: in VS Code, UIs "make the IDE look like a mash up of different tools". An opinionated UI API exposing views that "appear native to Zed" would be "the best of both worlds".
- Asks: "if we wanted to get a small group together to work on this, what would it look like?" (no answer in the thread).

### luan-coelho (NONE), 2026-05-19
- "a robust extensions API is still not a clear priority for the Zed team".
- Heavy IntelliJ user. Praises the JetBrains SDK: deep IDE integration, UI customization, tools, panels, actions, inspections.
- Argues the team should prioritise a strong extensibility foundation over organisation, team and AI features.
- On 2026-07-26 replied to vitalics: "It appears the focus is still on AI capabilities".
- On 2026-09-11 about embedded_gpui: "This looks like a good sign to me; I hope it's not false hope."

### AirboZH (NONE), 2026-05-19
- Wants to build a JSONL preview extension and a SQLite viewer extension. Custom UI is "the essential first step".
- Currently has to jump back to VS Code or external tools. Offers help.

### macraig (CONTRIBUTOR; speaks for the Zed team), 2026-05-21 (13 upvotes, 6 replies)
Only team statement in the thread, quoted in full:
> "Hey everyone, thanks for the enthusiasm and willingness to work on this — we really appreciate it! This is closely tied to our extensions roadmap, and something that needs to be driven by our team to make sure it's done in a coherent way. It's a significant undertaking and not something we're likely to get to in the near future, but it's very much on our radar as part of the bigger extensions story. We'll update this thread as things progress!"

Replies:
- **RemiKalbe (CONTRIBUTOR), 2026-06-18**
  - Quotes "not something we're likely to get to in the near future".
  - "Zed is focusing way too much on AI when something as important as this has been so neglected."
  - Hearing that it is not coming soon is "really frustrating".
  - Was drawn to Zed for its editor-experience focus. Still loves Zed and appreciates ACP.
- **KiidxAtlas, 2026-06-18**: "Thanks for saying this man."
- **nakajimayoshi, 2026-06-18**: see above. Echoes the AI-focus criticism.
- **vitalics (NONE), 2026-07-25**: pings @macraig for an update, ETA or plans. No staff reply recorded.
- **luan-coelho, 2026-07-26**: see above.
- **R-Bower (NONE), 2026-07-30**: "without UI extensions, this limits Zed to something adjacent to an LSP-integrated text editor (albeit a powerful one). Still useful, but not what I need for a daily driver of complex software development."

### jmtrs (NONE), 2026-08-08
- "It's August already and this is still quiet."
- Joke/use case: a Nyan Cat in the status bar, like the old IntelliJ NyanProgressBar plugin. Attached a screenshot mock-up. Asks for an update.

### Muhammad-Owais-Warsi (NONE), 2026-09-10
- Asks for updates.
- **pleahmacaka (CONTRIBUTOR), 2026-09-11**: "Looks like they're experimenting with something https://github.com/zed-industries/embedded_gpui"
- **luan-coelho**: see above.

### mentenaz (NONE), 2026-10-05: fork and design report (11 replies)
Top comment:
- "I wanted panels like these badly enough that I forked Zed and built them directly into the app, since there's no extension route."
- A personal GPL-3.0, Windows-tested fork: "evidence of what the API would need to support, not a proposal to merge anything".
- Source and write-up: https://github.com/mentenaz/zed_Source
- Screenshots: Database schema graph, visual workflow editor and dashboard, under https://raw.githubusercontent.com/mentenaz/zed_Source/main/docs/screenshots/ (Database_Schema_Graph.png, Flow_Running.png, DashBoard.png).

Panels and the UI primitives each needed:

| Panel | What it does | Primitives needed |
| --- | --- | --- |
| Database | Live SQLite/Postgres/MySQL/MSSQL, schema explorer, SQL workbench, schema graph | tree view, data table, code-editor surface, node-graph canvas |
| Workflow editor | Executable flows with branching and error paths, per-node results | node-graph canvas with zoom/pan/fit-view, property forms |
| Package managers (npm, NuGet, pip) | Search, install, update, vulnerability scan, inline README | searchable lists, markdown rendering, badges, forms |
| System monitor and processes | CPU/RAM/disk/network, per-core view, process list with kill | charts, heatmap, sortable virtual table |
| Dashboard | Runtimes, security findings, outdated packages, git status | cards, status badges, embedded views |
| GitHub client | Repo search, clone with folder picker, repo browsing | nested navigation, lists, credential storage |

API lessons stated:
- **Graph/canvas primitive.** Two panels are node graphs. "GPUI had no zoom/pan/fit-view, so I added it and plan to offer it upstream separately."
- **Theme inheritance must be automatic.** Zed's theme tokens differ from the GPUI component library's, so mentenaz wrote an adapter.
- **Workspace context.** Panels act on the active project and read git and settings state, so mentenaz touched `workspace`, `project`, `git` and `settings`. "An API that only offers an isolated widget would not have been enough."
- **Permissions.** Panels read the filesystem, list and kill processes, and open network and database connections. A sandboxed API needs an explicit permission model.
- **Long-running work.** Scans, DB connections and polling need async tasks and a way to show unknown or failed states.
- Offers to test an API prototype against these panels.

Replies in this subthread:
- Muhammad-Owais-Warsi (2026-10-05): "That's really cool".
- mentenaz (2026-10-05): offers to share more on the database browser, workflow editor or package managers.
- orangethewell (NONE, 2026-10-05): wants a good SQLite DB viewer in Zed.
- mentenaz (2026-10-05):
  - Already built that: a DB panel connecting to SQLite, PostgreSQL, MySQL and MSSQL, with a schema explorer, a relationship graph and a SQL workbench that exports as JSON or SQL.
  - Screenshots show Zed's own local SQLite DB: 39 tables, 221 columns, 30 foreign keys.
  - Caveats: an experimental fork, not an extension, no packaged release, build from source, Windows only tested.
  - Asks what a SQLite viewer should do: row browsing and editing, query history, export.
- hbouhadji (NONE, 2026-10-13): asks how hard it is to keep up with upstream and what the build time is.
- mentenaz (2026-10-14): syncs the fork regularly and fixes conflicts. Release builds run on GitHub, "at 25.*" (version number unsure).
- mentenaz (2026-10-15): full build takes about 20 minutes and a release build about an hour locally. On GitHub it usually takes 2 hours.
- tiagoacarvalho (NONE, 2026-11-04): wants to join. Was planning to move from Codium and this was the only blocker. Asks about plans to get from the current state to a pull request.
- mentenaz (2026-11-04):
  - "I doubt Zed Industries would accept this as a feature."
  - Each engine backend is about 1,000 lines. With Oracle added "the scope is large for a codebase as big as Zed's".
  - Works in the fork as a daily driver: about 500 MB with 30 engines connected.
  - Links a second project: https://github.com/mentenaz/mentenaz.Forge.Super
- tiagoacarvalho (2026-11-04):
  - Will start on the Visual Extension API this RFC discusses.
  - Understands that "they do plan on it, but they're short on hands and it's not a priority".
  - Wants to try mentenaz's project to build a comprehensive requirements list.
- mentenaz (2026-11-04): "Thanks reach out whenever youre ready".

### magalhaesfelipe (NONE), 2026-10-14: minimized as spam
- Body was a one-line "Any updates?" (no spam content to reproduce).

### tiagoacarvalho (NONE), 2026-11-04 (3 replies)
- Wants to start work and asks if anyone has made progress. Notes the Zed team is experimenting in https://github.com/zed-industries/embedded_gpui and will look into it.
- **mentenaz**: "Okay I guess they first trying that bef9re incorporating it into zed".
- **tiagoacarvalho**:
  - Does not think "this kind of two-stage embedding would be the desired solution on an editor like Zed which already goes such lenghts to run fast and efficient", especially "the javascript plugins". Acknowledges JS has a strong point in sandboxing what an extension can do.
  - Suggests "concurrent approaches" might be worth exploring.
  - Shares a screenshot showing the Zed team's branch has not moved in two months: "one hell of a merge conflict waiting to happen".
  - Is using mentenaz's work as a benchmark, wants extensions to do this "without vscode's alarming performance and security issues".
- **mentenaz** (2026-11-04), design write-up "what my panels needed":
  - A panel declares: direction (left, right or bottom, same as built-in panels); an icon (one of Zed's already-registered icons); a stable ID and a title; a UI body (GPUI elements or a declarative schema).
  - In the Zed fork, each panel is split into two crates so upstream merges barely touch them: `panel_ui` (GPUI elements only) and `panel_backend` (DB, process, git, network logic).
  - In Forge (Tauri), the schema-driven extension system works like this:
    - A panel is a JSON schema plus an optional backend binary. The host renders the schema without running any extension UI code.
    - UI components: sections, status/metric rows, inputs, buttons, toggles, selects and templated lists. Components read from named bindings, and buttons trigger named actions with the current input values.
    - Backend calls are one-shot: `binary --function <name> --Arguments <raw>`. stdout carries one JSON object on success; errors go to stderr with a non-zero exit code. There is a timeout, and the process is killed if it overruns. `binary --introspect` lists the available functions.
    - Theming: the host passes a small set of theme tokens.
    - Portability: the same cached package is rendered by both the Tauri build and the GPUI build, so the schema is not tied to one UI toolkit.
  - Example schema: panel id `weather`, title `Weather`, position `right`, icon `cloud`, backend `./logic.exe`, with a `TextField` (id `city`), a `Button` (action `search`) and a `List` (binding `results`).
  - Limits:
    - Rich components: tables, charts, markdown and node-graph canvases needed native components, so a small set of rich built-ins is likely needed.
    - Streaming: one-shot calls do not suit a system monitor. A persistent mode (JSON lines over stdout) would be needed.
    - Permissions: the process boundary gives crash isolation, but backends run with full user permissions. A real API needs an explicit permissions model (filesystem, network, process control).
    - Workspace context: panels need the active project, git state and settings.

### tiagoacarvalho (NONE), 2026-11-06
- "Work started in new zed fork": https://github.com/tiagoacarvalho/extension-api-with-ui
- Plans step-by-step work: a round of planning, design and modelling first, then the status-bar item.
- Follows "5. Implementation Strategy" from the RFC (Phases 1 to 4, quoted in the comment).

## 3. UI extension capabilities wanted, with who asked
- **Sidebar panels** (left, right, bottom): nguyenphutrong, clementGilardy, mentenaz.
  - Service panels: Linear/Jira tickets, AWS explorer, CI/CD status, GitHub client, package managers.
- **Status bar items**: nguyenphutrong (phase 1), jmtrs (Nyan Cat).
- **Custom views and previews**: JSONL preview and SQLite viewer (AirboZH), DB viewer (nakajimayoshi, orangethewell, mentenaz), dashboard and system monitor (mentenaz).
- **Rich primitives**: tree view, data and virtual tables, code-editor surface, markdown rendering, charts and heatmaps, cards and badges, node-graph canvas with zoom/pan/fit-view, forms and property editors (mentenaz). The RFC adds image preview, color picker and plots.
- **Native look and theming**: automatic theme inheritance (RFC, mentenaz); opinionated native-looking views (jeremycastanza).
- **Panels with workspace context** (project, git, settings) and background or streaming work (mentenaz).
- **Webviews**: explicitly out of scope per the RFC. Modals were not discussed.
- JetBrains-style deep UI integration (luan-coelho): tools, panels, actions, inspections.

## 4. Concerns and objections
- **Backwards compatibility.** Adding `build_ui` to `Extension` breaks all existing extensions (nakajimayoshi). Proposal: a separate `UIExtension` trait.
- **Security.**
  - Running GPUI in-process defeats the sandbox (nakajimayoshi).
  - Backends run with full user permissions (mentenaz).
  - RFC mitigations: WASM sandbox, validated components, resource limits.
- **Performance.**
  - RFC frame budget and auto-disable.
  - tiagoacarvalho distrusts the two-stage embedding / JS plugin approach seen in embedded_gpui.
- **Maintenance and scope.**
  - New API surface needing stable versioning (RFC).
  - mentenaz doubts upstream would accept large per-engine scope.
  - Fork merge-conflict burden (hbouhadji's question, mentenaz's answer).
- **Prioritisation.** RemiKalbe, nakajimayoshi, luan-coelho and R-Bower say Zed over-focuses on AI relative to extensibility. No one from the team replied to this.
- **UI consistency.** The VS Code "mash-up" worry (jeremycastanza), and the RFC's restricted, themed components.

## 5. Links mentioned
- https://github.com/zed-industries/embedded_gpui (pleahmacaka, tiagoacarvalho)
- https://github.com/mentenaz/zed_Source (mentenaz's Zed fork)
- https://raw.githubusercontent.com/mentenaz/zed_Source/main/docs/screenshots/Database_Schema_Graph.png
- https://raw.githubusercontent.com/mentenaz/zed_Source/main/docs/screenshots/Flow_Running.png
- https://raw.githubusercontent.com/mentenaz/zed_Source/main/docs/screenshots/DashBoard.png
- https://github.com/mentenaz/mentenaz.Forge.Super (mentenaz's second project, "Forge", Tauri-based, with a schema-driven panel system)
- https://github.com/tiagoacarvalho/extension-api-with-ui (tiagoacarvalho's new fork)
- https://www.gpui.rs/ and https://github.com/longbridge/gpui-component (nakajimayoshi)
- Issues #21400 and #48947 (RFC)
- Screenshot attachments (jmtrs, mentenaz, tiagoacarvalho): https://github.com/user-attachments/assets/8a688037-a967-416a-9bd9-6473468c7b2e, 067e169b-9537-43e7-aac3-a80e33b5986c, 33895ce6-913e-4808-b1b4-6a99a59f09ff, 6829ce5f-941c-4eac-b318-93caa5d75edb and 2fee6096-e004-4d67-8e91-1b5b52ae13c2 (same host path)

## 6. Open questions and unresolved points
- No timeline from the Zed team beyond macraig's 2026-05-21 comment. The promised thread updates did not appear. vitalics' ETA question went unanswered.
- What `embedded_gpui` is and how it relates to the RFC. Per commenters it is a two-stage embedding with JavaScript plugins, but this is not confirmed by the team. The branch is said to be inactive for two months.
- GPUI-in-process vs WIT/IPC vs declarative schema as the extension-to-host UI boundary. nakajimayoshi: "How GPUI components can be translated across WIT has yet to be determined."
- The permissions model (filesystem, network, process control) and how it is surfaced to users.
- Streaming and long-running updates for panels, plus a small set of rich built-ins, and graph/canvas support in GPUI (mentenaz plans to offer it upstream).
- Workspace context access (project, git, settings) and how extensions get it safely.
- Whether upstream would accept community PRs for this; mentenaz is doubtful.
- Backwards compatibility: separate `UIExtension` trait vs a new method on `Extension`.
- jeremycastanza's question about how a contributor group should organise was unanswered.

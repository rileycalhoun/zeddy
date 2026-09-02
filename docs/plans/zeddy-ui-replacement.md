# Zeddy UI replacement implementation plan

## Status and design authority

This plan replaces Zed's current application chrome with the Zeddy shell shown in [`zeddy-agent-workspace-v3.png`](../mockups/zeddy-agent-workspace-v3.png). The mockup is the visual and interaction authority for the canonical 1586 × 992 state.

The replacement is a composition change, not an editor rewrite. Zeddy keeps Zed's editor, pane tree, project model, actions, keymaps, panels, diff engine, terminals, notifications, persistence, and agent runtime, but presents them through one agent-first shell. The legacy shell is not a selectable end-state layout.

At plan creation, the repository baseline is:

- Commit: `2551721adb5b5187bc27cfae0fbe47f0ed4c5397`
- `Cargo.lock` SHA-256: `4db815cc36ebff234fce8c04ec1d5230cca613a907602af217bcca1d2d6ae331`
- Tracked manifests: 262 `Cargo.toml` files

The per-capability owners and verification paths are tracked in [`zeddy-ui-parity-matrix.md`](zeddy-ui-parity-matrix.md). GitHub issues #1 through #9 mirror the numbered implementation phases below.

## Non-negotiable constraints

1. Do not add, remove, rename, or edit a dependency. Every `Cargo.toml` and `Cargo.lock` must remain byte-for-byte unchanged.
2. Do not replace the editor, `PaneGroup`, project model, `ProjectPanel`, `AgentDiffPane`, terminal implementation, action system, or workspace persistence with parallel implementations.
3. Remove the current sidebar/activity-rail presentation. Do not introduce a VS Code-style icon rail.
4. Remove the duplicate planning UI made up of the task/progress card and the `Plan / Context / Changes / Tests` navigation. Its space becomes the subagent section.
5. Keep one plan in the center agent workspace with these canonical rows: `Inspecting search pipeline`, `Implementing weighted ranking`, and `Running focused tests`.
6. Keep the `Ready to review` card and route it through the existing diff and edit-acceptance machinery.
7. Keep the project file picker on the left, below subagents.
8. Provide an `Agent output` view for the provider-visible output stream, but never make it the default workbench tab.
9. Preserve all existing editor functionality. Features not visible in the mockup remain reachable through commands, shortcuts, the workbench overflow, or their existing editor surfaces.
10. Final cutover leaves one production shell. Temporary construction paths used while developing must be removed before completion.

## Acceptance map

| Mockup region           | Required behavior                                                                                                                          | Existing implementation to retain                                                                            | Required change                                                                                            |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| Top bar                 | Zeddy identity, repository/branch, global agent prompt, Run, Review, notifications/settings, native window controls                        | `TitleBar`, `PlatformTitleBar`, workspace/project Git state, agent message submission and review actions     | Recompose the title bar and inject agent controls from `zed`; do not make `title_bar` depend on `agent_ui` |
| Left: Subagents         | Stable list of direct subagents with complete/working/queued state; expandable `Now`, `Latest`, and update time; click-through to activity | `AcpThread`, subagent tool calls, `ConversationView::navigate_to_thread`, existing expandable subagent cards | Export a read-only subagent presentation model and render it as the top left section                       |
| Left: Project           | Existing file tree and file operations, with the selected file synchronized to the editor                                                  | `ProjectPanel` and project/worktree models                                                                   | Add an embedded presentation mode and mount the same panel entity below subagents                          |
| Center: Agent workspace | Current task, one three-step plan, live state, review card                                                                                 | `AcpThread::plan`, `ThreadView` plan rendering, `ActionLog`, `AgentDiffPane`                                 | Extract reusable views; add native-agent plan emission; remove duplicate plan surfaces                     |
| Right: Editor           | Existing tabs, splits, buffers, diagnostics, minimap, diff decorations, navigation and editing                                             | `Workspace::center`, `PaneGroup`, `Pane`, `Editor`                                                           | Keep the existing editor tree and give it the flexible column; never model the dashboard as an editor pane |
| Lower workbench         | Terminal, Problems, Tests, Agent output; canonical fixture selects Tests                                                                   | Existing terminal panel, diagnostics, task/test state, agent transcript/tool output                          | Provide a compact tab host and adapters; keep every legacy panel in overflow                               |
| Status bar              | Branch/sync, language, diagnostics, agent count, token meter, local-context state                                                          | `StatusBar` and existing items; `AcpThread::token_usage`                                                     | Reorder existing items and add small agent-specific items without replacing the status model               |

The canonical fixture must reproduce the mockup's information hierarchy and interaction states. Pixel comparison is a regression tool, not permission to hard-code demo data into production.

## Target composition

```text
MultiWorkspace
└── Workspace
    ├── Zeddy title bar
    ├── Zeddy body
    │   ├── Left navigator
    │   │   ├── Subagents
    │   │   └── Embedded ProjectPanel
    │   ├── Agent column
    │   │   ├── Agent workspace
    │   │   └── Workbench tabs
    │   └── Existing PaneGroup editor
    ├── Existing modal, notification, toast, and overlay layers
    └── Reconfigured StatusBar
```

The application crate, `zed`, remains the composition root. It already depends on `agent_ui`, `project_panel`, `sidebar`, `workspace`, `title_bar`, `terminal_view`, `tasks_ui`, `diagnostics`, and `git_ui`. Lower-level crates must not import one another merely to assemble this layout.

### Ownership boundaries

| Crate                                     | Owns after the redesign                                                                                             | Must not own                                                     |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `workspace`                               | Type-erased shell slots, splitter/focus geometry, claimed-panel mounting, overlay ordering, shell persistence hooks | Agent, project-tree, terminal, test, or diagnostics domain logic |
| `zed`                                     | Construction and wiring of the concrete Zeddy shell; async panel handoff; compatibility action mapping              | Duplicate agent/project/editor models                            |
| `title_bar`                               | Generic Zeddy title-bar structure and type-erased center/action slots; platform window chrome                       | Direct `agent_ui` imports                                        |
| `project_panel`                           | Existing file-tree behavior plus an embedded visual mode                                                            | Subagent state or agent controls                                 |
| `acp_thread`                              | Provider-neutral plan/token/output state and events                                                                 | Workspace presentation                                           |
| `agent`                                   | Native plan tool/event production and native subagent lifecycle                                                     | GPUI shell presentation                                          |
| `agent_ui`                                | `AgentWorkspaceModel` and reusable Subagents, Plan, Review, Prompt, and Agent Output views                          | Project tree or outer workspace layout                           |
| Existing terminal/task/diagnostics crates | Their existing state and production views                                                                           | A second workbench framework                                     |

## State and event contracts

### Workspace shell

Add a type-erased shell contract in `crates/workspace/src/workspace.rs`, represented by a small slot/delegate structure rather than feature-crate imports. It needs handles for:

- the left navigator;
- the agent workspace;
- the lower workbench;
- optional title-bar controls;
- the focus handle and minimum size for each region;
- which registered panels are visually claimed by the shell.

`Workspace` continues to own and render the existing `PaneGroup`. The delegate supplies adjacent views only. `MultiWorkspace` stops rendering the old registered sidebar as a full-height sibling, while its old persisted state and actions are migrated as described below.

Only one parent may render a claimed panel entity. The embedded `ProjectPanel`, `TerminalPanel`, and any other rehosted panel stay registered with `Workspace` so calls such as `workspace.panel::<ProjectPanel>()` keep working, but their legacy `Dock` does not render them simultaneously.

### Agent workspace snapshot

Export a read-only, presentation-ready snapshot from `agent_ui` instead of exposing `ConnectedServerState.threads`. The snapshot should contain:

- root session ID, title, and `ThreadStatus`;
- current `Plan` and completed-plan snapshot;
- stable direct-child subagent rows;
- changed-buffer count, paths, and diff statistics from the linked root `ActionLog`;
- aggregate `TokenUsage`;
- permissions/errors that require a visible response;
- actions for navigation, stop, submit, review, keep, and reject.

Subagents are keyed by session ID and ordered by their first spawn occurrence. If a resumed session appears in multiple tool calls, use one row and the latest invocation state. Direct children appear in the root section; nested children appear after navigating into their parent. Do not use `ThreadMetadataStore`, which intentionally excludes subagents.

Map state as follows:

| UI state        | Data source                                                                            |
| --------------- | -------------------------------------------------------------------------------------- |
| Complete        | Child thread is idle and the latest spawn tool call completed without error            |
| Working         | Child thread is generating or its spawn tool call is in progress                       |
| Queued          | A real pending spawn tool call/session exists but has not started                      |
| Failed/canceled | Tool-call or thread terminal state, shown explicitly rather than as Complete           |
| Now             | Latest in-progress tool call label and first relevant location/file                    |
| Latest          | Latest completed child tool-call label                                                 |
| Updated         | Timestamp maintained by the presenter when a subscribed child emits a meaningful event |

Never synthesize a queued subagent from a future plan step. If an external ACP implementation exposes a subagent ID but cannot load its session, keep the row and show its parent tool-call activity as a degraded, non-clickable fallback.

### Native plan production

External ACP agents already populate `AcpThread::plan` through `SessionUpdate::Plan`; the native agent does not. Add a native update-plan tool and event path in `agent` using the existing `acp_thread::Plan` types:

1. Register an `UpdatePlanTool` with the native agent's existing tool registry.
2. Validate ordered entries and statuses without panicking on malformed model input.
3. Emit a plan update from the native `Thread`.
4. Forward the event through `NativeAgentConnection::handle_thread_events` to `AcpThread::update_plan`.
5. Preserve the current completed-plan snapshot behavior.

The center timeline subscribes only to this normalized `AcpThread` state, so native and external agents render identically. Do not infer progress from message text or tool names.

### Agent output

“Actual token output” means the exact provider-visible text deltas and structured tool input/output received by Zeddy, in arrival order. Current provider APIs do not expose tokenizer token IDs, so the UI must not claim that token IDs or hidden reasoning are available.

Add a bounded, in-memory `AgentOutputEvent` sequence at the normalized `AcpThread` update boundary before `StreamingTextBuffer` coalesces deltas. Each event contains a monotonic sequence number, event kind, visible payload, session ID, and timestamp. It must:

- preserve arrival order and repeated chunks;
- include message/thought deltas that the provider makes visible, tool input/output, plan updates, status transitions, and aggregate usage updates;
- reuse the repository's existing sensitive-data and tool-output presentation rules;
- have an explicit memory/entry cap with a visible truncation marker;
- never persist by default;
- emit one lightweight `OutputEventAppended` notification;
- avoid rebuilding a concatenated transcript on every event.

`AgentOutputView` is lazy. It does not subscribe or lay out rows until its workbench tab is first opened, uses a viewport-bounded list, and offers Follow, Wrap, Copy, filter, and Clear controls. `Tests` is selected in the canonical mockup fixture; production restores the last non-output workbench tab and falls back to `Terminal`, never `Agent output`.

## File-level implementation plan

### 1. Lock the contract and dependency baseline

Files:

- `docs/mockups/zeddy-agent-workspace-v3.png`
- `docs/plans/zeddy-ui-replacement.md`
- all `Cargo.toml` files and `Cargo.lock` as protected inputs

Work:

- Record the baseline commit, manifest count, and lockfile digest shown above.
- Turn every row in the acceptance map into a tracked implementation issue/test case.
- Capture current keyboard/action reachability for editor splits, project actions, all dock panels, Git, debug, collaboration, outline, terminal, tasks, diagnostics, notifications, settings, and agent sessions.
- Add a CI/review check that fails on any manifest or lockfile change. The authoritative guard is `script/check-zeddy-dependency-baseline`; it compares every stack branch with the fixed baseline rather than its immediate PR base.
- Keep the canonical task, plan, subagent, file, and review strings in `#[cfg(test)]` or visual-test fixtures only. Production views must consume live workspace and agent state.

Exit gate:

- The acceptance checklist has an owner and test for every visible mockup region and every preserved legacy capability.
- The dependency-invariant commands at the end of this document are clean.

### 2. Normalize agent state before moving UI

Files:

- `crates/acp_thread/src/acp_thread.rs`
- `crates/agent/src/thread.rs`
- `crates/agent/src/agent.rs`
- the existing native tool registry under `crates/agent/src/tools/`
- `crates/agent_ui/src/conversation_view.rs`
- `crates/agent_ui/src/conversation_view/thread_view.rs`

Work:

- Add the native plan update path and converge it on `AcpThread::update_plan`.
- Add the bounded output-event sequence and notification.
- Add public read-only `AgentWorkspaceSnapshot` and `SubagentSnapshot` APIs in `agent_ui`.
- Subscribe to root and child `AcpThreadEvent`s and update only the affected snapshot row.
- Keep the existing `ConversationView::navigate_to_thread` path for drill-down.
- Retain existing subagent live preview, stop, permissions, error, and session-resumption behavior behind reusable actions.

Tests:

- Native and external ACP plans produce the same normalized state.
- Plan transitions, completed snapshots, invalid input, and session reloads are correct.
- Subagent states cover complete, working, queued, failed, canceled, resumed, nested, and unloadable external sessions.
- Output events preserve exact arrival order, cap memory, insert one truncation marker, and are not persisted.

Exit gate:

- The new views can be driven entirely by public `agent_ui` state without inspecting private thread maps or parsing rendered text.

### 3. Extract reusable agent-first components

Files:

- `crates/agent_ui/src/agent_ui.rs`
- `crates/agent_ui/src/conversation_view/thread_view.rs`
- `crates/agent_ui/src/agent_diff.rs`
- `crates/agent_ui/src/message_editor.rs` and related existing composer modules
- focused new modules inside `crates/agent_ui/src/` only when a component is a distinct logical unit

Work:

- Extract `SubagentSection` from the existing subagent card and expanded-content rendering.
- Extract `AgentPlanView` from `render_plan_summary`/`render_plan_entries`; render exactly one active plan surface.
- Extract `ReadyToReviewCard` from the existing edits summary.
- Export an `AgentPromptComposer` that reuses the current message editor, model, context, permission, and submit paths.
- Add the lazy `AgentOutputView` over the normalized output-event sequence.
- Keep `ThreadView` as the full conversation/activity destination when a subagent row is clicked.

Action mapping:

- `Run` submits through the active conversation's existing send path.
- `Review` and `Review diff` dispatch `OpenAgentDiff`/`AgentDiffPane::deploy_in_workspace`.
- `Apply` dispatches existing `KeepAll`. Its tooltip and accessibility label must explain that edits already exist in buffers and the action accepts them.
- Reject remains available in the review surface/overflow even though it is not shown on the compact mockup card.

Exit gate:

- Component tests render the canonical three-step plan, three subagent states, two-file review card, and hidden Agent Output tab without an outer Zeddy shell.

### 4. Add the shell seam without importing feature crates into `workspace`

Files:

- `crates/workspace/src/workspace.rs`
- `crates/workspace/src/multi_workspace.rs`
- `crates/workspace/src/dock.rs`
- `crates/workspace/src/persistence/model.rs`
- `crates/workspace/src/persistence.rs`

Work:

- Add the type-erased Zeddy shell slots/delegate.
- Render the title bar full-width, then left/agent/editor columns, then the status bar.
- Preserve `PaneGroup` as the only center editor owner and preserve all file-open routing to `last_active_center_pane`.
- Add claimed-panel bookkeeping so registered/rehosted entities render once.
- Preserve modal, toast, notification, drag, zoom, resize, and client-decoration overlay order.
- Replace focus-region order with: title bar → subagents → project → agent workspace → workbench → editor → status bar, while keeping direct action-based focus for every panel.
- Make splitters keyboard reachable and persist widths/heights through existing workspace persistence, not a new storage system.

Suggested geometry at the canonical size:

- Left navigator: approximately 340 px, resizable.
- Agent column: approximately 535 px, resizable.
- Editor: consumes the remainder and honors the existing minimum pane width.
- Workbench: lower portion of the agent column, vertically resizable.

Responsive rules:

- At wide sizes, all three columns remain visible.
- When minimum widths cannot fit, collapse the left navigator to an overlay opened by a command/keybinding; never replace it with an icon rail.
- At the next threshold, let the agent column become an overlay while keeping the editor usable.
- Preserve state and focus when a region moves between inline and overlay presentation.

Tests:

- Editor splits, preview tabs, pane zoom, and file-open targets remain unchanged.
- Claimed panels never render twice, including restored sessions.
- All focus regions and splitters are reachable by keyboard.
- Resize tests cover 1280 × 800, 1586 × 992, and 1920 × 1200.

Exit gate:

- A test shell can surround a real editor without any dependency from `workspace` to `agent_ui`, `project_panel`, or other feature crates.

### 5. Rehost the project tree and construct the left navigator

Files:

- `crates/project_panel/src/project_panel.rs`
- `crates/project_panel/src/project_panel_settings.rs` only if existing presentation settings need an internal variant
- `crates/zed/src/zed.rs`

Work:

- Add an embedded/tree-only presentation mode to `ProjectPanel` that suppresses legacy dock chrome while keeping its filter, tree, context menus, drag/drop, selection, autoreveal, and file operations.
- Keep the same loaded `Entity<ProjectPanel>` registered in `Workspace`.
- In `zed::initialize_workspace`/`initialize_panels`, stack `SubagentSection` over the embedded project panel and inject the combined view into the left shell slot.
- Show a loading placeholder until asynchronous panel construction completes.
- Call `finish_dock_restoration` only after all existing panel-load tasks finish.
- Map the old sidebar toggle/focus actions to the new navigator. Old persisted visibility must not resurrect the old sidebar.

Tests:

- Expand/collapse, filter, select, open, preview, rename, create, delete, drag, autoreveal, and keyboard navigation still work in embedded mode.
- Clicking a subagent navigates to that exact session and back without changing project selection.
- Root rows remain stable as status events arrive.
- Panel restoration never mounts two project trees.

Exit gate:

- The old sidebar presentation is absent and the left side contains only Subagents and Project as shown in the mockup.

### 6. Recompose the title bar and center agent workspace

Files:

- `crates/title_bar/src/title_bar.rs`
- `crates/platform_title_bar/src/platform_title_bar.rs` only if a generic slot needs support
- `crates/zed/src/zed.rs`
- the extracted `agent_ui` components

Work:

- Preserve platform drag regions and native minimize/maximize/close behavior.
- Preserve repository/branch subscriptions, collaboration indicators, updates, notifications, settings, and existing shortcuts.
- Add type-erased center/action slots to `TitleBar`; build their concrete agent contents in `zed`.
- Route the global prompt to the active root thread. If a subagent activity view is selected, submitting still targets the root unless the UI explicitly says otherwise.
- Compose the center task card, exactly one plan timeline, and linked review card from `AgentWorkspaceSnapshot`.
- Ensure permission prompts, authentication errors, rate-limit/errors, cancellation, and model-selection states remain visible and actionable.

Tests:

- Prompt submission, Run dropdown, Review, window drag, and system controls work on each platform configuration.
- Native and external agent plans update the same three-row component.
- A two-file linked subagent edit produces one root review card; Review diff and Apply use existing actions.
- No task-progress/`Plan / Context / Changes / Tests` duplicate remains in the left region or center.

Exit gate:

- The top and center regions match the mockup hierarchy while preserving the complete agent control path.

### 7. Build the lower workbench and compact status bar

Files:

- `crates/zed/src/zed.rs`
- `crates/workspace/src/status_bar.rs`
- existing production views in `crates/terminal_view/`, `crates/tasks_ui/`, and `crates/diagnostics/`
- extracted `AgentOutputView` in `crates/agent_ui/`

Work:

- Create a type-erased workbench tab host with Terminal, Problems, Tests, and Agent output.
- Rehost existing entities/models; do not create a second terminal, diagnostics database, task runner, or test result model.
- Preserve each tab entity and its focus when switching.
- Put Git, Debug, Outline, Collaboration, and any other legacy panels in the workbench overflow and keep their existing actions/shortcuts working.
- Use `StatusBar`'s item registry. Reorder existing branch/language/diagnostic items and add agent-count, token-usage, and local-context items.
- Keep status information actionable and provide overflow behavior at narrow widths.

Tests:

- Tab switching preserves terminal sessions, problem selection, test rows, raw output position, and focus.
- Agent output is never the first/default tab and performs no row layout before first open.
- Large output and test lists remain viewport bounded.
- Every rehosted legacy panel remains command- and shortcut-reachable.

Exit gate:

- The mockup's lower/status surfaces are present and the feature-parity matrix is green.

### 8. Persistence, accessibility, and interaction hardening

Files:

- `crates/workspace/src/persistence/model.rs`
- `crates/workspace/src/persistence.rs`
- `crates/workspace/src/workspace.rs`
- `crates/workspace/src/multi_workspace.rs`
- affected component renderers

Persist:

- left and agent column widths;
- workbench height and active non-output tab;
- expanded subagent session IDs;
- navigator/agent overlay visibility at narrow sizes;
- last focused region;
- Agent Output Follow/Wrap/filter preferences, but not its event contents.

Migration:

- Preserve pane groups, open buffers, editor splits, active pane, docks, and panel entities from existing sessions.
- Translate old sidebar visibility into Zeddy navigator visibility.
- Translate old agent/classic panel-layout actions to focus/show the relevant Zeddy region; they must never restore legacy chrome.
- Ignore obsolete visual placement after migration while retaining compatible serialized fields where removal would break old data.

Accessibility:

- Give every interactive row a stable unique ID, role, label, and keyboard action.
- Use `aria_selected`/`aria_expanded`; never communicate state by color alone.
- Announce meaningful status/plan/review changes without announcing every streamed output delta.
- Keep virtualized/bounded accessibility children for Agent Output and long test/project lists.
- Restore focus deterministically after navigation, overlay dismissal, tab switching, Apply, or Reject.

Exit gate:

- Old sessions open without losing editors or panels; keyboard-only and VoiceOver passes cover all new regions.

### 9. Perform the single-shell cutover

Files:

- `crates/zed/src/zed.rs`
- `crates/workspace/src/multi_workspace.rs`
- `crates/title_bar/src/title_bar.rs`
- `crates/sidebar/src/sidebar.rs` callers and actions
- layout settings/action adapters in `agent_settings` and title-bar code
- default keymaps only where an existing action must be remapped

Work:

- Install the Zeddy shell for every normal workspace.
- Remove production rendering and construction of the old sidebar/activity rail and the duplicate planning navigation.
- Remove user-facing classic/agentic shell switching. Map compatible old actions/settings to Zeddy regions so old keymaps and serialized settings do not fail.
- Keep the `sidebar` dependency and every other dependency unchanged, even if a legacy crate no longer supplies visible chrome.
- Search for stale render entry points, labels, test fixtures, screenshots, help text, and settings descriptions.
- Do not delete underlying panels or actions merely because their old chrome is gone.

Exit gate:

- There is no reachable production path to the old shell, no VS Code-style activity rail, and no duplicate plan UI.
- Every parity item below passes through the Zeddy shell.

## Feature-parity checklist

The implementation is not complete until all items work in the replacement shell:

- Open, edit, save, close, preview, split, move, zoom, and restore editor panes.
- Project filter, tree navigation, create, rename, delete, drag/drop, context menus, and autoreveal.
- Search, command palette, language server actions, diagnostics, code actions, formatting, completion, and edit prediction.
- Git status, blame, branch, diff, merge-conflict, and review workflows.
- Terminal creation, persistence, focus, split, close, and task execution.
- Problems, tests/tasks, debug, outline, collaboration, remote, and extension-provided panels.
- Notifications, toasts, modals, menus, settings, updates, account/collaboration state, and native window controls.
- Agent new/resume history, prompt submission, model/context/permission controls, cancellation, errors, tool calls, queued messages, subagents, plan, review, token usage, and output.
- Existing action names and default keybindings, except where an obsolete visual-layout action is intentionally mapped to the equivalent Zeddy region.
- Workspace/session restore, multi-workspace routing, remote projects, window zoom, and narrow-window behavior.

## Verification plan

### Focused regression tests

```sh
cargo test --locked -p workspace test_toggle_docks_and_panels -- --nocapture
cargo test --locked -p workspace test_panel_activation_and_region_navigation_focus_activation_handle -- --nocapture
cargo test --locked -p workspace test_status_bar_visibility -- --nocapture
cargo test --locked -p project_panel test_visible_list -- --nocapture
cargo test --locked -p project_panel test_opening_file -- --nocapture
cargo test --locked -p agent_ui test_completed_plan_snapshot_keeps_list_state_in_sync -- --nocapture
cargo test --locked -p agent_ui test_conversation_subagent_scoped_pending_tool_call -- --nocapture
cargo test --locked -p agent_ui test_single_file_review_diff -- --nocapture
cargo test --locked -p agent test_subagent_tool_call_end_to_end -- --nocapture
cargo test --locked -p acp_thread test_usage_update_populates_token_usage_and_cost -- --nocapture
cargo test --locked -p terminal_view test_terminal_panel_starts_open_follows_setting -- --nocapture
```

New GPUI tests should use stable element IDs and `VisualTestContext` interaction helpers. Scheduler-dependent timers must use the GPUI executor. Reproduce nondeterministic failures with `SEED`, `ITERATIONS`, and `PENDING_TRACES`.

### Affected-package and repository gates

```sh
cargo nextest run --locked \
  -p workspace -p project_panel -p agent_ui -p acp_thread -p agent \
  -p editor -p terminal_view -p title_bar -p tasks_ui \
  --no-fail-fast --no-tests=warn

./script/clippy \
  -p workspace -p project_panel -p agent_ui -p acp_thread \
  -p editor -p terminal_view -p title_bar -p tasks_ui -p zed

cargo build --locked -p zed
cargo nextest run --locked --workspace --no-fail-fast --no-tests=warn
cargo test --locked --workspace --doc --no-fail-fast
```

### Visual QA

Add deterministic fixtures for:

- the canonical 1586 × 992 mockup state;
- collapsed and expanded subagents;
- working, complete, queued, failed, and canceled states;
- no edits, ready-to-review, and open diff states;
- Tests, Terminal, Problems, and Agent output tabs;
- 1280 × 800 and 1920 × 1200 responsive states;
- light theme, high-contrast/theme extremes, long repository/file names, and localization expansion.

Run the macOS Metal visual runner:

```sh
UPDATE_BASELINE=1 VISUAL_TEST_OUTPUT_DIR=target/zeddy-visual \
  cargo run --locked -p zed --bin zed_visual_test_runner --features visual-tests

VISUAL_TEST_OUTPUT_DIR=target/zeddy-visual \
  cargo run --locked -p zed --bin zed_visual_test_runner --features visual-tests
```

The runner's local baseline comparison is not sufficient by itself because its fixture baselines are gitignored. Require a human side-by-side and overlay review against `docs/mockups/zeddy-agent-workspace-v3.png`, including intermediate loading/error states.

### Performance gate

Profile current Zed and the Zeddy candidate with identical small, medium, and severe sessions: many project files, several active subagents, sustained streaming, large Agent Output, large test results, editor splits, and an open diff.

```sh
cargo build --locked --profile release-fast -p zed

xcrun xctrace record \
  --template "Time Profiler" \
  --time-limit 60s \
  --output /tmp/zeddy-ui.trace \
  --launch -- target/release-fast/zed
```

Record p50/p95/p99/max foreground work and frame intervals, render invalidation count, memory growth during streaming, time to switch tabs, and task completion correctness. At 120 FPS, foreground frames have an 8.33 ms budget. Specifically reject:

- whole-shell invalidation for each output chunk;
- transcript concatenation on every render;
- layout/accessibility work for hidden Agent Output;
- unbounded non-virtualized output, test, subagent, or project rows;
- recreation of editor or diff entities on agent updates;
- resize feedback loops or continuously dirty inactive spinners.

### Accessibility/manual interaction gate

With accessibility active, use `dev: copy accessibility tree` or `dev: dump accessibility tree` and verify roles, labels, selected/expanded state, supported actions, row uniqueness, and focus order. Complete a keyboard-only pass for every acceptance-map region and verify that complete/working/queued/error states remain understandable without color.

### Dependency invariant

Run at every milestone and before merge:

```sh
./script/check-zeddy-dependency-baseline
```

The guard compares every protected file with fixed baseline commit `2551721adb5b5187bc27cfae0fbe47f0ed4c5397`, rejects tracked and untracked manifest changes, verifies the manifest fingerprint/count and `Cargo.lock` digest, and runs locked offline metadata. Comparing only with a stacked PR's immediate base is not sufficient. All supported cargo verification commands use `--locked`.

## Principal risks and mitigations

| Risk                                         | Mitigation                                                                                      |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Lower-level dependency cycle                 | Keep concrete composition in `zed`; pass only type-erased slots into `workspace`/`title_bar`    |
| Project or terminal entity renders twice     | Mark shell-claimed panels and assert one visual owner during restoration                        |
| File opens into the fixed agent dashboard    | Keep dashboard outside `PaneGroup`; retain `last_active_center_pane` routing tests              |
| Old persisted state resurrects legacy chrome | Version/migrate shell presentation while preserving pane/panel data; remap old layout actions   |
| Native plan remains empty                    | Implement an explicit native plan tool/event path, never heuristic message parsing              |
| Subagents vanish after completion            | Build rows from persisted parent tool-call/session links, not only `running_subagents`          |
| Resumed subagents duplicate                  | Deduplicate by session ID and use the latest invocation state                                   |
| “Token output” overpromises provider data    | Label provider-visible deltas accurately; do not imply tokenizer IDs or hidden reasoning        |
| Streaming makes the entire app jank          | Isolate notifications, keep output lazy and bounded, virtualize rows, profile production builds |
| Compact chrome hides existing features       | Maintain and test a capability-to-command/workbench-overflow parity matrix before cutover       |
| Apply semantics are misleading               | Explain that edits are live and map Apply to existing KeepAll acceptance semantics              |

## Definition of done

The redesign is complete only when:

1. Every normal workspace uses the Zeddy shell and the old shell is unreachable.
2. The left region contains Subagents over Project, with no activity rail and no duplicate planning navigation.
3. The center contains one live plan and one linked Ready to review surface.
4. Clicking a subagent opens its live activity, including terminal/error states.
5. Agent output is available, accurate to provider-visible events, bounded, lazy, and non-default.
6. The existing editor/pane tree and every feature-parity item work through the new composition.
7. Restored workspaces retain buffers, panes, panels, focus, and useful layout state without restoring old chrome.
8. Canonical, responsive, error/loading, accessibility, and keyboard QA pass.
9. A release-fast performance comparison shows no unacceptable frame, memory, or completion regression.
10. Focused tests, affected-package tests, full repository tests, clippy, build, visual review, and dependency checks pass.
11. All 262 manifests and `Cargo.lock` remain unchanged, with the recorded lockfile digest intact.

# Per-client selection (refs discussion #651)

Posted as a comment on https://github.com/ogulcancelik/herdr/discussions/651

---

Same need here: two Alacritty windows attached to the default session, want window A on workspace 2 and window B on workspace 1 at the same time. Today both mirror.

Dug into why. Selection is shared session state, so there is one selection for every attached client:

- `AppState.active: Option<usize>` (active workspace) — `src/app/state.rs:1274`
- `Workspace.active_tab` and `Workspace::focused_pane_id()`
- `render_virtual()` takes `&AppState` and draws that one active view, so every client gets the same frame modulo terminal size — `src/server/render_stream.rs:271`

So mirroring is structural, not a config gap. Independent windows need selection to become per-client.

The interesting part: `AGENTS.md` already classifies selection as client state, not shared runtime:

> Sidebar layout, token placement, colors, selection, modals, mouse/viewport state: TUI/client.

and the project is "migrating toward a server-owned runtime protocol with the TUI as one client." So this feature is the endpoint of a migration already in progress, not a new concept. Workspaces/tabs/panes stay shared session organization. Only the view into them (which ws/tab/pane each client looks at) moves to the client layer, where the guardrail says it belongs.

Sketch of the seam:

- A per-client `ClientView { active_workspace, active_tab_by_workspace, focused_pane_by_tab }`, living next to the existing per-client state in `ClientConnection` (`src/server/clients.rs`).
- `render(&AppState, &ClientView)` so "what is active" comes from the requesting client; pane contents and session structure stay shared. Render stays pure.
- Navigation/input mutates the issuing client's `ClientView` instead of the global selection.

Phaseable: Phase 1 introduces `ClientView` with a single default view, byte-identical for one client (pure refactor, testable without PTYs). Phase 2 routes input per client and turns on independence. Phase 3 makes focus-derived behavior per-client (cursor reveal, host focus reporting, active-tab notification suppression, hyperlink visibility).

Open questions that need a maintainer call before any code:

1. Scope of v1: per-client workspace only (smallest seam, just `AppState.active`), or full per-client tab and pane focus too?
2. `auto_switch` on agent state change: move every client, only the owning client, or none?
3. Is `ClientView` ephemeral per connection, or restored on reattach? Interaction with `resume_agents_on_restore`.

Happy to build Phase 1 as a no-behavior-change PR if the seam looks right to you. Mainly want your read on direction and the v1 scope before writing it.

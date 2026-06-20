# Claude GUI

When `claude` (Claude Code CLI) runs inside a terminal panel, a **Claude GUI** button appears in the bottom-right corner of that panel. Clicking it opens a GUI overlay that covers the panel and renders the claude conversation as bubbles — mirroring the cc2 (claude native TUI) layout exactly. When claude exits, the button and overlay are automatically removed.

Motto: **Port cc2 TUI behavior into a GUI as-is.** All claude domain logic is owned by this plugin; the core only provides a generic socket (decoupling — the core has no knowledge of claude).

## Three Data Planes

- **① Conversation history (JSONL)** — Incrementally parses `~/.claude/projects/<enc-cwd>/<sessionId>.jsonl` via offset tail + core `fs.watch` (no polling). Renders messages, tool calls, tool results, thinking, todos, system entries, and subagents as bubbles. Uses cc2 glyphs (`⏺` · `✻` · `∴` · `⎿`).
- **② Live stream (terminal buffer)** — In-progress responses are not yet in JSONL, so `app.terminal.readBuffer` reads the screen for live display. Replaced by structured JSONL bubbles on completion.
- **③ workflow/agent** — Scans `<sessionId>/subagents/agent-*.jsonl` + `*.meta.json` + `workflows/<runId>/` to build nested panels with accurate progress lines (tool count · tokens · last tool). No fake progress bars.

## Three-Layer Input Queue

GUI input is injected into the claude PTY. If claude is in a **dialog (`/status`, etc.) or thinking** state, naive injection causes lost messages or dialog misoperation. Three layers guard against this:

- **L1 PTY write** — Byte injection (attempt only).
- **L2 TUI buffer** — Confirms via `readBuffer` that the input actually appeared in claude's input field or queue.
- **L3 JSONL** — A `user` line appearing in the session JSONL is the **confirmed input signal = item removed from the GUI queue** (single removal point).

Modal gate (detects dialogs via `readBuffer` dismiss hints such as `Esc to cancel`) → injection held while modal is open. When the modal closes, the queue **drains automatically in FIFO order**. Four states (`held` → `injecting` → `awaiting` → removed), 1:1 dedup, double-injection blocked. Pending items survive overlay close/reopen.

## Commands

- `toggle [paneId]` — Show/hide the GUI (omit for the most recent claude panel)
- `open [paneId]` / `close [paneId]` — Open / close
- `send {text, [paneId]}` — Enqueue input and **immediately return queue state** (JSON). Returns `{ classify, queue:[{text,state,reason}] }`. Async — final "actual input (L3)" must be polled via `queue`. Auto-opens the GUI if not open.
- `focus [paneId]` — Move focus to the GUI overlay and **focus the input textarea**. Returns `{ focused }`.
- `type {text, [paneId]}` — **Actually type into the input field + Enter**. Unlike `send` (which enqueues directly), sets the textarea value and dispatches a real Enter keydown event, exercising the GUI's own input handler (ta.value → queue). Use for verifying the real GUI input path.
- `queue [paneId]` — Snapshot the current input queue (for progress polling)

```bash
sok plugin.soksak-plugin-agent-claude-gui.send '{"text":"hello"}'
# → { "classify":"modal", "queue":[{"text":"hello","state":"held","reason":"modal"}] }
sok plugin.soksak-plugin-agent-claude-gui.focus      # move to GUI input field
sok plugin.soksak-plugin-agent-claude-gui.type '{"text":"hello"}'   # type + Enter in input field
sok plugin.soksak-plugin-agent-claude-gui.queue
```

Queue chip states: `⧖ dialog wait` (front item blocked by dialog) · `⧖ queued (N)` (subsequent items) · `⤴ sending` (injecting) · `⏳ awaiting input` (waiting for L3 in claude queue). Chip disappears = actual input confirmed (L3).

## Permissions

`terminal` (claude detection) · `terminal:read` (② live stream · input readiness) · `terminal:write` (input injection) · `fs:read` (JSONL · session-env) · `ui:statusbar` · `ui:overlay:pane` · `commands`.

Without `terminal:read`, the input queue and live stream are disabled (falls back to legacy immediate injection).

## DOM Exposure (Structural Addresses)

Hosts access overlay elements via structural path addresses (`.../view/<pluginId.viewId>/node/<nodePath>`) rather than arbitrary CSS selectors — for clicking, measuring, and E2E testing. Only elements intended for exposure receive a `data-node` attribute, and their types are declared in `plugin.json` `contributes.nodes` (shown on the consent screen). Elements without `data-node` are inaccessible (explicit error).

Dynamic lists use `data-node="<kind>/<stable-key>"` — the stable key is a persistent identifier, not a counter index (messages use JSONL uuid; subagents use agent file id). Elements without a stable key do not receive a node (no fake indexes).

| Node | data-node | Description |
|------|-----------|-------------|
| input | `input` | claude input field (textarea). DOM target for `focus`/`type` commands. |
| send | `send` | Input send button (↩). |
| close | `close` | Overlay close button (✕). |
| verbose | `verbose` | Toggle thinking display/hide (∴). |
| session | `session` | Session id chip (click to copy full id). |
| msg | `msg/<uuid>` | User message row. Stable key = message entry uuid. |
| agent | `agent/<id>` | Subagent panel header (click to expand/collapse). Stable key = agent file id. |

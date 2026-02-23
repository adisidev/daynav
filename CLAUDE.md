# daynav

Personal day planner. Single-file web app at `index.html`.

## Philosophy

- **Single file, no build tools.** Everything lives in one `index.html` — HTML, CSS, JS. No frameworks, no dependencies, no npm. This is intentional. Keep it this way.
- **Keyboard-first.** The app is designed to be driven entirely from the keyboard. Mouse/click support exists but is secondary. Vim-inspired navigation (j/k to move, h/l to reorder).
- **Command palette as the primary interface.** All actions can be performed by typing commands. Unrecognized input is parsed as an implicit `add <name> <duration>`.
- **Minimal, clean UI.** Dark theme, monospace font, no emojis, no icons beyond the checkmark. Information density over decoration.
- **localStorage only.** No server, no database, no accounts. State persists in the browser. This is a personal tool, not a SaaS product.
- **Actual time tracking over estimates.** When you start a task (go) and complete it (space), the app records actual start/end times and shows how you did against your estimate.

## Architecture

```
index.html
├── <style>   — All CSS, uses CSS custom properties (--bg, --surface, etc.)
├── <body>    — Command bar, summary bar, task list, help overlay
└── <script>  — All JS, no modules, globals are fine
```

### State Model

```js
{
  tasks: [{ id, name, durationMinutes, scheduledStart, scheduledEnd, completed }],
  buffer: 10,        // minutes between tasks, snaps to multiples of 5
  selectedIdx: -1,   // currently highlighted task (-1 = none)
  timer: {           // persisted timer state (survives refresh)
    taskId, elapsed, running, firstStarted
  }
}
```

State is saved to `localStorage` key `"daynav"` on every mutation via `saveState()`.

### Timer

In-memory timer variables: `timerRunning`, `timerStartedAt`, `timerElapsedBefore`, `timerTaskId`, `timerFirstStarted`, `timerInterval`.

- `doGo()` — starts timer on the first incomplete task. Records `timerFirstStarted` (used as actual start time on completion).
- `doPause()` — accumulates elapsed time into `timerElapsedBefore`, stops ticking.
- Timer survives pause/resume (elapsed accumulates) and page refresh (persisted in localStorage).
- Timer counts down from estimate. Goes negative (red) when overrun.

### Scheduling / Buffer Logic

`reschedule()` places all incomplete tasks sequentially starting from now (rounded up to next buffer boundary). Completed tasks are skipped — they keep their actual start/end times.

`nextBufferSlot(endMins, buf)` determines when the next task starts after one ends:
- If end time falls exactly on a buffer boundary → add one full buffer.
- If more than half the buffer remains in the current slot → round up to next boundary.
- If half or less remains → skip to the boundary after that.

Example with buffer=10: task ends at 1:42 → next starts 1:50. Ends at 1:40 → next starts 1:50 (full buffer). Ends at 1:46 → next starts 1:50 (4min < half of 10). Ends at 1:44 → next starts 2:00 (6min > half, so rounds to 1:50... wait no, 1:44 has 6min remaining to 1:50 which is > 5, so next boundary).

Buffer values are always multiples of 5 (input is rounded up via `Math.ceil(n/5)*5`).

### Task Completion

When a task is marked complete (space or click):
- `scheduledEnd` is set to now.
- `scheduledStart` is set to `timerFirstStarted` (when go was first pressed for that task).
- Remaining tasks are rescheduled from now.
- Actual time taken is shown next to the estimate (green if under/matched, red if over).

## Keyboard Shortcuts

When command bar is **not focused**:

| Key | Action |
|-----|--------|
| `/` or `Cmd+K` | Focus command bar |
| `c` / `a` | Focus command bar with "add " prefilled |
| `i` | Focus command bar (empty) |
| `b` | Focus command bar with "buffer " prefilled |
| `e` | Edit selected task (name + duration in command bar) |
| `Escape` | Close help overlay, or deselect task |
| `g` | Start timer (go) on current task |
| `p` | Pause timer |
| `r` | Reset timer on selected task |
| `R` | Reschedule all tasks from now |
| `j` / `J` | Select next task |
| `k` / `K` | Select previous task |
| `l` | Move selected task down |
| `h` | Move selected task up |
| `1-9` | Jump to task by number |
| `Shift+1-9` | Reposition selected task to that slot |
| `d` / `D` | Delete selected task |
| `Space` | Toggle task complete |
| `?` | Show/hide help overlay |

When command bar **is focused**:

| Key | Action |
|-----|--------|
| `Enter` | Execute command, blur command bar |
| `Escape` | Blur command bar |
| `J` / `K` | Blur and jump to first/last task |

## Commands

| Command | Example | Notes |
|---------|---------|-------|
| `add <name> <duration>` | `add Standup 15m` | Duration supports: `30m`, `2h`, `1.5h`, `1h30m` |
| `<name> <duration>` | `Standup 15m` | Implicit add (fallback for unrecognized commands) |
| `buffer <minutes>` | `buffer 10` | Rounds up to nearest multiple of 5 |
| `go` / `play` | | Start timer on current task |
| `pause` / `stop` | | Pause timer |
| `delete <N>` | `delete 3` | Also: `del`, `rm` |
| `rename <N> <name>` | `rename 2 New name` | |
| `edit <N> <name> <dur>` | `edit 2 New name 45m` | Change name and/or duration |
| `clear` | | Remove all tasks |
| `reschedule` / `rs` | | Reschedule from now |
| `?` | | Show help |

## Rendering

Three render functions called by `render()`:
- `renderSummary()` — task count, done count, remaining estimate, finished actual time, end time, buffer, inline timer with go/pause button.
- `renderTaskList()` — task items with index, checkbox, name, scheduled time range, estimate, and actual time (for completed tasks).
- Timer ticks via `setInterval` at 1s updating the summary.

## Deployment

- **GitHub Pages**: https://adisidev.github.io/daynav/
- **Caddy** (local): served at `adi.my/daynav/` via `handle_path /daynav/*` in `/Users/adi/github/diegest-cluster/Caddyfile`.
- **Landing page**: linked from `/Users/adi/github/diegest-cluster/www/index.html` under "apps" section.

## Conventions

- No frameworks, no build step. Keep it a single `index.html`.
- No emojis in the UI.
- Dark theme using CSS custom properties in `:root`.
- Monospace font stack: SF Mono, Fira Code, JetBrains Mono.
- Vim-style keybindings. Don't add bindings that conflict with existing ones.
- `saveState()` after every mutation. `render()` after every state change.
- Duration format in display: `45m`, `1h`, `1h30m`. Parsing accepts all of these plus `2hr`, `90min`, etc.

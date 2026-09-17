# Quick reference

Every tool takes an optional `computer` id (sticky switch). Every action
receipt is JSON: `ok`, plus what was sent. Verify effects by observing.

## Observe
- `request_access` — permissions + capabilities; call once per session.
- `list_apps {all?}` — running apps (names, pids). Default: user-facing apps.
- `list_windows {app_ref?}` — windows with indices for `window_id`.
- `get_app_state {app_ref?, query?, role?, limit?, detail?}` — elements +
  `state_id`. The targeting tree.
- `find_elements {state_id?, query?, role?}` — filter a cached observation.
- `wait_for {query|role, state, timeout?}` — poll until UI appears/disappears.
- `screenshot {app_ref?|region?|display?}` — raster for visual work.
- `zoom {region}` — magnify the last raster.
- `get_value {target}` — read an element's value.
- `cursor_position` — hardware pointer.
- `list_sessions` — who is driving this machine: live sessions with bound targets, modes, and held pointers.
- `clipboard {action:"read"}` — user clipboard text (ask before reading if unsure).

## Act
- `click {target, button?, clicks?}` — left (1–3 clicks), right, or middle.
- `type {text, target?, press_enter?}` — unicode-safe; verifies by read-back where possible.
- `key {text, repeat?|duration?}` — chords like `cmd+s`; `duration` holds the key.
- `set_value {target, value}` — semantic write with read-back.
- `select_text {target, text_range?}` · `focus {target}` · `perform_action {target, action}`
- `scroll {target, direction, amount?}` · `left_click_drag {from_target, to}`
- `invoke_menu {path}` — app menu items through accessibility (exact for app-level commands like New/Save/Quit; see the close recipe for windows).
- `pointer {action, target?}` — move/down/up primitives (foreground/shared only).

## Apps & computers
- `open_application {name|bundle_id|pid, activate?}` — bind the input target; `app_not_found` when the selector resolves nowhere.
- `kill_app {name|bundle_id|pid, force?}` — quit an app; refuses an ambiguous name match (pass pid); never the helper itself.
- `preview {enabled}` — floating panel: captured window + agent/user cursors; live while bound.
- `computer {action, id?}` — list / switch / register / remove.
- `recording {action, …}` — start / stop / status / list (opt-in screen recordings).

## Browser (CDP)
- `browser {action:"start", url?}` — self-owned Chromium profile + this session's tab; the user's own browser is never touched.
- `browser {action:"navigate", url}` — http(s) or about:blank; waits for load (`verified`).
- `browser {action:"click", selector | point}` — CSS selector's box center, or page-viewport pixels from a browser screenshot.
- `browser {action:"type", text, selector?, enter?}` — inserts text (unicode), optional focus selector and Enter.
- `browser {action:"screenshot", full?}` — page PNG (inline image); viewport is the click-point space.
- `browser {action:"status"}` · `browser {action:"stop"}` — tabs/active tab; close this session's tab (last one out closes the browser).

## Session
- `list_sessions` — live sessions on this machine (content-free) and the user's control mode.
- `stop_computer_control {reason?}` — kill switch; input for this session ends.

## Recipes

Type into the document body:
1. `open_application {name:"TextEdit", activate:false}`
2. `get_app_state {role:"AXTextArea"}` → index
3. `type {target:{type:"element",index}, text:"…"}`
4. `get_value {target:{type:"element",index}}` → confirm

Close a window without borrowing focus:
1. Press the window's close-button element (`click` on the window's
   `AXButton`), or use `key cmd+w` (which borrows focus briefly and says so:
   `front_lease` / `front_restored`).

Fill and submit a web form (CDP, no pixels):
1. `browser {action:"start", url:"https://…"}`
2. `browser {action:"type", selector:"#email", text:"…"}`
3. `browser {action:"click", selector:"button[type=submit]"}`
4. Verify with `browser {action:"status"}` (url/title) or a fresh
   `browser {action:"screenshot"}` — never assume the click landed.
2. `list_windows` → window gone

> `invoke_menu` is exact for app-level commands (New, Save, Quit).
> Window-targeted items like Close can validate against a key window that a
> background app does not have and legitimately no-op — verify the effect
> before retrying or reporting.

Move between apps mid-task: `open_application` retires the previous app's
indices; always `get_app_state` after switching.

> The per-action wire names (`left_click`, `read_clipboard`, `recording_start`,
> `computer_list`, `hold_key`, …) remain callable as aliases; `tools/list`
> advertises the merged set above.

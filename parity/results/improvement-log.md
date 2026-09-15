# Improvement log — Codewhale Computer Use vs kimi-cu

Append-only record of the overnight measure → fix → verify loop started 2026-09-15.
Reference: kimi-cu v0.5.4 (`/Applications/KimiCU.app`, plugin dir
`~/.kimi-code/plugins/managed/kimi-cu/`). Target: parity, then past it.

## Baseline (2026-09-15, v0.6.0, HEAD 1088197)

- `npm test`: 260 pass / 0 fail / 15 skipped (275 total). Green.
- kimi-cu tool surface (10 tools): list_apps, get_app_state, click, type_text,
  press_key, scroll, set_value, perform_secondary_action, select_text, drag.
  Codewhale surface is already a superset (adds wait_for, find_elements, focus,
  zoom, hold_key, double/triple/right/middle click, clipboard, recording,
  remote computers, batching, preview panel).
- kimi-cu binary strings extracted to a working file for behavior diffing;
  notable behaviors to match/hunt: AXScroll{Up,Down,Left,Right}ByPage actions,
  AXScrollToVisible, AXShowMenu/AXOpen/AXConfirm/AXCancel, file-panel handling
  ("native file panel AXOpen failed and cmd+down fallback did not enter the
  item", "go-to-folder return reported a navigation"), menu-bar handling
  ("menu-bar menu items of a background app cannot be committed"), honest
  AXValue refusal strings for Electron/Web, observer-based notifications
  (AXObserver + window/focus/value notifications), axFocusOnly snapshot mode.

## Round 1 — parity runner steals the operator's focus (fixed)

- **Gap (ours, harness-side):** every fixture launch activated the new Chrome
  window and took the user's typing focus. Reported live by the operator.
  Verified experimentally: both direct `spawn` of the Chrome binary and
  `open -g -na` leave Chrome frontmost (Chrome self-activates on launch).
- **Fix:** `parity/darwin-probe.m` gains `--restore-focus <pid>` (polite
  `NSRunningApplication activateWithOptions:0`, no TCC needed);
  `scripts/lib/desktop-darwin.mjs` captures the frontmost pid before any
  fixture launch and restores it once the fixture reports ready.
- **Verify:** compiled probe + launched a throwaway Chrome `--app` instance:
  frontmost before `1024 Terminal`, after launch+restore `1024 Terminal`.
- Remaining: rerun full suite to confirm foreground-change receipts stay clean.

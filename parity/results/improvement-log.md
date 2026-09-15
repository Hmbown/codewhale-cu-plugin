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

## Round 2 — window-routed background pointer (the kimi-cu crown jewel, replicated and verified)

- **Gap:** background drag, clicks on non-pressable points, triple-click and
  wheel scrolling where no AX scrollbar exists all refused with
  `shared_pointer_required`. kimi-cu delivers these cursor-free via private
  SkyLight APIs.
- **Reverse engineering (receipts in /tmp/cu-exp experiments):**
  - `CGEventPostToPid` mouse events never reach AppKit (confirmed by fixture
    event monitor); HID-tap posts deliver but move the real cursor
    (mode 2: drag landed, cursor followed — unacceptable).
  - `CGEventSetWindowLocation` + `SLEventPostToPid` + signed
    `SLSEventAuthenticationMessage` (+ primer) delivered nothing.
  - Disassembly of kimi-cu (`BackgroundInput`, `SkyLight`, `SignedKeyboard`
    Swift enums) revealed the real recipe: lease the front process
    (`SLPSSetFrontProcessWithOptions(&psn,0,0x400)`, no window raise), post a
    hand-built 0xf8-byte window-focus record (`[0x24]=0xf8,[0x28]=0x0d,
    window id big-endian at 0x5c, [0xaa]=1`) via
    `SLPSPostEventRecordTo(&psn, rec)`, then post each CGEvent **as its raw
    event record** (pointer at `event+0x18`) through the same channel with
    window id fields `0x33/0x5b/0x5c`, subtype 3, and a window-space location.
    Restore front afterwards.
  - winloc space solved empirically: `winloc = point - CGWindow frame origin`
    (frame includes the titlebar; sweep matched AppKit
    `locationInWindow` to the canvas center).
- **kimi-cu ground truth found on the way:**
  - Its coordinate click delivers (down:2/up:2 — dual delivery), but its
    `drag` does NOT reach the fixture's custom canvas (same macOS 26.1):
    `ok:true` with zero delivered events. Its `bg-click` CLI also delivers
    nothing here. Its front restore leaves the target app frontmost.
  - Codewhale's record route delivers drags to the same canvas kimi-cu's
    drag cannot reach — this is past parity, not just at it.
- **Implementation:** new `bg_pointer` tool in
  `src/backends/darwin-accessibility.m` (SkyLight resolution with honest
  `bg_dispatch_unavailable` fallback, smallest-containing-window ownership
  guard, front lease + focus record + per-step record posts + `@finally`
  restore with AX re-assertion). `src/backends/darwin.mjs` routes background
  coordinate clicks, `left_click_drag`, and scrollbar-less `scroll` through
  it; receipts report `strategy:"window-record"`, `pointer_moved:false`,
  `front_lease:true`. `input_capabilities` advertises `window_record`.
- **Verify:** npm test 260/0/15. Live through repo `mcp/server.mjs` against
  the AppKit fixture: drag moved the square into the zone (`in_zone:true`,
  down/12 dragged/up, cursor position unchanged, frontmost back to
  Terminal); wheel scroll moved `scroll_top` 0→12 (dy=-3); kimi-cu
  comparison: its drag delivered nothing to the same canvas.
- **Honesty note:** the front lease is a momentary (tens of ms for clicks,
  gesture-length for drags) front-process swap with no window raise. A
  keystroke in exactly that window would go to the target app — reported in
  every receipt instead of hidden, matching the product's receipts rule.

## Round 3 — five targeted fixes (verified incrementally, parity pending)

- **stale_element oracle was fixture-stale (task fix).** The fixture redesign
  (premature-click hold) made `expect dyn==ready` impossible: nothing sends
  the premature click in this task. Updated the task to assert the real
  safety property — after `element_stale`, no click may land: dyn must stay
  `loading` through a 1 s dwell. Filtered run: 5/5.
- **AX-lag misclick (dynamic_content).** The Confirm click resolved to
  `AXFocused` on a container `AXGroup` because Chrome's AX tree lagged the
  DOM mutation. Focusing a group is not a click. `cuClickAction` now only
  takes `AXFocused` for explicit text-entry roles; everything else falls to
  the window-record real click. Web `AXMenuButton`/`AXPopUpButton` report
  unpressable for the same reason (AXPress does not open the native menu; a
  real mouse event does) — form_submit's `<select>`.
- **Astral-plane typing dropped under occlusion (unicode_type).** Measured:
  `CGEventPostToPid` key events keep BMP but lose surrogate-pair graphemes
  when the target window is not front-and-visible ("héllo wörld 日本 🐳" →
  "héllo wörld 日本 "). Same drop kimi-cu documents ("macOS defers the keys
  and nothing lands"); their answer is an opt-in visible raise. Ours is
  better: `type` now detects any multi-unit grapheme and routes the whole
  stream through the window-record channel (one front lease per type call).
  Live: `| 絵文字 🐳🎌 end` into the fully occluded fixture verifies true,
  delivery `window-record`, `front_lease:true`, cursor untouched.
- **Raising activation repaired (upload).** `axActivate` uses
  `SLPSSetFrontProcessWithOptions(&psn,0,0x200)` (kimi-cu's
  bringToFrontRaising recipe) with the AXFrontmost fallback.
- **Chrome fixture flags:** `--disable-background-timer-throttling`,
  `--disable-renderer-backgrounding`, `--disable-backgrounding-occluded-windows`
  so focus-restored fixtures don't clamp page timers.
- npm test stays 260/0/15 through each step.

## Round 4 — observation rebuild polling + activation runloop fix

- `app_info(activate:true)` reported `activation_not_confirmed` even when the
  activation worked: a one-shot helper never services its run loop, so
  NSWorkspace's frontmost view is seconds stale. The wait loop now pumps
  `[NSRunLoop runUntilDate:]`. browser.upload's activation step passes.
- Chromium tears down its accessibility tree across activation transitions
  (lazy renderer-a11y rebuild: 57 → 139 → 465 elements over ~3 s). Observes
  whose windows vend zero content now poll up to 8×300 ms rather than
  returning an empty UI; filtered observes (query/role) are exempt — an empty
  match is a legitimate answer there.

## Round 5 — the focus record is main, not key (swap-always)

- Chasing form_submit uncovered the real delivery rule on macOS 26.1: the
  hand-built window-focus record makes the target window *main* — events
  reach the process and are swallowed by first-mouse semantics (verified via
  a fixture NSEvent monitor: NSEvents arrive with correct window/location,
  views never fire). Only the front-process lease (NoWindows options) makes
  the window *key*, and only then do views receive the events. cuBgPointer
  and cuType now always take the lease; `front_lease:true` is reported.
- Menus opened under the lease close when it ends, so menu-opening clicks
  hold the lease across calls: state file + 6 s watchdog + restore at the
  next raw-input call, restoring only while the target is still frontmost
  (the user taking another app is never yanked back). Receipts say
  `menu_lease_held:true`.
- The no-swap recipe was also verified working when the window already held
  key status (reused/long-lived targets); it is documented in the helper
  comment but the lease is always taken for determinism.

## Round 6 — full form_submit flow verified live

- `<select>` popup: web `AXMenuButton`/`AXPopUpButton` reports unpressable
  (AXPress does not open the native menu), the record-route click opens it,
  the held lease keeps it observable, `AXPick` on "green" sets the value.
  Verified live end-to-end: color=green in the page oracle, frontmost
  restored to Terminal, cursor untouched.

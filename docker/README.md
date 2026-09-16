# The Linux box

A container that is a real, if headless, Linux desktop: Xvfb, a window manager,
the X11 command-line tools the backend shells out to, an AT-SPI accessibility
bus, and the parity suite's two fixtures (Chromium and Tk). It exists so the
Linux backend can be exercised from any machine with Docker, instead of only
from someone's Ubuntu install.

```sh
docker/run.sh            # npm test
docker/run.sh parity     # npm run parity -- --isolated
docker/run.sh smoke      # npm run smoke
docker/run.sh shell      # a shell on the X session
docker/run.sh -- npm run parity -- --isolated --task 'native.*' --repeats 5
```

Receipts land in `receipts/linux-docker/<timestamp>/` on the host. The source is
copied into the image rather than mounted, so every run reflects a build; the
dependency layer is cached, so edit-and-re-run costs a few seconds.

## What a green run here is, and is not, evidence for

It qualifies the X11 code paths and the isolated parity route on a live X
server with a live accessibility bus. Both are real: the backend runs the same
`xdotool`, `wmctrl`, `scrot` and `pyatspi` calls it runs anywhere.

It is not evidence for **Wayland** (no driver exists), for a **real login
session** — `native.modal_dialog` fails under KWin and passes here, so a
desktop's window manager changes the answer — or for anything the **macOS app**
holds, which is where permissions, human controls and background input live.
It also cannot speak to hardware input, multi-monitor or scaled displays.

Interference is measured, not assumed away. The entrypoint runs a second Xvfb
on `:0` as the "host" desktop, because the isolated route's claim is that it
leaves the host untouched — and a probe against a display that does not exist
reports zeroes whether or not anything moved.

## Two things the container gets to decide

**The browser is Chromium.** There is no arm64 Google Chrome for Linux, and
most distributions ship only Chromium. The driver reads the browser's
accessibility name from `--version` and the tasks target it by that name, so
both browsers work; the receipt records which one ran.

**Chromium's sandbox is off** (`/etc/chromium.d/00-container`). It needs
privileges the container does not have. This affects the fixture, not the code
under test.

## Known non-green rows

`docker/run.sh parity` at `--repeats 1` on 2026-09-16 (Chromium 152, Debian 12,
node 24.21, Xvfb 1600x1200) was **20/27**. Every row below passed 5/5 in
`parity/results/linux-xvfb-isolated-2026-09-07.json`, so each is either a
regression since that receipt or a difference between this container and the
Ubuntu host that produced it — telling those apart is the open work.

| row | what happens | reads as |
|---|---|---|
| `browser.drag_drop`, `native.drag_square` | `input_owner_required` | **product.** Held-input gestures require a connected desktop helper; Linux has none, so no in-process Linux run can pass them. The 2026-09-07 receipt says "demonstrated" — receipt and code now disagree. |
| `native.unicode_type` | `Ünïcödé ✓` arrives as `ünïcödé ✓` | **unresolved, and the most interesting.** Case is lost on accented capitals; the keysym trace shows no `Udiaeresis`. Either an xdotool/keymap difference or a real bug in non-ASCII typing. `browser.unicode_type` passes, so it is not simply broken. |
| `browser.upload` | no file chooser appears | **probably the container.** The task drives Chromium's GTK file dialog, which wants a desktop portal none is installed. |
| `native.modal_dialog` | dialog stays open | **probably the window manager.** `docs/LIMITATIONS.md` already records this failing under KWin and passing isolated; under openbox here it fails isolated too. |
| `browser.stale_element` | `dyn` stuck at `loading` | **timing.** The 3s expect window; `browser.form_submit` has been seen both green and red on the same image. Use `--repeats 5`. |
| `control.permission_denied` | error text lacks `DISPLAY` | **unresolved.** Accessibility still probes `ok` here, so the run takes a different branch than a machine with no session at all. |

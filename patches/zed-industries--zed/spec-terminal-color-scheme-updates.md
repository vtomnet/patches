# Spec: Terminal light/dark color-scheme update notifications (DEC mode 2031)

## Goal

Make Zed's integrated terminal participate in the de-facto "Color Scheme Updates"
protocol so TUIs (opencode, pi with `~/.pi/agent/extensions/terminal-theme.ts`,
nvim, etc.) can detect Zed's current light/dark appearance and react when the user
switches Zed's theme. Today these apps get no response and fall back to defaults.

This is a **host-side feature wiring** task. The terminal backend was recently
switched from `alacritty_terminal` to **libghostty-vt** (branch
`patched-libghostty-v1.6.3`, seam in `crates/terminal/src/ghostty.rs`). The new
backend *parses* DEC mode 2031 and exposes the query callback; alacritty did not,
which is why this was impossible before. But the backend cannot implement the
feature alone — it has no idea what Zed's appearance is or when it changes. That
plumbing is what this task adds.

## Background: why the backend swap alone didn't fix it

libghostty-vt is only the VT state machine. It tracks mode 2031 and will invoke an
`on_color_scheme` callback when it parses a `CSI ? 996 n` query, but:
1. No `on_color_scheme` callback is registered in the current seam, so queries are
   dropped.
2. The "push on change" half (mode 2031 subscription) is inherently host-driven —
   the host must write the notification to the PTY when the OS/Zed theme flips.
   Nothing in Zed does this.

Both halves are added here.

## Protocol reference (exact byte sequences)

- **Subscribe:** app sends `CSI ? 2031 h` (`\x1b[?2031h`). **Unsubscribe:** `CSI ? 2031 l`.
- **Query current scheme:** app sends `CSI ? 996 n` (`\x1b[?996n`).
- **Terminal reply / notification:** `CSI ? 997 ; 1 n` (`\x1b[?997;1n`) = **dark**,
  `CSI ? 997 ; 2 n` (`\x1b[?997;2n`) = **light**.
- While mode 2031 is set, the terminal emits the `997` notification on **every**
  appearance change. The `996` query returns the current scheme immediately and
  works regardless of whether 2031 is set.

## libghostty-vt API used (already a dependency)

From the `libghostty-vt` binding (crate already in the tree):
- `libghostty_vt::terminal::Mode::COLOR_SCHEME_REPORT` — the `2031` DEC private mode.
  Query with `term.mode(Mode::COLOR_SCHEME_REPORT) -> Result<bool>`.
- `libghostty_vt::terminal::ColorScheme` — `enum { Light, Dark }`.
- `Terminal::on_color_scheme(|term| -> Option<ColorScheme>)` — registers the callback
  fired when the parser sees `CSI ? 996 n`. Returning `Some(scheme)` makes libghostty
  format the `CSI ? 997 ; Ps n` reply and emit it through the **WRITE_PTY effect**.
  The seam already registers `on_pty_write` (which forwards WRITE_PTY output to the
  PTY via `TerminalBackendEvent::PtyWrite`), so the query reply path is complete once
  `on_color_scheme` is registered.

## Threading model / shared-state requirement

libghostty callbacks fire **synchronously inside `vt_write`, on the PTY reader
thread**, under the `FairMutex`. The `on_color_scheme` callback therefore cannot
touch GPUI (`cx`, themes). It must read the current appearance from a thread-safe,
lock-free value owned by the seam:

- Store `color_scheme: Arc<AtomicU8>` in `GhosttyTerm` (1 = dark, 2 = light).
- The `on_color_scheme` closure captures a clone of that `Arc` and reads it.
- The host (main thread) updates the atomic via a new seam function whenever Zed's
  appearance changes.

Keep all callback captures `Send` (the existing `unsafe impl Send for GhosttyTerm`
relies on this) — `Arc<AtomicU8>` is fine.

## Implementation

All line numbers are for branch `patched-libghostty-v1.6.3`; locate by symbol if they
have drifted.

### 1. `crates/terminal/src/ghostty.rs`

**a. Imports.** Add to the `libghostty_vt::...` use group:
`terminal::{ColorScheme}` (alongside the existing `Mode`), and
`std::sync::atomic::AtomicU8` (an `AtomicU32` is already imported for `CellPx`).

**b. `GhosttyTermConfig`** (~line 88): add a field
```rust
pub initial_is_dark: bool,
```
Set it in both constructors:
- `pty_term_config(...)` already receives `theme: &theme::Theme`; set
  `initial_is_dark: theme.appearance() == theme::Appearance::Dark`.
- `display_only_term_config(...)` has no theme; default `initial_is_dark: true`.

**c. `GhosttyTerm` struct** (~line 150): add
```rust
color_scheme: Arc<AtomicU8>,   // 1 = dark, 2 = light
```

**d. `new_term`** (~line 190, where `on_pty_write` / `on_bell` / `on_title_changed`
/ `on_size` are registered): create the atomic from config and register the callback.
```rust
let color_scheme = Arc::new(AtomicU8::new(if config.initial_is_dark { 1 } else { 2 }));
term.on_color_scheme({
    let color_scheme = color_scheme.clone();
    move |_term| Some(match color_scheme.load(Ordering::Relaxed) {
        2 => ColorScheme::Light,
        _ => ColorScheme::Dark,
    })
})
.ok();
```
Store `color_scheme` in the `GhosttyTerm { ... }` initializer.

**e. New seam function** (near `set_default_cursor_style` / `apply_config`):
```rust
/// Update the cached appearance used to answer `CSI ? 996 n` queries, and — if the
/// app has subscribed via DEC mode 2031 — return the `CSI ? 997 ; Ps n` notification
/// bytes the host should write to the PTY. Returns `None` if 2031 is not set.
pub(super) fn set_color_scheme(term: &mut GhosttyTerm, is_dark: bool) -> Option<&'static [u8]> {
    term.color_scheme.store(if is_dark { 1 } else { 2 }, Ordering::Relaxed);
    if term.term.mode(Mode::COLOR_SCHEME_REPORT).unwrap_or(false) {
        Some(if is_dark { b"\x1b[?997;1n" } else { b"\x1b[?997;2n" })
    } else {
        None
    }
}
```

### 2. `crates/terminal/src/terminal.rs`

**a. Import** `set_color_scheme` from the `crate::ghostty::{...}` use group.

**b. The `pty_term_config(scrolling_history, cursor_shape, theme.as_ref())` call**
(~line 1065) needs no change — `pty_term_config` now derives `initial_is_dark` from
the theme it already receives. Confirm the display-only path
(`display_only_term_config`) still compiles with the new field.

**c. Add a public method on `Terminal`** (model it on `set_cursor_shape`, ~line 1637):
```rust
pub fn set_color_scheme(&mut self, is_dark: bool) {
    if let Some(seq) = set_color_scheme(&mut self.term.lock(), is_dark) {
        self.write_to_pty(seq);
    }
}
```
`write_to_pty` is a no-op for display-only terminals, so this is safe there.

> Optional, related: on appearance change the backend's default palette (set via
> `set_default_color_palette` in `apply_config`/`new_term`) is not refreshed. Visible
> ANSI colors already follow the theme because `terminal_element::convert_color` maps
> `NamedColor` → theme color at render time, so this only affects OSC 4 color *query*
> responses. Refreshing the palette in `set_color_scheme` (or a dedicated method) is a
> nice-to-have, not required for this feature.

### 3. `crates/terminal_view/src/terminal_view.rs`

This view already observes settings changes and has `cx.theme()` access.

**a. Track last appearance.** Add a field to `TerminalView`, e.g.
`last_appearance: Option<theme::Appearance>` (import `theme::Appearance`). Initialize
`None`.

**b. Detect changes from two sources** — theme-setting changes and OS appearance
flips (the latter matters when the user's theme mode is "system"):
- In `settings_changed` (~line 559), call a new helper `self.sync_color_scheme(cx)`.
- Add a window-appearance subscription to the `subscriptions` vec (~line 270, next to
  `cx.observe_global::<SettingsStore>(Self::settings_changed)`):
  `cx.observe_window_appearance(window, |this, _window, cx| this.sync_color_scheme(cx))`
  (confirm the exact `observe_window_appearance` signature — see
  `crates/gpui/src/app/context.rs:462` and `crates/gpui/src/window.rs:1782`; it may
  need to be registered where a `&Window` is in scope, e.g. in the view's `new`).

**c. The helper:**
```rust
fn sync_color_scheme(&mut self, cx: &mut Context<Self>) {
    let appearance = cx.theme().appearance();
    if self.last_appearance == Some(appearance) {
        return;
    }
    self.last_appearance = Some(appearance);
    let is_dark = appearance == theme::Appearance::Dark;
    self.terminal.update(cx, |terminal, _| terminal.set_color_scheme(is_dark));
}
```

`theme::Appearance` is `{ Light, Dark }` (`crates/theme/src/theme.rs:54`);
`Theme::appearance()` returns it (`theme.rs:258`); `Appearance: From<WindowAppearance>`
already exists for the OS-appearance case.

## Edge cases

- **Initial query before any change:** the atomic is seeded from the theme at
  `new_term`, so a `CSI ? 996 n` sent immediately on shell start is answered correctly
  without waiting for an appearance change.
- **No subscription:** `set_color_scheme` returns `None` when mode 2031 is unset, so no
  stray `997` bytes are injected into apps that never subscribed.
- **Redundant updates:** `sync_color_scheme` guards on `last_appearance` so identical
  settings changes don't spam notifications.
- **Display-only terminals** (agent panels, REPL output, task output): no PTY, so the
  push is a no-op; the query callback still answers harmlessly.
- **Ordering:** the `996` reply flows through the existing async `PtyWrite` event path
  (same as DA/DSR responses today); the `2031` push is written synchronously on the
  main thread. Both are correct for these semantics.

## Testing

Build/run (the seam needs zig 0.15.2 on PATH for the libghostty-vt-sys build script):
```
PATH="/opt/homebrew/opt/zig@0.15/bin:$PATH" cargo build -p zed
```

1. **Query path** — in a Zed terminal:
   ```sh
   printf '\e[?996n'; sleep 0.2; printf '\n'   # then inspect with a tool that shows the raw reply,
   ```
   or use a small Python/uv snippet that puts the tty in raw mode, writes `\x1b[?996n`,
   and prints the response bytes. Expect `ESC [ ? 997 ; 1 n` under a dark theme,
   `;2` under a light theme.
2. **Subscription path** — run a known-good consumer (opencode, `pi` with
   `terminal-theme.ts`, or nvim with an auto-dark-mode plugin). Switch Zed between a
   light and a dark theme (Cmd-K Cmd-T / theme selector) and confirm the TUI flips.
   Also test with theme mode = "system" and toggle macOS appearance.
3. **Negative** — confirm an app that never sends `2031 h` receives no `997` bytes when
   you switch themes (no spurious input).

## Acceptance criteria

- `CSI ? 996 n` is answered with `CSI ? 997 ; 1 n` (dark) / `; 2 n` (light) matching
  Zed's active theme appearance, including on first query at shell start.
- With `CSI ? 2031 h` active, every Zed appearance change (theme switch, and OS
  appearance change when theme = system) pushes the matching `CSI ? 997 ; Ps n`.
- No notification when 2031 is not set.
- opencode / pi / nvim visibly follow Zed light↔dark switches.
- `cargo build -p zed` clean; display-only terminals unaffected.

# Quix Browser

A single static HTML page that loads other sites in an `<iframe>` and gives those frames
the permissions a browser game actually needs.

**https://ep1764.github.io/Quix_browser/Quix_system.html**

No build step, no dependencies, no server, no backend. One file.

---

## What it does

Most "put a game in an iframe" pages fail in a way that looks like the site is blocked when
it isn't. The frame loads, and then:

- **the mouse won't lock**, so mouse-look shooters are unplayable
- **WASD does nothing**, because the parent page still holds keyboard focus
- **the game's fullscreen button is dead**, because the frame was never granted fullscreen
- **controllers are invisible**, because `gamepad` was never granted

Quix fixes all four. Every game frame is created with an explicit Permissions Policy
(`fullscreen`, `gamepad`, `autoplay`, `screen-wake-lock`, `clipboard-write`, and more),
gets a click-to-play focus shield that hands the keyboard to the game, and deliberately
carries **no `sandbox` attribute** — see below.

It also ships a curated library of ~34 games, real per-tab browsing (each tab owns its own
frame and its own history), light/dark theming, a settings panel, and keyboard shortcuts
that are physically incapable of stealing a game's keys.

## What it cannot do, and why

A site can refuse to be framed with `X-Frame-Options` or
`Content-Security-Policy: frame-ancestors`. **The browser enforces that refusal in the
network stack, before any JavaScript on this page runs.** A page cannot read, rewrite or
remove another site's response headers. There is no attribute, sandbox flag or trick that
changes it.

Anything that *does* get around it is either a browser extension (which can rewrite response
headers) or a server that re-fetches the page for you. A static HTML file is neither.

> **Why a Chrome App like Leaf could.** Chrome Apps had the `<webview>` tag, which is not a
> frame inside the app's document at all — it's a separate, out-of-process browser view.
> `X-Frame-Options` and `frame-ancestors` only govern *framing*, so they never applied to it.
> Extensions could also strip those headers outright with the blocking `webRequest` API.
> Neither capability is available to a normal web page, and Chrome Apps have since been retired.

For sites that refuse, **Open in a real tab** is one click and always works.

### It can't even detect the refusal

Measured in Chromium: from the embedding page, a cross-origin frame that loaded perfectly
and one that was blocked outright are **indistinguishable**.

| signal | loaded fine | refused |
| --- | --- | --- |
| `load` event | fires | fires |
| `contentWindow.location.href` | `SecurityError` | `SecurityError` |
| `contentDocument` | `null` | `null` |
| `contentWindow.length` | `0` | `0` |
| Resource Timing `transferSize` / `responseStatus` | `0` / `0` | `0` / `0` |

The only exception is a site sending `Timing-Allow-Origin`, which game sites generally don't.

So Quix doesn't pretend. It asserts a verdict only when one is provable (same-origin, or a
real body size in Resource Timing); otherwise it waits a few seconds and offers a dismissible
"still blank?" hint. Library badges read **untried** until you actually play — they record
what happened on your device, they never guess.

## Why there is no `sandbox` attribute

This is the most important line in the file. Measured in Chromium, calling
`requestPointerLock()` inside a frame gives:

| frame | result |
| --- | --- |
| no `sandbox` attribute | `NotAllowedError` — needs a user gesture, i.e. **permitted** |
| `sandbox="allow-scripts"` | `SecurityError` — **forbidden outright** |

That difference is every mouse-look shooter working versus being permanently unplayable.
A cross-origin frame is already isolated by the same-origin policy, so `sandbox` can only
take capabilities away — and the two tokens you'd have to add back for a game to function
(`allow-scripts` + `allow-same-origin`) cancel out the security benefit anyway.

## Keyboard

Every shortcut requires <kbd>Ctrl</kbd>/<kbd>Cmd</kbd> or <kbd>Alt</kbd>, and **all of them
are ignored while a game frame has focus** — so <kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd>,
arrows, <kbd>Space</kbd> and <kbd>F</kbd> always reach the game.

| | |
| --- | --- |
| New tab / close tab | <kbd>Ctrl</kbd><kbd>T</kbd> / <kbd>Ctrl</kbd><kbd>W</kbd> |
| Next / previous tab | <kbd>Ctrl</kbd><kbd>Tab</kbd> |
| Focus address bar | <kbd>Ctrl</kbd><kbd>L</kbd> |
| Reload tab | <kbd>Ctrl</kbd><kbd>R</kbd> |
| Back / forward | <kbd>Alt</kbd><kbd>←</kbd> / <kbd>Alt</kbd><kbd>→</kbd> |
| Game library | <kbd>Ctrl</kbd><kbd>Shift</kbd><kbd>H</kbd> |
| Settings | <kbd>Ctrl</kbd><kbd>,</kbd> |
| Release the game frame | <kbd>Esc</kbd> |

## Privacy

Everything is stored in this browser's `localStorage` and nothing is uploaded anywhere.
The microphone is off by default. There is no analytics, no tracking and no backend —
there is nowhere for data to go.

## Development

It's one file. Open `Quix_system.html` in a browser.

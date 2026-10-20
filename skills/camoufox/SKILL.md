---
name: camoufox
description: >-
  Complete working knowledge of Camoufox, the open-source anti-detect browser for AI
  agents and web scraping: official Python and TypeScript launchers, CLI, proxies +
  GeoIP localization, fingerprint rotation, human cursor, virtual displays, remote
  servers, and multi-identity context farms. Use when writing, reviewing, running or
  debugging Playwright automation with Camoufox, converting plain Playwright code to
  stealth mode, installing/fetching browser builds (camoufox fetch/set/list),
  choosing headless vs virtual displays, or diagnosing fingerprint leaks (canvas,
  WebGL, WebRTC, fonts, navigator). Also use for any question about Camoufox
  options, errors, warnings, config keys, or best practices.
license: MIT
---

# Camoufox

Open-source anti-detect browser built for AI agents and scraping. Camoufox is a
debloated Firefox (~200 MB, no telemetry, no CSS animations) that spoofs the fingerprint
at the **C++ implementation level** — no JavaScript injection to detect — and runs
Playwright through a patched, sandboxed **Juggler** protocol instead of CDP, so pages
never see automation code. Identities are drawn by
[fpgen](https://github.com/scrapfly/fingerprint-generator) from the real-world
distribution of devices and coherence-checked so every value (UA, screen, WebGL, fonts,
voices, timezone, locale, WebRTC IP) describes **one plausible machine**.

Canonical sources: [camoufox.com](https://camoufox.com) ·
[GitHub](https://github.com/daijro/camoufox) ·
[Python README](https://github.com/daijro/camoufox/blob/main/python/README.md) ·
[TypeScript README](https://github.com/daijro/camoufox/blob/main/typescript/README.md) ·
[AGENTS.md](https://github.com/daijro/camoufox/blob/main/AGENTS.md)

## Read next

| Need | Open |
|---|---|
| Every launch option, defaults, types, warnings | [references/launch-options.md](references/launch-options.md) |
| Build a Python script this session | [references/python-api.md](references/python-api.md) |
| Build a Node/TypeScript script this session | [references/typescript-api.md](references/typescript-api.md) |
| CLI (`fetch`, `set`, `list`, `server`, `gpu-setup`, …) | [references/cli.md](references/cli.md) |
| Raw `config` / `CAMOU_CONFIG` keys and per-context setters | [references/config-keys.md](references/config-keys.md) |
| Proxies, `geoip`, timezone/locale/IPv6 behavior | [references/proxy-and-geoip.md](references/proxy-and-geoip.md) |
| Errors, `LeakWarning`s, pitfalls, version ceilings | [references/errors-and-pitfalls.md](references/errors-and-pitfalls.md) |
| Copy-paste recipes for common tasks | [references/recipes.md](references/recipes.md) |

## Install

Python 3.10+ (Playwright < 1.63 pulled in automatically):

```bash
pip install -U "camoufox[geoip]"   # [geoip] strongly recommended with proxies
camoufox fetch                    # downloads the paired browser build
```

Node.js 22.15+ (`playwright-core` < 1.63 as peer dependency; GeoIP is built in):

```bash
npm install camoufox playwright-core
npx camoufox fetch
```

`fetch` installs the **one browser build the package release is paired with** — not the
latest. Every build declares the interface it speaks; the package refuses one it cannot
drive (and names the package upgrade needed). `camoufox set` can pin or switch (see
[references/cli.md](references/cli.md)). Without `fetch`, the first launch downloads it.

## Core pattern

Camoufox returns ordinary Playwright objects: change only the launch, keep all your
Playwright code.

```python
from camoufox.sync_api import Camoufox

with Camoufox(headless=True) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
```

```javascript
import { Camoufox } from "camoufox";

const browser = await Camoufox({ headless: true });
const page = await browser.newPage();
await page.goto("https://example.com");
await browser.close();
```

Async Python: `from camoufox.async_api import AsyncCamoufox` → `async with`. TypeScript:
every call is awaited. Persistent profile → yields a `BrowserContext` with
`persistent_context=True` + `user_data_dir`. Many identities in one browser →
`NewContext(browser, os=..., proxy=...)` (each context gets its own fingerprint).
Details: [references/python-api.md](references/python-api.md),
[references/typescript-api.md](references/typescript-api.md).

## Golden rules (violating these causes detectable leaks)

1. **Never set identity values by hand.** No `user_agent`, `headers.User-Agent`,
   `screen.*`, `window.*`, `timezone`, navigator overrides via `config`. The identity is
   drawn as a whole; one manual override leaves the rest describing a different machine.
   Use `os`, `screen`, `window`, `locale`, `geoip`. Manual keys emit `LeakWarning`.
2. **`proxy=` with `geoip=True`.** Otherwise timezone/locale are the identity's, not the
   exit IP's — the #1 fingerprint-vs-IP mismatch WAFs check. Details:
   [references/proxy-and-geoip.md](references/proxy-and-geoip.md).
3. **`page.evaluate` is isolated** (invisible to the page, safe to read the DOM) and
   **cannot modify the page or read page globals**. For DOM changes:
   `main_world_eval=True` + `page.evaluate("mw:...")` — main-world code is detectable,
   use only when necessary; cannot return element handles.
4. **Leave the viewport alone.** Spoofed window ⇒ pages default to `no_viewport=True`;
   a forced `viewport` resizes the window to a size the identity does not claim.
5. **`is_mobile` is ignored** (warning). Desktop Firefox identities only — some WAFs
   test SpiderMonkey, so it can never pass as Chromium.
6. **Canvas belongs to the browser, contexts follow.** On the host's OS the identity
   claims the host's GPU and canvas output stays stock-Firefox clean (no noise). An
   identity on another OS, or claiming another GPU, gets LibreWolf/Tor-style randomized
   canvas/WebGL readback (plus a `LeakWarning`). `NewContext` with an OS/GPU different
   from the browser's raises unless the browser was launched with `canvas_noise=True`.
   Full logic: [references/launch-options.md](references/launch-options.md).
7. **Avoid fingerprint-hostile toggles.** `block_images`, `block_webgl`, `disable_coop`,
   `ff_version`, `custom_fonts_only`, `fingerprint=` warn because each creates a known
   anomaly. Silence all leak warnings only with `i_know_what_im_doing=True`.
8. **Windows/macOS/Linux identities only** (`os` is lowercase; anything else →
   `InvalidOS`). Mobile/tablet profiles are not supported by design.

## Choosing the fingerprint strategy

- **Default**: fpgen draws a random real-world device; the OS mix mirrors the real
  traffic distribution (e.g. a small Linux share), and the claimed Firefox version
  equals the browser's own by default — rotating FF versions is itself detectable;
  rotate Camoufox builds instead.
- `fingerprint_preset=True` (or a dict): recorded real-device presets instead of a
  synthesized draw.
- `screen=Screen(max_width=1920, max_height=1080)` bounds the generated display; do not
  use fixed `window=(w,h)` in production.
- `webgl_config=(vendor, renderer)`: force a GPU pair fpgen recorded for Firefox on
  that `os`; anything else raises `ValueError`.
- Per-context: `NewContext(browser, os=..., ff_version=..., geolocation=...,
  webrtc_ip=..., proxy=...)`.

## Headless decision

| `headless=` | Meaning |
|---|---|
| `False` (default) | Normal window; needs a display. |
| `True` | Firefox headless; Camoufox patches the headless tells it knows about. |
| `"virtual"` | **Recommended for servers (Linux):** headed browser on a private Xvfb display (needs `Xvfb` + `setpriv`); display is 1x1 unless `CAMOUFOX_VIRTUAL_DISPLAY_SIZE` (e.g. `1920x1080x24`); renders in software. |
| `"virtual-gpu"` | Linux: private headless Xorg bound to one GPU (`gpu=` / `CAMOUFOX_GPU`); real GPU rendering; setup once via `camoufox gpu-setup <gpu> \| sudo sh`; fails loudly (`GpuSoftwareFallback`) when the card is not used. |

Debug any launch: `debug=True` prints the `CAMOU_CONFIG` sent to the browser;
`camoufox test <url>` opens a headed browser with the Playwright inspector.

## Verifying & debugging leaks

Check identities against test sites before trusting a setup: browserscan.net,
browserleaks.com, creepjs, audiofingerprint.openwpm.com, and the bundled
camoufox.com/tests/* pages. If a site flags Camoufox but also flags stock Firefox, it is
the IP or rate limiting — not the browser. Diagnosis flow and reporting:
[references/errors-and-pitfalls.md](references/errors-and-pitfalls.md).

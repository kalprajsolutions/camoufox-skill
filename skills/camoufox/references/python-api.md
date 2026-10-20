# Python API — full reference

Package: [`camoufox` on PyPI](https://pypi.org/project/camoufox/) (current line 0.5.x).
Requirements: Python **3.10+** (`<4.0`), `playwright<1.63`
(installed automatically). Extras: `[geoip]` (GeoIP All-in-One db for `geoip=`),
`[gui]` (PySide6 manager UI).

```bash
pip install -U "camoufox[geoip]"
camoufox fetch                  # or: python3 -m camoufox fetch
```

> Pre-0.5 releases shipped a different API (BrowserForge fingerprints,
> `browserforge.fingerprints.Screen`, `from browserforge.fingerprints import
> FingerprintGenerator`). Prefer upgrading; this file documents 0.5.x.

## Imports

```python
from camoufox.sync_api import Camoufox, NewBrowser, NewContext
from camoufox.async_api import AsyncCamoufox, AsyncNewBrowser, AsyncNewContext
from camoufox import launch_options, DefaultAddons          # e.g. DefaultAddons.UBO
from camoufox.fingerprints import Screen
from camoufox.server import launch_server
from camoufox.server import launch_server
```

`DefaultAddons.UBO` — the only bundled default addon (uBlock Origin).

## Launch — sync & async

```python
from camoufox.sync_api import Camoufox

with Camoufox(headless=True) as browser:     # context manager; yields Browser
    page = browser.new_page()
    page.goto("https://example.com")
```

```python
import asyncio
from camoufox.async_api import AsyncCamoufox

async def main():
    async with AsyncCamoufox(headless=True) as browser:
        page = await browser.new_page()
        await page.goto("https://example.com")

asyncio.run(main())
```

`Camoufox(**opts)` accepts every option in [launch-options.md](launch-options.md) plus
`persistent_context=True, user_data_dir=...`. With a persistent context the context
manager yields a **`BrowserContext`**:

```python
from camoufox.sync_api import Camoufox

with Camoufox(persistent_context=True, user_data_dir="profiles/alice") as context:
    page = context.new_page()
    page.goto("https://example.com")
```

## Proxy with a matching location

```python
from camoufox.sync_api import Camoufox

with Camoufox(
    proxy={"server": "http://proxy.example.com:8080", "username": "user", "password": "pass"},
    geoip=True,
) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
```

`geoip=True` looks up the exit IP through the proxy, then sets timezone, locale
(speaker-distribution language), geolocation (with permission granted) and the WebRTC
IP. Pass an IP string to skip the lookup. Without a proxy, IPv6 stays on only when the
machine's IPv6 geolocates where the IPv4 identity does; otherwise the browser behaves as
IPv4-only and prints a `LeakWarning`. See
[proxy-and-geoip.md](proxy-and-geoip.md).

## One browser, many identities

```python
from camoufox.sync_api import Camoufox, NewContext

with Camoufox(headless=True, canvas_noise=True) as browser:
    alice = NewContext(browser, os="windows")
    bob = NewContext(
        browser,
        os="macos",
        proxy={"server": "http://proxy.example.com:8080", "username": "user", "password": "pass"},
    )
    alice.new_page().goto("https://example.com")
    bob.new_page().goto("https://example.com")
```

Per-context identity = navigator, screen, WebGL, fonts, voices, audio seed, timezone,
WebRTC IP, applied via `add_init_script` before any page script runs.

| `NewContext` argument | Effect |
|---|---|
| `preset` | Fingerprint preset dict instead of an fpgen draw. |
| `os` | `"windows"` \| `"macos"` \| `"linux"` for this context. On a browser without canvas noise (host OS + host GPU), anything but the host's OS raises `ValueError`. |
| `ff_version` | Firefox major version to claim; defaults to the browser's own. |
| `webrtc_ip` | IPv4/IPv6 address WebRTC reports (per context). |
| `proxy` | Per-context proxy (Playwright format). Unless `webrtc_ip` **and** `timezone_id` are both given, both are looked up from the exit IP; lookup failure raises `InvalidIP`. |
| `geolocation` | `{"latitude": ..., "longitude": ...}`; also grants the `geolocation` permission. |
| anything else | Forwarded to Playwright's `browser.new_context()` and overrides the generated value (e.g. `viewport`, `timezone_id`, `user_agent` — only where it does not break identity coherence). |

`AsyncNewContext(browser, **kw)` is the async twin.

## Own Playwright instance

```python
from playwright.sync_api import sync_playwright
from camoufox import NewBrowser, launch_options

with sync_playwright() as playwright:
    browser = NewBrowser(playwright, os="linux", headless=True)
    browser.close()

    browser = playwright.firefox.launch(**launch_options(os="linux", headless=True))
    browser.close()
```

Prefer `NewBrowser()`: it also defaults new pages to `no_viewport` when the window size
is spoofed and uses the host's color scheme and motion-reduction settings. Also accepts
`persistent_context=True, user_data_dir=` and `from_options=` (a dict previously built
by `launch_options()`, used as-is — skips re-drawing the identity).

## Screen constraint

```python
from camoufox.sync_api import Camoufox
from camoufox.fingerprints import Screen

screen = Screen(max_width=1920, max_height=1080)
with Camoufox(screen=screen) as browser:
    ...
```

## Addons

```python
from camoufox.sync_api import Camoufox
from camoufox import DefaultAddons

with Camoufox(
    addons=["/path/to/extracted-addon", "/path/to/another"],
    exclude_addons=[DefaultAddons.UBO],
) as browser:
    ...
```

Extracted addons = folders containing a `manifest.json`. To load an `.xpi`: rename to
`.zip`, extract, pass the folder. uBlock Origin (with custom privacy filters) is
installed by default; exclude with `DefaultAddons.UBO`.

## Remote server

```python
from camoufox.server import launch_server

launch_server(
    headless=True,
    geoip=True,
    proxy={"server": "http://example.com:8080", "username": "username", "password": "password"},
    port=1234,
    ws_path="hello",
)   # blocks; prints "Websocket endpoint: ws://localhost:1234/hello"
```

CLI equivalent: `python -m camoufox server`. Connect from any Playwright client:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.firefox.connect("ws://localhost:1234/hello")
    page = browser.new_page()
```

Servers run **one browser instance** — fingerprints do not rotate per connection;
rotate the server between sessions at scale. Persistent contexts cannot be served.

## Request-level identity generation (advanced)

`camoufox.fingerprints.generate_context_fingerprint()` draws a coherent identity and
builds the per-context init script; the launchers call it for you. Prefer launchers
over hand-built init scripts. If you must drive the browser binary yourself: launch
with `executable_path`, then call the window setters
([config-keys.md](config-keys.md) — per-context setters) from `add_init_script`.

## Exceptions (module `camoufox`)

| Exception | Raised when |
|---|---|
| `InvalidOS` | `os` not in (`windows`, `macos`, `linux`). |
| `InvalidIP` | Public/exit-IP lookup fails, or `webrtc_ip` is not a valid IPv4/IPv6 address. |
| `InvalidProxy` | Proxy `server` string not parseable. |
| `UnknownIPLocation` | IP not in the GeoIP database → pass `locale` and set coordinates another way. |
| `NotInstalledGeoIPExtra` | `geoip=` used without the `[geoip]` extra. |
| `CannotFindXvfb`, `VirtualDisplayNotSupported` | `headless="virtual"` missing `Xvfb`/`setpriv` or not Linux. |
| `AmbiguousGpuSelection` | Several GPUs, no `gpu=` selector for `"virtual-gpu"`. |
| `GpuNotFound` / `NoGpuFound` | Named card does not exist / no GPUs. |
| `GpuSoftwareFallback` | Browser did not render on the named card (warns if `require_gpu=False`). |
| `GpuVtTakeover` | Active VT changed during Xorg startup; launch aborted & cleaned up. |
| `ValueError` | `browser=` with `executable_path` (or `CAMOUFOX_EXECUTABLE_PATH` set); `NewContext` OS mismatch on a no-noise browser; unrecorded `webgl_config` pair. |

`LeakWarning` (a warning subclass) flags potentially-detectable choices; see
[errors-and-pitfalls.md](errors-and-pitfalls.md).

## Debugging helpers

```python
with Camoufox(headless=True, debug=True) as browser: ...   # print CAMOU_CONFIG identity
```

```bash
camoufox version        # include this output in GitHub issues
camoufox test https://example.com   # headed browser + Playwright inspector
camoufox path           # data directory (default linux: ~/.cache/camoufox)
```

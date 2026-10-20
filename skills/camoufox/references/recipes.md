# Recipes — copy-paste starting points

All recipes use the current API (package line 0.5.x). Options full list:
[launch-options.md](launch-options.md).

## 1) Minimal scrape (sync)

```python
from camoufox.sync_api import Camoufox

with Camoufox(headless=True) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
```

## 2) Async crawl entry (asyncio)

```python
import asyncio
from camoufox.async_api import AsyncCamoufox

async def main():
    async with AsyncCamoufox(headless=True) as browser:
        page = await browser.new_page()
        await page.goto("https://example.com")
        html = await page.content()

asyncio.run(main())
```

## 3) Proxy + locality (the standard scraping launch)

```python
from camoufox.sync_api import Camoufox

PROXY = {"server": "http://gate.example.com:8080", "username": "u", "password": "p"}

with Camoufox(proxy=PROXY, geoip=True, headless=True) as browser:
    page = browser.new_page()
    page.goto("https://ipinfo.io")   # sanity: country/timezone match the proxy
```

## 4) Identity farm — one browser, N identities

```python
from camoufox.sync_api import Camoufox, NewContext

with Camoufox(headless=True, canvas_noise=True, os=["windows", "macos", "linux"]) as browser:
    for i, country_proxy in enumerate(PROXIES):
        ctx = NewContext(browser, proxy=country_proxy)   # geoip implicit per context
        try:
            page = ctx.new_page()
            work(page, i)
        finally:
            ctx.close()
```

Notes:

- `canvas_noise=True` is required when contexts claim an OS different from the host's
  (and recommended for farms anyway — the host's own GPU/CPU stays off most checks).
- Contexts each get their own UA, screen, WebGL, fonts, voices, audio seed, timezone,
  WebRTC IP. The **IP hygiene stays your job** — unique proxies per ctx, sticky
  sessions so one identity keeps one location.
- Per-context opts: `os`, `ff_version`, `webrtc_ip`, `proxy`, `geolocation`,
  `preset`, plus Playwright context overrides.

## 5) Persistent login / stateful session

```python
from camoufox.sync_api import Camoufox

with Camoufox(
    persistent_context=True,
    user_data_dir="profiles/alice",
    proxy=PROXY,
    geoip=True,
) as context:
    context.new_page().goto("https://example.com/login")
    # cookies/localStorage survive here across runs
```

Cookies + storage persist per profile. Use one profile ↔ one identity ↔ one proxy, so
the account's device fingerprint stays stable forever (rotating a logged-in account's
device is itself suspicious). For lighter state sharing, export/import cookies via
`context.cookies()` between plain contexts.

## 6) Anti-challenge flow (Turnstile-style checkbox)

```python
# disable_coop lets scripts click elements inside cross-origin iframes
with Camoufox(disable_coop=True, window=(1280, 720), humanize=True) as browser:
    page = browser.new_page()
    page.goto("https://nopecha.com/demo/cloudflare", timeout=60000)
    page.wait_for_load_state("domcontentloaded")
    page.wait_for_load_state("networkidle")
    page.wait_for_timeout(5000)
    page.mouse.click(210, 290)   # coordinates vary per page; locate & click the checkbox
```

`humanize=True` gives mouse moves recorded-human trajectories (Cursory), which
challenge widgets monitor. `window` is okay for demos; prefer generated sizes in prod.

## 7) Main-world DOM surgery (rare, risky)

```python
# Reading the DOM needs nothing special and cannot leak:
html = page.evaluate("document.documentElement.outerHTML")
h1_text = page.evaluate("document.querySelector('h1')?.innerText")

# Modifying the page's real DOM:
with Camoufox(main_world_eval=True) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
    page.evaluate("mw:document.querySelector('h1').remove()")   # main world
```

- Isolated world (default): invisible to the page; cannot modify the page or read
  page-defined globals.
- Main world (`mw:`): detectable by the page; no returning of element handles; returns
  JSON-serializable data (`page.evaluate("mw:{key: 'value'}")` works).

## 8) Battery/bandwidth saving (rate-limited sources)

```python
with Camoufox(headless=True, block_images=True, extra_...) as browser: ...
# block_images warns: some WAFs detect image-less browsing — accept or don't use.
```

Pair with `block_webrtc=True` when the site does not need WebRTC at all. Avoid
`block_webgl` (WAFs check for its absence).

## 9) Screenshot/PDF farm on a Linux server

```python
from camoufox.sync_api import Camoufox

with Camoufox(headless="virtual") as browser:   # headed-in-Xvfb > headless realism
    page = browser.new_page()
    page.goto("https://example.com")
    page.screenshot(path="out.png", full_page=True)
```

Env: `sudo apt-get install xvfb` (+ `setpriv`); export
`CAMOUFOX_VIRTUAL_DISPLAY_SIZE=1920x1080x24` for larger buffers. GPU fidelity →
`headless="virtual-gpu"` with `gpu=` + one-time
`camoufox gpu-setup <PCI-addr> | sudo sh`.

## 10) Remote browser service (other language/other machine)

```bash
python -m camoufox server            # prints "Websocket endpoint: ws://localhost:PORT/…"
# or: npx camoufox server
```

```python
from camoufox.server import launch_server

launch_server(headless=True, geoip=True, port=1234, ws_path="hello")
```

Clients (any language with Playwright): `browser = p.firefox.connect("ws://host:1234/hello")`.
Single browser instance: one identity for all connects — rotate servers for scale; no
persistent contexts over remote servers.

## 11) Steal-the-identity debugging

```python
with Camoufox(headless=True, debug=True, os="windows") as browser:
    ...
# debug=True prints the exact CAMOU_CONFIG identity (UA, screen, GPU, fonts, voices, …)
```

CLI: `camoufox test https://my-camouflaged-url` opens the browser with the Playwright
inspector for step debugging. For a bug report attach output of `camoufox version`.

## 12) Recorded real-device presets

```python
Camoufox(fingerprint_preset=True)          # random real device recording
Camoufox(fingerprint_preset=my_preset_dict)  # replay/share one recorded device
NewContext(browser, preset=my_preset_dict)
```

Presets never freeze fonts/voices/audio — those remain generated & seeded by identity
(and the UA version is rewritten to the running browser's).

## 13) Verification harness before a real run

```python
CHECK_URLS = [
    "https://browserscan.net",          # IP/timezone/locale/geo coherence + score
    "https://browserleaks.com/webrtc",  # no real IP
    "https://browserleaks.com/canvas",  # canvas behavior (noise vs host-GPU)
    "https://abrahamjuliot.github.io/creepjs/",  # deep lies detection
]

def verify(proxy):
    with Camoufox(proxy=proxy, geoip=True, headless=True) as b:
        pg = b.new_page()
        for url in CHECK_URLS:
            pg.goto(url, timeout=60000)
            pg.wait_for_timeout(1500)
```

Treat any incoherent pair (IP country vs timezone vs language vs geolocation) as a bug
in your options — usually the missing `geoip=True`.

## 14) Plain Playwright → Camoufox diff

```diff
- from playwright.sync_api import sync_playwright
- with sync_playwright() as p:
-     browser = p.firefox.launch(headless=True)
+ from camoufox.sync_api import Camoufox
+ with Camoufox(headless=True) as browser:      # add proxy=/geoip=/os=... as needed
      page = browser.new_page()
      page.goto("https://example.com")
```

Keep every other Playwright call. Only browser-launch lines change; delete custom
anti-detection scripts (JS `add_init_script` spoofing, UA tricks) — they fight the
C++-level spoofing and add traces.

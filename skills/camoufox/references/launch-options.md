# Launch options — complete reference

Applies to both launchers with identical names (snake_case in both languages; user docs
keep one set of names). Valid on `Camoufox`/`AsyncCamoufox` (Python),
`Camoufox()` (TypeScript), `NewBrowser`/`AsyncNewBrowser`, `launch_server`/`launchServer`,
and `launch_options`()/`launchOptions()` — except where marked as excluded. Any other
keyword is forwarded to Playwright's `firefox.launch()` / `launch_persistent_context()`
(camelCase keys from TypeScript).

> Defaults shown are the current package (PyPI/npm `camoufox` 0.5.x). Non-spoofing
> options that create detectable anomalies print a `LeakWarning`.

## Browser & identity

| Option | Type | Default | Effect & notes |
|---|---|---|---|
| `os` | `str` or `list[str]` | fpgen's real-world mix | `"windows"`, `"macos"`, `"linux"`, or a list to pick from. Lowercase; anything else raises `InvalidOS`. |
| `ff_version` | `int` (TS: `number`) | the browser's own | Firefox version claimed in the UA. Warns: mismatch with the engine is detectable. Rotate browser builds instead of versions. |
| `fingerprint_preset` | `bool` or dict | `None` | Recorded real-device preset instead of an fpgen draw. `True` = random bundled preset. Presets never provide fonts/voices/audio — those are still generated and seeded from the identity. |
| `fingerprint` | fpgen fingerprint dict | drawn | Use this fpgen fingerprint instead of drawing. Warns. (Legacy BrowserForge objects from pre-0.5 code are not valid here.) |
| `webgl_config` | `(vendor, renderer)` tuple | drawn | Force a GPU pair **fpgen has recorded for Firefox on `os`** (lists: `camoufox.webgl.firefox_gpus`; TS bundle mirrors it). Any other pair raises `ValueError`. Requires `os`. Overriding the host GPU switches the canvas to noise mode — see canvas table. |
| `screen` | `Screen` object / plain object | real display when headed | Bounds the generated screen: `Screen(max_width=1920, max_height=1080)`; also `min_width`, `min_height`. Headful launches derive the max from the physical monitor unless given. |
| `window` | `(int, int)` | generated | Fixed outer window size. Warns: fixed sizes are fingerprint-friendly; for production let it generate (it also centers `window.screenX/Y` on the generated screen). |
| `fonts` | `list[str]` | none | Extra font families on top of the spoofed OS's bundled fonts. Randomize custom additions to reduce font fingerprinting. |
| `custom_fonts_only` | `bool` | `False` | Use only `fonts`, not the OS set. Warns: the font set no longer matches the claimed OS. |
| `locale` | `str` or `list[str]` | from `geoip`, else drawn identity's | e.g. `"en-US"` or `["fr-FR", "en-US"]`; first entry drives `Intl`. A bare region (`"US"`) picks a language from the real speaker distribution of that country. |
| `pin_cpu_cores` | `bool` | `False` | Linux/Windows: pin the browser to as many cores as `navigator.hardwareConcurrency` claims, so parallel-timing checks measure the claimed number. Costs CPU; serializes concurrent launches. |

## Network, proxy & location

| Option | Type | Default | Effect & notes |
|---|---|---|---|
| `proxy` | dict | none | Playwright format: `{"server": "http://gate.example.com:8080", "username": ..., "password": ..., "bypass": ...}`. |
| `geoip` | `bool` or `str` | none | `True` = look up the public IP (through `proxy` if set); a string = use that IP. Sets timezone, locale (speaker distribution), geolocation (permission granted), and WebRTC IP. Without a proxy it also decides whether IPv6 stays on. Needs the `geoip` extra (Python). |
| `geoip_db` | `str` | `camoufox set --geoip` choice | Database name, default bundle `"GeoIP AIO by daijro"` (auto-refreshed when a week old). |
| `block_images` | `bool` | `False` | Block all image requests. Saves bandwidth; warns: some WAFs detect no-image loading. |
| `block_webrtc` | `bool` | `False` | Disable WebRTC entirely (no leak to spoof). |
| `block_webgl` | `bool` | `False` | Disable WebGL. Warns: many WAFs check for its absence. |
| `disable_coop` | `bool` | `False` | Drop Cross-Origin-Opener-Policy so elements inside cross-origin iframes (e.g. the Turnstile checkbox) can be clicked. Warns. |

## Stealth & automation behavior

| Option | Type | Default | Effect & notes |
|---|---|---|---|
| `humanize` | `bool` or `float`/`number` | off | Human-like cursor movement for `mouse.move()` (replayed recordings of real people, Cursory). A number caps one move at that many seconds (default cap 1.5). Cursor highlighter is browser-internal, never visible to the page. |
| `main_world_eval` | `bool` | `False` | Allow `page.evaluate("mw:...")` and `mw:` init scripts in the page's real world. Without it, a `mw:` evaluate fails with "Main world evaluation is disabled". `mw:` scripts cannot return element handles. |
| `enable_cache` | `bool` | `False` | Enable Firefox page/request caches (uses more memory). Without it, `page.go_back()` / `page.go_forward()` do nothing. |
| `allow_addon_new_tab` | `bool` | `False` | Let addons open tabs (blocked by default). |
| `config` | dict | none | Raw Camoufox properties (see [config-keys.md](config-keys.md)). Overrides the generated identity; manual identity keys emit `LeakWarning`. Advanced use only. |
| `firefox_user_prefs` | dict | none | Extra Firefox prefs passed at launch. |
| `args` | `list[str]` | none | Extra browser command-line arguments. |
| `env` | dict | copy of environment | Environment variables for the browser process. |
| `debug` | `bool` | `False` | Print the identity/config sent to the browser (`CAMOU_CONFIG`). |

## Browser binary & display

| Option | Type | Default | Effect & notes |
|---|---|---|---|
| `headless` | `bool` / `"virtual"` / `"virtual-gpu"` | `False` | See table below. Not handled by `launch_options()`/`launchOptions()` — use those only for plain headless/headful launches, or use `NewBrowser`. |
| `gpu` | PCI address / index / node / `vendor:device` / vendor name | `CAMOUFOX_GPU`, or the only card | Selects the card for `headless="virtual-gpu"`. Never guessed: several GPUs with no selector raise `AmbiguousGpuSelection`. Not accepted by `launch_options()`. Excluded in Python's `launch_options`. |
| `require_gpu` | `bool` | `True` | With `"virtual-gpu"`: fail the launch if WebGL work is not accounted to the card within `CAMOUFOX_GPU_WAIT_S` (default 30 s); `False` warns (`GpuSoftwareFallback`). Not taken by `launch_options()`. |
| `virtual_display` | `str` | none | Attach to an existing X display, e.g. `":99"`, instead of spawning one. |
| `executable_path` | path str | the installed build | Use this binary (also via `CAMOUFOX_EXECUTABLE_PATH`). Mutually exclusive with `browser` → `ValueError`. |
| `browser` | `str` | the paired build | Launch a different **installed** build: `"official/beta.20"`, `"beta.20"`, `"134.0.2-beta.20"`. Never downloads; warns when it differs from the paired build (same as `camoufox set`). |
| `canvas_noise` | `bool` | on when identity gives the host's OS/GPU | Force LibreWolf/Tor style random canvas + WebGL readback (`privacy.resistFingerprinting`, canvas only). Warns whenever it switches away from the default. Per **browser**, not per context. |
| `i_know_what_im_doing` | `bool` | `False` | Silence the leak warnings above. |

## Persistent profile

| Option | Value | Effect |
|---|---|---|
| `persistent_context` | `True` | Launch `launch_persistent_context` instead; the context manager / promise yields a **`BrowserContext`**, not a `Browser`. Requires `user_data_dir`. Cannot be served by `launch_server`. |
| `user_data_dir` | path str | Profile directory to persist (cookies, storage, addons). |

## Headless modes in detail

| `headless=` | What runs | Requirements | Gotchas |
|---|---|---|---|
| `False` | Normal window | A display | Default; best realism. |
| `True` | Firefox headless | none | Camoufox patches the headless tells it knows about (fine pointer, wheel ticks, etc.), but a WAF may still find new ones. |
| `"virtual"` | Headed browser on a private Xvfb display | Linux; `Xvfb` and `setpriv` (util-linux) | 1x1 display unless `CAMOUFOX_VIRTUAL_DISPLAY_SIZE=1920x1080x24`. Xvfb has no GPU ⇒ canvas/WebGL render in software (identity keeps a drawn GPU: reads as a machine whose driver failed — see canvas table). Handled by `Camoufox`/`AsyncCamoufox`/`NewBrowser`/`launch_server`. |
| `"virtual-gpu"` | Headed browser on a private headless Xorg bound to one GPU | Linux; Xorg + the card's driver | Real GPU rendering; the identity claims that card. One-time machine setup without root: `camoufox gpu-setup <GPU> \| sudo sh`. Labels: PCI address is stable (`0000:01:00.0`), or index in PCI order, `renderD128`/`/dev/dri/card1`, `10de:2c02`, or a vendor name (`nvidia`). Errors `GpuNotFound`, `NoGpuFound`, `AmbiguousGpuSelection`, `GpuSoftwareFallback`, `GpuVtTakeover` — all `VirtualDisplayError` subclasses. The launch verifies WebGL work lands on the card; no silent fallback to Xvfb. |

`python -m camoufox gpu-xorg-config <GPU>` prints just the Xorg config for the card.

## Canvas & WebGPU rules (read before customizing GPUs)

What a page can read from `canvas`/WebGL readback comes from the **host's** OS, driver
and GPU. The launcher decides by two facts: does the identity claim the host's OS? is
the host on a GPU?

| Situation | Behavior |
|---|---|
| Identity on the host's OS, host has GPU driver (or software host) | No canvas noise — reads exactly like stock Firefox. All contexts share the host GPU ⇒ less identity diversity. |
| Identity on another OS, or claims another GPU (`webgl_config`, preset GPU, unrecorded host GPU) | LibreWolf/Tor-style random readback automatically, **with a `LeakWarning`** — reads as a privacy-hardened Firefox, slightly rarer than stock. |
| `canvas_noise=True` | Force noise on any OS. Needed browser-wide when contexts claim other OSes. |
| `canvas_noise=False` | Never noise (only sensible on the host's OS + host GPU; else pixels betray the real OS ⇒ worst case). |

Noise is set at browser launch. A browser launched **without** noise makes every
`NewContext` claim the host's OS and GPU; `NewContext(os=...)` with anything else raises
`ValueError`.

WebGPU: `navigator.gpu` exists only where Firefox on the claimed device has it (Windows
and Apple Silicon macOS); the adapter is the host's only when the identity claims the
host's GPU — otherwise `requestAdapter()` resolves `null`, as Firefox does for a blocked
or software GPU.

## Environment variables

| Variable | Effect |
|---|---|
| `CAMOU_CONFIG` | The identity blob the browser reads (written by the launchers; used when driving the browser binary directly). |
| `CAMOUFOX_EXECUTABLE_PATH` | Default for `executable_path`. |
| `CAMOUFOX_GPU` | Default GPU selector for `"virtual-gpu"`. |
| `CAMOUFOX_VIRTUAL_DISPLAY_SIZE` | Xvfb size for `"virtual"`, e.g. `1920x1080x24`. |
| `CAMOUFOX_GPU_WAIT_S` | Seconds (`"virtual-gpu"`) to wait for kernel GPU usage by the browser (default 30). |

## Paired-build rules

- Each package release is **paired with one browser build** it was tested against.
  Launches use it even when newer builds are installed; a launch with the paired build
  missing downloads it first (package upgrade never runs an untested browser).
- `camoufox set` overrides persistently (warns at launch); `browser=` overrides one
  launch; `camoufox set --release` restores pairing.
- Every build declares the interface it speaks; a package too old to drive it fails at
  `sync`/`fetch`/launch and names the required upgrade.
- Development checkouts (installed from the repo, not PyPI) follow their channel
  (`official/stable` by default) instead of a pairing.

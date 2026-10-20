# Errors, warnings, pitfalls & troubleshooting

## Exceptions (Python; TS throws the same names)

| Exception | Cause | Fix |
|---|---|---|
| `InvalidOS` | `os` not one of `windows`/`macos`/`linux` (values are case-sensitive lowercase). | Lowercase the value. |
| `InvalidIP` | Exit-IP or public-IP lookup failed, or `webrtc_ip` is not a valid IPv4/IPv6 address. | Check the proxy; or pass `webrtc_ip=` and `timezone_id=` to skip the lookup. |
| `InvalidProxy` | Proxy `server` string not parseable. | Use Playwright's dict format with scheme + port. |
| `UnknownIPLocation` | IP absent from the GeoIP database. | Pass `locale=`; set geolocation via `NewContext(geolocation={...})`. |
| `NotInstalledGeoIPExtra` | `geoip=` without the `[geoip]` extra. | `pip install -U "camoufox[geoip]"`. |
| `CannotFindXvfb` | `headless="virtual"` without Xvfb. | `sudo apt-get install xvfb` (Debian) / `sudo pacman -S xorg-server-xvfb` (Arch); also needs `setpriv` (util-linux). |
| `VirtualDisplayNotSupported` | `"virtual"` outside Linux / no setpriv. | Use Linux, or plain `headless=True`. |
| `AmbiguousGpuSelection` | Multiple GPUs, no selector with `"virtual-gpu"`. | Pass `gpu=` PCI address (`0000:01:00.0`) or set `CAMOUFOX_GPU`; list with `python -m camoufox gpu-xorg-config`. |
| `GpuNotFound` / `NoGpuFound` | Named card missing / no cards. | Fix the selector; on headless cloud servers prefer `"virtual"`. |
| `GpuSoftwareFallback` | WebGL work not accounted to the chosen card in `CAMOUFOX_GPU_WAIT_S` (default 30 s). | Fix driver/Xorg; or `require_gpu=False` to warn and continue (software canvas). |
| `GpuVtTakeover` | Active VT switched during Xorg startup (desktop vs server conflict). | Launch from a service/ssh session not attached to a desktop VT. |
| `ValueError: browser + executable_path` | Both given (or `CAMOUFOX_EXECUTABLE_PATH` set while passing `browser=`). | Remove one. |
| `ValueError` on `NewContext` OS/GPU | Context claims a different OS/GPU than the browser without canvas noise. | Launch browser with `canvas_noise=True`, or drop the explicit OS. |
| `ValueError` on `webgl_config` | Vendor/renderer pair not recorded for Firefox on that `os`. | Pick from the recorded list (`camoufox.webgl.firefox_gpus`). |

All virtual-display errors are subclasses of `VirtualDisplayError`.

## LeakWarnings (intentional, noisy, by design)

Warning → what it means → how to proceed:

| Warning | Meaning | Action |
|---|---|---|
| Manual identity key in `config` | You overrode one value of a coherent machine (e.g. `navigator.userAgent`, `headers.User-Agent`, `screen.*`, `window.*`, `timezone`, geolocation keys). | Delete it; use `os`/`screen`/`window`/`locale`/`geoip`. |
| `geoip=True` without proxy | Timezone/locale from identity, machine's IP elsewhere — classic mismatch. | Add `proxy=`; if IP is truly local, silence with `i_know_what_im_doing=True`. |
| `user_agent`/`viewport`/`is_mobile` forwarded to Playwright | Breaks or fights the generated identity (`user_agent` leaks a second machine; forced `viewport` resizes a spoofed window; `is_mobile` ignored). | Drop plain-Playwright overrides; rely on generated identity. |
| `block_images` | Some WAFs detect image-less browsing. | Keep for bandwidth-heavy tasks you accept the risk for. |
| `block_webgl` | Many WAFs treat missing WebGL as automation. | Spoof instead (`webgl_config`) unless the site has no WebGL checks. |
| `canvas_noise` switch | Reads as privacy-hardened Firefox (LibreWolf/Tor-style), rarer than stock. | Keep when identities ≠ host OS; else drop. |
| Disable-COOP (`disable_coop`) | Cross-Origin-Opener-Policy disabled (needed to click Turnstile-type checkboxes in cross-origin iframes). | Notable anomaly; enable per launch only. |
| `ff_version` mismatch | Claimed version ≠ engine version. | Only for testing; rotate builds instead. |
| `custom_fonts_only` | Font set no longer matches claimed OS. | Pass `fonts=` instead. |
| `fingerprint=` | Custom fpgen dict bypasses coherence guarantee. | Prefer `os`/`screen`. |
| Non-paired build (`set` or `browser=`) | Running an untested build pairing. | `camoufox set --release` to restore. |
| `i_know_what_im_doing=True` | Suppresses the warnings above — the choice still applies. | Extra care during reviews. |

## Pitfalls (from the project's own agent guidance)

1. **Don't hand-set identity values.** No `user_agent`, `navigator.*`, headers,
   screen, window, timezone, geolocation overrides. One override ⇒ inconsistent
   machine ⇒ detectable.
2. **`geoip=True` with every proxy'd launch.**
3. **Evaluate is isolated.** `page.evaluate("...")` can't read page globals; use
   `main_world_eval=True` + `mw:` prefix for page-world access; `mw:` cannot return
   element handles (`evaluate_handle`); without the option a `mw:` evaluate fails with
   "Main world evaluation is disabled".
4. **Viewport**: pages default to `no_viewport=True` under spoofed windows; don't force
   `viewport=` in production.
5. **`is_mobile` ignored** (warns). Desktop identities only.
6. **Persistent contexts** need both `persistent_context=True` and `user_data_dir`,
   can't be served via `launch_server`.
7. **Version ceilings**: Playwright `<1.63`, Node `≥22.15`, Python `≥3.10,<4`.
8. **Build pairing**: `camoufox set` overrides (warns at launch); `browser=` for one
   launch; `camoufox set --release` restores.
9. **`browser` and `executable_path` are mutually exclusive** (also with
   `CAMOUFOX_EXECUTABLE_PATH` env).
10. **Canvas is per browser**: without noise, all contexts claim host OS + GPU;
    `NewContext` raises for a different OS/GPU.
11. **Xvfb has no GPU**: `"virtual"` ⇒ software-rendered canvas; that identity reads as
    a machine with a failed GPU driver. Expect broad software rendering in scrapers'
    Docker images; use `"virtual-gpu"` when consistently great canvas matters.
12. **One browser instance per server**: remote-server connections share a single
    identity; rotate servers for scale.
13. **Addons are extracted folders** (with `manifest.json`) — not `.xpi` paths; and
    addons can't open tabs unless `allow_addon_new_tab=True`.
14. **`enable_cache` off by default** (memory). `page.go_back()`/`go_forward()` are
    no-ops without it.

## Diagnosing "site flags me"

Follow the maintainer's flow (docs/development/workflow):

1. Does the site flag **stock Firefox** too? → Not the browser: your IP/rate limit.
   Rotate proxy, retry.
2. Does it flag Camoufox only? → Run the leak ladder below, with `camoufox test` for
   live debugging:
   - Check identity consistency first: `browserscan.net` (timezone/locale/geo/WebRTC),
     `creepjs` (lies detection), `audiofingerprint.openwpm.com` (audio),
     `browserleaks.com/*` (fonts/canvas/webgl/webrtc/cdpr).
   - Look for the `LeakWarning`s printed at launch — they list your anomaly budget.
   - Set `debug=True`, read `CAMOU_CONFIG`, confirm os/GPU/screen/vendor/renderer
     plausibility (UA↔screen↔GPU↔OS must all agree).
   - Toggle suspects one by one (`disable_coop`, `block_images`, cache, coop, addons),
     retest after each.
   - Try `fingerprint_preset=True` (recorded real device vs synthesized draw), or
     `headless="virtual"` instead of `headless=True`.
   - File issues only after checking existing reports, with `camoufox version` output:
     https://github.com/daijro/camoufox/issues/new?template=camoufox-detected.md

## Release-state context (calibrating expectations)

- Package line 0.5.x (PyPI & npm, released together at the same version).
- Browser source fully public since v146.0.1-beta.25 (Jan 2026); older releases ≤
  v135.0.1-beta.24 shipped a closed-source canvas patch.
- Project under active development in 2026 after a maintenance gap; docs site may lag
  the READMEs. **READMEs + properties.json on GitHub are authoritative.**

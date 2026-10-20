# Config keys (`config` / `CAMOU_CONFIG`) & per-context setters

`config={...}` is the raw escape hatch: it overrides the generated identity.
Authoritative list of keys and types (current):
[browser/settings/properties.json](https://github.com/daijro/camoufox/blob/main/browser/settings/properties.json).
The launchers populate all of this for you — the Python library reads `config` only for
features not yet exposed as options. Manual **identity** keys emit `LeakWarning`.

Keys are served by `MaskConfig`, a C++ reader of the `CAMOU_CONFIG` env var. Spoofing
happens at C++ level, so hijacked properties look native (`toString()` → `native code`,
no descriptor drift), and values agree in workers and iframes.

## Global identity & behavior keys (current)

| Key | Type | Purpose |
|---|---|---|
| `navigator.userAgent` | str | UA (defaults coherent with the drawn device) |
| `navigator.doNotTrack` | str | DNT header/JS value |
| `navigator.appVersion` | str | |
| `navigator.oscpu` | str | |
| `navigator.language` | str | |
| `navigator.platform` | str | |
| `navigator.hardwareConcurrency` | uint | |
| `navigator.maxTouchPoints` | uint | (0 ⇒ fine pointer on desktop) |
| `navigator.globalPrivacyControl` | bool | |
| `screen.height` / `screen.width` | uint | |
| `screen.availHeight` / `availWidth` / `availTop` / `availLeft` | uint | |
| `screen.colorDepth` / `screen.pixelDepth` | uint | |
| `window.outerHeight` / `outerWidth` | uint | Outer viewport can never be resized by the user. |
| `window.innerHeight` / `innerWidth` | uint | Prefer these over `document.body.*`. |
| `window.screenX` / `screenY` | int | |
| `window.devicePixelRatio` | double | |
| `headers.User-Agent` | str | falls back to `navigator.userAgent` |
| `headers.Accept-Language` / `headers.Accept-Encoding` | str | |
| `webrtc:ipv4` / `webrtc:ipv6` | str | Spoofs ICE candidates, SDP, `getStats()` (protocol-level). Private/loopback addresses are left alone. Candidate UDP ports use the **claimed OS's** ephemeral range. |
| `network:ipv4Only` | bool | geoip-derived: behave as an IPv4-only network. |
| `fonts` | array | Font family names the page can find. |
| `audio:seed` | uint | Deterministic audio transform seed (`AudioBuffer`/`AnalyserNode` outputs), per identity, never 0. |
| `geolocation:latitude` / `longitude` / `accuracy` | double | latitude pairs with longitude; accuracy auto-derived from coordinate precision if unset. Permission auto-granted. |
| `timezone` | str | IANA TZ (`America/Chicago`); changes `Date()`, `Intl`, workers. |
| `locale:language` / `locale:region` | str | Intl + system language (need each other). |
| `locale:script` / `locale:all` | str | Optional script (`Latn`); `all` overrides the accepted-languages list. |
| `humanize` | bool | Cursor humanization switch. |
| `humanize:maxTime` / `humanize:minTime` | double | Max (default 1.5 s) / min seconds per move. |
| `showcursor` | bool | Browser-internal cursor highlighter (not visible to pages). |
| `AudioContext:sampleRate` / `outputLatency` / `maxChannelCount` | uint/double | |
| `webGl:vendor` / `webGl:renderer` | str | `UNMASKED_VENDOR_WEBGL` / `UNMASKED_RENDERER_WEBGL`. |
| `webGl:supportedExtensions` / `webGl2:supportedExtensions` | array | |
| `webGl:parameters` / `webGl2:parameters` | dict | Parameter values (do NOT invent: WAFs hash WebGL fingerprints against datasets). |
| `webGl:shaderPrecisionFormats` / `webGl2:shaderPrecisionFormats` (+ `:blockIfNotDefined`) | dict/bool | |
| `voices` | array | Array of `{isLocalService, isDefault, voiceUri, name, lang}`; voices must match the claimed OS's set. |
| `voices:blockIfNotDefined` | bool | Block system voices when not spoofed. |
| `mediaDevices:enabled` / `micros` / `webcams` / `speakers` | bool/uint | Device counts (defaults 3/1/1). |
| `mediaDevices:microphoneLabels/Groups`, `webcamLabels/Groups`, `speakerLabels/Groups` | array | OS-style labels & device groups. |
| `media:spoof_codecs` | bool | `canPlayType` / `isTypeSupported` answers from the claimed OS (Linux host codec libs otherwise leak). |
| `allowMainWorld` | bool | `mw:` main-world evaluates (`main_world_eval=...` sets this). |
| `disableWorldIsolation` | bool | Advanced escape from isolated worlds. |
| `allowAddonNewtab` | bool | Addons may open tabs (`allow_addon_new_tab`). |
| `forceScopeAccess` | bool | Read closed shadow roots: `config={"forceScopeAccess": True}`. |
| `instantAnimations` | bool | Complete animations instantly so Playwright never waits. `config={"instantAnimations": True}` — **detectable, warns**. |
| `addons` | array | Extracted-addon paths. |
| `certificatePaths` / `certificates` | array | Custom CA handling. |
| `debug` | bool | Verbose config debugging. |

## Removed keys (do not use on current builds)

Removed at **130.0.1-beta.12**: `webgl*:contextAttributes`,
`webGl*:parameters:blockIfNotDefined` (older dual names).
Removed at **146.0.1-beta.25**: `navigator.buildID`, `enableRemoteSubframes`.
Removed at **156.0.1-beta.32**: `battery:*`, `canvas:*` (`seed`, `aaOffset`,
`aaCapOffset`), `document.body.*`, `fonts:spacing_seed`, `memorysaver`,
`navigator.appCodeName/appName/cookieEnabled/languages/onLine/product/productSub`,
`pdfViewerEnabled`, `screen.pageXOffset/pageYOffset`,
`voices:fakeCompletion(-:charsPerSecond)`, `webrtc:localipv4/localipv6`,
`window.history.length`, `window.scroll*`, `disableInstantAnimations`.
Removed at **156.0.1-beta.37**: `disableTheming`, `webGl*:contextAttributes`.

→ Battery, PDF-viewer and body-size spoofing are **stock Firefox behavior now**; don't
pass them in `config`. (The older fingerprint pages on camoufox.com still document some
removed keys — properties.json wins.)

## Per-context window setters

`NewContext()` applies its identity via an init script calling `window.*` setter
functions that the patches remove (`window-setter-seal.patch`) before page script runs:

| Setter | Page-visible effect |
|---|---|
| `setAudioFingerprintSeed(seed)` | AudioBuffer/AnalyserNode outputs |
| `setTimezone(tz)` | `Date`, `Intl.DateTimeFormat`, all time APIs |
| `setScreenDimensions(w, h)` | `screen.width/height` + matching CSS media features |
| `setScreenColorDepth(depth)` | `screen.colorDepth` |
| `setNavigatorPlatform(platform)` | `navigator.platform` |
| `setNavigatorOscpu(oscpu)` | `navigator.oscpu` |
| `setNavigatorHardwareConcurrency(cores)` | cores (workers included) |
| `setNavigatorUserAgent(ua)` | `navigator.userAgent` (workers included) |
| `setWebRTCIPv4(ip)` / `setWebRTCIPv6(ip)` | ICE, SDP, `getStats()` |
| `setWebGLVendor(vendor)` / `setWebGLRenderer(renderer)` | WebGL unmasked ids |
| `setFontList(fonts)` | Font existence checks (comma-separated, case-insensitive) |
| `setSpeechVoices(voices)` | `speechSynthesis.getVoices()` |

Storage is a mutex-protected per-context store (serialized as `roverfox.s.`-prefixed
prefs, synced to content processes over sync IPC). Audio, navigator, screen, timezone
and WebGL values are also stored at browser scope (id 0) for workers that cannot
resolve their context; fonts, voices and WebRTC are not. `window-setter-seal.patch`
removes each setter from the global right after it runs, and each context can call a
given setter once (`IsSetterOffered` guard).

Values must describe one plausible machine — the launchers'
`generate_context_fingerprint()` builds the whole script; never mix hand-picked values.

## What stays stock Firefox (by design)

| Signal | Why | Consequence |
|---|---|---|
| Canvas pixels | Stock Firefox doesn't perturb pixels; noise would mark you as a spoofer | On the host OS identity, canvas hash is stable **per machine**; on other OSes readback is randomized (RFP) |
| Text widths | No glyph-spacing noise — real machines don't produce it | Same font + same OS measures identically everywhere |
| SpiderMonkey engine behavior | JS engine quirks can't be turned into V8 | Camoufox presents Firefox identities only |
| System colors (`AccentColor`), text/GPU rasterization, shader precision | Host OS/driver realities the patches cannot change | Known-limitation rows; mitigate per identity mismatch with `canvas_noise=True` |

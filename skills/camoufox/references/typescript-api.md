# TypeScript / Node.js API — full reference

Package: [`camoufox` on npm](https://www.npmjs.com/package/camoufox) (official; current
line **0.5.x** — anything earlier, e.g. the 0.1–0.4 unofficial `camoufox-js` port, is
deprecated). Requirements: Node.js **22.15+**, `playwright-core` **< 1.63** (peer
dependency). GeoIP is built in — no extra.

```bash
npm install camoufox playwright-core
npx camoufox fetch
```

Ports the Python package exactly: same options (snake_case), same data files, same
browser install directory, same identity draws — a pinned identity looks identical from
either language. Python is never called.

## Imports

```javascript
import {
  Camoufox,            // also AsyncCamoufox (parity alias)
  NewBrowser,          // also AsyncNewBrowser (parity alias)
  NewContext,          // also AsyncNewContext (parity alias)
  launchOptions,
  launchServer,
} from "camoufox";
```

`playwright-core` has one promise-based API, so all functions return promises.
`exclude_addons` takes `("UBO")[]` strings. `screen` accepts the package's `Screen`
helper or a plain object (`{ max_width, max_height, min_width, min_height }`).

## Launch

```javascript
import { Camoufox } from "camoufox";

const browser = await Camoufox({ headless: true });
const page = await browser.newPage();
await page.goto("https://example.com");
await browser.close();
```

`Camoufox(options)` takes every option from [launch-options.md](launch-options.md).
With `persistent_context: true` + `user_data_dir` it resolves to a **`BrowserContext`**:

```javascript
const context = await Camoufox({ persistent_context: true, user_data_dir: "./profiles/alice" });
const page = await context.newPage();
await context.close();
```

## Proxy with a matching location

```javascript
import { Camoufox } from "camoufox";

const browser = await Camoufox({
  proxy: { server: "http://proxy.example.com:8080", username: "user", password: "pass" },
  geoip: true,
});
```

`geoip: true` resolves the exit IP through the proxy and sets timezone, locale
(speaker-distribution language), geolocation (permission granted) and the WebRTC IP.
A string IP skips the lookup. IPv6 stays on only when the machine's IPv6 geolocates
where the IPv4 identity does; otherwise IPv4-only behavior + `LeakWarning`.

## One browser, many identities

```javascript
import { Camoufox, NewContext } from "camoufox";

const browser = await Camoufox({ headless: true, canvas_noise: true });
const alice = await NewContext(browser, { os: "windows" });
const bob = await NewContext(browser, {
  os: "macos",
  proxy: { server: "http://proxy.example.com:8080", username: "user", password: "pass" },
});
await (await alice.newPage()).goto("https://example.com");
await (await bob.newPage()).goto("https://example.com");
await browser.close();
```

| `NewContext` option | Effect |
|---|---|
| `preset` | Fingerprint preset object instead of an fpgen draw. |
| `os` | `"windows"` \| `"macos"` \| `"linux"`. On a no-noise browser (host OS + host GPU), another OS throws `ValueError`. |
| `ff_version` | Firefox major version to claim; defaults to the browser's own. |
| `webrtc_ip` | IPv4/IPv6 WebRTC reports. |
| `proxy` | Per-context proxy; unless `webrtc_ip` **and** `timezoneId` are both given, both are looked up from the exit IP (`InvalidIP` thrown on failure). |
| `geolocation` | `{ latitude, longitude }`; grants the `geolocation` permission. |
| anything else | Forwarded to Playwright's `browser.newContext()` **in camelCase** (e.g. `timezoneId`, `viewport`), overriding generated values. |

## Own Playwright instance

```javascript
import { firefox } from "playwright-core";
import { NewBrowser, launchOptions } from "camoufox";

const browser = await NewBrowser(firefox, { os: "linux", headless: true });
await browser.close();

const plain = await firefox.launch(await launchOptions({ os: "linux", headless: true }));
await plain.close();
```

Prefer `NewBrowser()`: defaults new pages to `no_viewport` when the window size is
spoofed and uses the host's color scheme / reduced-motion settings. `launchOptions()`
returns **Playwright camelCase keys** (`executablePath`, `firefoxUserPrefs`, …) and does
not handle `headless: "virtual"` / `"virtual-gpu"` or `gpu`. `NewBrowser` also accepts
`persistent_context`, `user_data_dir`, `from_options`.

## Remote server

```javascript
import { launchServer } from "camoufox";

const server = await launchServer({ headless: true, port: 1234, ws_path: "camoufox" });
console.log(server.wsEndpoint());   // e.g. ws://localhost:1234/camoufox
```

Non-blocking (unlike Python, which blocks and prints). Connect from any Playwright
client: `firefox.connect(endpoint)`. One browser instance — no per-connection
fingerprint rotation; rotate the server between sessions. Persistent contexts cannot be
served.

## Headless GPU display (Linux)

```javascript
const browser = await Camoufox({ headless: "virtual-gpu", gpu: "0000:01:00.0", os: "linux" });
```

Defaults on (`require_gpu: true`); selectors and one-time setup match Python — full
detail in [launch-options.md](launch-options.md). One-time machine setup:

```bash
npx camoufox gpu-setup 0000:01:00.0 | sudo sh
```

## Addons & toggles (parity notes)

```javascript
const browser = await Camoufox({
  addons: ["/path/to/extracted-addon"],
  exclude_addons: ["UBO"],
  humanize: 2.0,               // number = max seconds per cursor move
  block_images: false,
});
```

## Testing this package

The repo's TS tests replay golden fixtures recorded from the Python launcher
(byte-for-byte `launch_options()` output), plus end-to-end identity comparisons against
a real browser (`CAMOUFOX_E2E=1 CAMOUFOX_EXECUTABLE=/path/to/camoufox-bin pnpm test
tests/e2e.test.ts` from `typescript/`). Relevant only when contributing; user-visible
lesson: the two launchers are kept strictly in sync.

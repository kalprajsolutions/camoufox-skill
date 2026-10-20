# Proxies & GeoIP — matching your traffic to a coherent location

The most common detection vector after the fingerprint itself: browser values that
disagree with the exit IP. Every scraped-origin WAF compares IP→country/timezone/locale
against what the page reports. Camoufox's `geoip` layer closes this in one line.

## The one-line rule

```python
with Camoufox(proxy={...}, geoip=True) as browser: ...
```

`geoip=True` → looks up the **exit IP through the proxy**, then sets:

- timezone (`geolocation:...`, permission auto-granted),
- locale + `Accept-Language` (language chosen from the **real speaker distribution**
  of the IP's region — the browser does not speak English just because most bots do),
- geolocation: latitude/longitude derived from the IP,
- WebRTC IP → proxy IP (ICE, SDP, `getStats()` are rewritten at protocol level).

Pass a string instead to use that IP without lookup: `geoip="203.0.113.9"`.

## Proxy format (Playwright's)

```python
proxy={
    "server": "http://gate.nodemaven.com:8080",   # http(s)://host:port or socks5:
    "username": "username",                       # optional
    "password": "password",                       # optional
    "bypass": "example.com",                      # optional
}
```

```python
# Per-context in a farm:
NewContext(browser, proxy={"server": ..., "username": ..., "password": ...})
```

**Use residential proxies when stealth matters** — the docs underline this explicitly.
Datacenter ranges correlate with automation regardless of browser.

## What geoip actually does

| Step | Detail |
|---|---|
| 1. Resolve IP | Through `proxy` if set; else the machine's public IP. Failure → `InvalidIP`. |
| 2. Geolocate | Database: GeoIP All-in-One (default name `"GeoIP AIO by daijro"`), auto-refreshed when the copy is a week old. Not found → `UnknownIPLocation` (then pass `locale=` and set coordinates another way, e.g. `NewContext(geolocation=...)`). |
| 3. Populate identity | timezone, locale(s), geolocation, WebRTC IP, `network:ipv4Only` when needed. |
| 4. IPv6 sanity | Without a proxy: if the machine has IPv6 and it geolocates where the IPv4 identity does, IPv6 stays on; otherwise the browser acts as IPv4-only (`network:ipv4Only`) and prints a `LeakWarning` explaining why. This prevents IPv6 DNS fallback from exposing a different country. |

## Locale behavior

```python
Camoufox(locale="en-US")                  # single
Camoufox(locale=["en-US", "fr-FR", "de-DE"])   # ordered accepted languages; first = Intl
Camoufox(locale="en-US,fr-FR,de-DE")      # comma string form
Camoufox(locale="US")                     # country → realistic language distribution
```

Default without `locale`: drawn from `geoip` (proxy's region), else from the drawn
device identity. Do not combine `geoip=True` with a contradictory `locale` unless it
matches the exit IP's region.

## Per-context proxy & locality

```python
from camoufox.sync_api import Camoufox, NewContext

with Camoufox(headless=True, canvas_noise=True) as browser:
    ctx = NewContext(
        browser,
        os="macos",
        proxy={"server": "http://proxy.example.com:8080", "username": "u", "password": "p"},
    )
```

Rules:

- Unless **both** `webrtc_ip` and `timezone_id` are provided, the exit IP is looked up
  per context and timezone + WebRTC IP are set from it (gives per-context locality with
  zero config).
- Lookup failure → `InvalidIP`; bypass it by passing `webrtc_ip=` + `timezone_id=`
  yourself (per-context Playwright `geolocation={latitude, longitude}` is recommended
  for the coordinates and grants the permission automatically).

## WebRTC strategy options

| Need | Option |
|---|---|
| Report the proxy IP | default behavior with `geoip=True` (or pass `webrtc_ip`) |
| No WebRTC at all | `block_webrtc=True` — cleanest when the site doesn't need WebRTC |
| Both proxies and WebRTC checks | keep `geoip=True`; verify at browserleaks.com/webrtc |

## Proxy-source hygiene

Sponsors/docs recommend residential pools (NodeMaven, Node Proxy, Thordata, etc. — see
camoufox.com sponsors). Things to check on any provider: whether the IP is already
flagged/known to WAFs, whether session stickiness (ZIP/city targeting) is available so
the same identity keeps the same location, and whether the exit IP country changes
mid-session. IP quality is the part Camoufox **cannot** spoof.

## Verification checklist run this before scraping

1. `python -m camoufox version` — synced GeoIP db, correct build.
2. Open `https://browserscan.net` through the exact proxy+`geoip=True` config: IP,
   timezone, language, geolocation dots must agree.
3. `https://browserleaks.com/webrtc` and `/ip` — no real IP.
4. `https://www.browserscan.net` score sanity / creepjs for deeper analysis.
5. Then run the real workflow headless/virtual with a few test identities.

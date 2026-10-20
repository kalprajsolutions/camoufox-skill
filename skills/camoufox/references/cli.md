# CLI reference

`camoufox <command>` or `python -m camoufox <command>` (Python) ·
`npx camoufox <command>` (Node). One set of commands, identical behavior; Python and
Node share the browser install directory.

## Version & data management

| Command | Options | What it does |
|---|---|---|
| `fetch [VERSION]` | | Install the paired build — or, after `set`, the latest in the chosen channel — or `VERSION` without making it active. Syncs first automatically. When containerizing, run `fetch` as the user the environment will run as, so the browser install is readable at runtime. |
| `sync` | `--spoof-os auto\|mac\|win\|lin`, `--spoof-arch auto\|x86_64\|i686\|arm64` | Refresh the list of available builds from GitHub release repos (`repos.yml`: Official `daijro/camoufox`, CoryKing `coryking/camoufox`, JWriter20 `JWriter20/camoufox`). The `--spoof-os/--spoof-arch` flags target builds whose OS/arch differ from the machine (e.g. a Windows-marked asset from Linux). |
| `set [SPECIFIER]` | `--geoip`, `--release` | Pin a build (`official/stable/134.0.2-beta.20`) or follow a channel (`official/stable`, `official/prerelease`, or any repo's channel like `coryking/stable`). No argument opens an interactive picker. `--geoip` picks the GeoIP database instead; `--release` restores the paired build. Choices are persistent; launches warn when running a non-paired build. |
| `active` | | Print the active target (`official/stable`, or a pinned version marked `(not fetched)`). |
| `list [installed\|all]` | `--path` | Tree of installed builds (or every available build from synced repos; `--path` prints full paths). |
| `remove [VERSION]` | `--select`, `-y/--yes` | Remove one build, pick interactively, or (no arg) **the whole data directory**. |
| `version` | | Package versions (Camoufox, fpgen, Playwright), active build vs latest in channel, GeoIP database + last update, storage paths and sizes. **Paste this into bug reports.** |
| `path` | | Print the data directory (`~/.cache/camoufox` on Linux; platform cache dir elsewhere). |

## Run & debug

| Command | Options | What it does |
|---|---|---|
| `test [URL]` | `--executable-path PATH` | Open a headed browser with the Playwright inspector (the window may look broken — Camoufox is not meant for human use). |
| `server` | | Start a Playwright server with default options; prints the websocket endpoint (`launch_server` equivalent). |
| `gpu-setup [GPU]` | | Print the root script that prepares this machine for `headless="virtual-gpu"` on card GPU (or `CAMOUFOX_GPU`). Run: `camoufox gpu-setup 0000:01:00.0 \| sudo sh`. Idempotent. |
| `gpu-xorg-config [GPU]` | | Print just the Xorg config `virtual-gpu` needs for that card. |
| `gui` | | Qt UI manager (Python only; needs `pip install "camoufox[gui]"`): manage installs, active version, GeoIP db, pinned versions/channels. |

## Manifest / specifiers

```
<owner>/<channel>[/<version>]     e.g.
official/stable                   follow latest stable release of daijro/camoufox
official/prerelease               follow latest prerelease (or "alpha"-named)
official/stable/134.0.2-beta.20   pin one exact build
coryking/stable/142.0.1-fork.26   a fork repository
```

`fetch` accepts the same specifier to install without activating:
`camoufox fetch official/stable/135.0-beta.25`.

## Build pairing recap

- `camoufox fetch` = the **paired** build unless `set` changed the target.
- Launches always use the paired/active build; `browser=` launch option overrides for
  one launch only (no download).
- Interface mismatch (older package meets newer browser) fails loudly and names the
  required package upgrade.

## CI / Docker notes

- `camoufox fetch` as the user that will run the browser (or image user) — installs are
  user-scoped to the environment owner.
- Playwright `<1.63`, Python `>=3.10`, Node `>=22.15`.
- For Linux servers use `headless="virtual"` (needs `apt-get install xvfb` and
  `setpriv`; check `which Xvfb`), or `virtual-gpu` with `gpu-setup` for cards.
- `camoufox remove` without arguments wipes the data dir — convenient for image
  cleanup, destructive elsewhere; `remove <version>` is the surgical form.

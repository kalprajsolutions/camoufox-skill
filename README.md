# Camoufox Agent Skill

Agentic skill: complete working knowledge of [Camoufox](https://camoufox.com), the
open-source anti-detect browser for AI agents and web scraping — installation, the
official **Python** (PyPI `camoufox`) and **TypeScript** (npm `camoufox`) launchers, the
CLI, proxies + GeoIP locality, fingerprint rotation, per-context identities, human
cursor, virtual & GPU displays, remote servers, and leak debugging.

Built from the official docs and repositories (October 2026, package line **0.5.x**;
READMEs on GitHub are authoritative when docs lag).

## Contents

```
skills/
└── camoufox/
    ├── SKILL.md                        # entry point: core workflow + golden rules
    └── references/
        ├── launch-options.md           # every option: types, defaults, warnings, canvas rules
        ├── python-api.md               # full Python reference (sync/async/NewContext/server/…)
        ├── typescript-api.md           # full Node.js reference
        ├── cli.md                      # camoufox fetch/set/list/… + build pairing + Docker
        ├── config-keys.md              # config / CAMOU_CONFIG keys, per-context setters
        ├── proxy-and-geoip.md          # proxy setup, geoip semantics, IPv6, locale
        ├── errors-and-pitfalls.md      # exceptions, LeakWarnings, pitfalls, leak workup
        └── recipes.md                  # copy-paste recipes for common tasks
```

## Install this skill

Any agent that supports [Agent Skills](https://skills.sh):

```bash
npx skills add kalprajsolutions/camoufox-skill          # repo-local install
npx skills add kalprajsolutions/camoufox-skill -g       # global
```

Or copy the folder manually for Claude Code / Opencode / Cursor / Codex:

```bash
git clone https://github.com/kalprajsolutions/camoufox-skill
mkdir -p ~/.claude/skills   # or .claude/skills in your project
cp -r camoufox-skill/skills/camoufox ~/.claude/skills/
```

The skill only teaches; nothing runs at load time. Verify your agent picked it up by
asking: *"launch Camoufox with a proxy and geoip in Python"*.

## Quick start of the browser (not the skill)

```bash
pip install -U "camoufox[geoip]" && camoufox fetch       # Python 3.10+
# or
npm install camoufox playwright-core && npx camoufox fetch   # Node 22.15+
```

```python
from camoufox.sync_api import Camoufox

with Camoufox(
    proxy={"server": "http://proxy.example.com:8080", "username": "user", "password": "pass"},
    geoip=True,
    headless=True,
) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
```

See `skills/camoufox/references/recipes.md` for 14 more ready-made flows.

## What the skill covers (and gets right)

| Topic | Detail |
|---|---|
| Hardware-level stealth model | Why C++-level spoofing + sandboxed Juggler beats JS injection / CDP fighters |
| Two official launchers | Python (sync/async, context managers) & TypeScript (one promise API) — byte-compatible identities |
| All launch options | Defaults, types, and every option that prints a `LeakWarning` |
| Identity engine | fpgen draws, coherence checks, presets, OS mix, screen/window/fonts/voices/audio seeding |
| Per-context fingerprints | `NewContext` identity fields, canvas_noise coupling, worker coverage |
| Localities | `geoip=True` semantics, IPv6-only-when-consistent rule, locale-from-speaker-distribution |
| Displays | headless vs `virtual` Xvfb vs `virtual-gpu` Xorg; GPU selection & one-time setup |
| Servers | `launch_server` / `camoufox server` + `firefox.connect()` clients, rotation caveat |
| CLI reality | Build pairing channels & repos, `set/--release/--geoip`, `list all`, spoof-os/arch |
| Escape hatches | Raw `config` keys (with **removed** keys to avoid), per-context window setters |
| Failure mode playbook | Every exception name → cause → fix; warning → meaning → action; leak ladder |

## Compatibility

| Works with | Notes |
|---|---|
| Claude Code, OpenCode, Cursor, Codex, Gemini CLI, etc. | Standard `SKILL.md` format |
| camoufox Python ≥ 0.5 | BrowserForge-era code (≤ 0.4) differs; references note the deltas |
| camoufox npm ≥ 0.5.8 | npm < 0.5.8 = deprecated unofficial port (`camoufox-js`) |

## Publishing checklist (maintainers)

1. `git init && git add . && git commit -m "Camoufox agent skill v1"` and push to
   `kalprajsolutions/camoufox-skill`.
2. Repo topics: `camoufox`, `agent-skills`, `claude`, `web-scraping`, `anti-detect`.
3. Publicize the install command (`npx skills add kalprajsolutions/camoufox-skill`) —
   the [skills.sh](https://skills.sh) directory indexes repos automatically.
4. When Camoufox ships breaking changes, diff `python/README.md` +
   `browser/settings/properties.json` upstream and update the three load-bearing files:
   `launch-options.md`, `config-keys.md`, `errors-and-pitfalls.md`.

## Legal & responsible use

Camoufox is legitimate automation tooling. Respect the terms of service and laws of any
site you operate against; this skill asserts no warranties. Upstream licensing: launchers
MIT, browser MPL-2.0 (Cursory port: LGPLv3-or-later). This skill is MIT.

## Links

- Upstream: https://github.com/daijro/camoufox · https://camoufox.com
- Python: https://pypi.org/project/camoufox · Node: https://www.npmjs.com/package/camoufox
- Issues/detections upstream: https://github.com/daijro/camoufox/issues

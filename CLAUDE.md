# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

Single-page static web app — a browser emulation of Dries Depoorter's _Shortlife v3_
clock (displays `age / WHO-life-expectancy × 100`, ticking in real time). Vanilla
HTML/CSS/JS in one `index.html`. No build, no framework, no runtime dependencies,
no tests.

## Commands

```bash
task            # list tasks
task setup      # brew bundle --file=Brewfile (installs prettier, go-task)
task update     # brew upgrade the toolchain to current versions
task fmt        # prettier --ignore-unknown --write .
task check      # prettier --ignore-unknown --check .  (verify without writing)
task serve      # python3 -m http.server 8000  → http://localhost:8000
```

`task serve` is only needed for a localhost origin — otherwise just open
`index.html` in a browser.

Prettier is canonical: `task check` is expected to pass, and `.github/workflows/ci.yml`
enforces it on every push and PR. The Brewfile is unversioned, so `task update` always
pulls the latest prettier, while CI pins an exact `prettier@X.Y.Z` via `npx`. **After
`task update`, set the version in `.github/workflows/ci.yml` to match `prettier
--version`** — otherwise local `task fmt` and CI will disagree about the file.

The one exception is the `LE` table, which carries a `// prettier-ignore` so its
column alignment survives. Prettier reproduces an ignored node's body verbatim, so
that block's indentation is maintained by hand — keep new entries at the same
6-space depth as their neighbors.

## Architecture

Everything lives in `index.html`: markup, styles, and the tick logic.

- **`LE`** — WHO Global Health Observatory life expectancy at birth, bundled in the
  page; no network calls. One entry per country, `[male, female]` in years. Adding
  a country means adding one line. The country `<select>` is built from
  `Object.keys(LE)`, so **key order is dropdown order** — `"World"` is
  deliberately first, the rest alphabetical.
- **Render loop** — `render()` reschedules itself with `requestAnimationFrame` and
  redraws 7 decimal places every frame. `start(cfg)` / `stop()` own that loop via
  the `anim` handle; always cancel before restarting.
- **Over-expectancy state** — past 100% the display flips to "beyond expectancy"
  and recolors to the accent red via inline styles set in `render()` (and cleared
  by assigning `""` on the way back down).
- **Serial number** — deterministic `h * 31 + charCode` hash of the engraved name,
  so the same name always yields the same "no. NNNNNN / 1 000 000".
- **Device** — a CSS 3D flip card: the `↻` buttons toggle `.flipped` on `#device`
  to swap the front screen for the engraved back plate.
- **Brightness** — `+` / `−` write a `--brightness` custom property on `:root`,
  applied as `filter: brightness()` on `.screen` only, clamped to 0.35–1.4.

### Persistence

Two `localStorage` keys, both written eagerly:

- `shortlife.v3` — the config JSON (`name`, `sex`, `birth`, `country`), written by
  **Program device** and restored on load.
- `shortlife.v3.b` — brightness, written on every `+` / `−`.

**Reset removes only `shortlife.v3`** — brightness deliberately persists across a
reset. All personal data stays in the browser; nothing is transmitted.

`restore()` runs a stored payload through `usable()` before programming the device,
because that payload can outlive the code that wrote it. A country that is no longer
an `LE` key falls back to `"World"`; a bad `sex` or an unparseable/future `birth` is
treated as unprogrammed and leaves the setup card visible. **Renaming or removing an
`LE` key strands everyone who stored it** — they silently land on `"World"`, so treat
`LE` keys as a persistence format, not just display strings.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

Single-page static web app — a browser emulation of Dries Depoorter's _Shortlife v3_
clock (displays `age / WHO-life-expectancy × 100`, ticking in real time). Vanilla
HTML/CSS/JS in one `public/index.html`. No build, no framework, no runtime dependencies,
no tests.

## Commands

```bash
task            # list tasks
task setup      # brew bundle --file=Brewfile (installs prettier, go-task, bun)
task update     # brew upgrade the toolchain to current versions
task fmt        # prettier --ignore-unknown --write .
task check      # prettier --ignore-unknown --check .  (verify without writing)
task serve      # bunx wrangler dev --port 8000  → http://localhost:8000
task deploy     # task check, then bunx wrangler deploy
```

`task serve` runs the Worker locally, so it serves only `public/` and 404s
everything else, as production does. Opening `public/index.html` directly in a
browser also works.

Prettier is canonical: `task check` is expected to pass. Nothing here is version-pinned
on purpose — the Brewfile is unversioned, so there is no pin to keep in sync.

The accepted cost: a prettier release that changes formatting makes `task check`
fail, and so `task deploy` refuse, on a repo nobody touched. That is not a broken
build — recover with `task update && task fmt` and commit the reformat on its own.

The one exception is the `LE` table, which carries a `// prettier-ignore` so its
column alignment survives. Prettier reproduces an ignored node's body verbatim, so
that block's indentation is maintained by hand — keep new entries at the same
6-space depth as their neighbors.

## Deploy

Hosting is a Cloudflare Worker serving static assets from `public/` — only that
directory is published (`wrangler.jsonc`), so repo files never reach the site.
The hostname `shortlife.philoserf.com` must not change: every visitor's device
lives in `localStorage` under that origin, and any other hostname silently resets it.

No CI service is in the deploy path, and pushing deploys nothing. Run
`/release-gate` before a release, then `task deploy` from `almanac` or `sextant`
with `task setup` done. `sextant` authenticates with `wrangler login`; headless
`almanac` uses a Cloudflare API token from its login keychain
(`cloudflare-api-token`), which `task deploy` exports only when the lookup succeeds.

## Architecture

Everything lives in `public/index.html`: markup, styles, and the tick logic.

- **`LE`** — WHO Global Health Observatory life expectancy at birth, bundled in the
  page; no network calls. One entry per country, `[male, female]` in years. Adding
  a country means adding one line. The country `<select>` is built from
  `Object.keys(LE)`, so **key order is dropdown order** — `"World"` is
  deliberately first, the rest alphabetical.
- **Derived config** — `derive(c)` is the only place a stored or entered config
  becomes something the render loop can drive: it holds the only date **parse** in
  the file (the other `new Date()` is today's date, for the birthday input's `max`),
  the country fallback and the `LE` lookup, and returns a **new** object carrying
  `birthMs`, `le` and `lifespanMs`, or `null` for "treat as unprogrammed". The
  argument keeps the four-key shape that goes to storage — `cfg` is always the
  derived object, never the stored one, so nothing derived can leak into a write.
- **Render loop** — `render()` reschedules itself with `requestAnimationFrame`,
  computes 7 decimal places every frame and writes to the DOM only when a string
  actually changed. The display nodes (`#pctWhole`, `#pctFrac`, `#pctSign`, the
  four `#stat*` nodes) live in the markup and are only ever assigned
  `textContent` — **no `innerHTML` anywhere**. The `shown*` variables mirror
  what is on screen; `stop()` blanks the nodes, so it must clear them too or the
  next redraw is skipped. `start(cfg)` / `stop()` own the loop via the `anim`
  handle; always cancel before restarting.
- **Over-expectancy state** — past 100% the display flips to "beyond expectancy"
  and recolors to the accent red through the `over` class on `#screen`; the
  colours live in CSS, not in `render()`.
- **Serial number** — deterministic `h * 31 + charCode` hash of the engraved name,
  so the same name always yields the same "no. NNNNNN / 1 000 000".
- **Device** — the whole page is the device: a CSS 3D flip card. Only `showFace()`
  touches `.flipped` on `#device`, because it also sets `inert` on whichever face is
  turned away — both faces stay in the DOM, so without it the keyboard tabs straight
  into controls nobody can see. Add no `classList` call for `.flipped` outside it.
  The front is the screen; the **back has two
  states**, swapped by `showBack("plate" | "form")` — the engraved plate once the
  device is programmed, the **Program your device** panel (`#setup`) when it is not.
  `⚙` on the back reopens the panel; it hides while the panel is already showing.
  The faces are stacked in one grid cell, so the card is as tall as the taller face:
  `.face.front` carries the 120/62 proportions and the back only exceeds them while
  the panel is open. `#device` starts with `no-anim` so the opening face, chosen by
  script after first paint, does not animate a flip on load.
- **Which face opens** — `start()` shows the plate and turns to the front; `stop()`
  shows the panel and **turns to the back**. Every transition goes through one of
  those two, so a valid stored config opens on the ticking front and anything
  else opens on the panel.
- **Brightness** — `+` / `−` write a `--brightness` custom property on `:root`,
  applied as `filter: brightness()` on `.screen` only, clamped to 0.35–1.4.

### Persistence

Two `localStorage` keys, both written eagerly, and every access goes through
`readStore` / `writeStore` — `localStorage` throws outright when the browser
blocks site data, and an unguarded read in `restore()` leaves the page stranded
on a dead front face. Writing `null` removes a key. Add no raw `localStorage`
call outside those two functions:

- `shortlife.v3` — the config JSON (`name`, `sex`, `birth`, `country`), written by
  **Program device** and restored on load.
- `shortlife.v3.b` — brightness, written on every `+` / `−`.

**Reset removes only `shortlife.v3`** — brightness deliberately persists across a
reset. All personal data stays in the browser; it is never transmitted. The page
itself makes no network calls. Scripts on the live page that are not in the file
(Rocket Loader, the `/cdn-cgi/challenge-platform/` bot-detection script) are
injected by the Cloudflare zone, not by the page; no beacon or tracking script
belongs in `public/index.html`.

`restore()` and **Program device** both run their config through `derive()` before
programming the device, because a stored payload can outlive the code that wrote it. A country that is no longer
an `LE` key falls back to `"World"`; a bad `sex` or an unparseable/future `birth` is
treated as unprogrammed, which leaves the device on its back showing the panel.
**Renaming or removing an `LE` key strands everyone who stored it** — they silently
land on `"World"`, so treat `LE` keys as a persistence format, not just display
strings.

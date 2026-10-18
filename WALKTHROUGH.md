# Walkthrough

This document follows Shortlife's code in the order it runs: from the page reaching the browser, through the load sequence, to each button handler. For why the code is shaped this way, read `THEORY.md`. For commands and deploy steps, read `CLAUDE.md`.

## Overview

Shortlife is a browser emulation of Dries Depoorter's Shortlife v3 clock. The visitor enters their sex, birthday and country, and the device shows `age / life expectancy × 100`, updating in real time. The page stores those inputs in `localStorage` and makes no network calls of its own. A Content-Security-Policy in `<head>` keeps any script on the page from reaching another origin.

It uses plain HTML, CSS and JavaScript in a single file, with no build step, framework or runtime dependencies. A Cloudflare Worker serves the file as a static asset.

The page has two entry points:

- **The load sequence.** The one inline `<script>` runs from top to bottom. Most of it is definitions. The work happens in a few statements that run as the script loads, and above all in `restore()`, which chooses between programming the device from storage and showing the programming panel.
- **The buttons.** Seven click handlers: dimmer, brighter, two flip buttons, ⚙, **Program device** and **Reset**. A delegated handler also covers the two sex buttons.

## Architecture

| File                | Role                                                                                                              |
| ------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `public/index.html` | The whole application: styles in `<head>`, the device markup, and one script holding the data, logic and handlers |
| `wrangler.jsonc`    | Hosting: publish `public/` as static assets at one custom domain                                                  |
| `Taskfile.yml`      | `setup`, `update`, `fmt`, `check`, `serve`, `deploy`                                                              |
| `Brewfile`          | The toolchain: prettier, go-task, bun (which runs Wrangler through `bunx`)                                        |
| `.prettierrc.json`  | Prettier settings; `task check` is the only automated gate                                                        |

Inside the script, data moves in one direction:

```text
localStorage ──readStore──▶ JSON.parse ──▶ derive() ──▶ start(d) ──▶ render() ⟲ ──▶ DOM text
                                              │ null
                                              └────────▶ stop() ──▶ panel on the back
panel inputs ──Program──▶ derive() ──▶ writeStore(the four-key input) + start(d)
```

`derive()` is the only function that turns a config into something `render()` can run on. Storage holds what the owner entered, and everything computed is rebuilt on each load.

## How the page reaches the browser

The Worker has no script. It serves `public/` and nothing else, and only at the custom domain:

`wrangler.jsonc`

```jsonc
  "assets": { "directory": "./public" },
  // The custom domain is the only address: no workers.dev, no preview URLs.
  "workers_dev": false,
  "preview_urls": false,
  "routes": [{ "pattern": "shortlife.philoserf.com", "custom_domain": true }],
```

The second address is switched off because `localStorage` belongs to an origin. A device programmed at any other address would be invisible at the real one. `task deploy` runs the formatting check before uploading:

`Taskfile.yml` — `deploy`

```yaml
  deploy:
    desc: Check formatting, then deploy to Cloudflare
    cmds:
      - task: check
      - |
        if token=$(security find-generic-password -s cloudflare-api-token -w 2>/dev/null); then
          export CLOUDFLARE_API_TOKEN="$token"
        fi
        bunx wrangler deploy
```

`task serve` runs `bunx wrangler dev --port 8000`, so the local server matches production and also 404s anything outside `public/`. You can also open `public/index.html` directly from disk, because nothing in it depends on the server.

## The document before the script runs

### Styles that matter to the logic

Most of the stylesheet is visual. Five rules interact with the script.

The device is a flip card. Both faces share one grid cell, and `.flipped` rotates the inner element:

`public/index.html` — `.device-inner`, `.device.flipped`, `.device.no-anim`

```css
      .device-inner {
        position: relative;
        display: grid;
        /* minmax(0, 1fr), not the implicit auto track: an auto track would size to
           the panel's max-content and push both faces past the device's width. */
        grid-template-columns: minmax(0, 1fr);
        width: 100%;
        transform-style: preserve-3d;
        transition: transform 0.7s cubic-bezier(0.4, 0, 0.2, 1);
      }
      .device.flipped .device-inner {
        transform: rotateY(180deg);
      }
      /* The opening face is chosen by script after first paint — don't animate it. */
      .device.no-anim .device-inner {
        transition: none;
      }
```

The faces are stacked (`grid-area: 1 / 1`) with `backface-visibility: hidden`, and `.face.back` starts out rotated 180°. A `prefers-reduced-motion` query removes the transition, so the face still swaps but without the rotation. The front face takes the device's proportions:

`public/index.html` — `.face.front`

```css
      .face.front {
        aspect-ratio: 120 / 50; /* the engraved 120 x 50 x 20 mm */
      }
```

Brightness is a custom property on `:root` (`--brightness: 1`), and it only affects the screen: `.screen` carries `filter: brightness(var(--brightness));`. The beyond-expectancy colours are attached to a class, so the script only toggles the class:

`public/index.html` — `.screen.over`

```css
      .screen.over .pct {
        color: #ff7a5c;
        text-shadow:
          0 0 8px rgba(255, 90, 60, 0.55),
          0 0 20px rgba(255, 90, 60, 0.25);
      }
```

Finally, `.plate` and `.panel` both set `display`, which would override the browser's default `[hidden]` rule. The stylesheet puts it back:

`public/index.html` — `[hidden]`

```css
      /* .plate and .panel set display, which would beat the UA [hidden] rule. */
      [hidden] {
        display: none !important;
      }
```

### The markup holds the defaults

Every node the script writes already has the value it shows before the script runs. `stop()` later restores those same values by writing them again. The front face holds the screen and its three buttons:

`public/index.html` — `#faceFront`

```html
          <div class="screen" id="screen">
            <div class="pct" id="pct">
              <span class="whole" id="pctWhole">—</span
              ><span class="frac" id="pctFrac"></span
              ><span class="sign" id="pctSign"></span>
            </div>
            <div class="sub" id="sub">of life lived</div>
          </div>
```

The odd `</span\n><span` wrapping keeps whitespace out from between the three spans, so `35.1234567%` displays as one unbroken string.

The back face starts as `inert` and holds two alternatives: the engraved `#plate` (marked `hidden`), and the programming panel `#setup` (visible). The ⚙ button (`#bEdit`) also starts `hidden`. Below the device sits the stat line, outside the card and hidden until the device is programmed:

`public/index.html` — `#stat`

```html
    <div class="stat" id="stat" hidden>
      Age <b id="statAge">—</b> · WHO expectancy <b id="statLe">—</b> ·
      <span id="statLabel">remaining</span> <b id="statRemain">—</b>
    </div>
```

## The load sequence

The script is a classic, non-module `<script>` at the end of `<body>` with `"use strict"`. Its statements run in the order below.

### The data: `LE`

`public/index.html` — `LE`

```js
      // prettier-ignore
      const LE = {
      "World":          [70.6, 76.1],
      "Australia":      [81.2, 84.8],
```

There are 37 entries, each `[male, female]` years of life expectancy at birth. Prettier skips the table so its columns stay aligned, which is also why its entries sit at the same indentation as `const` rather than one level deeper. Each key does three jobs: it is the dropdown label, its position sets dropdown order, and it is the string a stored config uses to name its country. The comment above the table names the source: WHO GHO indicator `WHOSIS_000001`, 2023 estimates for every entry, fetched on 2026-10-18. It also says to re-fetch all rows together, because WHO revises past years with each release.

### Constants and the storage adapter

`public/index.html` — `MS_PER_YEAR`, `$`, `STORE`

```js
      const MS_PER_YEAR = 365.2425 * 24 * 3600 * 1000;
      const $ = (id) => document.getElementById(id);
      const STORE = "shortlife.v3";
```

Every `localStorage` access goes through two functions. `localStorage` throws, rather than returning `null`, when the browser blocks site data:

`public/index.html` — `readStore`, `writeStore`

```js
      function readStore(key) {
        try {
          return localStorage.getItem(key);
        } catch (_) {
          return null;
        }
      }
      function writeStore(key, value) {
        try {
          if (value === null) localStorage.removeItem(key);
          else localStorage.setItem(key, value);
        } catch (_) {}
      }
```

A blocked store therefore reads as empty and swallows writes. The device still works but forgets everything on reload. Passing `null` removes a key, and that is how Reset clears the config.

### Statements that run at load

Three statements populate the panel. The country `<select>` is built from the keys of `LE`, and Belgium is pre-selected as a nod to Ghent, where the real device is made:

`public/index.html` — country dropdown

```js
      const countrySel = $("country");
      for (const name of Object.keys(LE)) {
        const o = document.createElement("option");
        o.value = name;
        o.textContent = name;
        countrySel.appendChild(o);
      }
      countrySel.value = "Belgium"; // nod to Ghent
```

The birthday picker's `max` is set to today's **local** date. The `en-CA` locale formats a date as `YYYY-MM-DD`, which a date input accepts. `toISOString()` would give the UTC date instead, which is a day off east of Greenwich for part of each day. `max` only constrains the picker; `derive()` is what actually refuses a future birthday.

`public/index.html` — `$("birth").max`

```js
      $("birth").max = new Date().toLocaleDateString("en-CA");
```

The sex control is a pair of buttons with one delegated listener. The choice lives in the variable `sex`, and `aria-pressed` mirrors it:

`public/index.html` — sex toggle

```js
      let sex = null;
      $("sex").addEventListener("click", (e) => {
        const btn = e.target.closest("button[data-sex]");
        if (!btn) return;
        sex = btn.dataset.sex;
        for (const b of $("sex").children)
          b.setAttribute("aria-pressed", String(b === btn));
      });
```

The script then declares the two variables that hold the device's runtime state: `cfg`, the derived config or `null`, and `anim`, the pending `requestAnimationFrame` handle. Everything after that, down to `restore()`, defines functions or registers listeners. The sections below take each function when it is first called, starting from `restore()`.

### `restore()`: the real entry point

`restore()` is an immediately-invoked function at the bottom of the script. It runs once, and it is the one place where the page decides which state it opens in.

`public/index.html` — `restore`

```js
      (function restore() {
        const b = parseFloat(readStore(STORE + ".b"));
        if (!isNaN(b)) setBrightness(b);

        let c = null;
        try {
          c = JSON.parse(readStore(STORE));
        } catch (_) {}
        const d = derive(c);
        if (!d) {
          stop();
          return;
        }
```

Brightness comes first, from the `shortlife.v3.b` key. If the key is missing, `parseFloat(null)` returns `NaN` and the step is skipped. Then the config: when the key is missing, `readStore` returns `null`, `JSON.parse(null)` parses the string `"null"` and returns `null` without throwing, and `derive(null)` returns `null`. Corrupt JSON throws inside the `try`, leaving `c` as `null`. Either way the page reaches `stop()`, which shows the programming panel. Only a payload that `derive()` accepts continues to the second half:

`public/index.html` — `restore` (continued)

```js
        // Fill the panel from the derived config, so a country that fell back to
        // World shows as World rather than a name the dropdown no longer has.
        $("name").value = d.name;
        $("birth").value = d.birth;
        countrySel.value = d.country;
        sex = d.sex;
        for (const btn of $("sex").children)
          btn.setAttribute("aria-pressed", String(btn.dataset.sex === sex));
        start(d);
        $("hint").textContent = HINT_PROGRAMMED;
      })();
```

The panel is filled from `d`, not `c`, so it shows exactly what the device is running, including a country that has fallen back to World. The stored payload isn't rewritten. `start(d)` then programs the device.

### `derive()`: from stored input to runnable config

`public/index.html` — `derive`

```js
      function derive(c) {
        if (!c || typeof c !== "object") return null;
        if (c.sex !== "male" && c.sex !== "female") return null;
        const birthMs = new Date(c.birth + "T00:00:00").getTime();
        if (!Number.isFinite(birthMs) || birthMs > Date.now()) return null;
        const country = c.country in LE ? c.country : "World";
        const le = LE[country][c.sex === "male" ? 0 : 1];
        return {
          // A hand-edited payload can hold anything; everything downstream of
          // here treats name as a string.
          name: typeof c.name === "string" ? c.name : "",
          sex: c.sex,
          birth: c.birth,
          country,
          birthMs,
          le,
          lifespanMs: le * MS_PER_YEAR,
        };
      }
```

Appending `"T00:00:00"` with no zone makes ECMAScript parse the birthday as **local** midnight. A date-only string would be parsed as UTC. Sex, birthday and country fail in different ways: a bad sex or an unusable birthday returns `null`, while an unknown country quietly becomes World. The return value is a new object, so the caller's four-key input stays in the shape that goes to storage.

Here is a transcript of `derive()`, extracted from the page and run under bun with `TZ=America/New_York` (a run captured while writing this, not a live block):

```text
valid                                        country=Belgium le=84 name="Ada"
country no longer in LE                      country=World le=75.9 name="Ada"
name not a string                            country=Japan le=81.5 name=""
sex missing                                  null (unprogrammed)
birth unparseable                            null (unprogrammed)
birth in the future                          null (unprogrammed)
JSON.parse(null) → null                      null (unprogrammed)
```

The inputs were `{name:"Ada", sex:"female", birth:"1990-06-15", country:"Belgium"}`, followed by variants with country `"Atlantis"`, name `{x:1}`, no sex, birth `"15/06/1990"` and birth `"2999-01-01"`.

### `start()` and `stop()`: the two states

`restore()` calls exactly one of these, and so does every later change between programmed and unprogrammed.

`public/index.html` — `start`

```js
      function start(d) {
        cfg = d;
        $("engrave").textContent = d.name || "Unregistered";
        $("serial").textContent = serialFor(d.name);
        $("pctSign").textContent = "%";
        $("statLe").textContent = `${d.le.toFixed(1)}y`; // fixed for this config
        $("stat").hidden = false;
        showBack("plate");
        showFace("front"); // a programmed device shows its screen
        if (anim) cancelAnimationFrame(anim);
        render();
      }
```

`start()` does three things. It writes what is fixed for this config (the engraving, the serial, the `%` sign and the expectancy figure), shows the plate on the back, and turns the card to the front. Then it cancels any running loop and starts a new one by calling `render()` directly.

`stop()` is the inverse. It cancels the loop, clears `cfg`, writes every display node back to its markup default, hides the stat line, clears the `shown*` cache (covered under `render()`), and turns the card to show the panel:

`public/index.html` — `stop`

```js
      function stop() {
        if (anim) cancelAnimationFrame(anim);
        anim = null;
        cfg = null;
        $("pctWhole").textContent = "—";
        $("pctFrac").textContent = "";
        $("pctSign").textContent = "";
        $("sub").textContent = "of life lived";
        $("screen").classList.remove("over");
        $("engrave").textContent = "—";
        $("serial").textContent = "no. 000000 / 1 000 000";
        $("stat").hidden = true;
        shownPct = "";
        shownOver = null;
        shownAge = "";
        shownRemain = "";
        shownLabel = "";
        showBack("form");
        showFace("back"); // unprogrammed: present the panel
      }
```

### `showBack()` and `showFace()`: two independent switches

What the back shows and which way the card faces are set separately:

`public/index.html` — `showBack`, `showFace`

```js
      function showBack(which) {
        $("plate").hidden = which !== "plate";
        $("setup").hidden = which !== "form";
        $("bEdit").hidden = which === "form";
      }

      // The hidden face stays in the DOM and stays focusable, so Tab from the
      // screen would walk into controls nobody can see. inert has to track
      // .flipped exactly; every flip goes through here so the two cannot drift.
      function showFace(which) {
        const back = which === "back";
        $("device").classList.toggle("flipped", back);
        $("faceFront").inert = back;
        $("faceBack").inert = !back;
      }
```

⚙ hides whenever the panel is showing, because it exists only to open the panel. `showFace` is the only code that adds or removes `.flipped`, and it sets `inert` at the same time so the face turned away can't be reached with Tab.

### `serialFor()`: the engraved number

`public/index.html` — `serialFor`

```js
      function serialFor(name) {
        let h = 0;
        for (let i = 0; i < name.length; i++)
          h = (h * 31 + name.charCodeAt(i)) >>> 0;
        const n = String(h % 1000000).padStart(6, "0");
        return `no. ${n} / 1 000 000`;
      }
```

`>>> 0` keeps the hash an unsigned 32-bit integer on every step. A transcript from the same run:

```text
""               → no. 000000 / 1 000 000
"Ada Lovelace"   → no. 080523 / 1 000 000
"ada lovelace"   → no. 116107 / 1 000 000
```

The first line shows that `serialFor("")` produces the same placeholder `stop()` writes, which is why `start()` calls it unconditionally, even for an empty name.

### `render()`: the loop

`start()` calls `render()` once, and from then on `render()` schedules itself with `requestAnimationFrame` on every frame.

`public/index.html` — `render`

```js
      function render() {
        if (!cfg) return;
        const { le, lifespanMs } = cfg;
        const livedMs = Date.now() - cfg.birthMs;
        const pct = (livedMs / lifespanMs) * 100;

        // derive() refused a future birthday, so livedMs goes negative only if
        // the system clock is set back after programming; the clamps below cover that.
        const s = Math.max(0, pct).toFixed(7);
        if (s !== shownPct) {
          const dot = s.indexOf(".");
          $("pctWhole").textContent = s.slice(0, dot);
          $("pctFrac").textContent = s.slice(dot);
          shownPct = s;
        }
```

The percentage is formatted to seven decimals and split at the dot into the large whole part and the smaller fraction. The DOM is written only when the string has changed. `shownPct` and its four siblings hold what is currently on screen. Because `stop()` blanks the nodes directly, it has to reset these too, or the next `start()` would skip writing a node it believes is already correct.

How often a write actually happens depends on the lifespan, as this transcript shows:

```text
# seconds between changes of the 7th decimal
Japan female     2.73 s
World male       2.23 s
Nigeria male     1.94 s
```

The rest of the frame works the same way, comparing before writing:

`public/index.html` — `render` (continued)

```js
        const over = pct > 100;
        if (over !== shownOver) {
          $("screen").classList.toggle("over", over);
          $("sub").textContent = over ? "beyond expectancy" : "of life lived";
          shownOver = over;
        }

        const ageY = livedMs / MS_PER_YEAR;
        const remainY = le - ageY;
        const age = fmtYears(Math.max(0, ageY));
```

Past 100% the `over` class recolours the screen and the caption changes. The stat line's label switches from "remaining" to "over by" at the same instant (`remainY >= 0`), and the remaining time is shown as an absolute value. The last statement, `anim = requestAnimationFrame(render);`, keeps the loop going. The `if (!cfg) return;` guard at the top is what ends it: once `stop()` clears `cfg`, the next frame does nothing and schedules nothing.

As the comment in `render()` says, `derive()` refuses future birthdays, so `livedMs` can't be negative when the device is programmed. The two `Math.max(0, …)` clamps only matter if the system clock moves backwards afterwards.

### `fmtYears()`: years and days

`public/index.html` — `fmtYears`

```js
      function fmtYears(y) {
        let w = Math.floor(y);
        let d = Math.round((y - w) * 365.2425);
        // A year is 365.2425 days, so the last ~18 hours of one round to 365 and
        // would read "35y 365d". Carry instead: the year turns up to 0.74 days
        // early, which beats showing a day count no year ever reaches.
        if (d >= 365) {
          w += 1;
          d = 0;
        }
        return `${w}y ${d}d`;
      }
```

Transcript of the carry:

```text
35       → 35y 0d
35.5     → 35y 183d
35.997   → 35y 364d
35.9985  → 36y 0d
```

The age is measured in mean Gregorian years (`MS_PER_YEAR`), so "36y 0d" doesn't land exactly on the calendar birthday.

### The last statement

`public/index.html` — `no-anim` removal

```js
      requestAnimationFrame(() => $("device").classList.remove("no-anim"));
```

`#device` starts with `no-anim`, so the face that `restore()` picks is meant to appear without the 0.7-second rotation. Removing the class one frame later turns transitions back on for flips the user asks for. `THEORY.md` explains why it's unverified whether a single frame is always enough.

## User actions

### Flipping and ⚙

`public/index.html` — flip wiring

```js
      $("bFlip").addEventListener("click", () => showFace("back"));
      $("bFlipBack").addEventListener("click", () => showFace("front"));
      $("bEdit").addEventListener("click", () => showBack("form"));
```

These switch the card's face or the back's contents and nothing else. The loop keeps running while you look at the back. If the panel is opened with ⚙ and the card is turned back to the front, the front still ticks the old config, and the panel stays open until **Program device** or **Reset**.

### Program device

`public/index.html` — `#program` handler

```js
      $("program").addEventListener("click", () => {
        const name = $("name").value.trim();
        const birth = $("birth").value;
        const country = countrySel.value;
        if (!sex) {
          flash("Select a sex for the WHO table.");
          return;
        }
        if (!birth) {
          flash("Enter your birthday.");
          return;
        }
        const c = { name, sex, birth, country };
        // sex and birth are checked above and country comes from the select, so
        // the only way derive() can refuse what this panel produces is the date.
        const d = derive(c);
        if (!d) {
          flash("Birthday can't be in the future.");
          return;
        }
        writeStore(STORE, JSON.stringify(c)); // the stored shape, not the derived one
        start(d);
        $("hint").textContent = HINT_PROGRAMMED;
      });
```

The handler builds the same four-key object that `restore()` reads, runs it through the same `derive()`, and stores `c`, not `d`. A date input's `value` is either `""` or a valid `YYYY-MM-DD`, so once the two checks pass, a future date is the only remaining reason `derive()` could refuse. `start(d)` doesn't call `stop()` first, so the `shown*` cache survives. That's safe, because the cache still matches the DOM and the new config's first frame overwrites whatever differs.

### Reset

`public/index.html` — `#reset` handler

```js
      $("reset").addEventListener("click", () => {
        writeStore(STORE, null);
        stop();
        $("name").value = "";
        $("birth").value = "";
        sex = null;
        for (const b of $("sex").children)
          b.setAttribute("aria-pressed", "false");
        $("hint").textContent = HINT_PRIVACY;
      });
```

Reset removes only the config key. Brightness stays in `shortlife.v3.b`. The country dropdown keeps its current selection. `HINT_PRIVACY` duplicates the hint's text in the markup, and a comment above the constant asks you to keep the two in step.

### Validation messages

`public/index.html` — `flash`

```js
      function flash(msg) {
        const h = $("hint");
        h.textContent = msg;
        h.style.color = "var(--accent)";
        setTimeout(() => (h.style.color = ""), 2200);
      }
```

The message replaces the hint text and turns accent red for 2.2 seconds. The text itself stays until the next successful program or reset. Each call starts its own timer, so if a second message arrives within 2.2 seconds, the first call's timer clears its colour early. The hint has `aria-live="polite"`, so screen readers announce each message.

### Brightness

`public/index.html` — `setBrightness`

```js
      let brightness = 1;
      function setBrightness(b) {
        brightness = Math.min(1.4, Math.max(0.35, b));
        document.documentElement.style.setProperty(
          "--brightness",
          brightness.toFixed(2),
        );
        writeStore(STORE + ".b", brightness.toFixed(2));
      }
```

`+` and `−` step by 0.15 within the range 0.35–1.4. Every call writes the value back to storage, including the call `restore()` makes at load. The CSS variable it sets affects only the screen, so the case keeps its brightness.

## Tooling

The only check is prettier. `task check` runs `prettier --ignore-unknown --check .` and `task fmt` runs the same with `--write`. `.prettierrc.json` sets `proseWrap: "preserve"` and `embeddedLanguageFormatting: "off"`. Prettier reads both `.gitignore` and `.prettierignore`, so `.wrangler/` (in `.gitignore`) and `.issues/` (in `.prettierignore`, because the global git excludesfile that ignores it isn't visible to prettier) are both skipped. There are no tests and no CI, and pushing doesn't deploy anything.

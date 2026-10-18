# A theory of Shortlife

This is the understanding you need to change Shortlife without breaking what it is. For the commands, the deploy steps and a map of the file, read `CLAUDE.md`. This document covers why the code is shaped the way it is.

## What it is for

Shortlife models one object and one number. The object is Dries Depoorter's Shortlife v3, a small clock that belongs to one person. The number is `age / life expectancy at birth × 100`, where the expectancy is the WHO figure for that person's sex and country. The page has two jobs. It keeps the number honest to the second, and it makes visiting the page feel like owning the object rather than filling in a calculator.

The second job is a tighter constraint than it sounds, and it accounts for most of the layout. **The page is the device.** The front of the device is the screen. The back is either the engraved owner's plate or, before the device is programmed, the panel that programs it. The vocabulary follows the object. You don't save settings, you _program_ the device. The name is not a username, it is _engraved_. Each device has a _serial number_, and brightness is a pair of buttons on the case. Commit d4a5a58 is where this became the organizing idea: the setup form used to be a second card below the device, and "the page read as a web form with a device graphic above it." So when a feature request arrives, ask first whether the physical device would have the feature, and where on the case it would go.

### The domain, and the code's words for it

The only durable state is the **owner**: name, sex, birthday and country of residence. It is stored in the words a person would use about themselves: a calendar date, a country name, `"male"` or `"female"`. The other two inputs are the **`LE` table**, which maps a country name to `[male, female]` years of expectancy at birth, and **now**. Everything else is computed fresh on every load: the birth instant, the lifespan in milliseconds, the percentage, the age and the time remaining.

Several things about the domain are deliberate, and each is easy to "fix" into a different product:

- **Expectancy at birth, not remaining expectancy.** Expectancy at birth is a period average for a whole population. Anyone who survives past middle age has a conditional expectancy above it. The device makes no correction for this, which is why its **beyond expectancy** state is something ordinary owners reach, not just an edge case in the arithmetic. Switch to conditional expectancy and the percentage creeps toward 100 and almost never crosses it. That would be a different memento.
- **Sex is binary because the WHO tables are.** The label says so: "Sex (WHO tables)". The value has exactly one use, as an index into a two-element row (`c.sex === "male" ? 0 : 1`). The label is where the design states that the binary comes from the data source, not from the app.
- **Time is local, and a year is a mean year.** The birthday is a date, not an instant. `derive()` parses it as local midnight: `c.birth + "T00:00:00"` has no zone, so ECMAScript reads it as local time. The picker's `max` is today's local date, rendered through `en-CA`. An owner who changes time zone gains or loses a few hours, which matches how people think about birthdays. Ages are counted in Gregorian mean years (`MS_PER_YEAR`, 365.2425 days), not calendar years, so "35y 0d" can appear up to about 0.74 days before the calendar birthday. `fmtYears` carries the year rather than show "365d" (17dc6a8). This is a bounded drift that the code documents. Calendar arithmetic would swap it for time-zone and leap-day bugs.

## The organizing ideas

### Storage holds what the owner said, nothing the code computed

This is the invariant everything else depends on. `derive(c)` is the one gate from the stored shape to the object the render loop runs on. It adds `birthMs`, `le` and `lifespanMs`, normalizes the country and coerces the name to a string. It returns a new object, so its argument can still be stored. The Program handler writes `c`, never the derived object, and `cfg` only ever holds a derived one. Three consequences follow, and none of them would hold in a design that cached its computed values:

1. **A change to `LE` reaches every existing device on its next load.** No owner's figure is frozen at the value from the day they programmed. The same mechanism means a data refresh moves every owner's percentage in one step, and sometimes backwards (see uncertainties).
2. **`derive()` is the migration layer.** The payload has no schema version. The `v3` in `shortlife.v3` is the product's name, Shortlife v3, not a format revision. An old payload is handled by `derive()` accepting anything it can make sense of and treating everything else as unprogrammed, not by dispatching on a version. Any field added later has to stay optional in `derive()` permanently, because payloads outlive the code that wrote them.
3. **The country fallback is non-destructive.** If the stored country is no longer an `LE` key, the device shows World. `restore()` fills the panel from the derived object, so the dropdown shows World too. The stored payload is left alone, though, so if the key comes back, the owner's country comes back with it. Only re-programming writes World to storage.

`derive()` treats its inputs differently on purpose. An unknown country degrades to World. A bad sex, or a birthday that won't parse or is in the future, is refused. Country has an honest default; sex and birthday don't. Refusal always means **unprogrammed**, which leaves the device on its back with the panel open, the only state in which the owner can repair it. This was learned the hard way: a2a382b records the two failures that caused it, a TypeError in `render()` that left a dead "—" screen, and a screen ticking "NaN.undefined%".

There are two defaults here, and they must not be merged. The panel opens on Belgium ("nod to Ghent", where the device is made). An unknown stored country falls back to World. The first is a homage; the second is the honest answer to "we don't know."

### Each `LE` key is three things at once

Each key in `LE` is the label in the dropdown, the dropdown's order (built from `Object.keys(LE)`: World first, then alphabetical), and the persistence format, because a stored config names its country by this string. This is where a well-meaning change is most likely to do damage, and two changes are especially likely:

- **A data refresh that copies WHO's names.** WHO's current names for six of these keys are different: "Türkiye", "Russian Federation", "United States of America", "Republic of Korea", "United Kingdom of Great Britain and Northern Ireland" and "Netherlands (Kingdom of the)". Paste those in and every owner in those countries silently lands on World. Refresh the numbers, never the keys.
- **Localization.** To translate the dropdown you first have to separate the label from the key. Translating the keys themselves strands every owner.

The serial number is a format in the same sense. `serialFor(name)` is a Java-style string hash taken mod 1,000,000, and it is deterministic so that a given name always engraves the same number. Owners see it as their device's identity. Change the hash and every device gets a new number on its next load. It is computed, but it deserves the same care as the keys.

### The render loop has no tick rate

`render()` runs on every animation frame but only writes to the DOM when a formatted string has changed (ad680a4: "DOM mutations observed over two seconds: 360 before, 0 after"). The design's point is that the visible tick rate appears nowhere in the code. With seven decimals, the last digit of an 80-year lifespan turns roughly every 2.5 seconds, and roughly every 1.9 seconds on Nigeria's figure. The age line turns once a day. A `setInterval` tuned to one of those rates would be wrong for the others and would drift out of phase with the real digit changes. `requestAnimationFrame` also stops in background tabs without any extra code.

The price of this is the `shown*` mirror, five variables recording what is currently on screen. They are a cache of the DOM. `stop()` blanks the DOM without going through them, so `stop()` has to clear them too, or the next `start()` will skip a redraw. More generally, every node that `start()` or `render()` writes has a default in the markup that shows before the script runs ("—", "of life lived", the zero serial, the privacy hint), and `stop()` puts each default back by restating it. `stop()` is therefore a hand-written inverse of the markup that nothing checks. It is correct today. To add a display element you touch four places: its default in the markup, the writer (`render()` with a `shown*` entry, or `start()` if the value is fixed for the config, as `statLe` is), and `stop()`'s reset of both the node and its mirror. Only `HINT_PRIVACY` has a comment asking you to keep its two copies in step.

`render()` decides _whether_ the device is beyond expectancy and sets the `over` class. CSS decides what that looks like. Keep colours out of the script. The `Math.max(0, …)` clamps on the percentage and the age date from the first commit, before `derive()` refused future birthdays. They now guard only against the system clock moving backwards after the device is programmed, and a comment in `render()` says so.

### Two faces, two independent axes

Both faces stay in the DOM, stacked in a single grid cell. d4a5a58 explains why the track has to be `minmax(0, 1fr)`. The stacking has two consequences:

- **The card is as tall as its taller face.** The front's `aspect-ratio: 120 / 50` matches the plate's engraved "120 × 50 × 20 mm", and the CSS comment asks you to keep the two in step. The open programming panel is the face that normally sets a taller height.
- **A face turned away is still focusable unless it is `inert`.** `showFace()` is the only code that touches `.flipped`, because it sets `inert` on the other face at the same moment (fb12855). A new flip control that toggles the class directly brings back tabbing into controls nobody can see.

What the back shows (`showBack("plate" | "form")`) and which way the card faces (`showFace`) are separate axes. An owner can turn a programmed device over, open the panel with ⚙, turn back to a front still ticking the old config, and turn over again to find the panel still open. Only Program or Reset ends that state. The two meet at exactly two points: `start()` shows the plate and turns to the front, and `stop()` shows the panel and turns to the back. `restore()` calls exactly one of them. So "a usable stored config opens on the ticking front, anything else opens on the panel" holds by construction. There is no first-run flag and none should be added, because "first visit" and "stored config is unusable" are deliberately the same state.

### Storage that might not exist

Every `localStorage` access goes through `readStore` / `writeStore`. The failure that caused this rule is in 889b897: `localStorage` throws when the browser blocks site data, the first unguarded read was in `restore()`, and the page loaded "inert on a dead front face" because `stop()` never ran. The policy now is that a blocked store costs persistence and nothing else. The device still programs and ticks; it just forgets on reload. The brightness key is built from the config key (`STORE + ".b"`), so renaming `STORE` moves both. Reset removes only the config, and CLAUDE.md says brightness surviving is deliberate. My reading is that brightness is a setting for the screen, not the owner's personal data, but that is a guess.

## The seams

**The origin is a storage namespace.** `localStorage` is scoped to the origin, so `shortlife.philoserf.com` works less like a URL and more like a key prefix for every owner's device. That is why `wrangler.jsonc` publishes only the custom domain and switches off workers.dev and preview URLs explicitly (6656c69). Every extra address is somewhere an owner could program a device that the real address will never see. The rule was written down in 5118933. By then the site had already changed origin once, from `philoserf.github.io/shortlife/` to its own subdomain (f40c172). Under the browser's origin model, any device programmed at the old address stayed there: the redirect carries the visitor, not their storage. Nothing records whether anyone was affected. The later move from GitHub Pages to a Worker kept the hostname, and that is what made it safe. Treat any change of hostname, any second domain, or any preview deploy as a storage migration, not a DNS change.

**The Worker publishes `public/` and nothing else.** That boundary is principled. The repo root holds tooling and docs that must never reach the site, and `task serve` runs `wrangler dev` so the local server 404s on the same paths production does.

**The Cloudflare zone is the thinnest seam.** The page makes no network calls of its own, and the privacy promise in the hint, the README and the meta description rests on that. But the page that visitors get is not exactly the file. The zone adds markup at the edge. A fetch on 2026-10-13 found two additions: a hidden `/cdn-cgi/content` link after `<body>`, and an inline script that loads `/cdn-cgi/challenge-platform/scripts/precursor/main.js`. It found no Rocket Loader, which CLAUDE.md listed as injected until #19 replaced that list with a description by mechanism. Scripts added this way run in the page's origin and can read `localStorage` just as the page's own code can. Analytics has gone into the page and back out once already: a beacon was added in 7ade181 and removed the next day in ccd5ded, and the zone was still injecting Cloudflare Web Analytics' third-party beacon (`static.cloudflareinsights.com`) on 2026-10-18. Since #20 the page enforces the promise rather than relying on convention. A Content-Security-Policy `<meta>` lets scripts load only from the page's own origin and send network calls only there. That covers zone-injected scripts too, so the beacon is blocked while same-origin `/cdn-cgi/` traffic still works. What the policy can't stop is a same-origin script reading `localStorage` and sending it back to the zone's own `/cdn-cgi/` endpoints. The promise now rests on the CSP plus trusting the zone's first-party scripts, not on zone settings in general.

**Markup and script share the defaults.** The markup holds what shows before the script runs, and the script restates it in `stop()`. That is a convention rather than a principle. What it buys is a page that reads sensibly before, or without, its script.

**The only automated gate is formatting.** `task check` (prettier) is the only check, and `task deploy` runs it first. There are no tests and no CI, and pushing deploys nothing. The behaviours the authors treated as load-bearing are recorded in commit messages instead, often with the evidence used to confirm them: "confirmed against the pre-fix build", "360 before, 0 after". Read the history as the test suite. The `LE` block is exempt from prettier through `// prettier-ignore`, and its 6-space indentation is maintained by hand.

## What the design makes cheap, and what it doesn't

Cheap, because the design anticipates it:

- adding a country: one line in `LE`, in alphabetical order after World
- refreshing figures: change the values, keep the keys, and every device updates on its next load
- adding a derived display: the four places listed above
- adding a stored field: keep it optional in `derive()`
- restyling the beyond-expectancy state: CSS only

Requires rethinking something fundamental:

- translating country names, because each key is also a label
- more than one owner or device per browser: one storage key, one `cfg`, and `start` / `stop` assume a single device
- syncing or sharing a device across machines, which collides with the privacy promise and the no-network rule, the product's core ethic
- conditional life expectancy, which changes what the object means
- a new hostname, which is a storage migration

A maintainer who doesn't hold this theory would most likely do damage in one of these ways:

- renaming `LE` keys during a data refresh
- storing derived values to "save work"
- changing `STORE` or the serial hash
- toggling `.flipped` outside `showFace()`
- adding a display node that `stop()` never resets
- adding a route or alias in `wrangler.jsonc`
- writing the owner's name with `innerHTML`, where today every display node is set only through `textContent`
- putting a beacon back in the page

## Uncertainties

- **Refreshing `LE` makes the number jump.** Because nothing derived is stored, a refresh moves every owner's percentage the moment they next load the page. Before 1.1.0 the table held figures matching no single WHO year, with Nigeria 7 to 9 years low. #17 re-sourced all 37 entries from the GHO 2023 estimates, which lowered a 40-year-old Nigerian man's percentage from 73.4% to 65.1%. 1.1.0 shipped that jump with a note in the CHANGELOG and nothing on the page. Whether a memento that ticks should ever jump is still not recorded as a rule; the next refresh will raise the question again.
- **Whether `no-anim` does what its comments say.** The CSS comment and CLAUDE.md both say the script picks the opening face "after first paint" and that `no-anim` keeps that from animating. As I read the CSS Transitions model, removing the class one `requestAnimationFrame` later doesn't guarantee that. If a style was already computed before the script ran, the `.flipped` toggle and the class removal land in the same style change and the flip animates. If no style was computed yet, there is no before-change style and nothing animates, with or without the class. The live page currently runs the inline script during parsing (no Rocket Loader was observed), so the second case is probably the common one. I tried to watch this in Safari, but the automation tab was hidden, `requestAnimationFrame` never fired, and the test showed nothing. Treat the mechanism as unverified. Forcing a style flush before removing the class would make it hold in both cases.
- **Seven decimals.** The precision produces a visible tick every few seconds. Whether that is the reason, or whether it copies the physical device's display, the code doesn't say.
- **Why the stat line is outside the card.** I read the age, expectancy and remaining line as being _about_ the device rather than part of it, which would explain why the d4a5a58 move left it below the card. That is an inference from the layout, not something stated anywhere.
- **The physical product.** The plate's "Ghent, Belgium · 120 × 50 × 20 mm · USB-C", the "1 000 000" edition size and the formula are all taken by the code as facts about the real Shortlife v3. I haven't checked them against the product.
- **What the zone injects.** I observed it once, with `curl`, using two user agents. Which zone settings produce it, whether that changes over time, and whether Rocket Loader was ever on the Worker-served page are not visible from here.

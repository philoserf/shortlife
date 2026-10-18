# Changelog

## 1.1.0

### Changed

- Life-expectancy figures are re-sourced from the WHO Global Health Observatory's 2023 estimates for all 37 entries. The old table matched no single WHO year. **Every saved device's percentage will move on its next load.** Nigeria moves most: a 40-year-old man goes from 73.4% to 65.1%. Country names are unchanged, so every saved device keeps its country.
- A Content-Security-Policy now enforces "everything stays in your browser": no script on the page can load from or send to another origin. That includes the Cloudflare Web Analytics beacon the hosting zone was injecting, which no longer runs.
- The site moved from `philoserf.github.io/shortlife/` to `shortlife.philoserf.com`, now served by a Cloudflare Worker. The old address now returns 404, and a device saved there can't follow: browser storage belongs to an address, so program the device again at the new one.
- The screen redraws only the digits that changed instead of rebuilding every frame.

### Fixed

- The turned-away face of the device can no longer be reached with the keyboard, and the flip honours reduced-motion settings.
- A browser that blocks site data no longer leaves the page stuck on a blank screen.
- The age line no longer shows "365d" in the last hours before a birthday.
- A stored setup is checked once, the same way on load and on Program, so an unusable one opens the programming panel instead of a broken screen.

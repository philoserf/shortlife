# Shortlife

A browser emulation of Dries Depoorter's [Shortlife v3](https://driesdepoorter.be/product/shortlife-v3/) — a small clock that displays what percentage of your life you've lived.

**[Try it →](https://shortlife.philoserf.com/)**

Enter your sex, birthday, and country; the device shows `age / WHO-life-expectancy × 100` and ticks up in real time. Life-expectancy figures are WHO Global Health Observatory (life expectancy at birth, by country and sex), bundled in the page.

## Use

The device opens on its back. Fill in the panel, press **Program device**, and it flips to the ticking front — and opens there on every later visit.

- `↻` flips the device by hand; `⚙` on the back reopens the programming panel.
- Settings persist in `localStorage`; **Reset** clears them and returns the panel.
- `+` / `−` adjust screen brightness.
- All personal data stays in the browser — it is never sent anywhere. The page loads Cloudflare Web Analytics, which counts visits and sends no personal data.

## Run locally

Open `index.html` in any browser. No build, no server, no dependencies.

Everything — markup, styles, the tick logic, and the life-expectancy table — lives in that one file.

A memento, not medical advice.

## License

[MIT](LICENSE) © Mark Ayers

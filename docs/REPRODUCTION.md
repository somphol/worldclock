# Reproduction book

How to rebuild **worldclock** from scratch if the source is lost, or re-verify that a
rebuild matches the release. Read [ARCHITECTURE.md](ARCHITECTURE.md) first for the map.

> **The source is the truth.** `index.html` at HEAD on `main` is the release. Rebuild
> only when the file is unavailable or you are deliberately making a variant.

## 1. What is being reproduced

A single-file **world clock dashboard**:

- **Layout** — header with brand, local time and controls; a responsive grid of city
  cards; a searchable slide-over picker; an empty state with "restore defaults"; footer
  and toast.
- **Cards** — city name, zone subtitle, day/night icon, SVG analog clock, digital time
  (12h or 24h), date line, and chips for UTC offset, relative difference and zone
  abbreviation.
- **Data** — 141 curated cities plus every IANA zone reachable through
  `Intl.supportedValuesOf('timeZone')`.
- **Behaviour** — live tick via `requestAnimationFrame`, 30-second offset resync, tab
  pause, smooth or stepped second hand depending on `prefers-reduced-motion`.
- **Persistence** — cities, theme and 12h/24h choice in `localStorage`.
- **Delivery** — one `index.html`, no build, no dependencies, no runtime network calls.
  GitHub Pages, `main` / root.
- **Release** — v1.0.0.

## 2. Requirements matrix

| # | Requirement | Implementation | Where |
| --- | --- | --- | --- |
| 1 | Show several world cities at once with live clocks | `DEFAULTS`, `createCard`, `frame` | script |
| 2 | Analog and digital read-outs per card | `clockSvg`, `updateCardText` | script |
| 3 | Accurate offsets/DST without a tz library | `zoneOffsetMs`, `syncOffsets` | script |
| 4 | Add any city or IANA zone | `searchCities`, `addCity`, `Intl.supportedValuesOf` | script |
| 5 | Remove a city; restore defaults | `removeCity`, `restoreDefaults` | script |
| 6 | Light/dark theme with no flash | head script, `applyTheme` | head + script |
| 7 | 12h/24h switch, remembered | `setHour12`, `worldclock.format.v1` | script |
| 8 | Persist the dashboard locally | `readCities`/`saveCities` | script |
| 9 | Responsive, keyboard accessible, reduced motion | CSS breakpoints, picker keys, media query | CSS + script |
| 10 | Static hosting | `.nojekyll`, `sitemap.xml`, `robots.txt` | repo root |

## 3. Build order

1. **Skeleton** — `index.html` with head meta, the pre-paint theme script, and token
   blocks for both themes.
2. **Shell** — header (brand, local time, theme/format buttons), grid container, empty
   state, footer, toast.
3. **Data** — `RAW_CITIES` tuples, then `CITY_LIST`, `DEFAULTS`, `POPULAR`, `LS`,
   `state`.
4. **Time core** — `zoneOffsetMs` and `syncOffsets` first, then the cached formatters.
   Get the offset maths right before drawing anything.
5. **Cards** — `createCard` markup and `clockSvg`, then `updateCardText`.
6. **Loop** — `frame`, `updateHeader`, reduced-motion handling.
7. **Controls** — theme, format, remove, restore defaults.
8. **Picker** — search scoring, IANA fallback, keyboard and focus management.
9. **Docs and metadata** — README, CHANGELOG, sitemap, social card, this book.

## 4. Acceptance checks

A rebuild is done when all of these pass in a desktop Chromium browser:

1. Six default cities render, each with a moving analog clock and a digital read-out
   that agrees with the system clock to the second.
2. Bangkok shows `+0` relative to a Bangkok local machine; London/San Francisco show the
   right signed difference and, across a date boundary, `Yesterday`/`Tomorrow`.
3. Offset chips stay correct after changing the OS time zone and reloading.
4. `/` opens the picker; typing `kath` finds Kathmandu (`Asia/Kathmandu`, +05:45);
   Enter adds it and the card appears.
5. A city can be removed; the empty state appears when the last one goes; "restore
   defaults" brings the six back.
6. Theme and format survive a reload; with storage blocked the app still runs on
   defaults.
7. Enabling OS reduced motion makes the second hand step instead of sweep.
8. `python3 -m http.server` serves it, and `file://` opens it too.

Record what you ran in [TESTING.md](TESTING.md) rather than assuming.

## 5. Provenance

| Item | Value |
| --- | --- |
| Initial build | `b48a9f2` "Build real-time world clock dashboard" |
| Live link added | `0f0457f` |
| Social card + README header | `d628cc5` |
| Repo docs and community files | `4af6e80` |
| Release docs | this commit (AGENTS.md + `docs/`) |
| Author | Somphol Boonjing, MIT |

Do not invent a version history: `CHANGELOG.md` is the record.

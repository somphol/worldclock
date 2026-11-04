# Architecture

`worldclock` is one HTML document. There is no build, no module system and no
dependency. This document maps the inside so a change can be made without reading all
1,887 lines.

## 1. Document layout

| Lines | Block | Contents |
| --- | --- | --- |
| 1–23 | Head | meta, description, colour-scheme hints, data-URI favicon, pre-paint theme script |
| 24–912 | CSS | design tokens, base, backdrop, header, picker, grid/cards, empty state, footer/toast, responsive, reduced motion |
| ~914–1077 | Data | `RAW_CITIES`, `CITY_LIST`, `DEFAULTS`, `POPULAR`, `LS`, `state`, DOM refs, `cards` |
| ~1077–1887 | Script | storage, time math, formatters, rendering, controls, picker |

The CSS has two token sets, `html[data-theme="dark"]` and `html[data-theme="light"]`.
Components only reference the tokens, so a theme change never needs component edits.

## 2. Data model

```js
RAW_CITIES = [ ['Bangkok', 'Thailand', 'TH', 'Asia/Bangkok'], ... ]   // 141 entries
CITY_LIST  = RAW_CITIES.map(([city, country, cc, zone]) => ({
  city, country, cc, zone, id: zone + '::' + city
}))
DEFAULTS   = CITY_LIST.slice(0, 6)      // Melbourne, San Francisco, London, Bangkok, Tokyo, New York
POPULAR    = 14 named cities used as picker suggestions
```

`state` is the whole runtime model:

```js
state = {
  cities: [],        // [{city, country, cc, zone}] — order defines grid order
  hour12: true,
  localZone: 'UTC',  // resolved once from Intl
  localOffset: 0,
  headerBucket: -1,  // last second rendered in the header
  theme: 'dark'
}
```

Per-card runtime state lives on the DOM node (`card.offset`, `card.bucket`,
`card.chipsSig`); there is no separate card object.

## 3. Storage

| Key | Shape | Default when missing/invalid |
| --- | --- | --- |
| `worldclock.cities.v2` | JSON array of `{city, country, cc, zone}` | `DEFAULTS` (first six) |
| `worldclock.theme.v1` | `"light"` or `"dark"` | system preference |
| `worldclock.format.v1` | `"12"` or `"24"` | `"12"` |

All reads and writes are wrapped in `try/catch`: unavailable storage falls back to
in-memory defaults instead of throwing. `readCities()` validates zones with
`isValidZone()` before accepting them.

## 4. Time math

`zoneOffsetMs(zone, date)` is the only place offsets are derived:

1. Format the date in `zone` with `hourCycle: 'h23'` and pull the parts out.
2. Rebuild that wall-clock reading as if it were UTC.
3. Return `wallAsUtc - date.getTime()`.

There is no DST table. `syncOffsets(now)` recomputes the local offset and every card
offset, then clears each card's `chipsSig`, forcing the next tick to redraw chips. It
runs at startup and again whenever `frame()` notices `now - lastSync > 30000`.

Formatters are cached per zone in four `Map`s (`timeFmtCache`, `dateFmtCache`,
`shortFmtCache`, `abbrFmtCache`), because constructing `Intl.DateTimeFormat` is the
expensive part of a per-second update.

## 5. Rendering

`frame()` runs continuously via `requestAnimationFrame`:

1. Re-arm the next frame, then bail if `document.hidden` — a background tab costs
   nothing.
2. Every 30 s, resync offsets.
3. Update the header clock.
4. For each card, compute `localMs = now + card.offset` and set the three hand
   rotations. With reduced motion, `handMs` is floored to the second so the hand steps
   instead of sweeping.
5. When the whole-second bucket changes, call `updateCardText()` — digital time, date
   line, day/night icon, offset and abbreviation chips.

Card DOM is created once by `createCard()` (from `render()`); updates only touch text
nodes and transforms, so layout is not rebuilt every second.

## 6. Controls and picker

- **Theme** — `applyTheme()` sets `documentElement.dataset.theme` and persists the
  choice; `toggleTheme()` flips it. The head script applies the stored/system value
  before first paint.
- **Format** — `setHour12()` updates `state`, storage, the toggle UI and every card's
  text.
- **Picker** — `openPicker()`/`closePicker()` manage focus and an `aria-expanded` state.
  `searchCities(query)` scores curated entries (`scoreCity`), then appends matches from
  `Intl.supportedValuesOf('timeZone')` when fewer than 12 results exist. Keyboard:
  `/` opens, `↑`/`↓` move, `Enter` adds, `Escape` closes.
- **Empty state** — when `state.cities` is empty, the grid hides and the empty panel
  offers "restore defaults" (`restoreDefaults()`).

## 7. Change guide

| To change… | Touch |
| --- | --- |
| Colours, radii, spacing | the two token blocks near the top of the `<style>` |
| Built-in city list | `RAW_CITIES` (keep tuple order `[city, country, code, zone]`) |
| Default dashboard | `DEFAULTS` |
| Picker suggestions | `POPULAR_NAMES` |
| Update cadence | `frame()` and the 30 s resync threshold |
| Storage shape | bump the key version and update `readCities`/`saveCities` |
| Card layout | `createCard()` plus the `.card` CSS rules |

Keep the file self-contained. A new external asset or fetch is a product change, not a
refactor.

# AGENTS.md — WORLD CLOCK

> Handoff for the next maintainer or agent. Rebuild detail is in
> [docs/REPRODUCTION.md](docs/REPRODUCTION.md); the technical map is
> [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## 1. What this is

A static, single-file **world clock dashboard**: one HTML document containing markup,
CSS and JavaScript, with 141 built-in cities, a searchable time-zone picker, SVG analog
clocks, digital readouts, relative-offset badges, light/dark themes and a 12h/24h
switch. All time math comes from `Intl.DateTimeFormat`; there is **no build step, no
package manager, no dependency and no network request at runtime**. State lives in
`localStorage` only.

It is **not** a server-backed service, not a scheduling or alarm app, and not a
framework starter. Live copy: <https://somphol.github.io/worldclock/>.
Local master: `~/Projects/worldclock`. Current release: **v1.0.0**.

## 2. Commands

```bash
cd ~/Projects/worldclock

# run (no install, no build)
open index.html                   # works from file:// too
python3 -m http.server 8000       # or http://localhost:8000/

# verify
curl -s -o /dev/null -w '%{http_code}\n' https://somphol.github.io/worldclock/
bash -c 'for f in preview.png docs/social-card.png; do test -f "$f" || echo "missing $f"; done'

# deploy — Pages serves the repository root of main; there is no workflow
git push origin main
```

The verification matrix is in [docs/TESTING.md](docs/TESTING.md). There is no CI and no
test runner in the repository; checks are manual plus the live HTTP probe.

## 3. Architecture

One file, `index.html` (≈1,887 lines), in four blocks:

1. **Lines 1–23 — head.** Meta, favicon data-URI, and a pre-paint inline script that
   reads `worldclock.theme.v1` (falling back to `prefers-color-scheme`) so the correct
   theme is set before first paint.
2. **Lines 24–912 — CSS.** Design tokens under `html[data-theme="dark"|"light"]`, then
   header, picker panel, grid, cards, responsive and reduced-motion rules.
3. **Lines ~914–1077 — data and state.** `RAW_CITIES` (141 `[city, country, cc, zone]`
   tuples), `CITY_LIST` (adds `id = zone + '::' + city`), `DEFAULTS`, `POPULAR`, the
   `LS` storage keys and the single `state` object.
4. **Lines ~1077–1887 — behaviour.** Storage helpers, time-zone math, formatter caches,
   card rendering, the animation loop, theme/format controls, picker search and
   keyboard handling.

Key paths:

- **Time math** — `zoneOffsetMs(zone, date)` derives an offset by comparing the zone's
  `Intl` wall clock with UTC; nothing is hard-coded. `syncOffsets` recomputes every card
  every 30 seconds, so DST transitions are picked up without a reload.
- **Rendering** — `frame()` runs on `requestAnimationFrame`, returns immediately when
  `document.hidden`, and moves second/minute/hour hands by transform each frame. Text
  (digital time, date, chips) is refreshed once per second per card, keyed by
  `card.bucket`.
- **Reduced motion** — when `prefers-reduced-motion: reduce` matches, the second hand
  steps once per second instead of sweeping.
- **Persistence** — `worldclock.cities.v2` (array of `{city, country, cc, zone}`),
  `worldclock.theme.v1` (`"light"`/`"dark"`), `worldclock.format.v1` (`"12"`/`"24"`).
  Every access is wrapped in `try/catch`, so private mode degrades to defaults.
- **Picker** — `searchCities` scores the curated list, then tops up with
  `Intl.supportedValuesOf('timeZone')` so any IANA zone is reachable. `/` opens it,
  arrow keys move, Enter adds, Escape closes and restores focus.

## 4. Decisions and constraints

- **Single file is the product.** Do not split it into modules or introduce a bundler;
  the offline, copy-anywhere property is the point.
- **Never hand-roll offsets or DST rules.** Always go through `Intl`; the browser's tz
  database is the source of truth.
- **IDs are `zone::city`.** The same zone may be added twice under different labels;
  keying by zone alone would collide.
- **Storage keys are versioned.** Changing the shape means `cities.v3`, not an in-place
  migration of `v2`.
- **No runtime fetches.** The only absolute URLs in the file are the SVG namespace and
  canonical/Pages links; adding a font or API call breaks the offline guarantee.
- **Theme is set before paint** by the inline head script. Keep that script early and
  dependency-free, or the page flashes the wrong theme.

## 5. Known issues

- The README says "150+ cities"; the curated list is **141**. The rest of the reachable
  set comes from `Intl.supportedValuesOf`. Numbers in the README are approximate.
- No automated tests and no CI. `docs/social-card.png` and `preview.png` are committed
  images; nothing regenerates them.
- Browser coverage in this release: Chromium only. Safari, Firefox and real mobile
  devices are **not verified**.
- Offset badges assume modern `Intl`; very old browsers without full tz data will show
  wrong offsets rather than fail.

## 6. Continuity

- Live: <https://somphol.github.io/worldclock/> (HTTP 200 at the time of writing).
- Pages: `main` branch, repository root, `.nojekyll` present, no Actions workflow.
- Local master copy: `~/Projects/worldclock`.
- First build: commit `b48a9f2`; documentation release: see `CHANGELOG.md`.

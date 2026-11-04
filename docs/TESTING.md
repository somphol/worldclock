# Testing

## Status of this release

| Evidence | When | Result |
| --- | --- | --- |
| Live Pages probe (`curl`, HTTP status) | 2026-11-03 | **200** |
| Headless Chromium DOM render, `file://` | 2026-11-03 | **6/6 cards, clocks populated** |
| Local HTTP server pass | — | **Not run** |
| Manual interactive matrix (§3) | — | **Not run** in this release |
| Safari / Firefox / real mobile | — | **Not verified** |

The documentation commit added Markdown only and did not change `index.html`. The
headless pass confirms the app boots, builds the six default cards and fills their
digital read-outs; it does **not** exercise the picker, storage, theme switching or
pointer/keyboard behaviour. Run §3 before claiming a behavioural change.

There is no CI workflow and no committed test runner. The checks below are the gate.

## 1. Commands

```bash
cd ~/Projects/worldclock

# live probe
curl -s -o /dev/null -w '%{http_code}\n' https://somphol.github.io/worldclock/

# headless render (macOS, Chrome installed)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --virtual-time-budget=4000 \
  --dump-dom "file://$PWD/index.html" > /tmp/wc-dom.html

grep -c 'data-id=' /tmp/wc-dom.html                 # expect 6
grep -o 'data-id="[^"]*"' /tmp/wc-dom.html          # expect the six defaults

# local server
python3 -m http.server 8000                         # http://localhost:8000/
```

## 2. Static results (2026-11-03)

- Live site returns `200`; the served document is ~60 KB.
- Headless Chromium produced ~80 KB of rendered DOM from `file://`.
- `data-id` attributes: **6** —

  `Australia/Melbourne::Melbourne`, `America/Los_Angeles::San Francisco`,
  `Europe/London::London`, `Asia/Bangkok::Bangkok`, `Asia/Tokyo::Tokyo`,
  `America/New_York::New York`.
- Digital read-outs are populated (for example `3:20`, `8:20`, `4:20` at test time),
  and chips carry derived text such as `+19 hrs ahead` and `Tomorrow, 3:20 PM`.
- No missing local asset: `preview.png` and `docs/social-card.png` exist and are the
  only images referenced by the README.

## 3. Manual matrix

Run in a desktop browser, ideally once over `file://` and once over
`http://localhost:8000/`.

| # | Check | Expected |
| --- | --- | --- |
| 1 | Load the page | six cards, each clock advancing once per second |
| 2 | Compare a card with the OS clock for that zone | same minute, same second |
| 3 | Offset chip vs your machine | correct sign and value; `Yesterday`/`Tomorrow` across a date boundary |
| 4 | Press `/` | picker opens, focus in the search field |
| 5 | Type `kath` | Kathmandu appears; Enter adds it as a card |
| 6 | Add an odd zone (e.g. `Pacific/Chatham`) | card renders, offset is `+12:45`-style, not a crash |
| 7 | Remove cards until none remain | empty state appears |
| 8 | "Restore defaults" | the six default cities return |
| 9 | Toggle theme | switches instantly, no flash on reload |
| 10 | Toggle 12h/24h | every card and the header follow, choice persists |
| 11 | Reload | cities, theme and format retained |
| 12 | Block storage (private window / site data off) | app still runs on defaults, no error dialog |
| 13 | Enable OS reduced motion | second hand steps once per second |
| 14 | Narrow the window to ~360 px | single-column grid, no horizontal scroll |
| 15 | Tab through the page | visible focus, controls reachable, picker traps focus while open |

## 4. What a failed check means

- **Wrong offset** — suspect `zoneOffsetMs` or stale formatter caches, not the UI.
- **Clocks frozen** — check `frame()` early return (`document.hidden`) and that
  `requestAnimationFrame` is actually scheduled.
- **Cards missing after reload** — check `readCities()` validation: an unknown zone in
  `worldclock.cities.v2` is dropped by design.
- **Flash of wrong theme** — the inline head script must stay before the stylesheet.

## 5. Not verified

- Safari, Firefox, Edge, and real iOS/Android devices.
- Screen-reader output (markup has labels and live regions, but no AT pass was run).
- Long-run drift: no soak test beyond a few minutes of headless rendering.

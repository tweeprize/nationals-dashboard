# Nationals Dashboard

A single-file static web app (`index.html` — no build step, no server) that shows
Washington Nationals stats against the rest of MLB. All data comes from the public
MLB Stats API (`statsapi.mlb.com`), fetched client-side; the CSP in `<head>` locks
`connect-src` to that one host on purpose — don't add other data sources without
updating it.

Opening `index.html` directly (`file://`) works fine: the MLB API allows
cross-origin requests from a `null` origin, so no local server is required to run
or test this app.

## Features and expected behavior

**Team select (header)** — Switch which of the 30 MLB teams the dashboard is
about. Defaults to the Nationals (team id 120). Reordering: if the selected
team has a game today, that opponent is pinned second in the list (right after
the Nationals) so it's quick to switch to.

**Season / Last 7 toggle** — Season stats load by default. "Last 7" is disabled
until at least 7 completed games exist in the schedule, and aggregates the
selected team's last 7 games' box scores into rate stats client-side.

**Team Rankings tab** — Bar chart ranking all 30 teams by a chosen stat
(Offense: OPS/AVG/OBP/SLG/HR/R/SB, Pitching: equivalent stats), selected
team highlighted, gold dashed line = league average.

**Players tab** — Sortable table of the selected team's individual batters/pitchers
for the current season/window. Minimum 20 PA (batting) or 5 IP (pitching) to
appear, so call-ups with token appearances don't clutter it.

**Pitching tab — pitch location chart**:
- Pick a pitcher (dropdown, sorted by innings pitched), then a specific game
  they appeared in, to see a scatter chart of every pitch's location
  (catcher's-view coordinates), colored/shaped by pitch type, with outcome
  legend and type/outcome click-to-filter toggles.
- A "Current Inning" vs "Full Game" toggle appears only for the pitcher's most
  recent appearance if it's today's game.
- **Current-pitcher marker (⚾)**: during a live Nationals game, whoever is
  actively pitching right now is prefixed with ⚾ and moved to the top of the
  pitcher dropdown. Before first pitch, this is the probable starter; once the
  game is live, it's whoever the live feed says is on the mound *at the moment
  the marker was last computed* — see refresh caveat below.

**Refresh button (↺, header)** — Re-pulls whatever the current view needs
without reloading the whole page:
  - Team/Players tab → refetches season/last-7 stats.
  - Pitching tab, pitcher+game already selected → refetches just that game's
    pitch-by-pitch data (fast path, doesn't refetch season stats).
  - In both pitching-tab cases, it must *also* re-check who's currently
    pitching (`refreshCurrentPitcher()`) and re-render the pitcher dropdown —
    this was previously missed on the fast path, which is why the ⚾ marker
    would go stale mid-game until a full page reload. Any future change to
    `refreshData()` should preserve this.

**Persistence** — Selected team, tab, and (on the pitching tab) selected pitcher
and game are saved to `localStorage` and restored on next load.

## Testing

No automated tests exist yet. To verify a change actually works, drive a real
browser against the file — do not rely on reading the code alone:

```bash
npm install playwright   # in a scratch dir; browsers via `npx playwright install chromium`
```

Then a Playwright script can `page.goto('file:///…/index.html')` directly — no
dev server needed. For anything touching the live-game pitcher logic, mock the
`**/api/v1.1/game/*/feed/live` route to simulate a pitching change rather than
waiting for a real one; see git history around the ⚾-marker refresh fix for a
worked example of this pattern.

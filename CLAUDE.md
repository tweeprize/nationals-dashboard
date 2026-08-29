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
appear, so call-ups with token appearances don't clutter it. Columns are driven
by `PLAYER_HIT_STATS`/`PLAYER_PITCH_STATS` (index.html) — adding a stat is a
one-line addition as long as the MLB Stats API returns that field on the
hitting/pitching split (batting currently: OPS, AVG, OBP, SLG, HR, SB). The SB
column displays as "made/attempted" (e.g. `17/19`) via a stat config's
`attemptsKey: 'caughtStealing'` — sorting and the vs-league-average color still
key off the raw SB number alone, not the attempt total. That `attemptsKey`
mechanism is generic (see `renderPlayerTable()`'s `attempts()` helper), so any
future made/attempted-style stat can reuse it the same way.

**Pitching tab — pitch location chart**:
- Pick a pitcher (dropdown, sorted by innings pitched), then a specific game
  they appeared in, to see a scatter chart of every pitch's location
  (catcher's-view coordinates), colored/shaped by pitch type, with outcome
  legend and type/outcome click-to-filter toggles.
- A "Current Inning" vs "Full Game" toggle appears only for the pitcher's most
  recent appearance if it's today's game.
- **Current-pitcher marker (⚾)**: works for *whichever team is selected*, not
  just the Nationals — during that team's live game, their own active pitcher
  is prefixed with ⚾ and moved to the top of the dropdown. Before first pitch,
  this is the probable starter; once live, it's that team's own most recently
  used pitcher (`computeCurrentPitcherId()` reads
  `liveData.boxscore.teams.{home,away}.pitchers`, keyed to whichever side is
  the selected team — deliberately *not*
  `liveData.plays.currentPlay.matchup.pitcher`, which is whoever's on the
  mound *right now* and therefore only ever correct for the team currently on
  defense; the other team's own pitcher would look unmatched/missing the
  moment they're batting instead). See refresh caveat below.
- The pitcher dropdown normally hides anyone under 5 IP for the selected team
  this season (see Players tab), but always includes the current pitcher
  regardless — otherwise a recent trade/call-up who's actively pitching today
  wouldn't have an `<option>` to be marked at all, not just a missing ⚾.
- **Current Batter scope button**: a third button ("vs <name>") joins Full
  Game / Current Inning — all three are one mutually-exclusive scope
  (`state.pitchScope`, see `pitchInScope()`), so clicking it filters the chart
  down to just the pitches of the at-bat in progress, the same way Current
  Inning filters to the current inning. It only appears when that filter would
  actually show something: the selected game is today's *live* game, the
  selected pitcher is the one `liveData.plays.currentPlay.matchup.pitcher`
  says is actually on the mound right now (`computeCurrentBatter()` returns
  the batter/pitcher/`atBatIndex` from `currentPlay`; `setPitcherGame()` then
  checks `cb.pitcherId === state.pitcherId`), and at least one pitch of that
  at-bat has already landed in `state.pitchData` (via its `atBatIndex`, added
  in `extractPitches()`). Gating on the pitcher match (unlike the informational
  ⚾ marker, which is deliberately *not* gated this way) is intentional here:
  the button filters *this pitcher's own chart*, so it's only ever meaningful
  when they're the one actually facing that batter — for the other ~half of a
  live game, the selected team is batting and their own pitcher is on the
  bench, so the button simply stays hidden. Recomputed on pitcher/game
  selection and by the refresh button's fast path (both funnel through
  `setPitcherGame()`); auto-resets to Full Game if the scope was `'batter'`
  and the button disappears (pitching change, inning ended, refresh landed on
  a non-live game).
- **ABS challenge rings**: a pitch that was challenged under the 2026
  Automated Ball-Strike challenge system gets a gold ring around its point
  (a non-interactive overlay dataset, `pointHitRadius:0`, flagged
  `_isChallengeRing` so the customPointShapes plugin and the tooltip filter
  both skip it) instead of a new color/shape, since the chart already uses
  color for outcome and shape for pitch type. Hovering the real point
  underneath shows who challenged and the ruling as a second tooltip line.
  Source: `ev.details.hasReview` / `ev.reviewDetails` on each pitch event from
  `playByPlay` — same endpoint already used for the chart, no extra fetch.
  A season-wide challenge stat (e.g. team challenge success rate) would be a
  much heavier feature — MLB's stats endpoints don't aggregate this, so it'd
  require pulling full play-by-play for every game played, not just the one
  selected — see git history / conversation context before attempting that.

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

## Keeping this file current

There is no other spec for this app — this file is the only record of intended
behavior beyond the code itself. Whenever a change alters, adds, or removes
user-facing behavior described above, update the relevant section in the same
commit as the code change. If a change is purely internal (refactor, styling,
no behavior change), this file doesn't need touching.

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

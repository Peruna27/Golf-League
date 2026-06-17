# Golf League — Fantasy Leaderboard for Golf Majors

A mobile-first fantasy leaderboard for golf majors. One repo, one subfolder per tournament.
Three friends (Mike, Reed, Joe) draft 8 players each; lowest combined score of their top 4 wins.

> **READ THIS BEFORE STARTING A NEW TOURNAMENT.** Every design and functionality decision below
> was settled through trial and error across the 2026 Masters and 2026 PGA Championship.
> Don't re-litigate them — copy what works and only deviate when there's a real reason.

---

## Product Spec

- **League rules**: 3 teams, 8 players each. Lowest combined score of each team's top 4 counting players wins (golf — low score wins).
- **Phone-friendly**: optimized for one-handed use on a phone. Everything fits without horizontal scrolling on a 360–414px viewport.
- **Auto-refresh** every 60 seconds + manual Refresh button. Pauses when tab is hidden (visibility API), fires once immediately when tab becomes visible again.
- **Tap any player row** (in standings card or full leaderboard) → TV-style scorecard modal with hole-by-hole detail per round, color-coded birdies/eagles/bogeys/doubles.
- **Tournament-themed branding**: each major gets its own colors, logo, and copy matching the official event branding.
- **Cache last good data** in localStorage so the page still works if the API is down. Cache key is namespaced per tournament.
- **No dummy data.** When the API returns nothing, show an empty leaderboard with scheduled tee times in the Thru column — never fabricate fake scores. ESPN's `/scoreboard?dates=<tournament-day>` returns the full 156-player field with status="Scheduled" days before play begins, and tee times come from the per-competitor `/status` endpoint, so a "scheduled" state is fully populated without making anything up.

---

## Repo Layout

```
Golf-League/
├── index.html                    # Landing page — cards linking to each tournament
├── 2026_Masters/index.html       # Masters: green theme, "Woo Woo!" league name
├── 2026_PGA_Championship/        # PGA Championship: navy theme
├── Source/                       # Excel team rosters (user maintains)
│   └── 2026_<Tournament>.xlsx
└── CLAUDE.md                     # this file
```

- **GitHub repo**: `Peruna27/Golf-League` (was `Masters-League` originally — renamed during the PGA Championship build).
- **Live URL**: `https://peruna27.github.io/Golf-League/`
- **Git identity**: `Local User <local@local>` (placeholder, set per global CLAUDE.md rules).
- **Pages**: serves from `main` branch root. Push → auto-deploys in ~60 seconds. Landing page lives at root.

---

## Data Sources (this is the gold)

### Masters Tournament
**Use the official Masters.com feed.** Best data of any tournament — per-hole scores, tee times, scoreboard-ready everything.

```
https://www.masters.com/en_US/scores/feeds/{YEAR}/scores.json
```

Structure: `data.player[]`, each with `full_name`, `pos`, `topar`, `today`, `thru`, `teetime`, and `round1.scores` … `round4.scores` (18-element arrays of hole scores). Sort by `pos` parsed numerically, not by the pipe-delimited `sort_order` field (its format changes between rounds).

### PGA Championship (and likely other PGA-of-America events)
**ESPN's main `/golf/pga/leaderboard` endpoint returns ZERO events for the PGA Championship** — it's run by the PGA of America, not the PGA Tour. The fix is the `/scoreboard` endpoint with an explicit date:

```
https://site.api.espn.com/apis/site/v2/sports/golf/pga/scoreboard?dates=YYYYMMDD
```

This DOES return the event. Try today first, then each tournament day (Thu–Sun) as fallbacks. The bare `/leaderboard` endpoint can be the final fallback in case it ever starts working.

### Tee Times (any tournament)
ESPN's scoreboard does NOT include tee times. Use the per-competitor status endpoint on the core API:

```
https://sports.core.api.espn.com/v2/sports/golf/leagues/pga/events/{EVENT_ID}/competitions/{EVENT_ID}/competitors/{ATHLETE_ID}/status?lang=en&region=us
```

Returns `teeTime` (ISO 8601 UTC), `startHole`, `thru`, `position.displayName` (with T-prefix already), and `type.state` (`pre` / `in` / `post`).

**Don't fetch 156 of these in parallel** — browsers cap concurrent connections per origin at ~6. Batch in groups of 6–8, and only fetch for players who haven't teed off (`thru === '--'`). Cache per day in `localStorage` (tee times don't change once published).

### Finding the ESPN event ID for a new tournament

```bash
curl "https://site.api.espn.com/apis/site/v2/sports/golf/pga/scoreboard" \
  | jq '.leagues[0].calendar[] | select(.label | contains("Masters") or contains("PGA Championship"))'
```

Save that ID into the `ESPN_EVENT_ID` constant in the tournament's `index.html`.

---

## ESPN Data Structure Quirks (the parser landmines)

The `/scoreboard?dates=...` response is NOT the same shape as the bare `/leaderboard` you may know from other ESPN sports.

- **No `status` block on competitors.** Don't read `c.status.position.displayName` — it doesn't exist. Compute position from `c.order` (sequential rank) and post-process to add `T` prefix when multiple players share the same `c.score` string.
- **Score is at `c.score`** as a string: `"E"`, `"-3"`, `"+4"`.
- **`c.linescores[i]` is the i-th round** with:
  - `value`: total strokes when complete; partial strokes during play; 0 if not started.
  - `displayValue`: score-to-par as a string ("-2", "E"); `"-"` if round not started.
  - `linescores[j]`: nested per-hole array, each with `period` (hole #), `value` (strokes on that hole), `scoreType.displayValue` (cumulative score-to-par at that hole).
- **`period` is the hole number, NOT the order of play.** A player starting on hole 10 (typical morning wave) will have their first hole entry with `period: 10`. Map by `period` to put scores in the correct hole positions on the scorecard.
- **A round with no per-hole entries means the player hasn't started it yet.** Critical for the "today" / "thru" logic — use globally-determined "today's round" (max round index with any data across the field), not per-player last-round-with-data.
- **Round is complete only when its per-hole array has 18 entries.** Don't display partial stroke counts as round totals.
- **Athlete ID** is `c.id` — keep it on parsed player data so you can hit the core status endpoint for tee times.
- **Name normalization**: ESPN uses unicode characters (e.g., `Ludvig Åberg`). Normalize via `name.toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g, '')` before matching against roster search terms.
- **Roster matching is greedy-longest-first.** Sort search-term entries by length descending so `'patrick reed'` matches before `'reed'` would. Avoid one-token search terms unless the surname is unique (e.g., `'rose'` matches everyone with Rose in their name — use `'justin rose'`).

---

## Adding a New Tournament — Checklist

1. **Copy** an existing tournament folder (`2026_PGA_Championship/` is the cleanest template).
2. **Find the ESPN event ID** via the calendar query above. Update `ESPN_EVENT_ID` constant in JS.
3. **Update title, subtitle** in the header — tournament name + course + year. Tab `<title>` too.
4. **Swap colors** — change CSS variables `--masters-green`, `--counting-green`, `--counting-bg` to the tournament's brand. Keep variable names as-is to minimize diff (yes, they're misnamed at this point — fine).
5. **Logo** — Wikipedia event pages reliably have a hot-linkable PNG. Pattern: `https://upload.wikimedia.org/wikipedia/en/...`. Hot-linking is fine for a personal site for 3 friends; trying to host the logo ourselves risks trademark issues. **Don't apply `filter: invert()` blindly** — it can render a colored logo as a white square.
6. **Course pars** — update `COURSE_PARS` for the host venue (18-element array, total should equal championship par). US Open & PGA Championship typically 70, Masters 72, Open Championship 71 or 72.
7. **Read the Excel rosters** from `Source/<Tournament>.xlsx`. The file is a zip — parse it with Python's stdlib `zipfile` + `xml.etree.ElementTree` if openpyxl/pandas aren't installed (don't `pip install` without user approval). Update the `TEAMS` object. **Display order in `TEAMS` should match the Excel column order** (e.g., for 2026 PGA: Mike / Reed / Joe).
8. **`CACHE_KEY`** — change to avoid colliding with other tournaments' localStorage data.
9. **Tournament-name check** in `parseESPNData` — used to flag the status banner when the API is serving a different event. For PGA Championship: `tn.includes('pga championship') || tn.includes(courseName)`.
10. **ESPN fallback dates in `ESPN_API_URLS`** — when copying a tournament template, replace the previous tournament's date list (e.g., May 14-17 for PGA) with the new tournament's dates (e.g., June 18-21 for US Open). Today's date is tried first; the per-tournament-day URLs are fallbacks. If you forget this step, the page silently falls back to "no data" because the wrong tournament's dates only ever return the wrong tournament. **Do not** keep `loadDummyData()` as the empty-state — delete the call. ESPN populates the field days early.
11. **Landing page card** in root `index.html` — add a new card linking into the new subfolder, with course + dates and an event status tag (`Up Next` / `Live` / `Final`).
12. **Commit & push** — Pages rebuilds automatically in under a minute.

---

## Design Conventions (do not redebate every tournament)

### Typography (loaded from Google Fonts)
- **Libre Baskerville italic** — tournament title in header + section headers ("STANDINGS", "LEADERS"). Matches the Masters' classic italic serif.
- **Source Serif 4** — large numeric displays (team totals, rank numbers, leaderboard score totals). Heavy weights (800–900).
- **DM Sans** — labels, table headers, body text, and the scorecard hole/par/score cells. Weights 400–700.

### Score colors (TV golf convention)
- **Under par = red** (`#c0392b`), **even/over = dark text**. Apply via `!important` on `.score-under`/`.score-even`/`.score-over` to win specificity battles.
- Applies to: leaderboard "Tot" column, leaderboard "Tdy" column, team standings player scores, team total in standings, scorecard round headers.
- The `--score-blue` (`#2874a6`) is reserved for the **scorecard modal only** (bogey/double markers) — NOT the main leaderboard.

### Header (sticky, compact)
- Single inline row: logo image (height ~46px) + title text + subtitle.
- **NOT a vertical banner.** We tried the big centered banner with stacked title/subtitle and the user pushed back: too much vertical space.
- Background: tournament primary color (Masters green / PGA navy).
- Sticky at top during scroll. Refresh button + spinner + "Updated X:XX PM" on the second line.

### Landing page (root)
- Black/dark background with subtle radial gradient.
- Each tournament gets a card with: logo on a small white tile, name, "Course · Dates" line, and a status tag (`Live` red, `Up Next` gold, `Final` dim).
- Cards have a colored left border matching each tournament's primary color.
- Title: "Golf League" in italic Libre Baskerville gold.

### Standings card (per team)
- One card per team. **Sorted by team total ascending** (lowest = 1st).
- Rank number: gold for 1st (`#c8a951`), silver for 2nd (`#8a8a8a`), bronze for 3rd (`#a67c52`). Source Serif weight 900, ~30px.
- Team name: uppercase, Source Serif 900, 18px.
- Sub-label: "Best 4 of 8" in small DM Sans uppercase.
- Team total: Source Serif 900, ~32px, color-coded.
- Tap the team header to expand/collapse the roster (chevron arrow rotates 180°).
- **All three teams open by default** on initial render.

### Player rows inside the standings card
- **Sort order within a team**: by `scoreToPar` ascending. Cut/WD players push to the bottom regardless of their score.
- Columns: position (`T13`), status dot, player name (shortened: `R. McIlroy`), per-round mini totals, Thru (stacked), Today, Total.
- **Top 4 (counting players)**: green-tinted background + filled green dot, full opacity.
- **Bench (players 5–8)**: outlined green dot, 0.4 opacity.
- **Cut / WD**: red dot, 0.3 opacity, strikethrough on name. Sorted to bottom of the team list.

### Leaderboard (full field)
- Columns: `Pos | Team | Player | R1 R2 R3 R4 | Thru | Tdy | Tot`.
- Header row uses same column widths as data rows so columns line up.
- **All rows share `min-height: 38px`** so the Thru column's stacked label-over-value doesn't make some rows taller than others.
- **Rostered rows**: 3px solid `--masters-green` left border + light tint (~`rgba(0,103,71,0.08)`). No alternating row stripes — they fight visually with the rostered highlight.
- **Player names**: first initial + last name (`R. McIlroy`) so they fit on one line on mobile. Use the `shortName()` helper.
- **Team tag**: small uppercase team owner name in the Team column, color-coded (Mike = blue, Reed = gold, Joe = pink). Empty cell for non-rostered players.
- Show **the entire field**, not just top 30.

### "Thru" column
- **Stacked label-on-top, value-below** format. Use the `fmtThru()` helper.
- During play: `Thru` / `14`
- Round complete: `Thru` / `F`
- Not teed off yet: just the time `1:20 PM` (NO "Tee" word above — empty `&nbsp;` placeholder to keep vertical layout).
- Eliminated: `CUT` / `WD` / `DQ` (no upper label).

### "Today" column
- Color-coded the same as Total (red for under par, dark for even/over).
- Shows `--` when the player hasn't teed off in today's round (NOT the previous round's score — this was a bug we fixed; use globally-determined "today's round").

### Position display
- Tie-aware `T` prefix (`T1`, `T5`, `T11`). Compute in JS — don't trust the feed's tie behavior.
- For ESPN data: post-process after sorting by `c.order`, group by `scoreStr`, assign `T<firstOrderInGroup>` when group size > 1.

### Section header bar
- Green band (tournament primary color) with white text.
- Italic Libre Baskerville, weight 700, ~17px.
- "Standings" (NOT "Fantasy Standings" — user trimmed it) and "Leaders" (NOT "Tournament Leaderboard").

### Footer
- One line of small DM Sans muted text: "Auto-refreshes every 60 seconds · Scores via ESPN".
- Wordmark line below: tournament name in italic Libre Baskerville, colored in primary.

---

## Scorecard Modal (TV-style hole-by-hole)

Opens on tap of any player row (standings card or full leaderboard).

### Header
- Tournament primary color band.
- Player name in uppercase Source Serif 800.
- Below the name: `T13 · -7 · Team Mike` (position, total, team tag).
- Close button (×) circular, top-right.

### Body — one block per round (1 through 4)
- Round header: `Round 1` label on the left, total strokes + score-to-par on the right.
- For in-progress rounds: `Round 2 · In Progress` and the strokes show `33 thru 6` with smaller secondary text.
- Empty round: shows `Not yet played` placeholder so all four rounds always render.

### The grid (the part that took 10 iterations to get right)
- Three rows per nine: **HOLE / PAR / SCORE**. Two grids per round (front 9 then back 9).
- Grid template: `58px label | 9 hole columns (1fr each) | 38px Out/In total`.
  - Label column needs to be wide enough for "SCORE" (~58px). Don't go smaller.
- **All three rows the same font size, same font family, same weight** (12px DM Sans 700). When fonts differ at the same px size they appear different visual sizes; using one font + background tint is more reliable than fighting font metrics.
- The score row uses **regular weight (400)** so it doesn't compete visually with the colored circle/square markers.
- **Backgrounds differentiate rows**: HOLE darkest (`#dde1e8`), PAR lighter (`#eef0f4`), SCORE white.
- **Horizontal separator lines**:
  - **Dark line ABOVE the HOLE row** (top edge of the header block).
  - **Dark line BELOW the HOLE row** (between HOLE and PAR).
  - **Light line between PAR and SCORE** (NOT dark — user explicitly pushed back on this).
  - Vertical lines between cells stay light.
- All cells flex-centered vertically. Avoid `padding: Npx 0` for vertical alignment — use `display: flex; align-items: center;` so cells line up regardless of content height.
- Label cells (HOLE/PAR/SCORE in the leftmost column) get the background + uppercase + letter-spacing distinction. They inherit font size from the row so they match the numbers next to them. Don't apply a separate small font to the label cell — it'll feel out of scale.
- **Only the leftmost cell in a row is a label.** Don't class hole-number cells as `sc-row-label` — the override rule will shrink them to 9px.

### Score markers
- **Birdie** (−1): red number in red-outlined circle (20×20px).
- **Eagle** (−2 or better): white number on **solid red filled** circle. Don't use double-outline for eagle — too easily confused with birdie.
- **Bogey** (+1): blue number in blue-outlined square (20×20px).
- **Double bogey or worse** (+2): blue number in **double-outlined** square (border + outline with offset). Keep this as outlined, NOT solid — user prefers it.
- Par: plain number, no marker.

### In-progress rounds
- The scorecard populates as soon as a player tees off — don't gate it on round completion.
- **Score-to-par for in-progress rounds is computed against par-of-holes-played**, not the full course par. Otherwise a player who's −2 through 6 holes looks like they're −45 (because 25 strokes minus course par 70).
  ```js
  let strokes = 0, parPlayed = 0;
  scores.forEach((s, i) => { if (s != null) { strokes += s; parPlayed += COURSE_PARS[i]; } });
  const toPar = strokes - parPlayed;
  ```
- Round header reads `33 thru 6` (current strokes + holes played) with "In Progress" after the round label.

### Modal interactions
- Tap outside (backdrop) closes. Tap the modal itself doesn't propagate.
- Tap the × button closes.
- Body scroll is locked while the modal is open (`document.body.style.overflow = 'hidden'`).
- Fade-in animation (~220ms scale+opacity pop).
- Lookup uses `leaderboardData.find(x => x.name === playerName)` first, then falls back to `rosterName` (the TEAMS roster name) so the click works from both the standings card and the leaderboard.

---

## Auto-refresh + caching

- **Interval**: 60 seconds.
- **Visibility API**: when `document.hidden` becomes true, clear the interval. When it becomes false again, immediately call `fetchData()` once and restart the interval.
- **Refresh button**: top-right of header. Shows a spinning border while loading.
- **Last-updated text**: shown next to the refresh button. Format: `Updated 3:14 PM` or `Cached (5m ago)` when serving stale data.
- **Cache on every successful fetch**: `localStorage.setItem(CACHE_KEY, JSON.stringify({ ts, data: rawApiResponse }))`. Each tournament's `CACHE_KEY` MUST be unique (e.g., `'pga_championship_2026'`).
- **Tee-time cache** uses a separate key like `tee_times_<EVENT_ID>_<YYYYMMDD>` since tee times don't change but the schedule changes day-to-day.

---

## Hosting

- GitHub Pages, free, public repo.
- Push to `main` → auto-deploys in ~60 seconds.
- No custom domain. After a repo rename, GitHub auto-redirects the old URL for a while, but tell users the new link explicitly.

---

## Gotchas / Lessons Learned

- **`onclick` attributes that include `JSON.stringify(...)` must be in single quotes**, not double quotes. JSON.stringify produces inner double quotes that break a double-quoted attribute. Half a session was lost to this — every click was a no-op.
- **CSS specificity bites you with row-scoped overrides.** When `.sc-row-score .sc-cell` (0,2,0) and `.sc-row-label` (0,1,0) both apply to the same cell, the descendant rule wins. Use compound `.sc-row-score .sc-cell.sc-row-label` (0,3,0) to force the label styling — or better, don't try to make label cells smaller than the row; let them inherit.
- **Different fonts at the same px size render at different visual sizes** (especially serif vs sans). At 12px DM Sans 700 vs Source Serif 4 800, the serif looks noticeably bigger. Either unify the font or compensate empirically.
- **The hole row's number cells must NOT have `sc-row-label` class.** Only the leftmost "Hole" cell is a label. If you class every cell in the hole row as a label, the override rule shrinks the numbers to 9px and they look tiny next to the par row.
- **`localStorage` is per-origin**, so a single `CACHE_KEY` will collide between tournaments. Namespace with the tournament name + year.
- **Hard refresh on mobile**: close tab and reopen. iOS Safari aggressively caches HTML, so users won't see fixes until then.
- **Tee time fetch must skip already-started players** (`thru === '--'` filter) — otherwise you're making 156 needless requests per refresh.
- **Sort_order in some feeds (Masters.com) changed format between rounds.** Don't trust pipe-delimited rank fields — sort by `pos` (or the equivalent display position) parsed numerically with cut/WD players sent to bottom.
- **The Masters.com feed and ESPN /scoreboard data have totally different shapes.** Write a parser per source. Don't try to coerce one into the other's shape.
- **GitHub Pages rebuild time**: usually <60s, sometimes 2–3 minutes. Tell the user to hard-refresh if a change isn't visible.
- **`onclick=` handlers on stringified objects**: always pass a primitive (the player name) and look up the full record in the handler. Don't try to embed JSON into the attribute.
- **Trademark images**: hot-link from Wikipedia or the official CDN. Don't commit logos to the repo.
- **The `Source/` folder Excel files are not committed** (user maintains locally). The `TEAMS` object in each tournament's `index.html` is the authoritative copy at deploy time.

---

## Useful Snippets

### Read Excel without openpyxl/pandas

```python
import zipfile, xml.etree.ElementTree as ET
ns = {'s': 'http://schemas.openxmlformats.org/spreadsheetml/2006/main'}
z = zipfile.ZipFile(r'C:\path\to\file.xlsx')
ss = []
with z.open('xl/sharedStrings.xml') as f:
    root = ET.parse(f).getroot()
    for si in root.findall('s:si', ns):
        ss.append(''.join(t.text or '' for t in si.iter('{http://schemas.openxmlformats.org/spreadsheetml/2006/main}t')))
with z.open('xl/worksheets/sheet1.xml') as f:
    root = ET.parse(f).getroot()
    for row in root.find('s:sheetData', ns).findall('s:row', ns):
        for c in row.findall('s:c', ns):
            v = c.find('s:v', ns)
            val = ss[int(v.text)] if v is not None and c.attrib.get('t')=='s' else (v.text if v is not None else '')
            print(c.attrib['r'], val)
```

### YYYYMMDD for today (JS)

```js
function todayYYYYMMDD() {
    const d = new Date();
    return d.getFullYear() + String(d.getMonth()+1).padStart(2,'0') + String(d.getDate()).padStart(2,'0');
}
```

### Tie-aware position display

```js
parsed.sort((a, b) => a.position - b.position);
const firstByScore = {}, countByScore = {};
for (const p of parsed) {
    if (!(p.scoreStr in firstByScore)) firstByScore[p.scoreStr] = p.position;
    countByScore[p.scoreStr] = (countByScore[p.scoreStr] || 0) + 1;
}
for (const p of parsed) {
    const first = firstByScore[p.scoreStr];
    p.posDisplay = countByScore[p.scoreStr] > 1 ? 'T' + first : String(first);
}
```

### Greedy-longest-first name matching with accent normalization

```js
const searchEntries = [];
for (const [team, data] of Object.entries(TEAMS)) {
    for (const p of data.players) {
        for (const s of p.search) {
            searchEntries.push({ term: s.toLowerCase(), team, playerDef: p });
        }
    }
}
searchEntries.sort((a, b) => b.term.length - a.term.length);

function matchPlayer(name) {
    const lower = name.toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g, '');
    for (const e of searchEntries) {
        if (lower.includes(e.term)) return { team: e.team, playerDef: e.playerDef };
    }
    return null;
}
```

### In-progress score-to-par (only counts holes played)

```js
let strokes = 0, parPlayed = 0, holesPlayed = 0;
scores.forEach((s, i) => {
    if (s != null) { strokes += s; parPlayed += COURSE_PARS[i]; holesPlayed++; }
});
const toPar = strokes - parPlayed;
const inProgress = holesPlayed < 18;
```

### Format a Thru cell (label-over-value, special cases)

```js
function fmtThru(val) {
    if (!val || val === '--')
        return '<span class="lb-thru-label">Thru</span><span class="lb-thru-val">--</span>';
    if (val === 'CUT' || val === 'WD' || val === 'DQ')
        return '<span class="lb-thru-label">&nbsp;</span><span class="lb-thru-val">' + val + '</span>';
    if (/[AP]M/i.test(val))   // tee time — no "Tee" label, just the time
        return '<span class="lb-thru-label">&nbsp;</span><span class="lb-thru-val">' + val + '</span>';
    const num = val.replace(/^Thru\s*/i, '');
    return '<span class="lb-thru-label">Thru</span><span class="lb-thru-val">' + num + '</span>';
}
```

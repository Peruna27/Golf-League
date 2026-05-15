# Golf League — Fantasy Leaderboard for Golf Majors

A mobile-first fantasy leaderboard for golf majors. One repo, one subfolder per tournament.
Three friends (Mike, Reed, Joe) draft 8 players each; lowest combined score of their top 4 wins.

---

## Product Spec

- **League rules**: 3 teams, 8 players each. Lowest combined score of each team's top 4 players wins (golf — low score wins).
- **Phone-friendly**: optimized for one-handed use on a phone in a recliner.
- **Auto-refresh** every 60 seconds + manual refresh button + pauses when tab is hidden.
- **Tap any player row** (in standings card or full leaderboard) → TV-style scorecard modal with hole-by-hole detail per round, color-coded birdies/bogeys.
- **Tournament-themed branding**: each major gets its own colors, logo, and copy matching the official event branding.

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

- **GitHub repo**: `Peruna27/Golf-League`
- **Live URL**: `https://peruna27.github.io/Golf-League/`
- **Git identity**: `Local User <local@local>` (placeholder, set per global CLAUDE.md rules)
- Pages serves from `main` branch root; landing page lives at root.

---

## Data Sources (this is the gold)

### Masters Tournament
**Use the official Masters.com feed.** Best data of any tournament — per-hole scores, tee times, scoreboard-ready everything.

```
https://www.masters.com/en_US/scores/feeds/{YEAR}/scores.json
```

Structure: `data.player[]`, each with `full_name`, `pos`, `topar`, `today`, `thru`, `teetime`, and `round1.scores` … `round4.scores` (18-element arrays of hole scores).

### PGA Championship (and likely other PGA-of-America events)
**ESPN's main `/golf/pga/leaderboard` endpoint returns ZERO events for the PGA Championship.** This is because the PGA Championship is run by the PGA of America, not the PGA Tour. The fix is the `/scoreboard` endpoint with an explicit date param:

```
https://site.api.espn.com/apis/site/v2/sports/golf/pga/scoreboard?dates=YYYYMMDD
```

This DOES return the event. Try today first, then each tournament day (Thu–Sun) as fallbacks. The bare `/leaderboard` endpoint can be the final fallback in case it ever starts working.

### Tee Times (any tournament)
ESPN's scoreboard does NOT include tee times. Use the per-competitor status endpoint on the core API:

```
https://sports.core.api.espn.com/v2/sports/golf/leagues/pga/events/{EVENT_ID}/competitions/{EVENT_ID}/competitors/{ATHLETE_ID}/status?lang=en&region=us
```

Returns `teeTime` (ISO 8601 UTC), `startHole`, `thru`, `position.displayName` (with T-prefix already!), and `type.state` (`pre` / `in` / `post`).

**Don't fetch 156 of these in parallel** — browsers cap concurrent connections per origin at ~6. Batch in groups of 6–8, and only fetch for players who haven't teed off (`thru === '--'`). Cache per day in `localStorage` (tee times don't change once published).

### Finding the ESPN event ID for a new tournament
The PGA scoreboard returns a calendar listing every tournament with its event ID:

```bash
curl "https://site.api.espn.com/apis/site/v2/sports/golf/pga/scoreboard" | jq '.leagues[0].calendar[] | select(.label | contains("Masters") or contains("PGA Championship"))'
```

Save that ID into the `ESPN_EVENT_ID` constant in the tournament's `index.html`.

---

## ESPN Data Structure Quirks (the parser landmines)

The `/scoreboard?dates=...` response is NOT the same shape as the bare `/leaderboard` you may know from other ESPN sports. Specifically:

- **No `status` block on competitors.** Don't read `c.status.position.displayName` — it doesn't exist. Compute position from `c.order` (sequential rank) and post-process to add `T` prefix when multiple players share the same `c.score` string.
- **Score is at `c.score`** as a string: `"E"`, `"-3"`, `"+4"`.
- **`c.linescores[i]` is the i-th round** with:
  - `value`: total strokes when complete, partial strokes during play, 0 if not started
  - `displayValue`: score-to-par as a string ("-2", "E"); "-" if round not started
  - `linescores[j]`: nested per-hole array, each with `period` (hole #), `value` (strokes on that hole), `scoreType.displayValue` (cumulative score-to-par at that hole)
- **`period` is the hole number, NOT the order of play.** A player starting on hole 10 (typical morning wave) will have their first hole entry with `period: 10`. Map by period to put scores in correct hole positions on the scorecard.
- **A round with no per-hole entries means the player hasn't started it yet.** Critical for the "today" / "thru" logic — use globally-determined "today's round" (max round index with any data across the field), not per-player last-round-with-data.
- **Round is complete only when its per-hole array has 18 entries.** Don't display partial stroke counts as round totals.
- **Athlete ID** is `c.id` — keep it on your parsed player data so you can hit the core status endpoint for tee times.
- **Name normalization**: ESPN uses unicode characters (e.g., `Ludvig Åberg`). Normalize via `name.toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g, '')` before matching against roster search terms.

---

## Adding a New Tournament — Checklist

1. **Copy** an existing tournament folder (`2026_PGA_Championship/` is the cleanest template).
2. **Find the ESPN event ID** via the calendar query above. Update `ESPN_EVENT_ID` constant.
3. **Update title, subtitle** in the header — tournament name + course + year.
4. **Swap colors** — change CSS variables `--masters-green`, `--counting-green`, `--counting-bg` to the tournament's brand. Keep variable names as-is to minimize diff (yes, they're misnamed at this point — fine).
5. **Logo** — Wikipedia event pages reliably have a hot-linkable PNG. Pattern: `https://upload.wikimedia.org/wikipedia/en/...`. Hot-linking is fine for a personal site for 3 friends; trying to host the logo ourselves risks trademark issues.
6. **Course pars** — update `COURSE_PARS` for the host venue. Total should equal the championship par (US Open & PGA Championship typically 70, Masters 72, Open Championship 71 or 72).
7. **Read the Excel rosters** from `Source/<Tournament>.xlsx`. The file is a zip — parse it with Python's stdlib `zipfile` + `xml.etree.ElementTree` if openpyxl/pandas aren't installed (don't `pip install` without user approval). Update the `TEAMS` object. Order in the object is Mike / Reed / Joe to match the Excel layout.
8. **`CACHE_KEY`** — change to avoid colliding with other tournaments' localStorage data.
9. **Tournament-name check** in `parseESPNData` — used to flag the status banner when the API is serving a different event.
10. **Dummy data** — refresh the player list to match the new rosters. Useful for previewing while the tournament hasn't started yet, and so the page isn't broken if ESPN goes down.
11. **Landing page card** in root `index.html` — add a new card linking into the new subfolder.
12. **Commit & push** — Pages rebuilds automatically in under a minute.

---

## Design Conventions (do not redebate every tournament)

- **Typography**: Libre Baskerville italic for tournament title and section headers (Masters-style classic serif). Source Serif 4 for large numeric displays (team totals, rank numbers). DM Sans for labels and table data.
- **Score colors** (TV golf convention): **under par = red**, even/over = dark text. Apply via `!important` on `.score-under`/`.score-even`/`.score-over` to win specificity battles.
- **Player names on the leaderboard**: first initial + last name (`R. McIlroy`) so they fit on one line on mobile.
- **Position display**: tie-aware with `T` prefix (`T1`, `T5`, `T11`).
- **"Thru" column**: stacked label-on-top format. Shows `Tee 1:20 PM` before round, `Thru 14` during play, `F` at completion, `CUT` / `WD` for eliminated.
- **"Today" column**: same color coding as total (red for under). `--` when not teed off.
- **Rostered rows on leaderboard**: green left border (3px solid `--masters-green`) + light tint. No alternating row stripes — competes with the rostered highlight.

### Scorecard modal (the TV-style hole-by-hole)

- All three rows (HOLE, PAR, SCORE) the **same font size** (12px, DM Sans 700) — when fonts differ at the same px size they appear different visual sizes; using one font + background tint is more reliable.
- **Backgrounds** differentiate rows: HOLE darkest (#dde1e8), PAR lighter (#eef0f4), SCORE white.
- **Dark horizontal lines above and below the HOLE row** mark the header section. **Light line** between PAR and SCORE. Don't darken every line.
- **Birdie** = red number in red-outlined circle. **Eagle** = white number on **solid red** circle.
- **Bogey** = blue number in blue-outlined square. **Double+** = blue number in double-outlined square (outline + inner border).
- **In-progress rounds populate the scorecard.** Compute score-to-par against par-of-holes-played (not full course par 70), and show "In Progress · thru N" in the round header.
- Score-to-par for in-progress rounds uses played holes only, otherwise -2 thru 6 holes looks like the player is -52.
- Modal opens on tap (player rows in standings AND full leaderboard). Lookup is by `name`, fallback to `rosterName`.

---

## Hosting

- GitHub Pages, free, public repo.
- Push to `main` → auto-deploys in ~60 seconds.
- No custom domain. Old URL after a repo rename is auto-redirected by GitHub for a while, but tell users the new link explicitly.

---

## Gotchas / Lessons Learned

- **`onclick` attributes that include `JSON.stringify(...)` must be in single quotes**, not double quotes. JSON.stringify produces inner double quotes that break a double-quoted attribute.
- **CSS specificity bites you with row-scoped overrides.** When `.sc-row-score .sc-cell` and `.sc-row-label` both target the same cell, the descendant selector (0,2,0) ties with a compound selector (0,2,0) — whichever is declared last wins. Use `.sc-row-score .sc-cell.sc-row-label` (0,3,0) to be explicit.
- **Different fonts at the same px size visually render at different sizes** (especially serif vs sans). Either unify the font or accept different metrics.
- **`localStorage` is per-origin**, so a single `CACHE_KEY` will collide between tournaments. Namespace with the tournament name + year.
- **Hard refresh on mobile**: close tab and reopen. iOS Safari aggressively caches HTML.
- **Tee time fetch must skip already-started players** (`thru === '--'` filter) — otherwise you're making 156 needless requests per refresh.
- **Sort_order in some feeds (Masters.com) changed format between rounds.** Don't trust pipe-delimited rank fields — sort by `pos` (or the equivalent display position) parsed numerically with cut/WD players sent to bottom.

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

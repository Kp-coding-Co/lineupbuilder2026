# Ice Scheduler — Handoff

Reference doc for the next coding session on `ice.html`. Last updated: 2026-08-04.

---

## What this is

A clone of the Jacks lineup builder's interface, repurposed to allocate **multiple teams
into ice-time openings across multiple rinks**. Same shape as the lineup app: one
self-contained HTML file, React + Babel from CDN, no build step, four tabs, dark mode,
a constraint solver behind one big button.

The difference from an off-the-shelf rink booking tool (RinkBook and friends) is the
allocator. Those tools are a shared calendar with conflict detection — you still decide
who goes where. This one takes "these 12 teams need this much ice, here are the blocks we
bought, here are the age-group rules" and produces the week.

- **File**: `ice.html` (sibling of the lineup app's `index.html`, same repo, same Pages site)
- **URL once deployed**: `https://kp-coding-co.github.io/lineupbuilder2026/ice.html`

---

## Tabs

1. **Ice** — the inventory calendar. Every block of ice the association has purchased,
   by rink and date. Month grid across the Sep–Mar season, per-day chips with a fill bar
   showing how much of each block is allocated. Add blocks one at a time, or as a weekly
   recurring pattern across a date range (the way ice actually gets bought). Blocks can be
   marked **blocked** — tournament, public skate, maintenance — and the allocator skips them.
2. **Allocate** — the setup tab. Pick a week, review supply (ice available per day per
   rink) against demand (per-team session counts, defaulting to each team's weekly target
   and overridable for this week only), then **Allocate**. Shows what it couldn't place and why.
3. **Schedule** — the board. Day by day, rink by rink, with open gaps shown inline so
   you can see what's left. Swap two teams, drop a team into an open gap, nudge a start
   time, change length or session type, delete. Undo / redo / reset week. Validation panel.
   Publish/unpublish (gated). Exports: printable sheet, CSV, .ics.
4. **Teams & Rules** — teams (age group, level, weekly target, home rink, blackout days
   and dates), rinks, the age-group rule matrix, and the season usage report.

---

## Data model

All arrays, all stored under `ice_*` keys in localStorage (and mirrored to Supabase when
sharing is on).

### `rinks` — `ice_rinks`
```js
{ id, name, short }
```

### `teams` — `ice_teams`
```js
{
  id, name,
  ageGroup: "u7" | "u9" | "u11" | "u13" | "u15" | "u18",
  level: string,               // free text: "A", "B", "House"
  sessionsPerWeek: number,     // default weekly target
  homeRinkId: string | "",     // preference, not a constraint
  blackoutDows: number[],      // 0 = Sunday
  blackoutDates: string[],     // "YYYY-MM-DD"
  notes: string,
}
```

### `slots` — `ice_slots` — a block of purchased ice
```js
{ id, rinkId, date: "YYYY-MM-DD", start: "HH:MM", end: "HH:MM", note, blocked, blockLabel }
```

### `assignments` — `ice_assignments` — one scheduled ice time
```js
{
  id, slotId, rinkId, date, start, end,
  teamIds: string[],           // 1 team, or 2 when the band allows shared ice
  kind: "practice" | "game" | "skills",
  shared: boolean,
}
```

### `rules` — `ice_rules` — keyed by age group
```js
{
  durationMin: 60,
  weekday: { earliest: "17:00", latest: "19:15" },   // earliest/latest START time
  weekend: { earliest: "07:00", latest: "19:00" },
  allowShared: false,
  maxPerDay: 1,
  maxPerWeek: 4,
}
```

**The window is about when a team may take the ice, not when it must be off.** A U9 team
with a 6:15p latest start can skate until 7:05p; it just can't be handed a 6:30p slot.
This is the constraint the whole tool exists to enforce.

### `meta` — `ice_meta`
```js
{ publishedWeeks: string[], version }   // week starts (Mondays) that have been published
```

---

## The allocator

`solveSchedule()` in `ice.html`. Greedy with restarts — same family as the lineup app's
`multiRunSolve`, tuned for a different problem.

1. Expand the request list: one entry per team per ice time needed.
2. Order **hardest-first** — the team whose age band has the narrowest start window has
   the fewest places to go, so it picks first. Ties are broken randomly, which is what
   makes restarts explore.
3. For each request, enumerate every legal placement: inside a purchased block, inside the
   band's start window, long enough to fit, not colliding with the rink or the team, not
   on a blackout day. Plus **join** placements — sharing an existing ice time of the same
   length when both bands allow it.
4. Score each placement and take the cheapest (with jitter). Cost terms, in rough order of
   weight:
   - **stranded ice** — a placement that leaves an unusable sliver behind is the most
     expensive mistake, priced per wasted minute
   - **fairness** — a team already holding more than its share of prime ice pays to take
     more; a team stuck with off-peak ice gets a discount on prime
   - **spacing** — two ice times in one day is heavily penalized, back-to-back days mildly
   - **window position** — mild bias toward the front of the band's window
   - **home rink** — a tie-breaker, nothing more
5. Score the finished week (unplaced requests dominate, then stranded ice, then variance in
   prime/off-peak holdings, then same-day doubling). Keep the best of 60 runs.

Each click passes a fresh random `seed`, so re-running gives a different — but still
well-scored — arrangement. That's deliberate: "I don't like this week, try again" is a
real workflow.

**Hard vs soft.** The start window, the ice length, rink collisions and team collisions are
hard — the solver leaves a request unplaced rather than break one, and hand-edits that break
one show up as `error` in the validation panel. Everything else is a preference.

Performance: ~1s for 12 teams / 30 requests / a week of ice at 60 runs; ~1.4s for 40 teams
at 30 runs. All in the browser, no worker.

---

## Validation

`validateSchedule()` — warn, don't block, exactly like the lineup app. The only things
that surface as `error` are genuine conflicts (double-booked rink, double-booked team,
start outside the age window, an ice time that escaped its purchased block). Everything
else is `warn` (wrong length for the band, over the daily max, scheduled on a blackout,
short of the requested count) or `info` (stranded ice, ice still open).

The panel header doubles as the ticker when collapsed, same pattern as the lineup app.

---

## Persistence

localStorage is always the working store, keyed `ice_rinks`, `ice_teams`, `ice_slots`,
`ice_assignments`, `ice_rules`, `ice_meta`.

**Cloud sharing is optional and off until the table exists.** On load the app tries to read
`ice_data` row 1 from the same Supabase project the lineup app uses. If that fails — table
missing, offline, CDN blocked — it quietly falls back to local-only and the header badge
reads *Local only* instead of *Synced*. No error, no broken app.

To turn sharing on, run once in the Supabase SQL editor:

```sql
create table if not exists public.ice_data (
  id int primary key,
  rinks jsonb, teams jsonb, slots jsonb,
  assignments jsonb, age_rules jsonb, meta jsonb
);
alter table public.ice_data enable row level security;
drop policy if exists "ice_data_anon_select" on public.ice_data;
create policy "ice_data_anon_select" on public.ice_data for select to anon using (true);
drop policy if exists "ice_data_anon_update" on public.ice_data;
create policy "ice_data_anon_update" on public.ice_data for update to anon using (true) with check (true);
insert into public.ice_data (id) values (1) on conflict (id) do nothing;
```

Same trade-off as the lineup app: the anon key is public by design, there's no login, and
the passphrase gate (`EDIT_PASSPHRASE`, currently `coldsteel`) is client-side only. It
guards publishing/unpublishing a week and restoring a backup — a speed bump against
accidents, not security.

---

## Code locations

| Thing | Search for |
|---|---|
| Theme palette | `const C_LIGHT`, `const C_DARK`, `function applyTheme` |
| Age bands and defaults | `const AGE_GROUPS`, `const DEFAULT_RULES` |
| Prime / off-peak definition | `PRIME_WEEKDAY`, `function slotQuality` |
| Seed data | `DEFAULT_RINKS`, `DEFAULT_TEAMS`, `SEED_PATTERN`, `buildSeedSlots` |
| Free-space math | `function freeSegments`, `function slotUtilization` |
| Allocator | `function candidatePlacements`, `placementCost`, `solveOnce`, `solveSchedule` |
| Validation | `function validateSchedule` |
| Exports | `exportCSV`, `exportICS`, `exportPrintable` |
| Ice calendar | `function IceTab`, `SlotEditorModal`, `RecurringIceModal` |
| Allocate tab | `function AllocateTab` |
| Board | `function ScheduleTab`, `BoardView`, `RinkColumn`, `TeamWeekView` |
| Teams / rules / usage | `function TeamsTab`, `RulesEditor`, `UsageReport` |
| Passphrase gate | `EDIT_PASSPHRASE`, `function EditGate` |

---

## Design decisions to remember

- **Minutes since midnight** is the internal time unit everywhere. Strings ("HH:MM") are
  only for storage; `fmtTime` is only for display. Don't mix them.
- **Dates are parsed at local noon** (`parseDate`) so DST and timezone offsets can never
  shift a calendar day.
- **Weeks start Monday** (`weekStartOf`). Associations plan Mon-to-Sun.
- **Sharing is modelled as two teamIds on one assignment**, not two assignments. That's
  what makes "this block of ice served two teams" true in every report and export.
- **A gap shorter than `MIN_USEFUL_BLOCK` (50 min) is stranded ice** — shown greyed on the
  board, counted in validation, and priced heavily in the solver.
- **The .ics export uses floating local times** (no TZID). Every ice time is local to the
  rink, and floating means the calendar app shows exactly what's written.
- **Tab IDs** (`ice`, `allocate`, `schedule`, `teams`) are stable; labels have already
  changed once.

---

## Open work

- **Season-level allocation.** Everything is week-at-a-time today. Running a whole season in
  one pass would let fairness balance across months rather than within a week, but needs a
  different UI for reviewing the result.
- **Games vs practices.** `kind` exists on an assignment and shows on the board and in
  exports, but the allocator treats all requests the same. Games realistically want longer
  ice, an opponent, and home/away — none of which is modelled.
- **Goalie / skills sessions across teams.** Currently just another session for one team.
- **Travel time between rinks.** A team can be given ice at two different arenas on the same
  day with nothing stopping it (the same-day penalty is the only guard).
- **Per-team public page.** The per-team .ics covers the subscribe case; a shareable
  read-only page per team would be the next step.
- **Audit log.** No record of who changed what. Would need real auth to be worth much.
- **Usage report date range** defaults to the whole configured season, not the actual data
  range — fine today, slightly wrong if the season dates move.

# Jacks 11U AA Lineup System

Browser app for managing the Jacks 11U AA team's roster, schedule, lineups, pitch counts, and arm care. Single self-contained HTML file, no build step. Team data lives in a shared Supabase row — anyone with the link sees the same live roster/schedule/lineups, no login required. A handful of "goes live for every coach" actions are gated by a shared passphrase plus a confirmation step; everything else is open.

**Live site:** <https://kp-coding-co.github.io/lineupbuilder2026/>

## What's in it

- **Schedule** — May–Aug calendar with multi-game days, practices, and played-game flags.
- **Create a Lineup** — attendance, pitching plan (drag-to-reorder on desktop, arrow buttons on mobile), catching plan, bench-tier targets, constraint-solver-driven Generate.
- **Lineup Editor** — list of every saved lineup as cards with quick PNG / Gameday HTML exports. Click in to edit one and you get a swap-mode grid, validation panel, undo/redo/reset, prev/next nav between games, Finalize/Lock, and an in-place **Edit pitching/catching** modal that regenerates without leaving the page. Past games are tinted blue but stay editable.
- **Team Reports** — usage heatmap (toggle Total ↔ Avg per game, fold in projected future-game innings, BN column), Arm Care pitcher status with optional projected pitches folded into 7d/14d/30d totals, depth chart, positional priority matrix.
- **Exports** — game-ready PNG and a self-contained interactive HTML lineup card (tap-to-swap, dark mode, hide-innings, "reset to plan") that works offline during the game.
- **Dark mode** persisted per device.
- **Mobile-friendly** — touch-target sizing on plan editors, mobile-safe pitcher reordering, collapsed Setup sections by default, shortened tab labels under 600px.

## Run locally

Open `index.html` in any modern browser. No build step.

## Deploy via GitHub Pages

1. Push this repo to GitHub.
2. In **Settings → Pages**, set the source to `main` branch, root folder.
3. Site is served at `https://<username>.github.io/<repo>/`.

Every push to `main` redeploys within ~30 seconds. No build step, no deploy quota.

## Editing

The app is a single self-contained HTML file (`index.html`) with React + Babel-standalone loaded via CDN. To make changes, edit `index.html` and reload — no compile or install step.

## Sharing data with co-coaches

No login. Anyone with the site link sees the live shared roster, schedule, lineups, and pitch logs — the app hydrates from a single Supabase row on load. There's no more "download a file, send it, upload it" step.

**Editing is gated, not login-walled.** A short list of actions that go live for every coach immediately are protected by a shared passphrase plus an explicit "this will be visible to all coaches" confirmation, re-prompted every time (nothing is remembered between prompts):

- Finalizing (locking) or unlocking a lineup
- Saving Roster Priority changes (the Roster tab batches position/bench-tier edits into a draft — nothing goes out until you click **Save Changes**)
- Generating a lineup from the Setup tab
- Restoring a backup JSON file, or bootstrapping a device's local data into the cloud

Everything else — viewing, adding/editing the schedule, swapping players in an unfinalized lineup, logging pitches — is open to anyone, no prompt.

This is a soft speed bump, not real security: the passphrase check lives in the page's own JavaScript, so it's visible to anyone who looks at the page source. It's there to stop accidental edits and add a moment's pause before something goes out to the whole team, not to keep a determined person out.

### The ⚙ menu

Settings now only holds a manual backup tool, independent of the live sync:

- **Download team-data.json** — exports the current shared data as a JSON snapshot, for your own safekeeping.
- **Choose file…** — restores a backup JSON file, overwriting the shared cloud data for every coach. Gated by the passphrase + confirmation.

## Ice Scheduler (`ice.html`)

A second app in the same repo, built from this one's interface but pointed at a different
problem: allocating **multiple teams into ice-time openings across multiple rinks**.

**Live site:** <https://kp-coding-co.github.io/lineupbuilder2026/ice.html>

Same shape as the lineup builder — single self-contained HTML file, four tabs, dark mode,
a constraint solver behind one big button — with hockey's constraints in place of baseball's:

- **Ice** — calendar of every block of ice the association has purchased, per rink. Add
  blocks singly or as a recurring weekly pattern. Mark blocks blocked for tournaments or
  public skates.
- **Allocate** — pick a week, set who needs how much ice, and the allocator fills the
  blocks. It respects each age group's **ice length** and **earliest/latest start times**
  (separately for weeknights and weekends) as hard rules, shares half-ice between the
  youngest bands where allowed, spreads prime-time and off-peak ice fairly across teams,
  keeps teams off back-to-back days, and avoids leaving gaps too short for anyone to use.
  Anything it can't place is listed with the reason.
- **Schedule** — the week board, day by day and rink by rink, with open gaps shown inline.
  Swap two teams, drop a team into an open gap, nudge a start time, undo/redo/reset,
  publish. Conflict detection is live. Exports: printable sheet, CSV, and .ics calendar
  files (whole association or one team).
- **Teams & Rules** — teams with weekly targets, home rinks and blackout dates; the rinks
  list; the editable age-group rule matrix; and a season usage report showing hours per
  team and how the prime ice got divided.

Runs local-only out of the box. Shared cloud sync turns on once the `ice_data` table
exists — see `ICE_HANDOFF.md` for the one-time SQL and the rest of the engineering notes.

## Files

- `index.html` — the lineup app
- `ice.html` — the ice scheduler
- `ICE_HANDOFF.md` — engineering handoff for the ice scheduler
- `archive/` — older versions, design mocks, and sample game-day exports kept for reference
- `HANDOFF.md` — engineering handoff doc: data model, code locations, anti-footguns
- `NEXT_PHASE_BRIEF.md` — brief for a future Vite/normalized-DB rebuild. Kept for reference; not pursued.

## Data persistence

The shared source of truth is a single row in a Supabase table (`team_data`), read by everyone on load and written by the `load`/`save` layer in `index.html`. Every write also mirrors to the browser's `localStorage` as an offline-friendly cache. There's no login — the Supabase anon key is public by design, and the passphrase gate described above is enforced client-side only. See `HANDOFF.md` for the one-time Supabase RLS setup this depends on.

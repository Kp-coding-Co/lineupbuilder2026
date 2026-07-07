# Jacks 11U AA Lineup System

Browser app for managing the Jacks 11U AA team's roster, schedule, lineups, pitch counts, and arm care. Single self-contained HTML file, no build step. Multi-user access is powered by a Supabase project: coaches sign in with a magic link and share one live `team_data` row, so changes made by one coach show up for everyone else. `localStorage` is still used as an offline cache per device, and a lightweight "publish via the repo" path remains available as a manual fallback.

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

## Multi-user access (Supabase)

`REQUIRE_AUTH` (near the top of `index.html`) is `true`, so the app requires sign-in and syncs to a shared Supabase `team_data` row instead of per-device `localStorage`. Every signed-in coach reads and writes the same roster, schedule, lineups, and pitch logs — no manual export/import needed.

**One-time setup (done once per team, in the Supabase dashboard for the project referenced by `SUPABASE_URL`):**

1. **Custom SMTP** — Supabase's built-in email sender is capped at 4 magic-links/hour, too low for a team. Under **Project Settings → Auth → SMTP Settings**, wire up a provider (e.g. [Resend](https://resend.com)'s free tier) so login emails aren't rate-limited.
2. **Allowlist each coach** — only emails in the `allowed_emails` table can sign in. In the Supabase SQL editor:
   ```sql
   insert into public.allowed_emails (email) values ('coach@example.com');
   ```
3. **Sign in** — each coach visits the deployed site, enters their email, and clicks the magic link sent to their inbox.

The first coach to sign in with an empty cloud row gets an "import to cloud" banner offering to push their local data up as the shared baseline; after that, everyone's edits sync automatically.

To go back to local-only (no login, no cloud sync) for fast iteration, flip `REQUIRE_AUTH` back to `false` in `index.html`.

## Sharing data with co-coaches (manual fallback)

This mechanism predates the Supabase sync above and still works as a manual override — useful for offline sharing or a one-off snapshot outside the live sync.

**The mechanism:** a `team-data.json` file lives in the repo root and is served alongside `index.html`. On every page load the app fetches it. If its `publishedAt` timestamp is newer than what the visitor has applied locally, a banner appears: *"New team data published [date]. Apply it? [Apply] [Skip]"*. **Apply** overwrites the visitor's localStorage with the snapshot and reloads. **Skip** suppresses the banner until you publish a newer version.

### Head coach workflow (you publish)

1. Make your changes in the app as normal — schedule games, build lineups, log pitches.
2. Click the **⚙** icon in the header → **Download team-data.json**.
3. Upload that file to the repo root (overwrite the existing `team-data.json`) via the GitHub web UI's "Upload files" → drag-and-drop → commit.
4. GitHub Pages updates within ~30 seconds.
5. Co-coaches refresh, see the banner, click Apply.

### Co-coach workflow (they view)

1. Visit the deployed site.
2. On first visit, or whenever you publish, a banner offers to apply the latest team data. Click **Apply**.
3. They're now looking at the same roster, schedule, and lineups you published. They can navigate freely, run reports, and play with hypothetical lineups locally — those local edits stay on their device and don't propagate back.

### Importing a coach's suggested changes

If a co-coach wants to suggest changes (e.g. they exported their JSON after playing with hypothetical lineups):

1. Have them click **⚙ → Download team-data.json** and send you the file.
2. On your device, click **⚙ → Choose file…** and pick their file. The app prompts for confirmation, then replaces your localStorage with theirs and reloads.
3. Review. If you want to broadcast it, click **⚙ → Download team-data.json** again and upload it to the repo.

### The ⚙ menu

- **Download team-data.json** — exports your current view as a JSON snapshot.
- **Choose file…** — imports a JSON file (typically one a coach sent you), overwriting your local view.
- **Sync now** — re-fetches `team-data.json` from the deployed site and offers to apply it. Useful if you skipped an earlier banner.

The modal also shows two timestamps: when you last applied a snapshot locally, and the current `publishedAt` on the server.

### `team-data.json` format

```json
{
  "publishedAt": 1714752000000,
  "roster": [ ... ],
  "schedule": [ ... ],
  "pitchLogs": [ ... ],
  "benchTiers": { ... }
}
```

`publishedAt` is a Unix milliseconds timestamp set at export time. The app uses it to decide whether the snapshot is newer than what the visitor has locally.

## Files

- `index.html` — the app
- `team-data.json` — the published team snapshot (created on your first Export). Not in the repo until you publish.
- `archive/` — older versions, design mocks, and sample game-day exports kept for reference
- `HANDOFF.md` — engineering handoff doc: data model, code locations, anti-footguns
- `NEXT_PHASE_BRIEF.md` — brief for a future Vite/normalized-DB rebuild. Kept for reference; not pursued.

## Data persistence

State (roster, schedule, lineups, pitch logs, bench tiers) syncs to a shared Supabase `team_data` row when signed in (see **Multi-user access** above), with `localStorage` as an offline-per-device cache. The `team-data.json` mechanism remains available as a manual, no-login fallback for bridging devices.

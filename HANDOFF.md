# Dugout — Session Handoff

A reference doc for the next coding session on this app. Date of last update: 2026-04-27.

---

## What this is

Single-file (`index.html`) React app for managing an 11U baseball team's roster, schedule, lineups, and pitch counts. Runs entirely in the browser. Used by the head coach (you) and eventually a small group of co-coaches.

- **Repo**: <https://github.com/ken-peterson-2112/jacksdugout>
- **Live URL**: <https://ken-peterson-2112.github.io/jacksdugout/>
- **Deploy method**: GitHub web UI upload of `index.html` to the repo. Pages serves it from `main` branch root, ~30s after upload.

---

## Tech stack

- **Single HTML file** with React + ReactDOM + Babel-standalone loaded from CDN. No build step.
- **JSX in `<script type="text/babel">`** — Babel transpiles in-browser at load.
- **Supabase JS SDK** loaded from CDN; project ref: `oewaghxnwqzeneavontg`. Anon key is hardcoded (safe — RLS protects writes).
- **Persistence**:
  - Currently `localStorage` only. The cloud-aware `load`/`save` layer is wired but `REQUIRE_AUTH = false` short-circuits hydration so writes don't reach Supabase.
  - When `REQUIRE_AUTH = true`, magic-link login + `team_data` JSON-blob row in Supabase becomes the source of truth.

---

## File layout

```
.
├── index.html              ← the entire app
├── README.md
├── .gitignore
├── HANDOFF.md              ← this file
├── files.zip               ← local backup, .gitignored
└── archive/
    ├── pin-and-solve.md          ← removed feature; revival recipe
    ├── dugout.html               ← old single-file version
    ├── dugout-lineup.jsx         ← older JSX-only attempt
    ├── lineup_candidates.html
    ├── sample_gameday_optC.html
    ├── lineup_option_A.png … D.png
    └── solid 4 innings.png
```

---

## Tab structure (in this exact order)

1. **Schedule** — May–Aug 2026 calendar, 7-day Sun-Sat grid per month. Tap empty day to add game/practice. Tap entry to edit/open. Multiple games per day supported (tournaments). Each entry has time + opponent + home/away + game type.
2. **Create a Lineup** — was "Game Settings". The setup form: date/time/opponent/home-away/game-type, attendance, pitching plan, catching plan, bench priority. Big "Generate Lineup" button. On mobile, Pitching/Catching/Bench sections start collapsed.
3. **Lineup Editor** — was "Lineup". Read-only ticker + collapsed validation panel up top. Swap-mode interactive grid. Bench-score golf badges (eagle/birdie/par/bogey/2bogey) at top-right of each cell. Undo/Redo/Reset row. Player Summary heatmap at bottom. Finalize Lineup as its own row. Download Gameday HTML in secondary action row.
4. **Team Reports** — was "Team", was "Roster". Three sections vertical:
   1. **Usage Report**: heatmap of innings played per (player × position). Date range picker; defaults to whole season.
   2. **Arm Care**: pitcher status table with as-of-date scrubber. *(formerly its own top-level tab)*
   3. **Depth Chart**: 8 columns (no P, no BN). Players stacked vertically as tier-colored chips.
   4. **Positional Priority**: editable tier matrix (the original Roster table). Has a `?` button → modal with explanation.

Header: **"Jacks 11U AA · 2026"** · ☾/☀ theme toggle · Live beta badge (when REQUIRE_AUTH=false) or email + Sign out (when authed).

---

## Key data model

### `roster` (localStorage `dugout_roster`)
Array of `{ id, name, number, positionMap }`. `positionMap` keys are positions ("P", "C", "1B"…); values are `"<tier>"` or `"<tier>:<rank>"` (rank breaks ties within a tier). Tiers: `preferred`, `secondary`, `emergency`.

### `schedule` (localStorage `dugout_schedule`)
Array of entries:
```js
{
  id: "sch_<ts>_<rand>",
  date: "YYYY-MM-DD",
  kind: "game" | "practice",
  time: "HH:MM" | null,        // 24h, required for games
  opponent: string,            // required for games
  homeAway: "home" | "away",   // required for games (no default)
  gameType: "exhibition" | "season" | "tournament" | "playoffs",
  played?: boolean,            // set by Log Pitches modal
  lineup?: {
    grid: [{ P, C, 1B, ..., RF, BN: [] }, ...],   // one obj per inning
    pitcherAssignments: [{ id, innings, budget }],
    catcherAssignments: [{ id, innings }],
    absentIds: string[],
    gameInnings: number,
    isFinalized: boolean,
  }
}
```

### `pitchLogs` (localStorage `dugout_pitchLogs`)
`{ id, playerId, pitches, date, opponent, scheduleId? }`. `scheduleId` ties a log to a specific schedule entry (set by the Log Pitches modal); manual logs from Arm Care don't have it.

### `benchTiers` (localStorage `dugout_benchTiers`)
Map of `playerId → "regular" | "flex" | "dev"`. Note: stored IDs are still `flex` and `dev` even though the labels are "Reduced" and "Limited" — keeping IDs preserves existing data. `BENCH_TIER_MAP.iron` is aliased to `regular` as a safety net for legacy data.

### `gameMeta` (localStorage `dugout_gameMeta`)
Active game's metadata: `{ date, time, opponent, homeAway, gameType, scheduleId? }`. Mirror auto-saves changes back to the linked schedule entry via the `currentScheduleId` transient state.

### `team_data` (Supabase)
Single-row table. Columns are JSONB blobs: `roster`, `schedule`, `pitch_logs`, `bench_tiers`, `game_meta`. Plus `allowed_emails` table for the access allowlist.

---

## Major features built this session

| Area | What's there |
|---|---|
| **Multi-month calendar** | May–Aug 2026, 7-col grid. Multiple entries per day. Time + home/away + game type required for games. "Add & Edit" button creates entry and jumps to lineup builder in one click. |
| **Practices** | Same UI flow as games via "Set as practice" link. Orange chips, no opponent/home-away needed. |
| **Played-game logging** | Log Pitches modal on Lineup Editor for games linked to schedule entries. Replaces existing pitchLogs for that game on save. Sets `played: true` flag → ✓ on calendar. |
| **Pitcher availability** | Computed against `gameMeta.date` (not wall-clock today). Hard block only on **logged** past outings within the rest window. Soft warnings for: same-day other game (tournament), past planned-but-unlogged, future planned within 4d. Setup tab shows green / yellow / red availability chips + a separate "Unavailable" arms list. |
| **Bench-score golf badges** | Lineup grid cells. Par = 2 sits. Eagle (0) = double-circle green; birdie (1) = single circle; par = blank; bogey (3) = single square orange; double bogey (4+) = double square red. Cell badge has count inside in the Player Summary table. |
| **Validation panel** | Collapsed by default. Header doubles as the live ticker (severity chip + message + prev/next/dismiss/× controls) when there's an active warning. Refresh button always visible. |
| **Undo / Redo / Reset** | Lineup Editor. History+future stacks; Reset reverts to the originally-generated grid. Resets on every fresh Generate. |
| **Auth scaffolding** | Magic-link login screen, Supabase client wired, `team_data` row + `allowed_emails` allowlist + RLS schema applied. **Currently bypassed** via `REQUIRE_AUTH = false`. Toggle to true to re-enable cloud sync. |
| **Dark mode** | Full palette + `applyTheme(name)` mutates `C`, `chipStyle`, `thStyle`, `tdStyle`, `inputStyle`, `TIER_BG`, `TIER_COLORS` in place. Persisted to `dugout_theme`. ☾/☀ toggle in header. Includes the gameday HTML export. |
| **Gameday HTML export** | Self-contained interactive download. Tap any two cells to swap, ✓ marks swapped, "Reset to plan" button. Now also has dark-mode toggle and per-inning hide ("tap inning header to hide") with a "Show all" pill. |
| **Mobile pass** | `viewport-fit=cover`, `touch-action: manipulation`, `-webkit-tap-highlight-color: transparent`, `text-size-adjust: 100%`. 16px input font on phones to kill iOS zoom-on-focus. Tab labels shorten under 600px (CSS toggle). Pitching/Catching/Bench sections collapsed by default on mobile. Lineup Editor secondary action row stacks vertically on mobile. |

---

## Design decisions to remember

- **Light theme is default**, dark theme is a swap. Persisted in `localStorage["dugout_theme"]`.
- **Tab IDs vs labels**: IDs (`schedule`, `setup`, `lineup`, `roster`) are stable; labels have changed multiple times. Don't rename IDs without considering ripple effects.
- **`gameMeta` is the active game**, mirrored back to `schedule[currentScheduleId]` automatically while editing. Generating from Setup with date+opponent filled in but no `currentScheduleId` auto-creates a new schedule entry.
- **`Generate` always uses `multiRunSolve`** (25 attempts) for the bench-fairness gate; `pins` always `[]` since pin/solve UI is archived.
- **Bench tier names are aspirational**; IDs are stable: `regular`/`flex`/`dev`. UI labels say `Regular`/`Reduced`/`Limited`. Iron Man tier was removed but `BENCH_TIER_MAP.iron` is aliased to regular for legacy data safety.
- **Validation is "warn, don't block"**: most issues are surfaced in the ticker but never prevent saves. The only hard block is rest-from-logged-pitchLogs.
- **Pitcher availability uses planning date, not wall-clock today**. So re-opening an old game gives historically-accurate results.
- **Conditional formatting on Usage Report**: light blue → grey → bright orange heatmap, recalibrates min/max within the visible date range.
- **Add & Edit is the primary button** in the schedule modal (filled green); Add is secondary (outlined). The typical flow is "create entry, go straight to building the lineup."
- **Sections collapse on mobile** for the heavier Setup tab planning sections, default open on desktop. Driven by `useIsMobile()` hook.

---

## Removed / archived

- **Pin & Solve**: full code recipe in `archive/pin-and-solve.md`. Engine code (`generateLineup` accepts `pins`, `multiRunSolve` exists) is intact and dormant — passing `[]` is a no-op.
- **Iron Man bench tier**: removed from `BENCH_TIERS` array. Engine still has `id === "iron"` branches in bench-balance passes; those become unreachable.
- **Standalone `Arm Care` tab**: now lives inside Team Reports between Usage Report and Depth Chart.
- **Print Grid + Export JSON buttons**: removed from Lineup Editor action rows. `handlePrintGrid` and `handleExportJSON` functions still exist but are unused. About 100 lines of dead code; cheap to leave in for future revival.
- **Standalone validation ticker block**: merged into the validation panel header. Old code wrapped in `{false && ...}` and could be fully stripped on a cleanup pass.
- **"Test mode" badge text**: now reads "Live beta".
- **Hardcoded variation #**: Jude Charron is `#11`, not `#99`. Bumped `ROSTER_VERSION` from 2 → 3 to apply.
- **Validation messages dropped**: "X plays POS every inning" is gone. Emergency-assignment lines are clustered: `"Kaden at LF — 4 emergency innings (1, 4, 5, 6)"`. "— should swap" suffix dropped from inversion warnings.

---

## TBDs / open work

- **Auth**: `REQUIRE_AUTH = false`. Flip to `true` and **set up custom SMTP in Supabase** (Resend free tier or similar) before inviting other coaches — built-in email is rate-limited to 4/hour per project.
- **Allowlist**: Currently only `peterson.ken12@gmail.com`. Add other coaches via:
  ```sql
  insert into public.allowed_emails (email) values ('coach@example.com');
  ```
- **Import GC Data button**: placeholder modal. Needs a sample GameChanger CSV export to wire up the parser.
- **Import Pitch Count button**: placeholder modal. Needs sample / format spec from the league's proprietary tool.
- **`played` filter on Usage Report**: currently uses `lineup && date < today`. Could tighten to `played === true` if/when that flag is reliably set.
- **iOS notch padding**: `viewport-fit=cover` is set but I haven't added `padding-bottom: env(safe-area-inset-bottom)` anywhere. Probably fine for browser usage; matters if/when this becomes a PWA.
- **Game-day HTML export styling for paper**: the export is light-mode-friendly by design. If a coach prints from inside the export, dark-mode rules don't apply — already handled.

---

## Common code locations

| Thing | Search for |
|---|---|
| Theme palette | `const C_LIGHT`, `const C_DARK`, `function applyTheme` |
| Bench tiers | `const BENCH_TIERS`, `BENCH_TIER_MAP` |
| Default roster | `const DEFAULT_ROSTER` (line ~190) |
| Lineup engine | `function generateLineup`, `function multiRunSolve`, `function validateLineup` |
| Auth + cloud sync | `function App` top, `useEffect` blocks for `getSession()` and `onAuthStateChange` |
| `load`/`save` cloud-aware | Search `cloudCache`, `KEY_TO_COLUMN`, `pushToCloud` |
| Schedule entry editor | `function ScheduleEditorModal`, `ScheduleTab.openAdd`/`openEdit`/`saveEditor`/`saveAndEdit` |
| Pitch log modal | `function LogPitchesModal`, `App.handleSavePitchLog` |
| Pitcher availability | `const pitcherAvailability = useMemo` inside `App` |
| Bench-score badge | `function BenchScoreBadge` |
| Mobile detection | `function useIsMobile` |
| Gameday HTML template | `const GAME_DAY_TEMPLATE_B64` (base64 — decode/edit/re-encode) |

---

## Dev workflow

1. **Edit `index.html` locally** (any editor; no build).
2. Open the file directly in Chrome to test (`file://` URL). Note: localStorage is per-origin, so file-protocol is its own bucket — guest window for clean-state testing.
3. Magic-link login won't work from `file://` because Supabase redirect URLs are configured for the GitHub Pages domain. Test auth on the live site.
4. **Push by re-uploading** `index.html` to the repo via the GitHub web UI's "Upload files" → drag-and-drop → commit. (No git CLI required while OneDrive is in the path.)
5. Pages updates ~30s after the upload commits.

### Editing the gameday template

The exported game-day file is a base64-encoded HTML in `const GAME_DAY_TEMPLATE_B64`. To modify:
1. Decode: `awk '/GAME_DAY_TEMPLATE_B64 = "/{ match($0, /"([^"]+)"/, a); print a[1]; exit }' index.html | base64 -d > /tmp/template.html`
2. Edit `/tmp/template.html`
3. Re-encode: `base64 -w0 /tmp/template.html`
4. Replace the string in `index.html` (use Perl one-liner or careful manual splice to avoid messing up the long quoted string).

---

## Anti-footguns / things future me might forget

- **Don't bump `ROSTER_VERSION`** unless you mean to overwrite the user's stored roster with `DEFAULT_ROSTER`.
- **Hardcoded literal Unicode characters in JSX text** (em-dash `—`, bullet `•`, ⚠) sometimes get stored as `\uXXXX` escape strings in this file — JSX renders them as the literal escape text instead of the character. Fix is to wrap in `{"…"}` so they're parsed as JS string literals: `{"⚠"}` instead of `⚠`.
- **`tdStyle`/`thStyle`/`chipStyle`/`miniChipStyle`/`inputStyle`** are module-level mutable objects. They get rebuilt by `applyTheme` so dark mode picks up new colors. If you add new shared style objects, register them in `applyTheme` too.
- **`TIER_BG` and `TIER_COLORS`** are also mutated by `applyTheme`. Same caveat.
- **`generationId` in App** is bumped on Generate / Solve so child components (Lineup Editor) know to reset their internal state (history stack, ticker dismissals).
- The **swap-cancel `setSwapSel(null)`** is called from undo/redo/reset to prevent stale selection state.
- **Two separate validation defaults**: `defaultOpen` is `true` in the `Section` component, but the validation panel uses its own `validationOpen` state (default `false`).
- The **Usage Report wrapper has `background: C.surface`** to make the rounded card surface still extend the full width even though the inner table is `width: auto` and sits left-aligned.

---

## What worked particularly well

- The single-file architecture — no build step, fast iteration loop, deploys via web UI.
- localStorage as default with a clean cloud-sync path that can be flipped on by toggling `REQUIRE_AUTH`.
- Module-level mutable design tokens (`C`) plus pre-built style objects that get rebuilt on theme change — kept the existing inline-style codebase working without a CSS overhaul.
- Bench-score golf metaphor — the coaches who play golf instantly grok it.

Good luck future-me.

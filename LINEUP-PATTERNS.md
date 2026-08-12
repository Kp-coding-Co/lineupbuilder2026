# Lineup Patterns — What the Coach Actually Does

Derived from `team-data.json` (published 2026-08-12): 39 finalized lineups across the
season, with the **last 15 games** (2026-07-15 → 2026-08-31) analysed in depth.

- 15 games · 104 team innings · 178 player-games · 12-player roster
- Two of the 15 are placeholder entries (`Tbd` 08-14, `Fake` 08-31). Re-running every
  statistic on the 15 most recent *real* games moves nothing material — every figure
  below is stable to ±0.15. They're left in.

These grids are the **coach-approved final state** — whatever the generator proposed,
this is what got finalized. That makes them the right training signal.

---

## The headline

The engine currently builds each inning as a **fresh constrained optimisation**: score
every player against every position, solve, then rebalance bench counts globally until
the spread is ≤ 2. The coach does something structurally different — he runs a **fixed
defensive alignment and rotates players through it one at a time**. Positions are sticky;
players are the moving parts; bench time is a *hierarchy*, not something to equalise.

Nine of the ten patterns below fall out of that one difference.

---

## Pattern 1 — Every position has an owner and a designated backup

Share of innings at each position, last 15 games:

| Pos | Starter | Backup | 3rd | Top-2 coverage |
|---|---|---|---|---|
| **1B** | Reed 54% | Jude 43% | — | **97%** |
| **2B** | Strachan 60% | Henry 38% | — | **98%** |
| **3B** | Elliott 74% | Mason 19% | Everett 7% | **93%** |
| **SS** | Benji 63% | Cullen 33% | — | **96%** |
| **LF** | Owen 60% | Mack 19% | Cullen 11% | 79% |
| **CF** | Everett 63% | Cullen 11% | Mack 9%, Henry 9% | 74% |
| **RF** | Clay 62% | Mack 17% | Henry 11% | 79% |
| **C** | Mason 47% | Cullen 21% | Mack 18%, Benji 13% | 68% |

The infield is nearly a closed system — two names cover 93–98% of every infield slot.
The outfield is looser because Mack and Henry are deliberate floaters.

Seen from the player's side, most of the roster has exactly one job:

| Player | Home pos | Share of their field innings |
|---|---|---|
| Elliott | 3B | **99%** (77/78) |
| Benji | SS | **99%** (66/67) |
| Reed | 1B | **98%** (56/57) |
| Owen | LF | **98%** (62/63) |
| Clay | RF | **98%** (65/66) |
| Strachan | 2B | **97%** (62/64) |
| Everett | CF | 88% (66/75) |
| Jude | 1B | 74% |
| Henry | 2B | 61% |
| Cullen | SS | 58% |
| Mack | LF | 42% — true utility |

Average distinct field positions per player-game (excluding P/C): **1.07** for the six
anchors, 2.53 for Henry, 2.20 for Cullen. Six of twelve players played exactly one field
position in 14 of 15 games.

> **Rule to encode.** A player's home position is a near-hard constraint, not a +12
> tiebreaker. When an anchor is on the field, they are at their home position; if the
> alternative is playing them elsewhere, bench them instead.

---

## Pattern 2 — Bench load is a hierarchy, deliberately unequal

Sits per game, last 15:

| Player | Sits/game | | Player | Sits/game |
|---|---|---|---|---|
| Reed | **2.60** | | Mack | 1.80 |
| Owen | **2.53** | | Mason | 1.60 |
| Henry | 2.07 | | Strachan | 1.33 |
| Clay | 2.00 | | Everett | 1.21 |
| Jude | 1.80 | | Cullen | 1.13 |
| | | | Elliott | 1.13 |
| | | | Benji | **0.87** |

That's a 3× range, and it is stable game over game — not noise. Per-game bench spread
(max sits − min sits among present players) was **3 or 4 in 7 of 15 games**, with a
median of 2.

Two things the coach does that the engine currently forbids or penalises:

- **Zero-sit games happen on purpose**: 9 of 178 player-games (Benji 5, Cullen 3,
  Everett 1). Benji played all 7 innings in a third of his games.
- **The cap gets exceeded**: 3 of 178 player-games went over 3 sits (Reed 5 and 4,
  Owen 4).

Playing time still lands in a reasonable band because the bench hierarchy is offset by
pitching and catching — active innings per game run from Benji 6.07 down to Reed 4.33.

> **Rule to encode.** Fairness is a *per-player target*, not a flat spread. The existing
> `benchTiers` field is exactly the right mechanism and is essentially unused — only 4
> players have an entry and all say `regular`. Populate it from observed sits/game and
> let the target drive selection.

---

## Pattern 3 — Sits are single innings, alternated

Only **15 of 178 player-games (8.4%)** contain a back-to-back sit. The dominant shape is
strict alternation — Owen vs Peterborough on 07-26 went `BN LF BN LF BN LF BN`; Reed's
signature line is `BN 1B BN 1B 1B 1B 1B`.

Sits are also front-loaded and back-loaded rather than evenly spread. Across the twelve
7-inning games, innings 1, 3, 5 and 7 carry the bench rotation; innings 2, 4 and 6 are
where the regulars are back out.

> **Rule to encode.** Keep the no-back-to-back preference (already present) and add an
> explicit *alternating* bias — a player who sat in inning `i` should be strongly
> preferred to play in `i+1`, which is stronger than the current `−100` penalty implies
> relative to the other scoring terms.

---

## Pattern 4 — A player who sits comes back to the same spot

After a single-inning sit, players returned to the **same position 102 of 137 times
(74%)**. The bench inning is a breather inside a fixed assignment, not a reshuffle.

Field-slot churn averages **4.17 of 8 slots** per inning transition — which sounds high,
but three of those are the sub-out/sub-in pair plus the catcher rotation. Churn at a
pitching change is 4.51 vs 3.94 otherwise, so the pitcher change is *not* the main driver
of movement; the bench rotation is.

> **Rule to encode.** When a player returns from the bench, their previous position is
> the default. Reserve the position for them across the sit where possible — i.e. the
> backup should slot into the *vacated* position rather than triggering a cascade.

---

## Pattern 5 — Pitchers get a bench inning right before they take the mound

Of the 35 outings that didn't start the game, the pitcher was **on the bench the inning
immediately before in 21 (60%)**. The next most common predecessor is 3B/1B at 3 each.

Post-outing rest is weaker: 11 of 35 non-closing outings (31%) were followed by a sit,
and the more common pattern is dropping back into an infield spot (2B 6, 1B 5, SS 4).

Examples: Strachan `2B 2B BN P P P 2B`, Jude `1B BN P P BN 1B 1B`, Mason `P P P BN 3B BN 3B`.

Related, and already correct in the engine: **zero** instances of a player catching after
pitching in the same game, across all 15.

> **Rule to encode.** Once the pitching plan is set, pre-seed a bench inning immediately
> before each pitcher's first inning (when they aren't starting). The engine currently
> has no concept of this and produces it only by accident.

---

## Pattern 6 — Catching is a two-man split, interleaved not blocked

- 2 catchers in 11 of 15 games, 3 in the other 4.
- Innings: Mason 47, Cullen 27, Mack 16, Benji 16.
- **18 of 33 catching stints are non-contiguous** — the catcher goes back to a field
  position mid-game and returns.

Each catcher has a field job they interleave with: Mason ↔ 3B, Cullen ↔ SS, Benji ↔ SS,
Mack ↔ OF. The most common thing a catcher does immediately before their stint is play SS
(8), and after, sit (6) or play SS (4).

The engine's `catcherPlan` (non-contiguous, spread by remaining quota, no catching after
pitching) already matches this well. **No change needed.**

---

## Pattern 7 — Inning 1 is the A-alignment, with one exception

How often the anchor starts at their own position:

| | 2B | 3B | CF | SS | RF | C | LF | 1B |
|---|---|---|---|---|---|---|---|---|
| Inning 1 | 13/15 | 13/15 | 13/15 | 12/15 | 11/15 | 10/15 | 8/15 | **1/15** |
| Final inning | 10/15 | 10/15 | 10/15 | 9/15 | 6/15 | 8/15 | 5/15 | 8/15 |

The exception is deliberate and consistent: **Jude starts at 1B in 14 of 15 games and
Reed opens on the bench in 11 of 15**, even though Reed out-plays him at 1B over the full
game (56 innings to 45). Reed's standing pattern is sit inning 1, play inning 2, sit
inning 3, then play out the game.

Note that the closing inning is *less* locked-down than the opener — the coach is not
saving a defensive unit for the end.

> **Rule to encode.** Build inning 1 from the A-alignment first, then rotate. And treat
> the inning-1 bench as a specific slot in the rotation order rather than the output of a
> general scoring pass.

---

## Pattern 8 — Pitching plan shape

- **3 pitchers in 10 of 15 games** (4 in two, 5 and 6 once each, 2 once).
- Block lengths: 2 innings ×20, 3 innings ×14, 1 inning ×15, 4 innings ×2.
- Blocks are always contiguous.
- Workload last 15: Jude 18, Strachan 17, Mack 12, Elliott 10, Benji 10, Mason 7,
  Henry 7, Reed 7, Cullen 6, Everett 5, Clay 3, Owen 3.

Everyone on the roster pitched. The one-inning blocks cluster in the two tournament
games where the coach was spreading arms deliberately (07-21 used six pitchers).

> **Rule to encode.** Nothing structural — but the default plan the Setup tab offers
> should be 3 pitchers × (2, 2, 3) for a 7-inning game rather than a flat `innings: 2`.

---

## Pattern 9 — The position map has drifted out of date

Actual assignments last 15 games rated against each player's declared `positionMap`:

- preferred **76.6%**, secondary 18.8%, emergency 3.1%, **not in the map at all 1.6%**

That last bucket is 13 innings the engine would refuse to generate and flags as a
high-severity warning after the fact. The map is describing a roster from May.

Concrete corrections implied by usage:

| Player | Change | Evidence |
|---|---|---|
| Henry | 2B `secondary` → **`preferred`** | 40 innings, the everyday backup 2B |
| Cullen | LF `emergency` → **`secondary`** | 11 innings |
| Jude | add **RF `emergency`** | 7 innings, currently unmapped |
| Everett | add **LF `emergency`** | 1 inning, unmapped |
| Reed | add **RF `emergency`** | 1 inning, unmapped |
| Strachan | add **CF `emergency`** | 1 inning, unmapped |
| Owen | add **2B `emergency`** | 1 inning, unmapped |
| Clay | add **CF `emergency`** | 1 inning, unmapped |
| Cullen | add **RF `emergency`** | 1 inning, unmapped |
| Benji | C `preferred` → `secondary` | 14 innings; Mason owns the job |
| Mack | C `preferred` → `secondary` | 19 innings; same |
| Cullen | SS `preferred` → `secondary`, C `preferred` → `secondary` | he's the backup at both |
| Henry | CF/LF `secondary` → `emergency` | 9 and 6 innings, spot duty |
| Mack | CF `secondary:2` → `emergency` | 9 innings |
| Jude | CF `secondary:3` → `emergency` | 7 innings |
| Benji | 1B `secondary:2` → `emergency` | 1 inning |
| Strachan | SS `secondary:2` → `emergency` | 1 inning |
| Owen | drop RF `emergency` | 0 innings |
| Clay | drop LF `emergency` | 0 innings |
| Elliott | drop LF `secondary`, RF `secondary` → `emergency` | 0 and 1 innings |

The `preferred`/`secondary` split should mean *owner* / *designated backup* — that reading
matches the usage almost perfectly and makes Pattern 1 expressible in the existing data
model without a schema change.

---

## Pattern 10 — Where the engine and the coach disagree

| # | Engine rule | Location | Coach's actual behaviour |
|---|---|---|---|
| 1 | Reject any lineup with bench spread > 2 | `scoreLineup` / `multiRunSolve` (`index.html:1743`, `:1797`) | Spread 3–4 in **7 of 15** games |
| 2 | "Never sits" = high-severity warning | `validateLineup` (`index.html:1623`) | 9 zero-sit player-games on purpose |
| 3 | `HARD_BENCH_CAP = 3`, enforced through 3 escalating relaxation levels | `index.html:1343` | Exceeded in 3 player-games (Reed 5, Reed 4, Owen 4) |
| 4 | Position continuity = `+12` tiebreaker, swamped by scarcity `+50` and tier `200/80/25` | `fieldScore` (`index.html:789`) | Home position held **97–99%** of innings |
| 5 | Bench chosen by scarcity + fairness score each inning | `index.html:687` | Fixed rotation order through a bench-priority list |
| 6 | Rebalance passes accept tier downgrades "freely" to flatten counts | `tryBenchBalance` (`index.html:1494`) | Never trades defensive alignment for bench evenness |
| 7 | No pitcher pre-rest concept | — | 60% of non-opening outings preceded by a sit |
| 8 | Catcher plan: non-contiguous, quota-spread, no catch-after-pitch | `index.html:566` | **Matches.** Leave alone |

Items 1–3 are all the same disagreement seen from three angles, and all three trace back
to one explicit prior decision recorded in the code: *"user ask 2026-04-23: spread =
max(sits) − min(sits) across non-iron players must not exceed 2."* The last 15 games say
that rule is now stricter than how the team is actually run. **That's a coaching call, not
a code call** — it should be changed deliberately, not inferred.

---

## Suggested implementation order

1. **Refresh the data first.** Apply the Pattern 9 `positionMap` corrections and populate
   `benchTiers` from observed sits/game. Roughly half the gap between engine output and
   coach output is stale inputs, not bad logic — and this step needs no engine change.
2. **Promote home-position continuity** from tiebreaker to dominant term in `fieldScore`,
   at a weight above the scarcity bonuses. This is the single highest-leverage change.
3. **Reframe bench fairness as tiered targets** — replace the flat `spread ≤ 2` gate in
   `multiRunSolve` with per-player sit targets derived from `benchTiers`, and drop the
   corresponding validation warnings for players whose target is met.
4. **Add pitcher pre-outing rest** as a bench pre-seed once the pitching plan is known.
5. **Seed inning 1 from the A-alignment** and rotate outward, rather than solving inning 1
   like every other inning.
6. **Default the pitching plan** to 3 × (2,2,3) for 7 innings.

Steps 3 and 5 change behaviour the coach previously asked for explicitly — worth
confirming before building.

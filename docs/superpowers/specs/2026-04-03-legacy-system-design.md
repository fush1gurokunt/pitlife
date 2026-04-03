# Legacy System Design
**Date:** 2026-04-03
**Project:** PitLife — F1 Career Simulator
**File:** `index.html` (single-file app)

---

## Overview

A generational legacy system allowing players to retire and continue as their child, chaining up to 3 generations. Each child inherits stat bonuses, nationality, helmet colours, and a seeded rival. The LOG tab surfaces family history, and the final career summary compares all generations side by side.

---

## 1. State Shape

Two new top-level fields added to game state `S`:

```js
S.generation   // integer 1–3, defaults to 1 on new game
S.legacy       // array of ancestor records, max length 2
```

Each legacy entry (appended when a generation ends):
```js
{
  name,            // full name string
  natFlag,         // emoji flag
  nat,             // nationality string
  helmet,          // helmet config object (colour, visor, pattern)
  stats: { Pace, Racecraft, Consistency, Fitness, Reputation, Morale },
  earnings,        // total career earnings (integer)
  championships,   // integer
  wins,            // integer
  podiums,         // integer
  seasonsInF1,     // integer
  mainRivalName,   // string — for seeding rival child
  mainRivalFlag    // emoji flag
}
```

`S.legacy.length` determines generation number at the start of each career. Generation 3 suppresses the legacy choice.

---

## 2. Retirement Trigger

**Voluntary retire button:**
- Shown on the Season tab, below the driver dossier stats
- Visible only when `S.stageIdx === 4` (F1) OR `S.age >= 28`
- On click: sets `S.gameOver = true`, `S.voluntaryRetire = true`, calls `save()`, then shows the retirement choice screen directly — `processSeasonEnd()` is NOT called; the current season is abandoned (no earnings recorded, last completed season stands in the log)
- The button uses the existing dimmed/bordered style to avoid visual prominence

**Existing forced endings** (age >= 40, burnout, fired twice) flow into the same retirement choice screen.

---

## 3. Retirement Choice Screen

Shown immediately after `processSeasonEnd()` completes, before the career summary. Uses the existing `champ-outcome` full-screen dark overlay pattern.

Content:
```
"Your story doesn't have to end here."

[Retire and pass the torch]    ← hidden if S.generation === 3
[This is my story]
```

- **Pass the torch:** appends parent record to `S.legacy`, triggers child setup
- **This is my story:** goes directly to career summary (`renderGameOver()`)
- If `S.generation === 3`: choice screen is skipped entirely, career summary shown with Dynasty heading

---

## 4. Child Career Setup

### Cinematic
A brief cinematic runs via `showCinematic()`:
```
"[Surname]. The name lives on."
"Meet [FirstName] [Surname]."
"Your legacy continues."
```

### Name Generation
- **Surname:** last word of parent's name (fallback: full name if single word)
- **First name:** coin flip selects same-gender or opposite-gender pool; random pick from that pool
- Two hardcoded pools of 8–10 names (gender-neutral enough to work for both genders)
- Pre-filled in the start screen name input, editable by player

### Start Screen
- Nationality step skipped — inherited from parent (flag and nat string copied)
- Helmet creator pre-filled with parent's helmet config — editable as normal
- `S.generation` incremented, `S.legacy` already updated before state reset

### Stat Inheritance
Applied after base stat initialisation:
```js
each stat += Math.floor(parentStat * 0.15)
S.earnings += Math.floor(parentEarnings * 0.10)
```
Stats are clamped to `[0, 100]` after inheritance using the existing `clamp()` helper.

### Rival Seeding
- `S.legacyRivalName = parentRecord.mainRivalName + ' Jr'`
- `S.legacyRivalFlag = parentRecord.mainRivalFlag`
- This rival appears in the F1 AI pool with base stats +5 across the board
- First F1 season opener cinematic gets an extra line:
  *"[RivalName] is already being compared to their parent. The old rivalry lives on."*
- `mainRivalName` on the parent record is populated at season end from the top AI rival in `S.seasonRivals[0]`

---

## 5. Family Legacy — LOG Tab

If `S.legacy.length > 0`, a "FAMILY LEGACY" section renders at the top of the Log tab above the season list.

One card per ancestor. Card contents:
- Name + flag
- Titles / Wins / Podiums / Earnings (four stat chips)
- A single generated legacy quote (first matching rule):

| Condition | Quote |
|-----------|-------|
| `championships >= 3` | "Three world championships. A true legend." |
| `championships >= 2` | "Multiple world titles. One of motorsport's greats." |
| `championships === 1` | "World champion. One of the greats." |
| `wins >= 5 && championships === 0` | "Never won the title. Came close more than once." |
| `seasonsInF1 >= 4` | "Spent [n] seasons at the top. A paddock institution." |
| default | "Went from karting to F1. The family name made it count." |

---

## 6. Career Summary

### Two-Generation (gen 2 career ends)
Below the normal career summary stats, a "vs Parent" comparison panel shows two columns:

| | Parent | You |
|---|---|---|
| Titles | n | n |
| Wins | n | n |
| Podiums | n | n |
| Earnings | £x | £x |

### Three-Generation (gen 3 career ends)
- Heading replaced with: **"The Dynasty — Three Generations of [Surname]"**
- Three-column table: Grandparent / Parent / You
- Normal career summary stats above the table as usual

---

## 7. What Is Not Changing

- All existing event logic, race simulation, stat progression
- The championship decider, qualifying overlay, team choice flow
- localStorage key and save/load format (new fields are additive)
- The start screen steps (nationality step is skipped for gen 2+, not removed)

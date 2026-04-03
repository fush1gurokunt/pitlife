# Manager Mode Design
**Date:** 2026-04-03
**Project:** PitLife — F1 Career Simulator
**File:** `index.html` (single-file app)

---

## Overview

A post-retirement team management mode unlocked by earning £50m+ during a driver career. The player buys a backmarker team, allocates budget, hires drivers, and races to win the Constructors Championship. Win = full cinematic + career summary. Lose (budget hits £0) = team folds, game over.

---

## 1. State Shape

New fields added to existing `S` object when manager mode begins:

```js
S.managerMode    // bool — gates all manager rendering
S.teamName       // string, max 20 chars, chosen at founding
S.teamBudget     // integer — S.earnings - 50_000_000 at purchase
S.teamCarRating  // integer, starts at 40 (backmarker)
S.teamDrivers    // array[2] of { name, flag, rating, age, salary }
S.teamLog        // array of season records:
                 //   { season, pos, pts, earnings, drivers[], carRating }
S.teamSeason     // integer, starts at 1
S.teamInfraLevel // integer 0–5, reduces costs each season by 5% per level
```

All fields saved to the same localStorage key as the driver career. No separate storage slot.

---

## 2. Unlock Flow

**Location:** Career summary screen (`renderGameOver`), below the normal career stats content.

**If `S.earnings >= 50_000_000` and not already in manager mode:**
```
"With £[X]m in the bank, you could buy your own team."
[Buy a team — £50m]   [Walk away]
```

**If `S.earnings < 50_000_000`:**
```
"Buy a team — £50m required"
"You retired with £[X]m. Not enough."
[button shown greyed out, non-interactive]
[Walk away]
```

**On "Buy a team":**
1. Deduct £50m from `S.earnings`, set remainder as `S.teamBudget`
2. Show team-naming screen (single input, max 20 chars, same style as driver name input)
3. Set `S.managerMode = true`, `S.teamCarRating = 40`, `S.teamSeason = 1`, `S.teamInfraLevel = 0`
4. `save()` then enter manager loop (show budget screen for Season 1)

**"Walk away"** → `startNewGame()` — completely fresh career, all state cleared.

---

## 3. Budget Screen

Shown as a full-screen overlay at the start of each team season, before driver market.

Three allocation rows with +/− steppers (£1m increments):

| Row | Effect |
|-----|--------|
| Car Development | Each £5m = +1 car rating, max +10/season |
| Driver Salaries | Amount available for driver market |
| Infrastructure | Each £3m = +1 infra level (max 5); each level = −5% costs next season |

A running "Remaining: £Xm" counter updates live. Cannot advance until remaining = £0.

At the start of each season, before the budget screen, an infra dividend is credited: `S.teamBudget += S.teamInfraLevel * 2_000_000`. This represents the cost savings from prior infrastructure investment and keeps the budget screen simple (player always allocates their full `S.teamBudget`).

---

## 4. Driver Market

Shown as a full-screen overlay after budget screen.

**Pool:** 12 candidates generated from combined `FEEDER_AI + F1_AI` name lists with randomised attributes:
- Rating: 25–90 (weighted toward 40–65)
- Age: 18–36
- Salary: scales with rating — approx `rating * 80_000` per season
- Flag: random from `RIVAL_FLAGS`

**Display:** 6 shown at a time (shuffle from pool of 12). Each card shows name, flag, rating, age, salary.

**Locking:** Drivers with rating > 70 show as locked (greyed, non-selectable) if `S.teamCarRating < 70`.

**Re-signing:** Previous season's drivers appear first with a "Re-sign" tag and 10% salary discount.

**Constraint:** Total salary of 2 selected drivers must not exceed the Driver Salaries allocation from the budget screen. Cannot advance until 2 drivers are selected within budget.

---

## 5. Race Season Simulation

Fully automatic — no player input during racing.

**Constructor score:**
```js
constructorScore = (S.teamCarRating * 0.6) + (avgDriverRating * 0.4) + rnd(-8, 8)
```

**AI field:** 9 AI constructor teams generated each season, ratings spread 30–95 (one dominant team at 85–95, two midfield at 60–75, rest 30–55).

Player's team is ranked against all 10 (including player). Championship points use `PTS_TABLE` per driver per round (24 rounds), summed for constructor total.

**Results display:**
- Constructor standings table (same style as existing `.standings` table)
- Drivers points breakdown (two rows, one per driver)
- No qualifying, no events, no decisions — pure results

---

## 6. End of Season

**Prize money by constructor position:**

| Pos | Prize |
|-----|-------|
| P1 | £40m |
| P2 | £28m |
| P3 | £20m |
| P4 | £14m |
| P5 | £10m |
| P6–P10 | £8m → £5m (linear) |

Prize money added to `S.teamBudget`. Season record appended to `S.teamLog`.

**Driver retention screen:**
- Each driver shown with their season stats and next-season salary (+10% raise if retained)
- "Keep" or "Release" button per driver
- Released drivers enter the pool next season; their slot filled from the market

**Carry-over:** Car rating and infra level carry over between seasons. No decay.

---

## 7. Win & Lose Conditions

**Win — Constructor P1** (triggers every season the player finishes P1; player can keep going until they choose "Walk away" or go bust):
- `showCinematic()` sequence:
  ```
  "SEASON FINALE"
  "CONSTRUCTORS' CHAMPION"
  "[Team Name] — built from nothing."
  ```
- Team career summary screen titled **"You Built a Champion"**
- Shows: seasons, constructor titles, total prize money, best/worst finish, drivers used
- "Walk away" → `startNewGame()`

**Lose — budget hits £0:**
- Checked at start of each season (after prize money credited)
- If `S.teamBudget <= 0`:
  - `showCinematic()`: *"The factory goes quiet. The dream is over."*
  - Team career summary (same screen, different heading: "The Dream Is Over")
  - "Walk away" → `startNewGame()`

---

## 8. UI Differences

`renderAll()` checks `S.managerMode` at the top and calls `renderManagerAll()` instead.

**Header (`renderManagerHeader`):**
- Team name replaces driver name
- Helmet SVG replaced with `🏭` icon
- Stage pill: "TEAM PRINCIPAL" in `var(--gold)`
- Driver age, veteran badge, and circuit badge hidden
- Season progress bar shows team season number

**Tabs:**
- "Season" → "Team" — shows team status card (car rating bar, budget, driver slots with ratings)
- "My Car" tab — hidden entirely (`display:none`)
- "League" → Constructors Championship table
- "Log" → team season history (same visual as driver log, fields swapped)

**No race button:** The bottom bar with "Race the Season" is hidden in manager mode. Racing is triggered by completing the budget + driver market flow.

**Budget/driver screens:** Implemented as full-screen overlays using the same `position:fixed;inset:0` pattern as existing overlays (`champ-overlay`, `qualify-overlay`).

---

## 9. What Is Not Changing

- All driver career logic, events, stats, race simulation
- The legacy system (`S.legacy`, `S.generation`)
- The start screen, helmet creator, nationality selector
- Save/load format (new fields are additive; existing driver saves load fine — `S.managerMode` will be `undefined` = falsy)
- The retirement choice screen (manager mode offer appears on career summary, after the legacy/retirement flow is complete)

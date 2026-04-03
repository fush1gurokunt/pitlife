# Economy & Shop System Design
**Date:** 2026-04-03
**Project:** PitLife — F1 Career Simulator
**File:** `index.html` (single-file app)

---

## Overview

A persistent in-game economy using PitCoins (🪙) earned through gameplay and purchasable with real money. A Shop tab lets players unlock premium helmet designs, gameplay features (Manager Mode, Legacy Dynasty, etc.), and consumable career boosts. Economy state persists across all careers in a separate localStorage key.

---

## 1. Economy State — Persistence

**localStorage key:** `pitlife-economy` (completely separate from `pitlife-save`)

**Default shape:**
```js
{
  pitcoins: 0,
  ownedDesigns: [],        // string IDs: 'stealth','tricolore','carbon','aurora','inferno','champion'
  ownedFeatures: [],       // string IDs: 'manager','legacy','rival_intel','weather','agent'
  ownedBoosts: {
    training: 0,
    sponsorship: 0,
    pr: 0,
    fitness: 0,
    secondchance: 0
  },
  lastLogin: null,         // 'YYYY-MM-DD' ISO string
  usedSecondChance: false  // reset to false at start of each new career (only cross-career exception)
}
```

**Helper functions:**
- `getEconomy()` — returns parsed object from localStorage, or defaults if missing
- `saveEconomy(e)` — serialises and saves
- `addCoins(n)` — get + add + save, updates header display
- `spendCoins(n)` — returns false if insufficient balance, otherwise deducts + saves
- `hasFeature(id)` — checks `ownedFeatures` array
- `hasDesign(id)` — checks `ownedDesigns` array

**New career:** `startNewGame()` calls `getEconomy()`, resets `usedSecondChance` to false, saves. All other economy data untouched.

---

## 2. PitCoins Earning

All awards call `addCoins(n)`. Coin balance shown in header updates immediately.

| Trigger | Where in code | Award |
|---------|--------------|-------|
| Complete a season | end of `processSeasonEnd()` | +50 |
| Race win | end of `processSeasonEnd()`, per win, max 4 | +25 per win |
| Championship win | end of `processSeasonEnd()` when `champPos === 1` | +200 |
| Reach F1 first time | `processSeasonEnd()` when `promoted && newIdx === 4` | +150 |
| Complete full career | `renderGameOver()`, guarded by `S.careerCoinAwarded` flag | +100 |
| Watch rewarded ad | `showMockAd()` callback | +75 |
| Daily login bonus | `checkDailyBonus()` on `DOMContentLoaded` | +20 |

**Daily login — `checkDailyBonus()`:**
Compares `new Date().toISOString().slice(0,10)` against `economy.lastLogin`. If different: `addCoins(20)`, update `lastLogin`, show toast. Toast is a small `position:fixed` div, slides up from bottom, auto-dismisses after 2.5s using the existing `@keyframes slideUp` animation.

---

## 3. Shop Tab

Fifth tab added to the tab bar: `🛒 Shop`. Tab labels shortened to fit: "Season" / "Car" / "League" / "Log" / "Shop".

PitCoins balance shown prominently at the top of the shop screen: `🪙 [balance]` in large Bebas Neue type.

Three sections, each a horizontally scrollable row (`display:flex; overflow-x:auto; gap:10px; padding-bottom:8px`):

---

### Section 1 — Helmet Designs

| ID | Name | Colours | Coin price | IAP only |
|----|------|---------|-----------|----------|
| `stealth` | Stealth | matte black + gold visor | 250🪙 | — |
| `tricolore` | Tricolore | blue/white/red | 150🪙 | — |
| `carbon` | Carbon | dark grey + carbon pattern | 300🪙 | — |
| `aurora` | Aurora | teal + purple gradient | 350🪙 | — |
| `inferno` | Inferno | deep red + orange | 200🪙 | — |
| `champion` | Champion | gold/white | — | £7.99 pack only |

Each card: small helmet SVG preview (using `buildHelmet()` with design colours), name, price, UNLOCK button (disabled if insufficient coins) or USE button if owned. Champion shows "🔒 Champion Pack only" unless owned.

Helmet design colour maps stored as a `HELMET_DESIGNS` const:
```js
const HELMET_DESIGNS = {
  stealth:   { base:'#111111', visor:'rgba(201,168,76,0.85)', detail:'#1a1a1a' },
  tricolore: { base:'#002395', visor:'rgba(255,255,255,0.9)', detail:'#ED2939' },
  carbon:    { base:'#2a2a2a', visor:'rgba(80,80,80,0.8)',   detail:'#444444' },
  aurora:    { base:'#008080', visor:'rgba(128,0,128,0.8)',  detail:'#20b2aa' },
  inferno:   { base:'#8B0000', visor:'rgba(255,140,0,0.85)', detail:'#cc2200' },
  champion:  { base:'#C9A84C', visor:'rgba(255,255,255,0.9)', detail:'#fff8dc' },
};
```

When a design is purchased and "USE" is clicked, it overwrites `S.helmet` fields and calls `save()` + `renderAll()`.

---

### Section 2 — Features

| ID | Name | Description | Coin price | IAP |
|----|------|-------------|-----------|-----|
| `manager` | Manager Mode | Buy a team on retirement | 800🪙 | £2.99 |
| `legacy` | Legacy Dynasty | 3-generation family careers | 600🪙 | £1.99 |
| `rival_intel` | Rival Intel | See rival stats in league table | 300🪙 | — |
| `weather` | Weather Forecast | See rain forecast before decisions | 200🪙 | — |
| `agent` | Agent | Team choice cards show exact numeric car rating (not just tier) | 400🪙 | — |

Each card: icon, name, one-line description, coin price, UNLOCK button (or OWNED badge). IAP price shown as secondary option where applicable.

---

### Section 3 — Career Boosts

| ID | Name | Effect | Price |
|----|------|--------|-------|
| `training` | Training Camp | +8 to one chosen stat | 150🪙 |
| `sponsorship` | Sponsorship Deal | +£500k to earnings | 100🪙 |
| `pr` | PR Blitz | +15 Reputation instantly | 120🪙 |
| `fitness` | Fitness Week | Skip next Burnout event this season | 130🪙 |
| `secondchance` | Second Chance | Convert one firing to a warning | 500🪙 |

Each card shows owned count, BUY button (multiples allowed except Second Chance — max 1 per career).

---

### IAP Section — "Support the Game"

Three cards below the three scrollable sections:

| Pack | Coins | Bonus | Price |
|------|-------|-------|-------|
| Starter Pack | 500🪙 | — | £0.99 |
| Pro Pack | 1800🪙 | — | £2.99 |
| Champion Pack | 5000🪙 | Champion helmet | £7.99 |

PURCHASE button on each: `console.log('PURCHASE: [pack]')` + `// WIRE TO STRIPE/ADMOB HERE` comment. Champion Pack immediately adds `'champion'` to `ownedDesigns` and awards 5000 coins.

---

## 4. Feature Gates

Checked at point of use. Each gate shows a shop prompt if the feature is not owned.

| Feature | Gate point | Locked behaviour |
|---------|-----------|-----------------|
| `manager` | Career summary buy-team button | Replace with: "Unlock Manager Mode in the Shop 🛒" (styled as disabled button) |
| `legacy` | Retirement choice screen | "Pass the torch" option hidden entirely; only "This is my story" shown |
| `rival_intel` | League tab rival rows | Stat columns replaced with `🔒` and `filter:blur(4px)` on values |
| `weather` | Header circuit badge + weather display | `S.weatherRainForecast` still simulated but badge never shown; rain icon never displayed |
| `agent` | Team choice cards (promotion + season-start) | Car rating bar hidden on team cards; salary still visible |

---

## 5. Boosts System

**Display:** A "BOOSTS" section rendered in the Season tab above the events list, only when any boost count > 0. Each owned boost is a small usable card with icon, name, and USE button.

**Boost behaviours:**

- **Training Camp:** USE opens an inline stat picker (6 buttons: Pace/Racecraft/Consistency/Fitness/Reputation/Morale). On pick: `S[stat] = clamp(S[stat]+8, 0, 100)`, save, re-render. Decrement count.

- **Sponsorship Deal:** USE immediately adds £500k to `S.earnings`, saves, shows toast "🪙 +£500k added to earnings". Decrement count.

- **PR Blitz:** USE immediately adds +15 to `S.Reputation` (clamped 0–100), saves, re-renders stats. Decrement count.

- **Fitness Week:** USE sets `S.fitnessWeekActive = true`, saves. In `genEvents()`, if `S.fitnessWeekActive` is true, the next Burnout-type event in the generated pool is removed; flag cleared after removal.

- **Second Chance:** USE sets `economy.usedSecondChance = true`, saves economy. In the `fired` path inside `processSeasonEnd()`, if `usedSecondChance` is true AND this is the first intercept, convert the firing to a warning: skip the fired state, show a results banner "⚠ Final Warning — your seat is safe this time." Reset `usedSecondChance` to false after use.

---

## 6. Rewarded Ads

**`showMockAd(callback)`** — single reusable function:
1. Shows full-screen overlay (grey `#888` background): "Ad playing… [countdown]"
2. Counts down 5 seconds (1s interval updating the number)
3. Shows "✓ Complete" with dismiss button
4. On dismiss: calls `callback()`
5. `// REPLACE WITH ADMOB REWARDED AD` comment at function definition

**Season tab banner** (above Race the Season button in bottom bar):
```
📺 Watch an ad — earn 75 🪙    [WATCH]
```
WATCH calls `showMockAd(() => addCoins(75))`.

**Post-bad-result offer** (in `renderResultsCard()` when `r.champPos > 8`):
```
"Tough season. Watch an ad to add +5 to one stat before next season starts?"
[📺 Watch Ad]
```
WATCH calls `showMockAd(() => openStatPicker(5))`. `openStatPicker(n, callback)` is a standalone function that renders a full-screen overlay with 6 stat buttons; on pick it applies +n (clamped 0–100), calls `save()`, dismisses the overlay, then calls `callback()` if provided. Shared by Training Camp boost and this ad offer.

---

## 7. Header & UI Changes

**PitCoins in header:** Shown in `renderHeader()` alongside driver name: `🪙 [balance]` in DM Sans 12px, colour `var(--gold)`. Updates on every `addCoins()` call via a direct DOM update (no full re-render needed).

**Tab bar:** Five tabs. Labels kept short. "My Car" → "Car". Tab pill font-size reduced to 9px if needed to fit all five.

**Shop tab render:** `renderShop()` called when `activeTab === 'shop'`. No bottom bar shown on shop tab.

---

## 8. What Is Not Changing

- All career game logic, race simulation, events, stats
- The legacy system and manager mode (economy gates them but doesn't change their internals)
- `S` state shape (no economy fields added to career save)
- `startNewGame()` beyond the single `usedSecondChance` reset
- Any existing localStorage key or save format

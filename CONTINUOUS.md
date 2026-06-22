# Fxaurum Rithmic — Continuous Futures Contract (back-adjusted rollover)

Pick a **root** (e.g. `ES`) → the chart **stitches the local contracts** behind the front month (`…ESH6 → ESM6 → ESU6`) into **one
continuous series**, **back-adjusted** so the price is seamless across each rollover (no gap at the contract change). Two
rollover rules (**date** / **volume**), and it applies to **candles + orderflow** (footprint / Volume Profile / big-trades) plus
**roll markers** at each switch. Built entirely from **local data** (expired contracts can't be replayed from Rithmic).

> Companion docs: **API.md** (`continuousInfo` + the `cont` param on `klines`/`aggTrades`/`bigTrades`/`volumeProfileRange`) ·
> **FOOTPRINT.md** / **VOLUMEPROFILE.md** / **BIGTRADES.md** (the orderflow it back-adjusts) · **STORAGE.md** (the Bar/Tick
> stores it reads) · **RECIPES.md**.

---

## 1. Which sockets to call

Continuous mode is **server-side stitching**: the client subscribes the **front** contract live, and adds a **`cont`** object to
every *historical* request — the gateway returns data already merged + back-adjusted, so the client renders it normally.

| Goal | Call | Notes |
|---|---|---|
| **Resolve the chain** (front + rollovers) | `continuousInfo {exchange, root, rule, adjust, rolls?}` | → `{ front, segments:[{sym, start, end, offset}] }`. The client subscribes `front` live and draws roll markers from `segments`. |
| **Continuous candles** | `klines {…, cont:{root,rule,adjust,rolls?}}` | Each per-contract segment is read + its **offset added to OHLC**, merged. |
| **Continuous footprint** | `aggTrades {…, cont:{…}}` | Closed-session ticks read per segment, **price shifted by offset**. |
| **Continuous big-trades** | `bigTrades {…, cont:{…}}` | Same shifted ticks → server-side cumulative-trade bubbles. |
| **Continuous Volume Profile** | `volumeProfileRange {…, cont:{…}}` | Per-session ticks shifted, binned by the **back-adjusted price**. |
| **Live** (front contract) | `@trade` / `@kline_<iv>` / `@bookTicker` on `front` | The front month, **offset 0** — normal subscribe, no `cont` needed. |

**`cont` object:** `{ root, rule:"date"|"volume", adjust:true|"1", rolls?:{ "<sym>":"<yyyymmdd>" } }`. Omit `cont` = a single
contract (normal chart).

---

## 2. The three modes (⚙ → "Chế độ")

| Mode | What it draws |
|---|---|
| **Đơn** (single) | One contract, exactly as you picked it (no stitching). |
| **Liên tục · date** | Continuous, rolling at each contract's **expiry-ish date** (or a date you type — see §4). |
| **Liên tục · volume** | Continuous, rolling when the **next contract's daily volume first exceeds** the current one's. |

Switching to a continuous mode: the client derives the **root** from the open symbol, calls `continuousInfo`, sets the live
symbol to the returned **front**, and passes `cont` on every historical request. The toolbar button shows the current mode.

---

## 3. Rollover rules & back-adjustment

**The chain.** The gateway lists every contract of the root that has **local bar data**, orders them by month/year, and the
newest is the **front**.

**Roll date** (where contract *i* hands off to *i+1*), per rule:
- **date** — the **last daily-bar date** of the older contract (≈ its expiry). Override with a typed date (§4).
- **volume** — the **first day** the newer contract's daily volume > the older's (falls back to the date rule if daily volume
  is missing).

**Back-adjustment** (when "Back-adjust" is on). At each roll, the gap between the two contracts is
`diff = close1d(newer, rollDay−1) − close1d(older, rollDay−1)` (the **prior day's settlement** from the Daily chart, the same
definition Sierra Chart uses). Offsets accumulate from the **front (0)** backwards: each older contract is shifted by the sum of
all `diff`s forward of it. Applying the offset to the older contract's **prices** makes the series continuous at the roll.

- **Back-adjust off** → you see the real **gap** at each contract change.
- Missing a daily settlement for a roll → that roll isn't adjusted (a yellow **warn** notice), the rest still are.

---

## 4. Settings ⚙ (gear next to "HĐ")

| Setting | Meaning |
|---|---|
| Chế độ (mode) | Đơn / Liên tục·date / Liên tục·volume |
| Back-adjust | Shift older contracts to remove the rollover gap (default **on**) |
| **Ngày rollover** (date mode only) | One **date input per rollover** (`ESM6 → ESU6 [date]`), populated from the chain. **Empty = auto** (shows the auto date as a hint); **type a date = force** that rollover day. |

The roll-date overrides are remembered per root (in the browser) and sent as `cont.rolls`. They apply **only in date mode**. The
roll boundary uses the **raw roll-day** (midnight UTC) so the date you type round-trips exactly. Editing a date **reloads the
chart** immediately.

The chain auto-lists **every** contract with data — when you later capture `ESZ6`, `ESH7`, … they appear as new rollover rows
automatically (no code change). The list is **exchange-aware** (gold `GC` → COMEX).

---

## 5. What's back-adjusted

| Layer | Source | Back-adjusted? |
|---|---|---|
| **Candles** | per-contract bars (`klines` cont) | ✅ OHLC + offset |
| **Footprint** | per-contract ticks (`aggTrades` cont) | ✅ price + offset |
| **Big-trades** | per-contract ticks (`bigTrades` cont) | ✅ price + offset |
| **Volume Profile** | per-contract ticks (`volumeProfileRange` cont) | ✅ binned by shifted price |
| **Roll markers** | `segments` boundaries | dashed vertical line + `⟲ old→new` label |
| **Live (front)** | `@trade`/`@kline`/`@bookTicker` | offset 0 (no shift) |

Scroll back to an older contract's region and the footprint/VP/big-trades line up with the back-adjusted candles; the dashed
lines mark exactly where contracts switch.

---

## 6. Data requirements & internals

- **Local only.** Expired contracts can't be replayed from Rithmic, so continuous uses what's in the stores. **Candles** need a
  contract's **1m + 1d** bars (1d also provides the back-adjust settlement); **orderflow** needs that contract's **ticks**. A
  region with bars but no ticks shows back-adjusted candles with empty footprint/VP there.
- **Resolver.** The chain (`front` + `segments`, each with a price `offset`) is computed once and cached ~5 min — recomputed when
  you force roll dates. The same split-and-shift logic backs every orderflow request: a range is split per contract and each
  segment's prices get its offset added.
- **Live = front.** Only the front contract is subscribed live (offset 0); everything older is historical local data. So the live
  candles/footprint align with the back-adjusted history at the most recent roll.
- The chain does **no month-cycle filtering** server-side (it lists any contract with data) — a leftover junk month would only
  appear if it has stale data in the store.

---

## 7. Changelog

- **2026-06-21** — Continuous futures contract: a server-side chain resolver + `continuousInfo` + a `cont` param on
  `klines`/`aggTrades`/`bigTrades`/`volumeProfileRange` (split per contract, back-adjust OHLC/price, merge); **two rollover
  rules** (date / volume) and **back-adjust** toggle; **candles + orderflow** (footprint/VP/big-trade) + **roll markers**; a ⚙
  gear with mode/back-adjust and a **per-rollover date editor** (auto or typed, round-tripped); auto-lists every contract with
  local data (exchange-aware). Live stays on the front month. Includes the expired-contract sync crash fix (see STORAGE.md).

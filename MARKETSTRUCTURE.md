# Fxaurum Rithmic — Market Structure & Contract Spec

Two lightweight, mostly low-frequency overlays built on Rithmic **R|API+** reference/session pushes: a **Market Structure**
overlay (settlement, daily price limits, market status, open interest, auction indicators) and a **Contract Spec** popup
(tick value, point value, margin, settlement method…). Neither needs history replay — both are **snapshot-on-subscribe** +
push-on-change, so they're cheap and appear instantly.

> Companion docs: **API.md** (`instrumentInfo`) · **RAW.md** (the raw `@On*` streams + `auxRefData` / `productRms` RPCs) ·
> **DOM.md** / **HEATMAP.md**.

---

## 1. Which sockets to call

### Market Structure (toolbar **MS** toggle)
Subscribes these **raw R|API+ streams** (`<exch>.<sym>@<callback>`, snapshot `Image` on subscribe, then push-on-change):

| Goal | Stream | Field(s) |
|---|---|---|
| **Settlement** price line | `@OnSettlementPrice` | `Price`, `PriceType` (`final`/`preliminary`), `Date` |
| **Projected settlement** (near close) | `@OnProjectedSettlementPrice` | `Price` (null outside the settlement window) |
| **Limit up / down** bands | `@OnHighPriceLimit` / `@OnLowPriceLimit` | `Price` (`"NaN"` = band not published) |
| **Market status** | `@OnMarketMode` | `MarketMode` (`Open`/`Closed`/`Pre-Open`/`Pause`/`Halt`/`Auction`…), `Reason`, `Event` |
| **Open interest** (futures) | `@OnOpenInterest` | `Quantity`, `QuantityFlag` (EOD figure — updates ~once/day) |
| **Auction indicative** open/close | `@OnOpeningIndicator` / `@OnClosingIndicator` | `Price` (IOP/ICP; `NaN`/null outside the auction window) |

### Contract Spec (**ⓘ** button next to the symbol)
Pull RPCs (reply inline):

| Goal | Call | Field(s) |
|---|---|---|
| Tick size, point value, currency, expiration | `instrumentInfo {exchange,symbol}` | `tickSize`, `pointValue`, `currency`, `expiration`, `description`, `instrumentType` |
| Settlement method, unit, trading dates | `auxRefData {exchange,symbol}` | `SettlementMethod` (`cash`/`physical`), `UnitOfMeasure`, `UnitOfMeasureQty`, `FirstTradingDate`, `LastTradingDate` |
| Account RMS limits | `productRms {accountId?}` | per-product `BuyLimit`/`SellLimit`, `MaxOrderQty`, `LossLimit` (often unset on demo) |

All raw streams + RPCs need the same entitlement as normal market data; `productRms` needs the trading entitlement (degrades
gracefully to "—" when absent).

---

## 2. Market Structure — what you see

Toggle **MS** (per-symbol, saved as `msOn`). On the chart:

- **Settlement line** — gold **solid** (`Settle`); **projected** settlement gold **dashed** (`Settle~`, only near the close).
- **Limit up** — red dashed (`Limit↑`); **limit down** — green dashed (`Limit↓`). A band Rithmic reports as `NaN` (not
  published) draws nothing — common for index futures where only one side is active.
- **Auction indicative** — purple `IOP` (indicative open) / blue `ICP` (indicative close), only inside the auction window.

And a **badge** next to the MS button:
- **Market status** — color-coded: **Open** green · **Halt/Pause** red · **Auction/Pre-Open** amber · **Closed** grey.
- **OI** — futures open interest (e.g. `OI 1.9M`).
- **stl ±x (±%)** — the current price vs settlement (the market's up/down on the day), updated live.

### Notes on the data
- **Settlement** is the exchange's official daily mark price (mark-to-market / daily % change reference) — *not* the last
  trade. Price above the line = up on the day, below = down.
- **Open interest** is an **end-of-day** figure — it updates ~once/day, not intraday (so there's no meaningful intraday OI
  chart; the badge value is the point). OI is symmetric (longs = shorts by definition) — it can't be split by side.
- **Limits** are the exchange circuit-breaker bands (e.g. index futures −7/−13/−20 % vs settlement), usually far from price →
  off-screen unless you zoom out.

---

## 3. Contract Spec — what you see

Click **ⓘ** (next to the symbol). A popup shows:

- **Contract**: description · symbol · exchange · type · expiration.
- **Tick size** · **tick value** (`= tickSize × pointValue`, e.g. GC $10, ES $12.50) · **point value** ($) · **currency**.
- **Settlement method** (cash / physical) · **unit of measure** (× qty) · **last trading date** (from `auxRefData`).
- **RMS limits** (from `productRms`): position buy/sell limit, max order qty, loss limit — shown when set (blank/"—" on
  data-only or demo accounts).

The core specs come from `instrumentInfo` (already cached per symbol), so the panel is instant; `auxRefData` / `productRms`
fill in asynchronously.

---

## 4. Internals

- Both are client-only overlays. Market Structure subscribes the six/eight raw streams on toggle, keeps a small `msData`
  state, redraws LWC **price lines** (`createPriceLine`) on each push, and updates the badge. Lines/streams are torn down on
  toggle-off and re-subscribed on symbol change. All values guard against `"NaN"` (parse → skip).
- Contract Spec is a popup rebuilt on open from the cached `instrumentInfo` plus two async RPCs.
- The raw `@On*` streams are the same **reflective OnAny passthrough** documented in **RAW.md** (ref-counted, snapshot-cached
  so a fresh subscribe gets the current value immediately). Feed rate is negligible (these push only on change).

---

## 5. Changelog

- **2026-07-01** — **Market Structure overlay** (`MS`): settlement + projected-settlement lines, limit up/down bands,
  market-status badge, futures open interest, price-vs-settlement, and opening/closing **auction indicators** (IOP/ICP) —
  all from R|API+ low-frequency raw pushes. **Contract Spec panel** (`ⓘ`): tick value / point value / currency / expiration
  (`instrumentInfo`), settlement method / unit / last-trading-date (`auxRefData`), account RMS limits (`productRms`).

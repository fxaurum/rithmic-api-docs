# Fxaurum Rithmic — DOM (ATAS SmartDOM-style)

A vertical **order book ladder** (depth of market) modelled on **ATAS SmartDOM**. Each row is a price level showing
resting bids/asks, executed **aggressive** volume at price, full-session volume, and order-book changes. The panel has a
**fixed cell size** (independent of the chart zoom) and **auto-centers** the current price.

> Companion docs: **API.md** (message schemas · `@depth`, `@trade`, `aggTrades`) · **MBO.md** (order-by-order / DBO — the
> per-order upgrade of this ladder) · **FOOTPRINT.md** · **RECIPES.md** §5 (DOM build-it-yourself) · **BIGTRADES.md**.

---

## 1. Which sockets to call

The DOM combines **3 sources** — the live book, volume-at-price (history + live), and best bid/ask (optional highlight):

| Goal | Call | Why |
|---|---|---|
| **Bid / Ask + Chg** (book) | `@depth` (stream) | resting size per level + delta between snapshots. The gateway maintains the book from quotes and emits ~30 Hz. |
| **Today / Sells / Buys** (volume at price) | `aggTrades` (history) + `@trade` (live) | full-session volume per price, split by aggressor. Loaded for the **whole day**, independent of the footprint window. |
| **Current price** (yellow row + line) | `@trade` | marks the last trade price; auto-centers on it. |
| **Best bid/ask highlight** (optional) | `@bookTicker` | brightens the two best levels. |
| **Historical day** | `aggTrades` → store | replay empty (closed session) → falls back to the **local tick store** for that day. |
| **Tick size + decimals + expiration** | `instrumentInfo` | authoritative from Rithmic — correct price decimals (e.g. GC 0.1), no inferring the tick from prices. |

**Minimum viable DOM** = `@depth` (book) + `@trade` (volume at price + last). `@bookTicker` only highlights best. Bid/Ask
requires **L2 (market depth)** entitlement from Rithmic.

---

## 2. Columns (ATAS SmartDOM-style)

Layout left→right: **Today · Price · Chg · Bid · Sells · Buys · Ask · Chg**. Bid = **green**, Ask = **red** (ATAS); Today =
**blue**; current price = **yellow** row + white horizontal line.

| Column | Shows | Source |
|---|---|---|
| Today | Full-session volume at price (buy+sell), blue bar | `aggTrades`+`@trade` |
| Price | Price ladder; current price highlighted | `@trade` |
| Chg (bid) | Bid size change between snapshots: + green added / − red pulled, fades over ~4 s | `@depth` |
| Bid | Resting bid size (green bar; best brighter) | `@depth` |
| Sells | Executed **aggressive sell** volume at price | `@trade` |
| Buys | Executed **aggressive buy** volume at price | `@trade` |
| Ask | Resting ask size (red bar; best brighter) | `@depth` |
| Chg (ask) | Ask size change (like Chg bid) | `@depth` |

**Today/Sells/Buys** are the same footprint data but aggregated for the **whole session** by price (not per bar). Buy/sell
split by the trade `side` (tick rule when empty) — same convention as the footprint.

---

## 3. Interaction & centering

| Action | Result |
|---|---|
| Mouse wheel / drag | Scroll the price axis up/down (disengages auto-center) |
| Double-click | Re-center on current price (= ATAS **C** button) |
| Auto-center | Pulls price back to the middle when it drifts **≥ 10 ticks** (no per-tick jitter) — like ATAS |
| Flash | The just-traded row lights up ~0.45 s then fades |

Fixed row height per tick (panel does **not** follow the chart's vertical zoom). With footprint on, the *chart* candles turn
hollow when zoomed in / solid when zoomed out — the DOM panel is separate and always fixed.

---

## 4. Settings (gear ⚙ next to the DOM toggle)

| Setting | Meaning | ATAS equivalent |
|---|---|---|
| Show/hide columns | Toggle **Today · Chg · Sells · Buys**. Price · Bid · Ask always shown; remaining columns auto-stretch. | column visibility |
| Depth | Levels each side of the current price to **render**; **0 = fill** the panel | (display depth) |
| View day | Pick a day to view history (reads store); "Hôm nay" returns to live | — |

> **Data depth** (how many levels the gateway actually streams, `@depth<N>`) is set in the **Heatmap ⚙ → "Độ sâu sổ lệnh"** (default **100**, max 1000) and is **shared** by the DOM and the heatmap — they subscribe one `@depth<N>` stream. The DOM's own *Depth* above only limits how many of the streamed levels are drawn. Raise the heatmap setting to see deeper resting liquidity (heavier feed).

---

## 5. Historical day & local tick store

Pick a day in ⚙ → the DOM shows **that day's volume profile (Today/Sells/Buys)**, read from the **local tick store**
(Rithmic can't replay a closed session). History mode is **frozen**: no live trades, no Bid/Ask (no historical book), with a
"📅 yyyy-mm-dd (history)" badge.

The **local tick store** records every live trade to disk (one file per **trading session** — CME roll 17:00 CT) and **auto-backfills the full session** each time you open the
chart during the session (from Rithmic replay → merged into the store). Consequences:

- Any day the app ran (and opened a chart) has history.
- A closed session the app missed is **unrecoverable** — Rithmic tick replay only serves the **currently open session**.
- Backfill covers the **current day only**; the store is the long-term archive, accumulating day by day.

---

## 6. Gateway internals

- `OnLimitOrderBook` is a one-shot snapshot; the live book is maintained from **`OnBidQuote`/`OnAskQuote`** (per-level
  updates) into a per-symbol `DomBook` (`SortedDictionary`, bids high→low / asks low→high), emitted as `@depth` (~30 Hz,
  coalesced) with a trailing flush and a ~1.5 s resync.
- Each emit wraps the book in a **`DepthBox`** (up to `DOM_MAX_LEVELS` = 1000 nearest-to-price each side); `Route` then
  serves each subscriber its own slice — `@depth<N>` → `Top(N)`, plain `@depth` → `Full()`. So per-client depth is honored
  on the live book, not just snapshots. Rithmic's MBP (`Quotes` flag) supplies hundreds of levels — enough for this
  **aggregated** ladder. For **order-by-order** (the individual orders behind each level, queue position, iceberg/spoof) the
  app has a separate **MBO** mode on Rithmic **DBO** — see **MBO.md** (separate entitlement; not required for this DOM).
- On a `@depth` subscribe the gateway calls `RebuildBook` for an immediate snapshot.
- `aggTrades` prefers **Rithmic replay** (full session, direct); a **full-session request** (window from the session open, 17:00 CT) is merged
  into the `TickStore` (one file per session) and marks the session backfilled; smaller requests trigger a background full-session backfill.

---

## 7. ATAS SmartDOM parity

| SmartDOM feature | Status |
|---|---|
| Today (session volume profile) | ✅ |
| Price · Bid · Ask · Sells · Buys | ✅ |
| Change Bid / Change Ask (Chg) | ✅ |
| Auto-center 10 ticks + C button (double-click) | ✅ |
| Latest-trade flash | ✅ |
| Show/hide columns, depth, view past day | ✅ |
| **Order-by-order (MBO / DBO)** | ✅ (DOM ⚙ "MBO" toggle — per-order blocks, queue, Vol/count, filters; see **MBO.md**) |
| Orders / PnL | ⬜ (needs order routing) |
| Liquidity Map (heatmap) | ⬜ (separate heatmap panel; MBO heatmap = MBO.md Phase 2) |
| Notes | ⬜ |

---

## 8. Changelog

- **2026-06-16** — Configurable book depth: `@depth<N>` now trims the **live** book per-subscription via `DepthBox`/`Route.Top(N)`
  (was a fixed 100-level cap). Data depth is a shared **Heatmap ⚙ "Độ sâu sổ lệnh"** setting (default 100, max `DOM_MAX_LEVELS`=1000),
  driving both the DOM ladder and the heatmap. Confirmed Rithmic MBP (`Quotes`) provides hundreds of levels — full ATAS-like depth
  without MBO.
- **2026-06-15** — DOM ladder: ATAS SmartDOM columns (Today/Price/Chg/Bid/Sells/Buys/Ask/Chg), green/red bid/ask, current-price
  line, auto-center (10-tick) + double-click recenter, latest-trade flash, column show/hide + depth settings, full-day Today,
  historical day viewer (day picker) from the local tick store, gateway book maintained from per-level quotes + full-day store
  backfill, authoritative tick size/decimals via `instrumentInfo` (correct price formatting, e.g. GC 0.1).

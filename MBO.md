# Fxaurum Rithmic — MBO (Market-By-Order, order-by-order DOM)

An **order-by-order** depth of market built on Rithmic **DBO (Depth-By-Order)**. Where the regular [DOM](DOM.md) shows the
**aggregated** size resting at each price (Level 2 / MBP), MBO shows the **individual orders** that make up that size — so you
can tell **one 500-lot order** from **fifty 10-lot orders** at the same price, see the **queue** (priority), and (Phase 2)
detect icebergs/spoofs.

> Companion docs: **DOM.md** (the aggregated ladder this extends) · **API.md** (`@mbo` / `mboProbe` schemas) · **HEATMAP.md**
> (Phase 2 will reuse this book engine) · **STORAGE.md**.

---

## 1. Which sockets to call

| Goal | Call | Why |
|---|---|---|
| **Order-by-order book** (per-level orders) | `@mbo` / `@mbo<N>` (stream) | each price level returns `[price, orderCount, total, [sizes…]]` — the individual resting orders, queue-ordered. Gateway maintains a per-order book from Rithmic DBO and emits ~20 Hz. |
| **Raw DBO events** (per order, no aggregation) | `@mboraw` (stream) | passthrough of **every** Rithmic DBO event — `New`/`Change`/`Delete`/`Image` with `ExchOrdId`, side, price, size, priority and exchange timestamp. No conflation, no book-building. Use when you need the raw order-flow tape, not a ladder. |
| **Aggregated total** (per level) | same `@mbo` (field `total`) **or** `@depth` | MBO `total` = Σ of the orders in the window; `@depth` gives the full-book total at every level (deeper than the MBO window). The ladder shows both. |
| **Verify entitlement / inspect raw DBO** | `mboProbe` (RPC, debug) | subscribes DBO around BBO for a few seconds and returns `{entitled, events, distinctOrders, hasExchOrdId, hasPriority, sample[]}`. |
| **Best bid/ask, last, tick size** | `@bookTicker`, `@trade`, `instrumentInfo` | same as the regular DOM. |

**Entitlement:** DBO is usually a **separate Rithmic market-data subscription** from L1/L2. If your plan lacks it,
`subscribeDbo` is rejected (surfaced via an alert) and `@mbo` stays empty. Use `mboProbe` to confirm before relying on it.

### `@mbo` payload

```json
{ "stream":"comex.gcq6@mbo20", "data":{
    "s":"GCQ6",
    "bids":[ [4152.3, 4, 31, [15,8,5,3]], [4152.2, 2, 20, [12,8]], … ],
    "asks":[ [4152.7, 1, 250, [250]], … ] } }
```

Each level = `[price, orderCount, totalSize, sizes[]]`. `sizes[]` are the individual orders **sorted by queue priority**
(front of queue first), **capped at 64 per level** (`orderCount`/`total` are still the full figures — so a level with >64
orders reports the true count/total but lists only the front 64 sizes). `@mbo<N>` = **N book levels each side** get per-order
DBO and are emitted; **`N` large (e.g. 1000) = the whole book**, order-by-order to the bottom (like ATAS). `N` drives both how
many levels the gateway subscribes DBO for (book-driven, see §4) **and** the slice width of the emit.

### `@mboraw` payload (raw DBO passthrough)

One message **per DBO event** (not conflated):

```json
{ "stream":"comex.gcq6@mboraw", "data":{
    "e":"mboraw", "s":"GCQ6",
    "type":"Change",        // New | Change | Delete | Image
    "side":"B",             // B | S
    "p":4152.3,             // price
    "q":8,                  // size  (-1 when the feed sends no size, e.g. Delete)
    "oid":"8246531",        // ExchOrdId — the order key
    "pri":"...",            // exchange queue priority (string as Rithmic gives it)
    "ts":1718000002360 } }  // exchange source time (ms)
```

**Window:** `@mboraw` rides the same per-price DBO window as `@mbo` — the window is **book-driven**: the gateway subscribes
DBO for the **nearest `N` levels each side that actually exist in the `@depth` book** (not a fixed ±tick band), where `N` =
the largest `@mbo<N>` request on that symbol, capped at `MBO_WIN_MAX` (1000). **`N` = 1000 (or the app's "Độ sâu MBO = 0")
subscribes the whole book** — every populated level to the bottom of the ladder. Same entitlement as `@mbo`.
Order-of-magnitude higher message rate than `@mbo` — only subscribe when you consume every event; the gateway skips the
passthrough entirely when no client holds `@mboraw`.

---

## 2. What you see (ladder, MBO mode)

Turn on the **DOM** panel, then **⚙ → "MBO — order-by-order"**. The Bid/Ask columns change from one solid bar to
**one block per order**:

- **Per-order blocks** — each resting order is its own block, **width ∝ its size** (a base "cell" width for size-1, growing
  proportionally), drawn in **queue order** from the price axis inward (bid grows left, ask grows right). The order's size is
  printed **inside** the block when it's wide enough.
- **`×N`** — the order **count** at that level (e.g. `×4` = four separate orders).
- **Vol** — a dedicated column with the per-level **total** size (no longer overlaps the blocks).

So a level reading `Vol 31 · ×4 · [15][8][5][3]` = 31 lots resting as 4 orders of 15/8/5/3, the 15 at the front of the queue.

---

## 3. Settings (DOM ⚙, MBO section)

| Setting | Meaning |
|---|---|
| **MBO — order-by-order** | Master toggle. Subscribes `@mbo<N>` (additive — does **not** disturb the `@depth`/heatmap streams). |
| **Độ sâu MBO (per-order)** | `N` levels each side that get per-order DBO. **`0` = full** (subscribe the whole `@depth` book, like ATAS). The plain `@depth` ladder is **always full** regardless — this only sizes the per-order layer (cost: one `subscribeDbo` per level, throttled — see §4). Shared with the heatmap MBO window. Default **0**. |
| **Cỡ ô mỗi lệnh** (cell size, px) | Base width of one order block (size-1). Larger orders scale up from this. 2–40 px. |
| **Tô đậm lệnh ≥** (highlight ≥) | Orders at/above this size render at full opacity (spot the big single orders). `0` = off. |
| **Ẩn lệnh <** (hide <) | Orders below this size are not drawn (de-clutter the size-1 noise). `0` = off. |

**Panel auto-width:** the DOM panel **widens to fit the densest row** in the MBO window (most/largest orders), so when you
show all orders nothing is clipped; the **other columns keep their normal width** (the panel grows, the chart reflows).
Raising the cell size widens it further. Per-symbol (saved in `domCols`).

---

## 4. How it works (gateway internals)

**DBO is per-PRICE.** Rithmic's `subscribeDbo(exch, sym, price)` subscribes the order-by-order feed **for one price level**.
There is **no "subscribe the whole book" call** in `rapiplus.dll` (verified by reflecting the DLL) — but it **does** serve
per-order at **any** depth. So to match ATAS's full-book MBO the gateway subscribes DBO for **every level it wants**, one call
each, and maintains a **book-driven window**:

- A `MboBook` per symbol holds `ExchOrdId → {price, size, side, priority, firstMs, lastMs}`.
- A 60 ms timer (`MboTick`, guarded by `Interlocked` re-entry so a slow fill can't double-subscribe) keeps DBO subscribed to
  the **nearest `WinTicks` levels each side that exist in the `DomBook`** (the real `@depth` ladder — not a fixed ±tick band),
  where `WinTicks` = the largest `@mbo<N>` request, capped `MBO_WIN_MAX` (1000). **`N=1000`/`0` = the whole book, to the
  bottom.** New levels are subscribed **nearest-first and throttled** (`MBO_SUB_PER_TICK` = 25 per pass) so the initial fill
  doesn't burst ~1375 `subscribeDbo` calls at once (that trips a Rithmic **rate limit** → `OMException`; a throttled fill
  reaches full in a few seconds). Ref-counted per symbol like `_mdRef` — N charts share one set of DBO subs.
- `OnDbo` events update the book: `Image`/`New` add, `Change` updates (price/size/priority), `Delete` removes. Prices are
  rounded to the tick grid (Rithmic returns floats like `4151.900000000001`).
- **Crossed-order prune:** if the market moves through a resting order (a bid ≥ the best ask, or ask ≤ best bid) the stale
  order is dropped from the book and a synthetic `Delete` is emitted on `@mboraw`, so neither `@mbo` nor the heatmap shows a
  bid stuck above the ask.
- Every tick the book is aggregated per price (group orders by price, sort by `Priority`) and emitted as `@mbo` (~20 Hz,
  coalesced), wrapped in an `MboBox` so `Route` can slice `@mbo<N>` → `Top(N)` per subscriber (mirrors `DepthBox`/`@depth`).
- On reconnect the engine loses all DBO subs → the book's subscribed-price set is cleared so the timer re-subscribes fresh.

`DboInfo` fields used: `ExchOrdId` (unique order id), `Priority` (queue position), `Size`, `Side`, `Price`/`PreviousPrice`,
`DboUpdateType {Image, New, Change, Delete}`, nanosecond source timestamps.

---

## 5. Entitlement probe (`mboProbe`)

`{"method":"mboProbe","params":{"exchange","symbol","levels","ms"}}` — subscribes DBO `levels` each side of BBO for `ms`,
then returns a summary and unsubscribes. Run it from the browser console **after opening the symbol's chart** (needs a BBO):

```js
rpc("mboProbe",{exchange:"COMEX",symbol:"GCQ6",levels:6,ms:8000}).then(r=>console.log(JSON.stringify(r,null,2)))
```

| Field | Meaning |
|---|---|
| `entitled` | `true` = account receives DBO (build/use MBO). `false` = rejected → reason in `alerts[]`. |
| `events` / `new` / `change` / `delete` / `image` | DBO events received in the window. |
| `distinctOrders` | distinct `ExchOrdId`s seen. |
| `hasExchOrdId` / `hasPriority` | whether the exchange supplies per-order id + queue priority (needed for queue + Phase-2 iceberg/spoof). |
| `sample[]` | first 40 events: `[type, side, price, size, exchOrdId, priority, tsMs]`. |

---

## 6. ATAS MBO DOM parity

| ATAS "MBO DOM" feature | Status |
|---|---|
| Per-order blocks (width ∝ size, queue order) | ✅ |
| Per-level **V** (total) + **C** (count) | ✅ (Vol column + `×N`) |
| Order-size number inside the block | ✅ (when wide enough) |
| Highlight large orders / hide tiny | ✅ (filters) |
| Level-2 fallback where MBO absent | ~ (overlays on the `@depth` ladder, which already covers all levels) |
| **Full-book per-order (whole ladder, not a ±band)** | ✅ (`Độ sâu MBO = 0`; `@depth` always full) |
| Per-order Ctrl-hover tooltip (price/vol/time/id/priority) | ⬜ |
| MBO **Heatmap** — per-order lifetime bars (lanes + size, on the chart) | ✅ (see [HEATMAP.md](HEATMAP.md) §5) |
| MBO **Heatmap** — per-order reconciled to the DOM (no drift) | ✅ (client reconciles `@mboraw` ↔ `@mbo` each 300 ms) |
| MBO Heatmap — time-weighted EMA + spoof subtraction | ⬜ (Phase 2) |
| Iceberg / spoof detector | ⬜ (Phase 2) |

---

## 7. Roadmap (Phase 2)

The book engine already keeps each order's **lifetime** (`firstMs`/`lastMs`), which is the input for:

- **MBO Heatmap** — ✅ **built** (lifetime-bar form): `@mboraw` drives one rectangle per order on the chart heatmap
  (width = lifetime, lane-stacked), **reconciled against `@mbo` every 300 ms so it matches the DOM tick-for-tick** (no
  event-stream drift), see [HEATMAP.md](HEATMAP.md) §5. Still on the roadmap: a **time-weighted EMA** form (resting liquidity
  brightens; flicker fades) with **spoof subtraction** (young + unfilled orders excluded).
- **Iceberg / spoof detection** — track each `ExchOrdId`: refills at the same price (iceberg), big unfilled orders pulled
  fast far from the BBO (spoof) → highlight + alert. The per-order `@mboraw` ids/lifetimes are **kept** (the reconcile only
  patches display gaps with `@mbo`), so the raw material for this is in place.
- **Per-order tooltip** — hover a block to read its id/priority/size/age.

---

## 8. Changelog

- **2026-07-01** — **Full-book per-order + heatmap reconcile.** DBO is now subscribed for the **whole `@depth` book**
  (book-driven window, `Độ sâu MBO = 0` = full to the bottom of the ladder, like ATAS) instead of a fixed ±20-tick band;
  the initial fill is **throttled** (`MBO_SUB_PER_TICK` = 25/pass, re-entry-guarded) to dodge the Rithmic rate-limit
  `OMException`, and crossed orders are pruned (synthetic `@mboraw` `Delete`). On the client, the heatmap's per-order bars are
  **reconciled against `@mbo` every 300 ms** (add-missing with real `@mbo` sizes / close-phantom) so they **match the DOM**
  despite `@mboraw` drift, plus a per-price **dead-order cap** (largest-kept). See [HEATMAP.md](HEATMAP.md) §5.
- **2026-06-30** — **MBO Heatmap (lifetime-bar form) shipped** on the chart: `@mboraw` now drives one rectangle per order
  (width = lifetime, lane-stacked, side-colored big-order highlight) in the heatmap's **MBO** mode — see
  [HEATMAP.md](HEATMAP.md) §5. Gateway pushes the current `MboBook` as `Image` events on `@mboraw` (re)subscribe so a chart
  enabling MBO mid-session sees the already-resting orders. (Time-weighted EMA + iceberg/spoof classification remain Phase 2.)
- **2026-06-26** — Added **`@mboraw`** stream: raw Rithmic DBO passthrough (one message per `New`/`Change`/`Delete`/`Image`
  event with `ExchOrdId`, side, price, size, priority, exchange ts). Rides the same per-price DBO window as `@mbo`;
  gateway skips the passthrough entirely when no client holds it. (DOM `@depth` remains aggregated-only — no raw form.)
- **2026-06-19** — MBO (Phase 1 + 1.5). Gateway: `MboBook` per-order engine on Rithmic DBO, rolling ±20-tick window around
  BBO (ref-counted, ~20 Hz emit), `@mbo`/`@mbo<N>` stream, `mboProbe` debug RPC. Entitlement confirmed on COMEX.GCQ6.
  Web app: DOM "MBO" toggle drawing per-order blocks (width ∝ size, queue order, in-cell size number), dedicated **Vol**
  column, `×N` count, highlight/hide filters, configurable cell size, and a panel that auto-widens to the densest row while
  keeping the other columns at their base width.

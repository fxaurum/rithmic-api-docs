# Fxaurum Rithmic — Big Trades (ATAS-parity tracker)

Large-order / block / sweep detection, modelled on **ATAS *Big trades*** (and equivalent to DeepChart's
big-order bubbles). This is a **living parity document**: each capability is mapped to its ATAS behaviour so
you can track, session after session, how close the reference web chart sits to ATAS — and which gaps remain.

> Companion docs: **API.md** (message schemas · `@trade`, `aggTrades`) · **RECIPES.md** §1 (build-it-yourself
> algorithm) · **FOOTPRINT.md**.

**Status legend:** ✅ matches ATAS · 🟡 approximated (close, not identical) · ⬜ not built yet · ❌ cannot be
matched exactly (ATAS does it server-side / proprietary).

---

## 1. What it does

A "big trade" is an unusually large aggressive order. The chart flags it where it happened — a **bubble** at the
trade price (sized by volume, with a pop animation), or, when one order **swept several price levels**, a
**vertical range box** spanning from the first to the last fill. Buy = green, sell = red. Works on **history**
(replayed/seeded) *and* updates **live**.

The core idea (same as ATAS *CumulativeTrades*): the fills of **one aggressor order** are merged into **one
logical trade**, then thresholded — so a single order walking through five prices shows as ONE marker, not five.

---

## 2. The data that makes ATAS parity possible

ATAS groups fills into a **`CumulativeTrade`** and keys it by its **`.Time`** (the exchange transact time) — so
in practice it groups by the **matching event**, not by the aggressor order id. Rithmic gives us both, and the
gateway forwards both:

- `@trade` field **`tu`** / `aggTrades` 6th element — exchange match time in **microseconds**
  (`SourceSsboe`·1e6 + `SourceUsecs`). Every print of one matching event (one aggressing sweep) shares it.
  **Primary key** — matches what ATAS keys on.
- `@trade` field **`o`** / `aggTrades` 5th element — `AggressorExchOrdId`, the aggressor order id (Rithmic *does*
  supply it, verified live). Useful, but **not** the primary key: a single order can fill across *several* match
  events (a resting/iceberg order eaten over time), so grouping by `o` would merge those into one oversized
  bubble that ATAS shows as separate trades. Kept only as a fallback when `tu` is missing.

The client groups consecutive prints by **same `tu`** → one cumulative trade. Fallbacks, in order: `tu` (matching
event) → `o` (aggressor order) → a fixed internal ~50 ms same-side window (only when neither survives). This is the
**default** (precise, no time field). An optional **`Cửa sổ gộp` (merge, ms)** slider switches to an ATAS-style
*cumulative window* — group **all same-side** aggressive prints within N ms regardless of `tu`/`o` → fewer, larger
bubbles, closer to how ATAS's "Cumulative Trades" looks. `0` keeps the precise default; dial it up to taste. The
gateway's `bigTrades` request takes the same **`merge`** param (and bypasses its per-session cache when set), so
history and live group identically.

> **Why a knob, not a fixed ATAS clone:** measured on one ESM6 candle, ATAS bubbles (69/61/56/54/50) sit *between*
> our matching-event grouping (max ~51) and a 100 ms window (max ~100) — ATAS's exact aggregation is proprietary,
> so an exact match would overfit one candle. The slider lets the user land on the look they want instead.

The local tick store persists `tu` **in-record** (v2 format: one `.tick` file per CME session, 29-byte records with
`tu` inside — the old `.fxu` sidecar is gone, which fixed the `.fxt`/`.fxu` alignment bug), so **history reviewed
from the store groups as precisely as live**. Only days recorded by very old builds are `tu`-less (ms window).

> Live capture confirming a sweep: three prints share one `tu` (here one `o` too), side B, 2+12+15 = **29** at one
> price — one matching event, one bubble. The trailing 1-lot prints each have a distinct `tu`, so they stay
> separate. (Grouping by `o` instead was measured to drift *higher* than ATAS — hence `tu` is primary.)

> The earlier ms-only path lost the sub-millisecond match time and had to guess with a 50 ms window, which clipped
> the **tail of a sweep at price reversals** (where the burst is followed by a lull) — e.g. ATAS 191 vs app 187.
> Passing `tu` through removes the guess.

---

## 3. Settings (gear ⚙ on the Big-trade toggle)

| Setting | Default | Meaning | ATAS equivalent |
|---|---|---|---|
| Size tối thiểu (`min`) | 50 | Minimum cumulative volume to flag | `MinVol` |
| Size tối đa (`max`, 0=∞) | 0 | Upper cap (exclusive); 0 = no cap | `MaxVol` |
| Gộp sweep (`sweep`) | **on** | Cumulative mode on/off (group by matching event `tu`). **On by default** — Rithmic is per-fill, so non-sweep almost never fires (a real "big trade" is a sweep of many small fills). | `CalculationMode` (Cumulative vs Separate) |
| Cửa sổ gộp (`mergeMs`) | 0 | `0` = group by matching event (precise). `>0` = ATAS-style **cumulative window**: merge all same-side prints within N ms → fewer, larger bubbles. Dial to match ATAS. | (approximates ATAS's looser cumulative merge) |
| Màu bóng (`buyCol`/`sellCol`) | teal / red | Bubble colours (buy / sell) — set distinct from the candle colours for legibility | — (app extra) |
| Bong bóng (`bubble`) | on | Bubble/range at price; off = ▲/▼ arrows | `VisualType` (Ellipse/Rectangle) |
| Size bóng (`size`) | 10 | Size scale factor | `Size` |
| min / max px | 5 / 50 | Bubble size clamp | `MinSize` / `MaxSize` |
| Cố định cỡ (`fixed`) | off | All bubbles same size, ignore volume | `FixedSizes` |
| Bóng tròn hết (`round`) | off | Force circles even for multi-level sweeps (drop the FirstPrice→LastPrice range box) | — (app extra; ATAS always range-boxes a sweep) |
| Vị trí giá (`priceLoc`) | Any | Restrict to a candle location (below) | `PriceLocation` |

**Volume filter** (identical to ATAS): keep a trade when `vol ≥ min` **and** (`max ≤ 0` **or** `vol < max`).

**Size formula** (identical to ATAS): `px = size × vol / min`, then `FixedSizes ⇒ px = size`, then clamp to
`[minPx, maxPx]`.

**PriceLocation** — using the cumulative trade's price span `[lo, hi]` (= min/max of FirstPrice & LastPrice) and
the candle body `[bodyLo, bodyHi]` (= min/max of Open & Close):

| Mode | Shows when |
|---|---|
| Bất kỳ / Any | always |
| Đỉnh nến / AtHigh | `hi == candle.High` |
| Đáy nến / AtLow | `lo == candle.Low` |
| Đỉnh hoặc đáy / AtHighOrLow | `hi == High` **or** `lo == Low` |
| Trong thân / Body | `hi ≤ bodyHi` **and** `lo ≥ bodyLo` |
| Râu trên / UpperWick | `lo > bodyHi` **or** `hi ≥ bodyHi` |
| Râu dưới / LowerWick | `hi < bodyLo` **or** `lo ≤ bodyLo` |

---

## 4. Algorithm (cumulative)

1. Walk trades in time order with a current group `{side, vol, tu, o, firstPrice, lastPrice, tLast, tArr}`.
2. **Same group** if the print's `tu` equals the group's `tu` (same matching event); else same `o` (same aggressor
   order, fallback); else (ms only) same side and within `sweepMs` of `tLast`. Match → `vol += q`, `lastPrice = p`.
   Otherwise **flush** the group and start a new one.
3. On flush, emit if it passes the volume filter (and, at render, the PriceLocation filter).
4. Render: if `firstPrice == lastPrice` → a **bubble** at that price; else a **range box** spanning
   `[min, max]` of first/last price — the ATAS multi-level sweep look. Label = total volume.
5. Live: the next print of a different event flushes the previous group; a timer flushes the final group after a
   `sweepMs` gap measured from **arrival time** (`tArr`, wall-clock) — not the exchange timestamp — so feed
   latency can't close a group early and clip a sweep's tail at a reversal.

Same function runs over the `aggTrades` history array and the live `@trade` stream, so big trades are reviewable
in history and update live seamlessly.

---

## 5. ATAS parity matrix

| ATAS capability | Status | Notes |
|---|---|---|
| CumulativeTrades grouping (by matching event) | ✅ | grouped on `tu` (exchange match µs, = ATAS `.Time` key); `o` then ms-window as fallbacks |
| SeparateTrades mode | ✅ | `sweep` off → threshold each print |
| MinVol / MaxVol filter | ✅ | identical predicate |
| Size = `vol×Size/MinVol`, clamp Min/Max | ✅ | identical formula |
| FixedSizes | ✅ | |
| FirstPrice→LastPrice sweep, drawn as a range | ✅ | range box for multi-level; bubble for single |
| PriceLocation (7 modes) | ✅ | identical logic |
| Buy/Sell/Between colours, volume label | ✅ | buy green · sell red |
| History back-fill + live update | ✅ | replay/store seed, then `@trade` |
| Time-of-day filter (TimeFrom/TimeTo) | ⬜ | roadmap |
| Alerts (sound + popup) | ⬜ | roadmap |
| AutoFilter (auto threshold) | ❌→🟡 | ATAS computes it **server-side** (requests cumulative trades with min/max = −1; their engine returns a curated "big" set). Not reproducible 1:1 from the public feed. Planned **approximation**: rolling percentile (flag above the Nth percentile of the last N cumulative trades). Will be labelled as an estimate. |
| VisualType shapes, transparency trackbars | ⬜ | cosmetic; bubble/range + arrows cover the common cases |
| NewBid/NewAsk volumes in tooltip | ⬜ | needs per-trade book context |

---

## 6. Roadmap

1. **Time filter** — `TimeFrom`/`TimeTo` (instrument-timezone aware), like ATAS. *Easy.*
2. **Alerts** — sound + on-chart popup when a flagged trade fires. *Medium.*
3. **AutoFilter (approx)** — rolling-percentile auto threshold; disables manual `min`. *Medium.* (See ❌→🟡 above
   for why it won't be bit-identical to ATAS.)
4. **VisualType / transparency** — extra shapes and opacity controls. *Cosmetic.*

---

## 7. Changelog

- **2026-06-18** — **Optional cumulative-window merge (`merge` ms) + ATAS-tuning.** Re-introduced a time-window
  merge as an **opt-in** `Cửa sổ gộp` slider (and a `merge` param on the `bigTrades` request, both client live and
  server historical) — `0` keeps the precise matching-event default, `>0` groups all same-side prints within N ms
  for ATAS-style larger bubbles. Server `merge>0` bypasses the `.bigcache`. **Sweep is now ON by default** (Rithmic
  is per-fill, so non-sweep practically never fires). Added **bubble colour pickers** (buy/sell, distinct from
  candles). Storage moved to **v2** — `tu` is now **in-record** (the `.fxu` sidecar is gone; `.fxt`→`.tick`,
  `.fxb`→`.bigcache`), history still groups by matching event exactly like live.
- **2026-06-15** — **Server-side `bigTrades` + full-session bubbles.** New gateway request groups raw trades by
  matching event server-side and returns only the bubbles, so the client seeds big trades for the **whole loaded
  session** (capped ~24 h) instead of the footprint window — without pulling millions of ticks. Render no longer
  caps at 150 bubbles (draws all in the visible range); the in-memory marker buffer holds up to 6000. Bubble size
  now scales with bar width (zoom), diameter bounded so it doesn't blanket neighbouring candles.
- **2026-06-15** — **Tick store now persists `tu`** in a parallel `.fxu` sidecar (8 bytes/record, lock-step with
  `.fxt`). Big-trade (and footprint) **history from the store groups by matching event** like live — not just the
  current session. Old `.fxt`-only days stay ms-window. The `.fxt` format is unchanged (backward compatible).

- **2026-06-15** — **Matching-event grouping (true ATAS parity).** Gateway forwards the exchange match time at
  **microsecond** resolution (`@trade.tu`, `aggTrades` 6th element); the client groups cumulative trades by
  identical `tu` — what ATAS's `CumulativeTrade` keys on (its `.Time`). Rithmic also supplies `AggressorExchOrdId`
  (`o`), but grouping by `o` was measured to drift *higher* than ATAS — one order can fill across several match
  events — so `o` is only a fallback when `tu` is missing (~50 ms window is last resort). Also fixed the live
  group-flush timer to use **arrival time**, not the exchange timestamp, so feed latency no longer closes a sweep
  early and clips its tail at reversals (was: ATAS 191 vs app 187, 173 vs 172).
- **2026-06-15** — Removed the `sweepMs` (merge-window ms) field from the settings UI to match ATAS: merging is by
  matching event, with a fixed internal ~50 ms fallback only when sub-ms time is unavailable. Settings expose merge on/off only.
- **2026-06-15** — Cumulative grouping by `AggressorExchOrdId`; FirstPrice→LastPrice sweep range; PriceLocation
  filter; FixedSizes; size/colour defaults aligned to ATAS. Verified live on CME ESU6.
- Earlier — Initial big-trade markers (time-window sweep) + volume bubbles with pop animation.

# Fxaurum Rithmic — Sweeps · Stops · Icebergs (MBO order-flow trackers)

Three order-flow indicators in the **Volume & OrderFlow** tab, ported from the **ATAS MBO Indicators** pack
(*SweepsTracker / StopsTracker / IcebergsTracker*). Each tracker is mapped to its ATAS behaviour (see the `ATAS` column in the setting tables below).

> Companion docs: **API.md** (`@trade`, `@mboraw`, `flowRecs`) · **MBO.md** (depth-by-order) · **BIGTRADES.md**.

---

## 1. What they are

All three read the same game — **aggressive orders meeting passive liquidity** — from three angles. Buy = green,
sell = red (Stops use cyan/pink to stay distinct from Sweeps).

| Tracker | Nature | Buy = | Sell = | Who (usually) |
|---|---|---|---|---|
| **① Sweeps** | **aggressor** (urgent) | BUY lifts offers across levels → up | SELL hits bids across levels → down | institution / HFT |
| **② Stops** | **triggered** (forced) | SHORT stops fire (buy-to-cover) → up (squeeze) | LONG stops fire (sell) → down (liquidation) | retail (being hunted) |
| **③ Icebergs** | **passive** (absorbing) | hidden BUY at bid → floor/support | hidden SELL at ask → ceiling/resistance | almost always institutional |

**The lens:** every fill has an **aggressor** (taker — sends a market order, pays the spread to fill *now*, *takes*
liquidity) and a **passive** side (maker — rests a limit, waits to be hit, *provides* liquidity). Sweep = the
aggressor. Iceberg = the passive side. Stop = an order forcibly triggered. Hold that and all three read cleanly.

---

## 2. Which sockets to call

Sweeps & Stops need only aggressor-tagged trades. Icebergs additionally require **MBO (depth-by-order)** because
the hidden quantity only shows in the per-order book. History comes from the gateway's **own recording** (stream-order
attribution), never by pulling millions of ticks to the client.

| Goal | Call | For | Why |
|---|---|---|---|
| Live fills + aggressor | `@trade` (stream) | ①②③ | Each print carries `side`, `o` (aggressor id), `tu` (µs match time). Feeds the MatchEngine that separates the aggressor from the passive resting orders. |
| Per-order MBO (hidden qty) | `@mboraw` (stream, DBO) | ③ | Per-order New/Change/Delete around the BBO → how the visible size changes, so we can infer the **invisible** filled quantity. |
| History (authoritative) | `flowRecs` (request) | ①②③ | Records the gateway wrote live: sweeps + cascades + icebergs (keyed by `oid`) + coverage. Client `mergeSeed`s to fill spans while the chart was closed. |
| Approx history (optional) | `flowHistory` / `bigTrades` | ①② | Where the gateway hadn't recorded: reconstructs clusters by max-order-id. Less exact than `flowRecs`, so it fills only **outside** coverage. |
| Candle backdrop (optional) | `klines` + `@kline_<iv>` | ①②③ | Maps marks onto the chart by **time**. Not required to compute the trackers. |

> **Icebergs are LIVE-only for fresh detection.** MBO can't be replayed (Rithmic gives no depth-by-order history),
> so the hidden part is only computable while the app is open. Iceberg history = whatever the gateway already
> recorded (`flowRecs.ice`) — the more the app runs, the fuller the store. Missing spans can be imported from ATAS
> IndicatorData jsonl (`import-atas-flow`).

---

## 3. ① Sweeps Tracker

A **large aggressive order sweeping across price levels** in one burst. The engine (`buildRuns` in
`orderflow.ts`) groups nearby same-side records into a **run** — window `winMs` measured **from the run's start**
(exactly ATAS's "Maximum aggregation window from the beginning of the sweep"; `long_0` doesn't slide). Buy and
Sell are **two independent flows** (`bool_0[2]`) — a buy run never cuts a sell run. Render: a highlight box over
`[minP,maxP] × [t0,t1]` + a pill of `Σ total lots`.

**Filters** (identical to ATAS): keep a run when `vol ≥ MinVolume`, `MinCount ≤ count ≤ (MaxCount or ∞)`, and
`(maxP−minP)/tick + 1 ≥ MinPriceRange`.

| Setting | Default | Meaning | ATAS |
|---|---|---|---|
| Aggregation window (`ofWinMs`) | 100 ms | max gap between fills in a run | `long_0` |
| Min volume (`ofMinVol`) | 100 | run total lots | `MinVolume` |
| Min/Max count (`ofMinCnt`/`ofMaxCnt`) | 1 / 0 | aggressor count (0 = no cap) | `Min/MaxTradesCount` |
| Min range ticks (`ofRange`) | 2 | price span to count as a sweep | `MinPriceRange` |
| Include Stops (`ofStops`) | off | add stop volume of the same matching event | `IncludeStops` |
| Style (`ofStyle`) | filled | filled / outline / dots / dashfill / dashline / none | `VisualMode` |
| Label Min · Cluster-by-X | 5 · 10px | zoom-out merges nearby pills into one Σ pill | `method_54` |
| Alert (`ofAlert`) | off | beeps when a **new** run passes the filter (live tail only) | `UseAlerts` |

---

## 4. ② Stops Tracker (stop cascade)

A **chain of triggered stop orders**: price hits a cluster of stops → they turn into market orders → cascade. The
engine catches the **triggering** aggressor + the same-side stops that fire (the matching event's stop field).
Unlike Sweeps, Stops aggregate as a **single flow** (`singleFlow`, ATAS *method_36* / `Struct18` scalar): an
**opposite-side** record **cuts** the current run — because a stop-run depends entirely on what triggered it.

**Counter-intuitive direction:**

- **Buy stops** fire as price **rises** = stop-losses of **shorts** (buy to exit) → adds upward fuel (short squeeze).
- **Sell stops** fire as price **falls** = stop-losses of **longs** (sell to exit) → adds downward fuel (long liquidation).

Settings share the Sweeps frame (`ofWinMs`/`ofMinVol`/`ofRange`/`ofMinCnt`/`ofMaxCnt`), but `cnt` = the cascade's
**stop count** ("Min/Max Stops Count"), and the card adds a **TRIGGER** field (the igniting order + price).

---

## 5. ③ Icebergs Tracker (hidden orders)

A **large limit order showing only a small "tip"**; as the tip fills it refills at the same price → hiding size so
the market doesn't move. Detected via MBO (`IcebergEngine`): at a match event,

```
hidden = totalTrades − (visBefore + volDelta − visAfter)
```

`hidden > 0` means an **invisible** fill happened. Confidence:

- **absolute** — a **native refresh** observed: a New order replenishing at that price within 50 ms of the hidden
  evidence, or a Delete at the same ms. The refill is directly seen.
- **medium** — only `hidden > 0`, no confirming refresh yet.

Each `IceRec` carries `hidden`, `filled` (passive fills), `aggFilled` (fills where it was itself the aggressor),
`refreshCnt`, `detCnt`, `maxVis` (peak visible tip), `conf`, and `multiPrice`. If the order **walks across prices**
it draws a **staircase line** (`segs`) and the card gets a per-event **Timeline** (New/Change/Delete/Trade, ordered
trade-before-change at the same ms; "same-price refresh" flagged). Hover a pill/line → that iceberg lights up and
the rest dims; a horizontal line pushed off the price axis still draws a vertical tick to the edge (hoverable).

| Setting | Default | Meaning |
|---|---|---|
| Min hidden (`icMinHidden`) | 1 | hidden lots threshold |
| Min total (`icMinTotal`) | 0 | total (filled+hidden) threshold |
| Min duration (`icMinDur`) | 0 | `lastMs−firstMs` (ms) — filters too-brief icebergs |
| Show trades (`icShowTrades`) | off | dots each fill on the iceberg line |
| Window Close Timeout | 100 ms | quiet gap before closing an open iceberg (= ATAS `WindowCloseTimeoutMs`, a `[Browsable(false)]` field; app `tickQuiet`/`icQuiet`) |

> **Why hidden drifts vs ATAS.** `filled`/`refreshCnt`/`detCnt` match ATAS exactly; only the absolute **hidden**
> count can drift (e.g. 374 vs 270). The app reconstructs event order from **two feeds** (`@trade` + `@mboraw`)
> sorted by µs then rank (New/Image `0` < trade/aggr `1` < Change/Delete `2`), whereas ATAS receives one
> pre-interleaved stream. When two events share a microsecond the tie-break can differ. Being tuned with live MBO
> capture (`__iceCap` → `IceRecorder.SetCapture` → replay harness).

---

## 6. Activity Accumulation sub-pane

All three trackers have a sub-pane charting accumulated activity over time (buy line, sell line, Delta), matching
ATAS's `ActivityAccumulationCalculator`:

| Mode | How it accumulates |
|---|---|
| `sum` | runs forever (cumulative total) |
| `half` | half-life decay `× 0.5^(dt_s/tSec)` = ATAS `exp(−ln2/tSec · dt)` |
| `reset` | resets to 0 each period |
| `win` | sliding window (last N only) |
| `side` | separate buy vs sell accumulation |

> **Deliberate divergence:** the app **filters the sub-pane by the on-chart display filter** (hidden on the chart
> → redrawn in the sub-pane), via `keepInRuns(buildRuns(...))`. ATAS does not re-filter (its `method_31` isn't used
> for activity). User's call: "ATAS not redrawing is ATAS's bug."

---

## 7. Trading meaning (educational — not advice)

**Who is retail, who is institutional?** The tape records no identity; this is behavioural inference:

- **Large sweeps** = size + speed, paying up to fill *now* → footprints of institutions / funds / HFT / prop.
- **Icebergs** = a size-hiding execution tool, almost always institutional; retail rarely uses true icebergs.
- **Stops** themselves are usually retail/small (SLs at obvious levels: round numbers, prior highs/lows) — but the
  *hunt* for them is often engineered by larger players ("stop hunt / liquidity grab").

**The real power = confluence** (one signal is noise):

| Confluence | Reads as |
|---|---|
| Sweep hitting an **opposite** iceberg | aggression meets absorption — e.g. relentless sell sweeps but price won't drop because a buy iceberg absorbs → **likely reversal up** (the "golden" setup) |
| Sweep + **same-side** stop cascade, no iceberg wall | genuine momentum → powered breakout, follow the trend |
| Iceberg holds / breaks | holds = strong S/R; breaks = level gone → continuation |
| Stop hunt (spike through a level then retract) | liquidity trap → fade it |

**How to apply:** (1) location first, signal second — read order-flow at meaningful zones (VWAP, value area, key
levels). (2) Sweep = who's urgent, Iceberg = who's defending, Stop = who's about to be flushed — weave the three
into one read. (3) Absorption (sweep eaten by iceberg) → lean reversal; momentum (same-side sweep+stop, no wall) →
lean with trend.

> **Risk:** these are probabilities, not certainties. Spoofing exists (fake icebergs/stops to bait), traps exist,
> and "retail vs institutional" is inference. No signal is a guarantee. This is educational market-microstructure
> description, **not personalized investment advice**.

---

## Changelog

- **2026-07-18** — First cut. Documents Sweeps/Stops/Icebergs trackers, the aggressor/passive/forced lens, sockets
  (`@trade` + `@mboraw` + `flowRecs`), per-tracker settings, the Activity Accumulation sub-pane (5 modes), the
  trading-interpretation section. Notes the known iceberg-`hidden` drift (2-stream µs
  tie-break) pending live-capture tuning.

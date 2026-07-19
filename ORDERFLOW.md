# Fxaurum Rithmic — Order-flow trackers (Sweeps · Stops · Icebergs · Absorption · Liquidity)

**Five** order-flow indicators in the **Volume & OrderFlow** tab. The first three are ported from the **ATAS MBO
Indicators** pack (*SweepsTracker / StopsTracker / IcebergsTracker*); **Absorption** and **Liquidity** are new,
modelled on the **Bookmap** absorption / SIT / liquidity engines. Each tracker maps to a shared render + engine layer
(`orderflow.ts` on the client, gateway recorders for history).

> Companion docs: **API.md** (`@trade`, `@mboraw`, `flowRecs`) · **MBO.md** (depth-by-order) · **BIGTRADES.md** · **HEATMAP.md**.

---

## 1. What they are

All five read the same game — **aggressive orders meeting passive liquidity** — from five angles. Buy = green,
sell = red (Stops use cyan/pink, Liquidity amber/grey, to stay distinct).

| Tracker | Nature | Reads | Buy / Support = | Sell / Resistance = | Feed |
|---|---|---|---|---|---|
| **① Sweeps** | **aggressor** (urgent) | breaks through | BUY lifts offers across levels → up | SELL hits bids across levels → down | `@trade` |
| **② Stops** | **triggered** (forced) | cascade fires | SHORT stops fire (buy-cover) → squeeze up | LONG stops fire (sell) → liquidation down | `@trade` |
| **③ Icebergs** | **passive HIDDEN** (absorbing) | hidden refill | hidden BUY at bid → floor | hidden SELL at ask → ceiling | `@mboraw` |
| **④ Absorption** | **outcome** (held) | aggression eaten at 1 price | buyers absorb (bid holds) → support | sellers absorb (ask holds) → resistance | `@trade` |
| **⑤ Liquidity** | **passive VISIBLE** (wall) | resting wall + fate | bid wall EXECUTED = strong / PULLED = spoof | ask wall EXECUTED / PULLED | `@mboraw` |

**The lens:** every fill has an **aggressor** (taker — market order, pays the spread to fill *now*, *takes*
liquidity) and a **passive** side (maker — rests a limit, waits to be hit, *provides* liquidity).
Sweep = the aggressor. Iceberg / Liquidity = the passive side (hidden vs visible). Stop = an order forcibly
triggered. Absorption = the **result** when aggression fails to break a price (mechanism-agnostic).

**Sweep vs Absorption (opposite outcomes of the same force):** aggression into one price either **breaks through**
(spreads across N levels → **Sweep**) or is **absorbed** (piles at one price, price holds → **Absorption**).
**Iceberg** and **Liquidity** are the two *mechanisms* (hidden refill vs visible wall) that can cause absorption —
so an actively-hit iceberg usually also fires Absorption, but Absorption fires even with no MBO (cheaper, broader).

---

## 2. Which sockets to call

Sweeps / Stops / Absorption need only aggressor-tagged trades (`@trade`). Icebergs / Liquidity additionally require
**MBO (depth-by-order)** because the hidden/visible resting size only shows in the per-order book. History comes from
the gateway's **own recording** (stream-order attribution), never by pulling millions of ticks to the client.

| Goal | Call | For | Why |
|---|---|---|---|
| **Live tracker records (recommended)** | `@sweeps` · `@stops` · `@iceberg` · `@absorption` · `@liquidity` (stream) | ①②③④⑤ | The gateway runs the engine and **pushes each finalised record live** — just subscribe (like `@bigTrade`). One stream feeds any number of charts / indicator instances (each filters + groups client-side). This is how the platform consumes all five. |
| Raw fills + aggressor (engine feed) | `@trade` (stream) | ①②③④⑤ | The raw trades the gateway's engine is built from. Each print carries `side`, `o` (aggressor id), `po` (passive id), `tu` (µs match time). Feeds MatchEngine + Absorption + credits Iceberg/Liquidity executed. |
| Per-order MBO | `@mboraw` (stream, DBO) | ③⑤ | Per-order New/Change/Delete around the BBO → hidden fill (iceberg) and visible resting size + fate (liquidity). |
| History (authoritative) | `flowRecs` (request) | ①②③④⑤ | Records the gateway wrote live: `sw` (sweeps) · `cs` (cascades) · `ice` · **`ab` (absorption)** · **`lq` (liquidity)** + per-array coverage. Client seeds to fill spans while the chart was closed. |
| Approx history (optional) | `flowHistory` / `bigTrades` / `aggTrades` | ①②④ | Where the gateway hadn't recorded: reconstructs from tick store. Absorption also self-backfills from `aggTrades` for closed candles before chart-open. |
| Candle backdrop (optional) | `klines` + `@kline_<iv>` | all | Maps marks onto the chart by **time**. Not required to compute the trackers. |

> **MBO is LIVE-only.** Rithmic gives no depth-by-order history, so Icebergs (hidden) and Liquidity (walls) can only
> be *computed* while MBO runs. Their history = whatever the gateway already recorded (`flowRecs.ice` / `.lq`) — the
> more the app runs with MBO on, the fuller the store. Sweeps / Stops / **Absorption** are `@trade`-based, so their
> history is robust: the gateway records them continuously (like sweeps), and Absorption additionally back-fills from
> replayable trades (`aggTrades`, ~90 days) for closed candles.

---

## 3. ① Sweeps Tracker

A **large aggressive order sweeping across price levels** in one burst. `buildRuns` (`orderflow.ts`) groups nearby
same-side records into a **run** — window `winMs` measured **from the run's start** (ATAS "Maximum aggregation window
from the beginning of the sweep"). Buy and Sell are **two independent flows**. Render: a highlight box over
`[minP,maxP] × [t0,t1]` + a pill of `Σ total lots`.

**Filters:** keep a run when `vol ≥ MinVolume`, `MinCount ≤ count ≤ (MaxCount or ∞)`, `(maxP−minP)/tick + 1 ≥ MinRange`.

| Setting | Default | Meaning |
|---|---|---|
| Max Window (`ofWinMs`) | 100 ms | max gap between fills in a run |
| Min Volume (`ofMinVol`) | 100 | run total lots |
| Min/Max Count (`ofMinCnt`/`ofMaxCnt`) | 1 / 0 | aggressor count (0 = no cap) |
| Min Range (`ofRange`) | 2 | price span (ticks) to count as a sweep |
| **Min Levels Swept** (`ofMinLevels`) | 0 | keep only sweeps that ate ≥ N **distinct** price levels (Bookmap `TreeSet<price>.size` — 300 lots over 5 levels ≠ 300 at one price) |
| Include Stops (`ofStops`) | off | add stop volume of the same matching event |

Card shows **Mức quét / levels** (distinct prices the sweep hit) when available.

---

## 4. ② Stops Tracker (stop cascade)

A **chain of triggered stop orders**: price hits a cluster of stops → market orders → cascade. The engine catches
the **triggering** aggressor + same-side stops that fire. Stops aggregate as a **single flow** — an **opposite-side**
record **cuts** the current run.

- **Buy stops** fire as price **rises** = short stop-losses (buy to exit) → upward fuel (short squeeze).
- **Sell stops** fire as price **falls** = long stop-losses (sell to exit) → downward fuel (liquidation).

Shares the Sweeps frame; `cnt` = the cascade's **stop count**; card adds a **TRIGGER** field (igniting order + price).

---

## 5. ③ Icebergs Tracker (hidden orders)

A **large limit order showing only a small "tip"**; as the tip fills it refills at the same price. Detected via MBO
(`IcebergEngine`): at a match event, `hidden = totalTrades − (visBefore + volDelta − visAfter)`. `hidden > 0` = an
**invisible** fill. Confidence:

- **absolute** — a **native refresh** (a New/Change replenishing at that price within 50 ms of the hidden evidence,
  or a Delete at the same ms). The refill is directly seen.
- **medium** — only `hidden > 0`, no confirming refresh yet.

Each `IceRec` carries `hidden`, `filled`, `aggFilled`, `refreshCnt`, `detCnt`, `maxVis`, `conf`, `multiPrice`.
Walking icebergs draw a **staircase line** (`segs`) + a per-event **Timeline** card.

| Setting | Default | Meaning |
|---|---|---|
| Min Hidden (`icMinHidden`) | 1 | hidden lots threshold |
| Min Total (`icMinTotal`) | 0 | total (filled+hidden) threshold |
| Min Duration (`icMinDur`) | 0 | `lastMs−firstMs` (ms) |
| Window Close Timeout (`icQuiet`) | 100 ms | quiet gap before closing an open iceberg |

---

## 6. ④ Absorption Tracker (NEW — held at a price)

**The opposite of a sweep.** Aggression **piles at one price and the price holds** → the passive side is absorbing
(a strong S/R zone forms). Computed from `@trade` only (no MBO): `AbsorptionEngine` sums aggressive volume per
`(matching-event, price)`; `buildAbsorption` groups same-price records inside `abWinMs`; a zone fires when
`Σvol ≥ abMinVol`. Mechanism-agnostic — the passive side may be an iceberg (hidden), a visible wall (Liquidity), or
many small limits; Absorption catches all of them, and works even without MBO entitlement.

- **Support** (▲, green) — sellers hit the bid but the **bid absorbs** (buyers hold the floor).
- **Resistance** (▼, red) — buyers lift the ask but the **ask absorbs** (sellers hold the ceiling).

**Price band (multi-level zone).** By default it groups one exact tick; set **`abBandTicks > 0`** to merge nearby
prices (≤ N ticks) into **one zone** — aggression spread over 3 adjacent ticks that each miss the threshold together
form one absorption zone that passes. The mark then draws a **rectangle covering the whole price band**
(`minP..maxP`), with the leader dot attaching to the band edge facing the label.

| Setting | Default | Meaning |
|---|---|---|
| Min Volume (`abMinVol`) | 150 | Σ same-price volume in the window |
| Window (`abWinMs`) | 300 ms | grouping window (same price) |
| **Band ticks** (`abBandTicks`) | 0 | merge prices ≤ N ticks apart into one zone (0 = one exact tick) |

Live from chart-open + **persisted** (reopen restores) + **gateway-recorded** (`AbsorptionRecorder` → `flowRecs.ab`)
+ back-fills from `aggTrades` for pre-open closed candles.

---

## 7. ⑤ Liquidity Tracker (NEW — visible walls + fate)

Tracks **large VISIBLE resting orders (walls)** in the book and their **outcome**. Different from Iceberg (hidden),
Heatmap (draws all liquidity as a gradient) and Absorption (`@trade` phenomenon): this flags **discrete visible
walls** and follows a lifecycle. Fed by `@mboraw` (+ `@trade`): `LiquidityEngine` aggregates resting size per price;
when it crosses `lqMinSize` a wall is tracked; a trade hitting that price credits **executed**; size removed without
a trade is **cancelled**.

**Lifecycle:** **PLACED** (size ≥ threshold) → **HELD** (rests ≥ `lqHoldMs`) → **EXECUTED** (traded into = a real,
strong level) **or PULLED** (removed before filling = **spoof / flip**). `cancelled = peak − current − executed`;
`executed ≥ cancelled` ⇒ *executed*, else *pulled*.

- **EXECUTED** (solid) = a genuine level that got consumed.
- **PULLED** (grey, dashed, ✕) = liquidity that vanished without filling → **spoof**.
- **ACTIVE** = a wall still resting (medium).

| Setting | Default | Meaning |
|---|---|---|
| Min Size (`lqMinSize`) | 200 | resting size at a price to flag a wall |
| Min Hold (`lqHoldMs`) | 1500 ms | rest time to count as HELD |
| Show EXECUTED / PULLED (`lqShowExec`/`lqShowPulled`) | on / on | filter by outcome |

The client engine is an approximation (`cancelled` = residual). The **exact** version uses the exchange
**passive-order id** at the gateway (touched ⇒ executed, untouched-leave ⇒ cancelled) — in progress. Persisted +
gateway-recorded (`LiquidityRecorder` → `flowRecs.lq`); MBO being live-only, history covers spans the gateway ran MBO.

---

## 8. Advanced options ("Nâng cao")

Each tracker's settings has a **Nâng cao** group. Turning it **off** keeps the classic (ATAS/Bookmap-parity)
behaviour; turning it **on** enables cross-tracker upgrades (opt-in, so defaults never change):

| Upgrade | Trackers | What it does |
|---|---|---|
| **SD auto-threshold** (`ofAutoThr` + `ofSdMult`) | all | threshold = `mean + N·SD` of recent record volumes instead of a fixed Min Volume — auto-scales with volatility (Bookmap default `N=10`). |
| **Adaptive window** (`ofAdaptWin`) | sweeps / stops | window = `3 × median tick gap` (clamped 10–1000 ms) instead of a fixed 100 ms — tight in fast markets, wide in slow. |
| **TBPL** (`ofTBPL`) | stops | keep only cascades that **Took/Broke a Prior Level** (swept through the recent high/low). |
| **Velocity** (`ofVel`/`ofVelMin`) | sweeps | lots/second of the run; filter "urgent" sweeps + show on card. |
| **Confidence** (`ofStopConf`) | stops | 0–100 score (walk-the-book: more stops over more ticks = higher). |
| **Absolute-only** (`icAbsOnly`) | icebergs | show only `conf = absolute` (drop `medium`). |

Sweeps/Stops v2 additionally **receive records via the gateway stream** (`@sweeps`/`@stops`/`@iceberg`), and
Icebergs v2 use the gateway's single-stream authoritative order (fixes the µs-tie-break `hidden` drift).

**Live streams (all five).** Every tracker has a server-computed **live stream** — `@sweeps` · `@stops` · `@iceberg`
· `@absorption` · `@liquidity` — pushing each record as the gateway finalises it (row = the matching `flowRecs`
field: `sw/cs/ice/ab/lq`), so thin clients / EAs can consume without running the engine (like `@bigTrade`).
`@absorption` needs only the trade feed; `@liquidity` turns MBO on itself (like `@iceberg`). The platform **subscribes
to all five** and prefers the gateway (server-computed) records when present — falling back to the local client engine
if the stream/history is absent. For Liquidity the client engine still supplies the **live ACTIVE walls** (the stream
only sends walls once they finalise), overlaid on the gateway's finalised records.

---

## 9. Rendering & interaction

- **Pill labels** — every tracker labels its marks with a dark rounded **pill** (coloured border by side, white
  number, a per-tracker icon: sweep dots, stop bars, iceberg glyph, absorption wall-and-arrow, liquidity bricks),
  offset off the candle with a dashed **leader line + dot** to the price level (or band edge).
- **Cluster Σ×N** — zoom out and nearby marks of the same side merge into one `Σvol ×N` pill (per-side; Liquidity
  keeps **real vs spoof** clusters separate). Hover a pill/cluster → its marks light up, the rest dim.
- **Highlight Style / Size / Thickness** — filled / outline / dashline… + bar thickness, on every tracker.
- **Multiple instances** — each tracker can be added **many times** (like Volume Profile): one engine, several
  filtered/coloured views (e.g. two Absorptions, one big-zone + one fine). Remove each via the legend ✕.

---

## 10. Activity Accumulation sub-pane

Every tracker has a sub-pane charting accumulated activity over time (two lines + Delta):

| Tracker | Line A | Line B | Delta |
|---|---|---|---|
| Sweeps / Stops | Buy | Sell | Buy − Sell |
| Icebergs | Bid | Ask | Bid − Ask |
| **Absorption** | Support | Resistance | Supp − Res |
| **Liquidity** | **Real** (executed+active) | **Spoof** (pulled) | Real − Spoof |

Modes: `sum` (cumulative) · `half` (half-life decay) · `reset` (per period) · `win` (sliding) · `side`. The sub-pane
is filtered by the same on-chart display filter.

---

## 11. Trading meaning (educational — not advice)

**Who is who?** The tape records no identity; this is behavioural inference: large **sweeps** = institutions/HFT
paying up to fill *now*; **icebergs** & big **walls** = size-hiding / size-showing institutional execution; **stops**
are usually retail SLs at obvious levels — but the *hunt* is engineered by larger players.

**The real power = confluence:**

| Confluence | Reads as |
|---|---|
| Sweep hitting an **opposite** iceberg / absorption | aggression meets absorption — relentless sell sweeps but price won't drop because a buy iceberg/absorption holds → **likely reversal up** (the "golden" setup) |
| **Absorption** zone that then **holds** | strong S/R; combine with Liquidity (is the wall real/executed?) and Sweeps (did it later break?) |
| **Liquidity EXECUTED** at a level | genuine defended level; **PULLED (spoof)** before price arrives | liquidity trap → don't trust the wall |
| Sweep + **same-side** stop cascade, no wall | genuine momentum → powered breakout |
| Stop hunt (spike through a level then retract) | liquidity grab → fade it |

**How to apply:** location first, signal second (read at VWAP / value area / key levels). Sweep = who's urgent,
Iceberg/Liquidity = who's defending (hidden vs visible), Absorption = did the defence hold, Stop = who's about to be
flushed. Absorption + no iceberg + no wall ⇒ many small limits, or a feed without MBO.

> **Risk:** probabilities, not certainties. Spoofing, traps, and "retail vs institutional" inference all exist. This
> is educational market-microstructure description, **not personalized investment advice**.

---

## Changelog

- **2026-07-19** — Expanded to **five** trackers: added **④ Absorption** (`@trade`, price-band zones, `flowRecs.ab`
  + `aggTrades` backfill) and **⑤ Liquidity** (`@mboraw` walls, EXECUTED/PULLED lifecycle, `flowRecs.lq`). New
  **Advanced ("Nâng cao")** group across trackers: SD auto-threshold, adaptive window, TBPL, sweep levels-swept,
  velocity/confidence/absolute-only. Rendering overhaul: off-candle **pill labels** with icons + leader lines,
  **cluster Σ×N**, hover-dim, Highlight Style/Size/Thickness, **multiple instances per chart**. History hardened via
  gateway `AbsorptionRecorder` / `LiquidityRecorder` (+ `IceRecorder` / `FlowRecorder`); **live streams** `@absorption`
/ `@liquidity` added so all five trackers stream server-computed. Persistence for all trackers.
- **2026-07-18** — First cut. Sweeps/Stops/Icebergs, the aggressor/passive/forced lens, sockets
  (`@trade` + `@mboraw` + `flowRecs`), per-tracker settings, the Activity Accumulation sub-pane, trading section.

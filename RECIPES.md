# Fxaurum Rithmic — Indicator & Order-Flow Recipes

A **cookbook**: for each popular futures indicator, *which* API streams/requests to use and the *algorithm*
to compute it. Each recipe is self-contained and language-agnostic — build it from the WebSocket API in JS,
Python, C#, anything. The goal is to give end users a reference to build indicators and, from there, trading
strategies/bots.

> Companion docs: **API.md** (full message schemas) · **FOOTPRINT.md** (footprint in depth) ·
> **BIGTRADES.md** (big trades — ATAS parity tracker).

This is a living document — recipes are added over time. Status legend: ✅ buildable from streams shipping
today · 🛠 needs a stream still on the roadmap.

---

## Data primitives (what every recipe builds on)

| Source | Kind | Gives you | Recipes that use it |
|--------|------|-----------|---------------------|
| `@trade` | stream | live trade prints `{p,q,side,T,tu,o,r}` (`o` = aggressor order id, supplied by Rithmic; `tu` = exchange match time in µs) | Big trades, CVD, VWAP, Volume Profile, Footprint |
| `aggTrades` | request | historical trades `[t,p,q,side,o,tu]` (Rithmic replay → local store) | seeding any trade-based indicator |
| `@bookTicker` | stream | best bid/ask `{b,B,a,A}` | spread, microprice, tick-rule side |
| `@depth` / `@depthN` | stream | order book `{bids[],asks[]}` | DOM, Heatmap, book imbalance |
| `@kline_<iv>` / `klines` | stream/request | OHLCV candles | bar overlay, time axis |

**Side convention** everywhere: `"B"` = buy aggressor (lifted ask), `"S"` = sell aggressor (hit bid), `""` =
unknown → apply the *tick rule* (price ≥ last trade ⇒ buy, else sell; or compare to best bid/ask).

**Session reset.** Most cumulative indicators (CVD, VWAP, Volume Profile) reset at the **session open**. Pick a
session boundary (e.g. CME equity index RTH open, or the exchange daily reset) and zero the accumulators there.

---

## 1. Big trades / block detection  ✅

Flag unusually large prints — institutional size, sweeps, icebergs surfacing. (Modelled on ATAS *Big trades*.)

**Data:** `@trade` (live) + `aggTrades` (history to back-fill markers).

**Two modes**
- **Cumulative** (default): merge prints that belong to the **same matching event** into one logical trade,
  then threshold the **summed** size. A single aggressive order that swept several price levels shows as ONE
  marker, not a cluster of small prints. This is how ATAS *CumulativeTrades* works.
- **Separate**: threshold each individual print.

**Grouping — use the matching-event timestamp (`tu`).** A matching event (one aggressing sweep) stamps every
print with the same exchange transact time; the gateway forwards it as `tu` (microseconds). Group the run of
consecutive prints with an **identical `tu`** into one cumulative trade — that's exactly what ATAS's
`CumulativeTrade` keys on (its `.Time`). Rithmic *also* supplies `o` (`AggressorExchOrdId`, the aggressor order
id), but **don't** make it the primary key: one order can fill across several match events (a resting/iceberg
order eaten over time), and grouping by `o` merges those into one oversized trade that ATAS shows separately
(measured to drift higher than ATAS). Use `o` only as a fallback when `tu` is absent, and a **time window**
(same side within `sweepMs`, e.g. 50 ms) when only millisecond `t` survives (old local-store replay). Order of
preference: `tu` → `o` → time window.

**Algorithm (cumulative)**
1. Pick `min` size (and optional `max`, 0 = no cap).
2. Walk trades in time order with a current group `{side, vol, o, tu, tLast, tArr}`.
3. **Same group** if the print's `tu` equals the group's `tu` (same matching event); else equal `o` (same
   aggressor order, fallback); else (ms only) same side **and** within `sweepMs` of `tLast`. Match → add;
   otherwise **flush** (emit if `min ≤ vol` and `max ≤ 0 || vol < max`) and start a new group.
4. Flush the final group at end of list (history) or after a gap > `sweepMs` (live: a timer; the next print of a
   different event flushes the previous group anyway). Measure the live gap from **arrival time**, not the
   exchange timestamp, so feed latency can't close a group early and clip a sweep's tail at a reversal.
5. Render markers (▲ below bar for buy, ▼ above for sell) or bubbles sized by volume; label = group size.

Apply the **same function** to the `aggTrades` history array and to the live `@trade` stream → big trades are
reviewable in **history** *and* update **live**.

```js
function buildBigTrades(trades, {min, max=0, sweepMs=50, cumulative=true}){  // trades: [{T,q,side,tu,o}] sorted by T
  const ok = v => v>=min && (max<=0 || v<max), out=[];
  if(!cumulative){ for(const t of trades) if(ok(t.q)) out.push({T:t.T, q:t.q, side:t.side||""}); return out; }
  let g=null; const flush=()=>{ if(g && ok(g.q)) out.push({T:g.T, q:g.q, side:g.side}); g=null; };
  const same=(t,s)=> g && ((t.tu&&g.tu) ? t.tu===g.tu             // same matching event (µs) — ATAS key (.Time)
                         : (t.o&&g.o) ? t.o===g.o                 // else same aggressor order id (fallback)
                         : (!g.o && s===g.side && (t.T-g.last)<=sweepMs));  // ms-only fallback
  for(const t of trades){ const s=t.side||"";
    if(same(t,s)){ g.q+=t.q; g.last=t.T; }
    else { flush(); g={T:t.T, q:t.q, side:s, tu:t.tu||0, o:t.o||"", last:t.T}; } }
  flush(); return out;
}
```

> **Tuning:** a fixed threshold misfires across contracts/sessions. A **rolling percentile** (flag size above
> the 99th of the last N trades) adapts to liquidity automatically. ATAS layers more filters: **price-location**
> (only at candle high/low/body/wick) and **marker size scaled by volume** are already in the reference chart;
> **time-of-day window** and **alerts** are on the roadmap. The full ATAS-parity breakdown — including the
> FirstPrice→LastPrice sweep range and why AutoFilter can't be matched 1:1 — is tracked in **BIGTRADES.md**.

---

## 2. CVD — Cumulative Volume Delta  ✅

The order-flow staple. Running sum of aggressive buy minus sell volume; divergence vs price signals absorption.

**Data:** `@trade` (the `side` field) + `aggTrades` to seed the session.

**Algorithm**
1. `cvd = 0` at session open.
2. Each trade: `cvd += (side==="B" ? +q : side==="S" ? -q : signByTickRule()*q)`.
3. Plot `cvd` as a line in a sub-pane under price. Compare slope vs price slope.

```js
let cvd = 0, last = null; const series = [];
function onTrade(d){
  let s = d.side;
  if (s!=="B" && s!=="S"){ s = (last===null || d.p>=last) ? "B" : "S"; }  // tick rule
  last = d.p;
  cvd += (s==="B" ? d.q : -d.q);
  series.push({ time: Math.floor(d.T/1000), value: cvd });
}
```

| Read | Meaning |
|------|---------|
| Price up, CVD up | healthy buy initiative |
| Price up, CVD flat/down | **bearish divergence** — buyers lifting but sellers absorbing |
| CVD steep step | aggressive sweep |

> **Per-bar delta** is the same signed sum reset each bar (already shown in the footprint). **CVD** is the
> never-reset cumulative version.

---

## 3. VWAP + standard-deviation bands  ✅

The most-used intraday fair-value line; bands mark stretched conditions.

**Data:** `@trade` (price × volume) + `aggTrades` to seed.

**Algorithm** — keep three running sums from the session open:
```
ΣPV  += price*qty
ΣV   += qty
ΣP2V += price*price*qty
VWAP  = ΣPV / ΣV
var   = ΣP2V/ΣV − VWAP*VWAP          # volume-weighted variance
σ     = sqrt(max(var, 0))
band± = VWAP ± k*σ                    # k = 1, 2, 3 …
```

```js
let pv=0,v=0,p2v=0;
function onTrade(d){ pv += d.p*d.q; v += d.q; p2v += d.p*d.p*d.q; }
function vwap(){ const m = pv/v; const sd = Math.sqrt(Math.max(p2v/v - m*m, 0));
  return { vwap:m, up1:m+sd, dn1:m-sd, up2:m+2*sd, dn2:m-2*sd }; }
```

> Anchored VWAP = same math but start `Σ` from a chosen event (open, a swing high/low, a news bar) instead of
> the session open.

---

## 4. Volume Profile (POC / Value Area)  ✅

Horizontal histogram of volume **by price** — finds high-volume nodes (support/resistance) and the value area.

**Data:** `aggTrades` (session/visible range) + `@trade` (extend live). (Or aggregate an existing footprint.)

**Algorithm**
1. Bucket volume by price: `vol[price] += q` for every trade in range.
2. **POC** = price with the max `vol`.
3. **Value Area (VA, default 70%):** start at POC, repeatedly add the *larger* of the two adjacent rows
   (above / below the current VA span) until the enclosed volume ≥ 70 % of total. The span's edges are **VAH**
   (high) and **VAL** (low).

```js
const vol = new Map();                                   // price -> volume
function onTrade(d){ vol.set(d.p, (vol.get(d.p)||0) + d.q); }
function valueArea(pct=0.70){
  const rows=[...vol.entries()].sort((a,b)=>a[0]-b[0]);  // ascending price
  const total=rows.reduce((s,r)=>s+r[1],0);
  let i=rows.reduce((mi,r,ix,a)=>r[1]>a[mi][1]?ix:mi,0); // POC index
  let lo=i, hi=i, acc=rows[i][1];
  while(acc < total*pct && (lo>0 || hi<rows.length-1)){
    const up   = hi<rows.length-1 ? rows[hi+1][1] : -1;
    const down = lo>0            ? rows[lo-1][1] : -1;
    if(up>=down){ hi++; acc+=up; } else { lo--; acc+=down; }
  }
  return { poc:rows[i][0], vah:rows[hi][0], val:rows[lo][0] };
}
```

> **Profile types:** by session = *Volume Profile*; by visible range = *VPVR*; split by time brackets = *TPO /
> Market Profile*. Same bucketing, different x-grouping.

---

## 5. DOM — order book ladder  ✅

The depth ladder scalpers read: resting bid/ask size at each price, plus imbalance.

**Data:** `@depth` (full book) or `@depthN` (top N).

**Algorithm**
1. Maintain two maps `bids[price]=size`, `asks[price]=size`; replace from each `@depth` snapshot (or apply
   deltas if you subscribe incremental).
2. Render a vertical ladder: price descending, bid size left / ask size right, best bid/ask highlighted.
3. **Book imbalance** at the top: `(bidSize − askSize) / (bidSize + askSize)` over the top K levels → pressure.

```js
let bids=new Map(), asks=new Map();
function onDepth(d){ bids=new Map(d.bids); asks=new Map(d.asks);
  const bb=d.bids[0]?.[1]||0, aa=d.asks[0]?.[1]||0;
  const imb=(bb-aa)/((bb+aa)||1);                 // −1 sell-heavy … +1 buy-heavy
}
```

> DOM size is **resting liquidity** (intent), not executed volume. Pair it with `@trade` to see what actually
> traded *into* the book (absorption / iceberg refills).

---

## 6. Liquidity Heatmap (Bookmap-style)  ✅

Resting liquidity over time — large resting orders glow, showing where price gets attracted/rejected.

**Data:** `@depth` sampled over time.

**Algorithm**
1. On a fixed cadence (e.g. every 250 ms) snapshot the book.
2. For each `[price, size]`, write a cell `(timeColumn, price) = size`.
3. Map `size` → color intensity (log scale works best); scroll columns left over time.
4. Overlay traded prints (`@trade`) on top to see liquidity being consumed vs pulled.

> This is a 2-D buffer (time × price) of resting size. Keep a ring buffer of the last W columns; recolor on each
> new snapshot. Spoofing shows as liquidity that appears then vanishes without trading through it.

---

## Advanced (roadmap) — API entry point + method

These need a stream/endpoint still being wired (🛠), but the method is here so the target is clear.

### 7. MBO DOM — depth-by-order  🛠
**Data:** `@dbo` (depth-by-order; per-order queue, not aggregated size). Requires the account to be entitled to
MBO from the exchange.
**Method:** keep an ordered queue of individual orders at each price; track add / modify / cancel / trade events
to see **queue position**, iceberg refills, and true order count behind a price — far richer than aggregated DOM.

### 8. Options chain  🛠
**Data:** `getOptionChain(exchange, product, expiry)` → instruments per strike; subscribe each for quotes/greeks
and open interest.
**Method:** build a strike × {call, put} grid with bid/ask, IV, OI; foundation for any options analytics.

### 9. Options GEX — Gamma Exposure  🛠
**Data:** option **open interest** per strike (from the chain) + the underlying/future price.
**Method:**
1. For each strike, compute **gamma** (Black-76 for futures options):
   `d1 = (ln(F/K) + 0.5σ²T) / (σ√T)`, `Γ = e^(−rT)·N'(d1) / (F·σ·√T)`.
2. Per-strike exposure: `GEX_K = Γ · OI_K · multiplier · F² · 0.01` (notional gamma per 1 % move).
3. Sum across strikes with a dealer-sign convention (commonly +calls, −puts). The **zero-gamma** flip level and
   large-GEX strikes act as volatility magnets / pins.
> The sign convention (who is long/short gamma) is a modeling choice — state it explicitly in your UI.

---

## General pitfalls (apply to all recipes)

- **Seed from history, then go live.** Replay `aggTrades` once to fill the session so far, then keep applying
  `@trade`. The boundary is seamless — don't double-count an overlapping window.
- **Tick rule for empty side.** Prefer the server `side`; when `""`, classify by price vs last trade (or vs
  best bid/ask from `@bookTicker`).
- **Session boundaries.** Reset cumulative indicators (CVD, VWAP, profile) at the session open, or they drift
  across days.
- **Resting vs traded.** `@depth` = intent (can be pulled/spoofed); `@trade` = done. Order-flow reads come from
  combining both.
- **Off-hours `aggTrades`.** For sessions Rithmic can no longer replay, history is served from the gateway's
  local tick store; if both are empty the market was closed with no recorded ticks — build live once it reopens.

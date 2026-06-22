# Fxaurum Rithmic — Footprint Guide

How to build an **order-flow footprint** (a.k.a. cluster / bid-ask chart) from the WebSocket API. Written for
any developer or AI to implement in any language.

> Companion docs: **API.md** — full WebSocket message schemas · **RECIPES.md** — cookbook of other indicators
> (CVD, VWAP, Volume Profile, DOM, Heatmap, …).

---

## What is a footprint

A footprint breaks each candle down by **price level** and shows, at every level, how much volume traded on
the **buy** (ask) side vs the **sell** (bid) side. A single candle becomes a column of price rows:

```
 sell | price   | buy
 ---- + ------- + ----
   8  | 5501.00 |  31
  42  | 5500.75 |  95   <- POC (most-traded price)
  60  | 5500.50 |  18
  12  | 5500.25 |   4
```

From this you read absorption, imbalances and delta.

---

## Which sockets to call

A footprint is built entirely from **individual trades that carry an aggressor side**. You need exactly two
data sources — one for history, one for live:

| Goal                          | Call                              | Why |
|-------------------------------|-----------------------------------|-----|
| **Footprint for past candles**| `aggTrades` (request)             | Replays historical trades `[time, price, qty, side]` — the raw input for past bars. |
| **Footprint for the live bar**| `@trade` (stream)                 | Each new trade updates the current bar's levels in real time. |
| OHLC overlay (optional)       | `klines` + `@kline_<iv>`          | Only if you also want the candle body/wick. **Not required** for the footprint. |
| Spread context (optional)     | `@bookTicker`                     | Best bid/ask — classify trades when `side` is empty (tick rule). |

> **Minimum viable footprint = `aggTrades` (history) + `@trade` (live).** Everything else is presentation.

---

## The data you need

Both sources give the same trade shape. Normalize to one record:

| Field     | From `aggTrades` | From `@trade` | Meaning |
|-----------|------------------|---------------|---------|
| time (ms) | `row[0]`         | `data.T`      | When the trade printed |
| price     | `row[1]`         | `data.p`      | Trade price |
| qty       | `row[2]`         | `data.q`      | Contracts |
| side      | `row[3]`         | `data.side`   | `"B"` buy aggr · `"S"` sell aggr · `""` unknown |

---

## Algorithm

Footprint is a nested tally: `bucket[barStart][price] = { buy, sell }`.

1. **Pick the interval** and compute its length in ms, e.g. `1m → 60000`.
2. **Find each trade's bar** by flooring its timestamp:
   ```js
   barStart = Math.floor(timeMs / intervalMs) * intervalMs;
   ```
3. **Add the trade to its price level**, split by side:
   ```js
   cell = bucket[barStart][price]            // {buy:0, sell:0}
   if (side === "B") cell.buy  += qty
   else if (side === "S") cell.sell += qty
   else { /* unknown -> tick rule, see Pitfalls */ }
   ```
4. **Seed history, then keep adding live.** Replay `aggTrades` once to fill past bars, then apply every
   `@trade` the same way — the current bar grows tick by tick.
5. **Render** each bar as a column: one row per traded price showing `sell-vol | buy-vol`, colored by volume,
   with POC/delta highlighted.

---

## Statistics (per bar)

| Metric              | Formula                       | Reads as |
|---------------------|-------------------------------|----------|
| **Delta**           | Σ buy − Σ sell                | Net aggressive pressure (positive = buyers led) |
| **POC**             | price with max (buy+sell)     | Where the most business happened |
| **Total volume**    | Σ (buy+sell)                  | Bar activity |
| **Bid/Ask imbalance** | buy[p] vs sell[p−1tick] ≥ 3× | Diagonal dominance → absorption / initiative |

---

## Full example — Python

```python
import asyncio, json, time, collections, websockets

URL="ws://127.0.0.1:9001/"; EXCH="CME"; SYM="ESU6"; IV_MS=60_000   # 1-minute bars
# bucket[barStart][price] = [buy, sell]
book = collections.defaultdict(lambda: collections.defaultdict(lambda: [0, 0]))

def add(t_ms, price, qty, side, last=[None]):
    if side not in ("B", "S"):                    # tick-rule fallback
        side = "B" if last[0] is None or price >= last[0] else "S"
    last[0] = price
    bar = (t_ms // IV_MS) * IV_MS
    cell = book[bar][price]
    cell[0 if side == "B" else 1] += qty

def show(bar):
    levels = book[bar]
    buy  = sum(c[0] for c in levels.values())
    sell = sum(c[1] for c in levels.values())
    poc  = max(levels, key=lambda p: sum(levels[p]))
    print(f"bar {time.strftime('%H:%M', time.localtime(bar/1000))}  "
          f"delta={buy-sell:+d}  POC={poc}  vol={buy+sell}")

async def main():
    async with websockets.connect(URL, max_size=None) as ws:
        now = int(time.time() * 1000)
        await ws.send(json.dumps({"method":"aggTrades","id":1,"params":{       # 1) history
            "exchange":EXCH,"symbol":SYM,"startTime":now-3600_000,"endTime":now}}))
        await ws.send(json.dumps({"method":"SUBSCRIBE","id":2,                  # 2) live
            "params":[f"{EXCH}.{SYM}@trade"]}))
        async for raw in ws:
            m = json.loads(raw)
            if m.get("id") == 1 and m.get("result"):           # aggTrades reply
                for t, p, q, s in m["result"]: add(t, p, q, s)
                print("seeded", len(m["result"]), "trades")
            elif m.get("stream", "").endswith("@trade"):       # live trade
                d = m["data"]; add(d["T"], d["p"], d["q"], d["side"])
                show((d["T"] // IV_MS) * IV_MS)

asyncio.run(main())
```

---

## Full example — JavaScript

```js
const URL="ws://127.0.0.1:9001/", EXCH="CME", SYM="ESU6", IV_MS=60000;
const book = new Map();                      // barStart -> Map(price -> {buy,sell})
let last = null;
function cell(bar, price){
  if(!book.has(bar)) book.set(bar, new Map());
  const m = book.get(bar);
  if(!m.has(price)) m.set(price, {buy:0, sell:0});
  return m.get(price);
}
function add(t, price, qty, side){
  if(side!=="B" && side!=="S") side = (last===null || price>=last) ? "B" : "S";  // tick rule
  last = price;
  const bar = Math.floor(t/IV_MS)*IV_MS;
  const c = cell(bar, price);
  if(side==="B") c.buy += qty; else c.sell += qty;
}
function metrics(bar){
  const m = book.get(bar); if(!m) return;
  let buy=0, sell=0, poc=null, pv=-1;
  for(const [p,c] of m){ buy+=c.buy; sell+=c.sell; const t=c.buy+c.sell; if(t>pv){pv=t;poc=p;} }
  console.log(`delta=${buy-sell}  POC=${poc}  vol=${buy+sell}`);
}

const ws = new WebSocket(URL); let id=1;
ws.onopen = () => {
  const now = Date.now();
  ws.send(JSON.stringify({method:"aggTrades", id:1, params:{
    exchange:EXCH, symbol:SYM, startTime:now-3600000, endTime:now}}));   // 1) history
  ws.send(JSON.stringify({method:"SUBSCRIBE", id:2, params:[`${EXCH}.${SYM}@trade`]}));  // 2) live
};
ws.onmessage = (e) => {
  const m = JSON.parse(e.data);
  if(m.id===1 && m.result){ m.result.forEach(([t,p,q,s])=>add(t,p,q,s)); }
  else if((m.stream||"").endsWith("@trade")){
    const d=m.data; add(d.T, d.p, d.q, d.side); metrics(Math.floor(d.T/IV_MS)*IV_MS);
  }
};
```

---

## Pitfalls

- **Side classification.** Prefer the server's `side` (`"B"`/`"S"`). When it is `""`, fall back to the *tick
  rule*: price ≥ last trade → buy, else sell. For better accuracy, compare against best bid/ask from
  `@bookTicker`.
- **Bar alignment.** `floor(time/intervalMs)*intervalMs` aligns to the epoch (matches minute/hour
  boundaries). Tick/volume/range bars are **not** time-aligned — footprint by time only makes sense for
  time-based intervals (`s/m/h/d`).
- **History + live overlap.** `aggTrades` ends at "now" and live `@trade` continues from there; the boundary
  is seamless. Don't double-count by replaying an overlapping window twice.
- **Float prices as keys.** Trade prices are exact tick multiples (e.g. 5500.25); using the number as a map
  key is fine. Format display decimals from the instrument's tick, not a fixed value.
- **Tick replay only covers the live/current session.** `aggTrades` is backed by Rithmic's intraday
  *tick* replay, which serves trades for the **session that is currently open**. Once a session closes
  (overnight, weekends, holidays) its per-trade ticks are **no longer replayable** — `aggTrades` returns an
  empty result even for a window that sits squarely inside that past session. This is a Rithmic data-plant
  limitation, not a bug, and widening the time window does **not** recover them.
  - OHLC **bars** (`klines`) *are* available historically — only the per-trade buy/sell split is not. So a
    closed market shows past candles fine, but their footprint cannot be reconstructed after the fact.
  - Practical consequence: from **Rithmic replay alone**, historical footprint is only available while the
    market is open (the current session). When `aggTrades` returns empty, the market is closed — build
    footprint **live** from `@trade` once trading resumes; don't treat the empty replay as an error.
  - **Local tick store (this gateway).** To work around the replay limit, the gateway records every live
    `@trade` to disk and `aggTrades` falls back to that store when Rithmic can't replay. So footprint **does**
    survive market close *for any session the gateway was running during* — it just can't reconstruct a session
    nobody recorded.

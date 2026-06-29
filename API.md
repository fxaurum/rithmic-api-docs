# Fxaurum Rithmic — WebSocket API Reference

A **Binance-style WebSocket API** over a single endpoint that streams live and historical futures market data
from **Rithmic (R|API+)**. A client subscribes to exactly the streams it wants and receives only those — no
client-side filtering needed. This document is written for any developer or AI to implement a client in any
language.

> Companion docs: **FOOTPRINT.md** — build an order-flow footprint from these sockets · **RECIPES.md** —
> cookbook of popular indicators (CVD, VWAP, Volume Profile, DOM, Heatmap, …) with API usage + algorithm.

---

## Overview

| Property        | Value |
|-----------------|-------|
| Transport       | WebSocket (RFC 6455), text frames, UTF-8 JSON |
| Default endpoint| `ws://127.0.0.1:9001/` |
| Encoding        | One JSON object per message |
| Auth            | None at the socket — the gateway is already logged into Rithmic. Bind to localhost only. |
| Symbol case     | Case-insensitive; the server upper-cases symbols (`esu6` → `ESU6`). Rithmic itself is case-sensitive, so prefer uppercase. |
| Timestamps      | Unix **milliseconds** (UTC) everywhere |

The same HTTP port also serves: the chart web app at `GET /`, the HTML docs at `GET /api.html` &
`GET /footprint.html`, and these Markdown docs at `GET /api.md` & `GET /footprint.md`.

---

## Connecting

No handshake/login step. Once the socket is open you may send requests and subscribe immediately.

```js
ws = new WebSocket("ws://127.0.0.1:9001/");
```

---

## Connection lifecycle & reconnect

There are **two independent connections** to keep in mind:

**1. Your WebSocket (client → gateway).** No auth. Server-side subscriptions are tied to your socket — if it
closes, they are dropped. So if *your* socket drops, reconnect **and re-`SUBSCRIBE`**. A simple retry loop
(e.g. every 2 s) is recommended.

**2. The gateway's upstream link to Rithmic.** The gateway keeps **one** logged-in R|API+ session that feeds
all clients. If that upstream link drops (network blip → Rithmic `ConnectionBroken`), the gateway:

- **auto-reconnects** — retries about **every 30 s until it succeeds** (matching Rithmic's own sub-minute
  reconnect cadence; there is no fixed attempt cap), and
- on success **auto-restores every active subscription** for you — you do **not** re-`SUBSCRIBE`.

While the upstream is down, **stream events simply pause** and then resume automatically once it reconnects —
no error frame is sent for a transient upstream blip. Requests (`klines`, `aggTrades`, …) sent *during* the
outage may time out; just retry them. The gateway stops retrying only if Rithmic **rejects** the re-login
(expired / disabled account) or the operator logs out — in those cases the upstream stays down.

> **Takeaway for clients:** keep your socket open and don't tear down just because events went quiet — the
> gateway is healing the upstream and will replay your subscriptions. Only handle reconnection for *your own*
> socket.

---

## Message format

Every message you **send** is a JSON object with a `method`. Attach a unique `id` (number or string) to
correlate the response.

```json
{ "method": "<name>", "params": { } , "id": 7 }
```

Two kinds of messages arrive **from** the server:

| Kind          | Shape                                              | When |
|---------------|----------------------------------------------------|------|
| Response      | `{ "id": 7, "status": 200, "result": ... }`        | Reply to a request with that `id` |
| Stream event  | `{ "stream": "cme.esu6@trade", "data": { ... } }`  | Pushed continuously after SUBSCRIBE |

Routing rule: check for `msg.stream` first (push event); otherwise match `msg.id` to a pending request.
Errors carry `error` and/or a `status` ≥ 400.

---

## Requests

### `klines` — historical candles

Returns OHLCV candles for a time window (backed by Rithmic bar replay).

```json
{ "method": "klines",
  "params": { "exchange":"CME", "symbol":"ESU6", "interval":"1m",
              "startTime": 1718000000000, "endTime": 1718003600000, "limit": 500 },
  "id": 1 }
```

| Param                  | Type    | Notes |
|------------------------|---------|-------|
| `exchange`             | string  | e.g. `CME`. Default `CME`. |
| `symbol`               | string  | Contract, e.g. `ESU6`. **Required.** |
| `interval`             | string  | See [Intervals](#intervals). Default `1m`. |
| `startTime` / `endTime`| int (ms)| Window. Default end = now, start = end − 1h. |
| `limit`                | int     | Keep at most the last N candles. Default 500. |

**Response** — array of `[openMs, open, high, low, close, volume, closeMs]`:

```json
{ "id":1, "status":200, "result":[
  [1718000000000, 5500.25, 5501.00, 5499.75, 5500.50, 182, 1718000060000],
  [1718000060000, 5500.50, 5502.25, 5500.25, 5501.75, 240, 1718000120000]
]}
```

> The first request per instrument+interval warms a bar handle (~2.5 s). An empty result means there were no
> bars in the window (e.g. market closed) — widen the window.

### `aggTrades` — historical trades

Returns individual historical trade prints **with aggressor side**. This is the data you replay to build a
footprint for past candles. Served from Rithmic trade replay when available; for sessions Rithmic can no
longer replay (market closed / older session) it falls back to the gateway's **local tick store** (trades
recorded live to disk), so footprint/big-trade history survives across restarts. Empty only if neither has data.

```json
{ "method":"aggTrades",
  "params":{ "exchange":"CME", "symbol":"ESU6",
             "startTime":1718000000000, "endTime":1718003600000, "limit":0 },
  "id": 2 }
```

**Response** — array of `[timeMs, price, qty, side, ordId, tu]` where side is `"B"` (buy aggressor / lifted ask),
`"S"` (sell aggressor / hit bid), or `""` (unknown). `tu` is the **exchange match time in microseconds**
(`SourceSsboe`·1e6 + `SourceUsecs`) — prints of one matching event share it, so you rebuild **cumulative trades**
by summing consecutive prints with the same `tu` (a single sweep across several price levels = one logical trade;
see RECIPES.md §1). That's the key ATAS keys on (its `CumulativeTrade.Time`). `ordId` (`AggressorExchOrdId`) is
the aggressor order id — Rithmic supplies it, but grouping by it over-merges an order that fills across several
events, so it's only a fallback when `tu` is missing. The local tick store persists `tu` **in-record** (v2 format:
one `.tick` file per CME session, 29-byte records with `tu` inside — the old `.fxu`/`.fxt` sidecar pair is gone),
so store-served history carries `tu` too; only days recorded by very old builds are `tu`-less/`ordId`-less
(`0`/`""`, milliseconds only) — group those by a time window instead.

```json
{ "id":2, "status":200, "result":[
  [1718000001120, 5500.25, 2, "B", "6417163530506", 1718000001120300],
  [1718000001450, 5500.00, 1, "S", "6417163530632", 1718000001450900],
  [1718000001450, 5500.00, 4, "S", "6417163530632", 1718000001450900]
]}
```

### `bigTrades`

Server-side **cumulative-trade detection**: the gateway pulls the raw trades for the range (Rithmic replay →
local store), **groups them by matching event** (`tu` → `o` → 50 ms window — the ATAS rule), filters by `min`,
and returns **only the resulting bubbles** — so a client can draw big trades for a whole session without pulling
millions of raw ticks. See BIGTRADES.md.

```json
{ "method":"bigTrades",
  "params":{ "exchange":"CME", "symbol":"ESU6",
             "startTime":1718000000000, "endTime":1718003600000, "min":50, "merge":0 },
  "id": 8 }
```

| Param                  | Type     | Notes |
|------------------------|----------|-------|
| `exchange` / `symbol`  | string   | `symbol` **required**. |
| `startTime` / `endTime`| int (ms) | Window. Default end = now, start = end − 6h. |
| `min`                  | int      | Return only cumulative trades with `vol ≥ min`. Default 1. |
| `merge`                | int (ms) | **Optional, default `0`.** `0` = group strictly by **matching event** (`tu` → `o` → 50 ms, precise — one aggressor sweep = one bubble). `> 0` = ATAS-style **cumulative window**: merge *all* same-side aggressive prints within `merge` ms regardless of `tu`/`o` → fewer, **larger** bubbles (closer to how ATAS's "Cumulative Trades" looks). `merge > 0` bypasses the per-session bubble cache (computed fresh each call). |

**Response** — array of `[timeMs, side, vol, firstPrice, lastPrice]` (one per cumulative trade with `vol ≥ min`).
`firstPrice`/`lastPrice` differ when the order swept several levels (draw a range box; equal = a bubble).

```json
{ "id":8, "status":200, "result":[
  [1718000001120, "B", 271, 7612.75, 7613.00],
  [1718000002360, "S", 84,  7611.50, 7611.50]
]}
```

### `volumeProfile` — current-session volume-at-price (native)

Rithmic's own `getVolumeAtPrice` for the **live session only** — no local history needed.

```json
{ "method":"volumeProfile", "params":{ "exchange":"CME", "symbol":"ESU6" }, "id": 9 }
```

**Response** — `{ rows:[[price,vol],…], total, vpoc, rp }` (rows price-ascending; `vpoc` = highest-volume price; `rp`
0=ok · 7=no-data · -1=timeout).

### `volumeProfileRange` — multi-session volume-at-price (local store)

Builds volume-at-price for **N closed sessions** from the local TickStore (a closed session with an empty store is
**backfilled once** via `replayBars(Tick,1)`, then served locally). `split:"1"` returns **one profile per session**;
omitted = **one merged** profile. See VOLUMEPROFILE.md.

```json
{ "method":"volumeProfileRange",
  "params":{ "exchange":"CME", "symbol":"ESU6",
             "startTime":1717800000000, "endTime":1718003600000, "days":6, "split":"1" },
  "id": 10 }
```

| Param                  | Type     | Notes |
|------------------------|----------|-------|
| `exchange` / `symbol`  | string   | `symbol` **required**. |
| `days`                 | int      | Number of **trading sessions** to include — the gateway walks back skipping weekends (`IsWeekendSession`), so a span hitting Sat/Sun still yields `days` profiles. When omitted, falls back to the `startTime`/`endTime` window. |
| `startTime` / `endTime`| int (ms) | Fallback window when `days` is absent. |
| `split`                | "1"/bool | `"1"` → per-session profiles; omitted → one merged profile. |

**Response (merged)** — `{ rows, total, vpoc, sessions:<count> }`.
**Response (`split`)** — `{ sessions:[ { start, end, rows, total, vpoc }, … ], total, split:true }` (each session
carries its `start`/`end` ms band so the client anchors it under that session's candles).

### `continuousInfo` — continuous-contract chain

Resolves a **back-adjusted continuous futures** chain for a root from the local contracts. See CONTINUOUS.md.

```json
{ "method":"continuousInfo",
  "params":{ "exchange":"CME", "root":"ES", "rule":"date", "adjust":"1",
             "rolls":{ "ESM6":"20260619" } },
  "id": 11 }
```

| Param | Type | Notes |
|---|---|---|
| `root` | string | Product root (e.g. `ES`, `GC`). Required. |
| `rule` | "date"/"volume" | Rollover rule. `date` = at each contract's last daily date; `volume` = when the next contract's daily volume first exceeds the current's. |
| `adjust` | "1"/bool | Back-adjust (shift older contracts by the cumulative settlement diff). |
| `rolls` | object | **Optional** force-roll dates `{ "<sym>":"<yyyymmdd>" }` (date rule only). |

**Response** — `{ front, segments:[ { sym, start, end, offset }, … ] }`. `front` = the live contract to subscribe; each segment is
a contract's active span (`end` = -1 for the front) and the `offset` added to its prices.

**The `cont` param.** Pass `cont:{ root, rule, adjust, rolls? }` on **`klines`, `aggTrades`, `bigTrades`, `volumeProfileRange`**
to get that data **stitched + back-adjusted** across contracts (the server splits the range per contract and shifts prices). Omit
`cont` for a single contract. Live streams stay on the front month (offset 0). See **CONTINUOUS.md**.

### `syncTicks` — backfill tick history into the local store

Queues a **background** job that pulls closed-session ticks from Rithmic (`replayBars(Tick,1)`, paginated) into the
gateway's **local tick store**, **recent-session-first**, so footprint/big-trade history is available offline and
across restarts. Returns immediately (the worker is single-threaded and paced); poll `syncStatus` for progress.

```json
{ "method":"syncTicks",
  "params":{ "exchange":"CME", "symbol":"ESM6",
             "startTime": 1716000000000, "endTime": 1718000000000,
             "expMs": 1718323200000, "priority": 1 },
  "id": 10 }
→ { "id":10, "status":200, "result":{ "queued": 12, "symbol":"ESM6", "kind":"tick" } }
```

| Param                  | Type     | Notes |
|------------------------|----------|-------|
| `exchange` / `symbol`  | string   | `symbol` **required**. |
| `startTime` / `endTime`| int (ms) | Span to cover. Only **closed** sessions are queued (the current session is built live). |
| `expMs`                | int (ms) | Optional contract **expiration**. When it has passed and the last session ≤ `expMs` is captured, the contract is **finalized** (never synced again). |
| `priority`             | int      | `1` = user-initiated → jumps ahead of the background archive queue. Omit/`0` = normal. |

Returns `{ queued, symbol, kind:"tick" }` (`queued` = sessions enqueued). If the contract is already finalized:
`{ queued:0, symbol, kind:"tick", final:true }`. The worker auto-skips **Sat/Sun** sessions (CME closed) without
calling Rithmic, and stops descending once it is **~10 calendar days before the first liquid session** (the
contract's effective start) — so a contract's dead pre-listing tail isn't dragged through. Sessions under ~100
ticks are treated as pre-liquid and dropped.

### `tickInfo` — local-store summary per symbol

What the local tick store holds for each symbol — drives a "sync history" management UI.

```json
{ "method":"tickInfo", "params":{ "exchange":"CME", "symbols":["ESU6","ESM6"] }, "id": 11 }
→ { "id":11, "status":200, "result":[
     { "symbol":"ESU6", "sessions":9, "ticks":1090000, "finalized":false, "syncing":true,  "first":"20260605", "last":"20260618" },
     { "symbol":"ESM6", "sessions":0, "ticks":0,       "finalized":false, "syncing":false, "first":"",         "last":"" } ] }
```

| Field        | Meaning |
|--------------|---------|
| `sessions`   | Number of closed sessions with data on disk |
| `ticks`      | Total ticks stored |
| `finalized`  | Contract fully captured to its last session → won't sync again |
| `syncing`    | A `syncTicks` job for this symbol is queued/running |
| `first`/`last` | First/last stored session tag (`yyyyMMdd`, CME trade-date) |

### `syncStatus` — overall sync progress

Progress of the single background sync worker (ticks **and** bars), for a "syncing…" indicator.

```json
{ "method":"syncStatus", "id": 12 }
→ { "id":12, "status":200, "result":{ "remaining":7, "total":12, "done":5, "pct":41, "running":true, "current":"ESM6 tick 06-12" } }
```

`pct` is computed against the **peak** queue depth so it rises monotonically as the queue drains.

### `syncHistory` — backfill bar history (klines) into the bar store

Same idea as `syncTicks` but for the **bar store** (klines), so switching timeframe/symbol later reads locally.
`{ "method":"syncHistory", "params":{ "exchange":"CME", "symbol":"ESU6" }, "id":13 }`. Listed contracts are also
prefetched in the background when a chart opens.

### `rawTicks` — probe Rithmic tick-history depth (debug)

A **diagnostic** call: it asks **Rithmic directly** (bypassing the local store) how far back its tick history actually
reaches, and returns a small head/tail sample instead of the full payload. Use it to verify what `mode` and date range
Rithmic will actually serve before relying on `syncTicks`/`aggTrades`. Like `mboProbe`, it has **side effects** (it
subscribes + replays from the feed) and is meant for debugging, not for the normal client path.

```json
{ "method":"rawTicks", "params":{ "exchange":"CME", "symbol":"ESU6",
                                  "startTime":1782000000000, "endTime":1782010000000, "mode":"trades" }, "id":14 }
→ { "id":14, "status":200, "result":{
     "mode":"trades", "exchange":"CME", "symbol":"ESU6",
     "reqStart":1782000000000, "reqEnd":1782010000000, "reqStartIso":"...Z", "reqEndIso":"...Z",
     "count":53120, "firstTs":1782000001000, "lastTs":1782009998000, "firstIso":"...Z", "lastIso":"...Z",
     "withOrderId":true, "capped":false,
     "sampleHead":[ [t,p,q,side,o,tu], ... ], "sampleTail":[ ... ] } }
```

| Param        | Type   | Notes |
|--------------|--------|-------|
| `exchange`   | string | Default `CME`. |
| `symbol`     | string | **Required**. |
| `endTime`    | number | ms; default = now. |
| `startTime`  | number | ms; default = `endTime − 10 min`. |
| `mode`       | string | `trades` (default → `replayTrades`) or `bars`/`tick`/`1t` (→ `getHistoryBars(Tick,1)`, the deeper ~90-day path). |

`count` is the **total** rows Rithmic returned (`capped:true` if more than the head+tail sample); `withOrderId` is true
when rows carry the aggressor order id (`o`); `firstTs`/`lastTs` (+ `…Iso`) bracket what actually came back. Replies on a
completion flag, or a **40 s** safety timeout. `replayTrades` only reaches the **current session**; the `bars` mode goes
back ~90+ days — see **STORAGE.md** for the depth notes.

> `account` & `exchangeInfo` are **pure raw 1-1** (returned as-is, no transform) → documented in **RAW.md §3** : `getAccounts` / `listExchanges`. The gateway still accepts the old Binance names via alias.

### `instrumentInfo`

Reference data for an instrument: **tick size**, price decimals, contract **expiration**, point value… Authoritative from
Rithmic (`getRefData` + `getPriceIncrInfo`), cached per instrument — don't infer the tick from prices.

```json
{ "method":"instrumentInfo", "params":{ "exchange":"COMEX", "symbol":"GCG6" }, "id":6 }
→ { "id":6, "status":200, "result":{
     "symbol":"GCG6", "exchange":"COMEX",
     "tickSize":0.1, "priceDecimals":1,
     "expiration":"202602", "expirationTime":"...",
     "pointValue":100, "currency":"USD",
     "description":"Gold", "productCode":"GC", "instrumentType":"Future", "tradingExchange":"COMEX" } }
```

Fields may be missing if the feed doesn't return them (client then falls back to inferring the tick from prices).

### `searchSymbol`

Search instruments by keyword (autocomplete/pick instead of typing). `exchange` optional — empty searches the main
exchanges sequentially (CME/CBOT/NYMEX/COMEX) and merges, because Rithmic requires one exchange per call. `field` defaults to `Symbol`,
`op` to `Contains`. `field`: Symbol | Description | ProductCode | InstrumentType | Exchange | ExpirationDate | Any. `op`:
Contains | Equals | StartsWith | EndsWith. From Rithmic `searchInstrument`; 8s timeout returns empty.

```json
{ "method":"searchSymbol", "params":{ "query":"ES", "exchange":"CME" }, "id":7 }
→ { "id":7, "status":200, "result":[
     { "symbol":"ESU6", "exchange":"CME", "description":"E-mini S&P 500", "expiration":"202609", "productCode":"ES", "instrumentType":"Future" },
     ... ] }
```

### `instrumentByUnderlying` — options chain (for the Options popup + GEX / 0DTE)

The options chain of a **futures contract** (the future is the underlying → strikes settle into it, spot-consistent). **Two-step**, and the gateway **serializes** these calls (single-in-flight at Rithmic — concurrent callers queue, so they don't time each other out):

1. **List expiries** — `expiration:""` → all option expiry **dates** (`CCYYMMDD`) for that future (daily/weekly/monthly; auto-corrects the underlying). `rows` empty.
2. **Drill one expiry** — `expiration:"<CCYYMMDD>"` → the option **instruments** at that date (the strike grid).

```json
{ "method":"instrumentByUnderlying", "params":{ "underlying":"ESU6", "exchange":"CME", "expiration":"" }, "id":8 }
→ { "id":8, "status":200, "result":{
     "expirations":["20260617","20260618","20260619", ...],   // option expiry dates
     "products":[], "rp":0, "rows":[] } }

{ "method":"instrumentByUnderlying", "params":{ "underlying":"ESU6", "exchange":"CME", "expiration":"20260619" }, "id":9 }
→ { "id":9, "status":200, "result":{ "expirations":["20260619"], "products":["E3D"], "rp":0,
     "rows":[ ["ESU6 C7000",7000,"C","20260619","E3D"], ["ESU6 P7000",7000,"P","20260619","E3D"], ... ] } }
```

`rows` = `[symbol, strike, "C"/"P", expiration, productCode]`. `rp`: `0`=ok, `7`=NO_DATA, `-1`=timeout (no reply in 12 s). **Filter out Event Contracts** (`productCode` matching `/^EC/` — e.g. `ECES` hourly events; gold EC also report a **10× strike**). Per-strike **OI + bid/ask** come from the `@bookTicker` stream (OI rides in the `oi` field); GEX batch-subscribes the strikes ±band, snapshots, unsubscribes. Falls back to `optionChain` (below) if no dates are returned. See **OPTIONS.md** (chain popup, GEX, 0DTE, Black-76).

### `optionChain` — options chain by **product + month** (fallback)

The other route to an options chain — Rithmic `getOptionList`, keyed by the **option product code** and a **month**
(`YYYYMM`) rather than a specific future. `instrumentByUnderlying` is **preferred** (spot-consistent, one call);
`optionChain` is the **fallback** the client uses when `instrumentByUnderlying` returns no expiry dates, and is handy
when you already know the monthly option product. Serialized like the other instrument calls; **8 s** timeout → `rp:-1`.

```json
{ "method":"optionChain", "params":{ "exchange":"CME", "product":"ES", "expiration":"202609" }, "id":10 }
→ { "id":10, "status":200, "result":{
     "expirations":["20260919", ...], "products":["EW","E3D", ...], "rp":0,
     "rows":[ ["ESU6 C7000",7000,"C","20260919","EW"], ["ESU6 P7000",7000,"P","20260919","EW"], ... ] } }
```

| Param        | Type   | Notes |
|--------------|--------|-------|
| `exchange`   | string | e.g. `CME` / `COMEX`. Default `CME`. |
| `product`    | string | **Required** — option/underlying **product code** (e.g. `ES`, `GC`). |
| `expiration` | string | Option **month** `YYYYMM`. Default = current month. |

`rows` = `[symbol, strike, "C"/"P", expiration, productCode]` — **same shape as `instrumentByUnderlying`** (the 3rd
element is the call/put flag). `expirations` are the option expiry **dates** (`CCYYMMDD`) Rithmic considers valid for
that product (returned even when `rows` is empty, so you can drill the right month); `products` = the option series
codes. `rp`: `0`=ok, `7`=NO_DATA, `-1`=timeout. Per-strike **OI + bid/ask** still come from `@bookTicker` (the `oi`
field) — same as `instrumentByUnderlying`.

### `chainSub` / `chainUnsub` — bulk-subscribe a whole option chain (Rithmic `subscribeByUnderlying`)

```json
{ "method":"chainSub", "params":{ "exchange":"COMEX", "underlying":"GCQ6", "expiration":"20260828",
                                  "symbols":["GCQ6 C4150","GCQ6 P4150", … ] }, "id":N }
```

One call subscribes **all options of the underlying for that expiration** on Rithmic (vs one `subscribe` per strike). The
quotes/OI/last still flow out via each strike's `@bookTicker` (and `@trade`) stream — so the client subscribes those streams
**as usual**; the gateway just **skips the per-strike Rithmic subscribe** for symbols a chain covers (the `symbols[]` you
pass). Ref-counted per `exchange|underlying|expiration`; `chainUnsub` (or disconnect) releases it. Used by **GEX** (brackets
the OI sweep) and the **options popup**. Trade-off: Rithmic streams **every** strike of the expiry to the gateway (heavier
feed) in exchange for **far fewer subscribe calls** (dodges the per-strike storm / "in use" throttle). See **OPTIONS.md**.

```json
{ "method":"chainUnsub", "params":{ "exchange":"COMEX", "underlying":"GCQ6", "expiration":"20260828" }, "id":N }
```

### `ping`

```json
{ "method":"ping", "id":5 }   →   { "id":5, "result":{} }
```

### `SUBSCRIBE` / `UNSUBSCRIBE` / `LIST_SUBSCRIPTIONS`

Stream names are `<EXCHANGE>.<SYMBOL>@<type>` (case-insensitive).

```json
{ "method":"SUBSCRIBE",
  "params":[ "CME.ESU6@trade", "CME.ESU6@bookTicker", "CME.ESU6@kline_1m", "CME.ESU6@depth10" ],
  "id": 6 }
→ { "id":6, "result":null }

{ "method":"UNSUBSCRIBE", "params":[ "CME.ESU6@trade" ], "id":7 }   →   { "id":7, "result":null }
{ "method":"LIST_SUBSCRIPTIONS", "id":8 }   →   { "id":8, "result":[ "cme.esu6@kline_1m", ... ] }
```

After SUBSCRIBE, matching stream events arrive continuously until UNSUBSCRIBE or socket close. Subscriptions
are reference-counted server-side and shared across clients.

---

## Streams

### `@trade` — last-trade prints

```json
{ "stream":"cme.esu6@trade",
  "data":{ "e":"trade", "s":"ESU6", "p":5500.25, "q":2, "side":"B", "T":1718000001120,
           "tu":1718000001120300, "o":"6417163530506", "r":1718000001140 } }
```

| Field  | Meaning |
|--------|---------|
| `p`    | Trade price |
| `q`    | Trade size (contracts) |
| `side` | `"B"` buy aggressor · `"S"` sell aggressor · `""` unknown |
| `T`    | Trade time (ms, exchange source time) |
| `tu`   | Exchange match time in **microseconds** (`SourceSsboe`·1e6 + `SourceUsecs`). Prints of one matching event share it — **primary** grouping key for ATAS-style cumulative trades / big trades (= ATAS's `CumulativeTrade.Time`; RECIPES.md §1) |
| `o`    | Aggressor exchange order id (`AggressorExchOrdId`) — Rithmic supplies it, but grouping by it over-merges an order filling across several events; **fallback** only when `tu` is missing |
| `r`    | Gateway receive time (ms) — measure feed latency: `r − T` |

> Only **real trade updates** are streamed. The last-trade *snapshot* Rithmic echoes on every (re)subscribe
> (its `Image` callback) is filtered out, so you never get a stale phantom print at an old timestamp.

### `@bookTicker` — best bid/ask

```json
{ "stream":"cme.esu6@bookticker",
  "data":{ "s":"ESU6", "b":5500.00, "B":12, "a":5500.25, "A":9, "v":1284530, "oi":2103445, "l":5500.25 } }
```

| Field        | Meaning |
|--------------|---------|
| `b` / `B`    | Best bid price / size |
| `a` / `A`    | Best ask price / size |
| `v`          | Session volume (cumulative, `TradeVolumeInfo.TotalVolume`); rides on the next quote |
| `oi`         | Open interest (`OpenInterestInfo.Quantity`); pushed immediately on change |
| `l`          | Last trade price — incl. the **snapshot** Rithmic sends on (re)subscribe (`CallbackType.Image`), so illiquid instruments (e.g. option strikes) show a last even with no live trade |

### `@depth` / `@depth<N>` — order book (DOM)

`@depth` = full book, `@depth10` = top N levels (the **N nearest-to-price** levels each side). The trim is applied **per subscription** on the live, continuously-maintained book — so two clients can watch the same symbol at different depths. The gateway keeps up to **1000** levels each side internally (`DOM_MAX_LEVELS`); `@depth` (no N) returns all of them. The web app subscribes `@depth<N>` with N from the Heatmap ⚙ **"Độ sâu sổ lệnh"** setting (default **100**, shared by the DOM ladder and the heatmap), so raising it deepens both at once.

```json
{ "stream":"cme.esu6@depth10",
  "data":{ "s":"ESU6", "bids":[[5500.00,12],[5499.75,40]], "asks":[[5500.25,9],[5500.50,33]] } }
```

| Field   | Meaning |
|---------|---------|
| `bids`  | Array of `[price, size]`, best first |
| `asks`  | Array of `[price, size]`, best first |

> Depth requires the instrument's DOM data to be available to the account; if not, the stream stays silent.

### `@mbo` / `@mbo<N>` — order-by-order book (MBO, Rithmic DBO)

Like `@depth` but **per individual order** instead of aggregated size. The gateway maintains a per-order book (Rithmic
**DBO / Depth-By-Order**) over a rolling **±20-tick** window around the BBO and emits ~20 Hz. `@mbo<N>` trims to the top *N*
levels each side. **Requires a separate DBO market-data entitlement** — probe with `mboProbe` first. See **MBO.md**.

```json
{ "stream":"comex.gcq6@mbo20",
  "data":{ "s":"GCQ6",
           "bids":[ [4152.3, 4, 31, [15,8,5,3]], [4152.2, 2, 20, [12,8]] ],
           "asks":[ [4152.7, 1, 250, [250]] ] } }
```

| Field | Meaning |
|-------|---------|
| `bids` / `asks` | Array of levels `[price, orderCount, totalSize, sizes[]]`, best first |
| `sizes[]` | Individual resting orders at that price, **sorted by queue priority** (front first), capped at 64/level |

### `mboProbe` — DBO entitlement probe (debug)

`{"method":"mboProbe","params":{"exchange","symbol","levels"(per side, def 6),"ms"(window, def 8000)}}` — subscribes DBO
around the BBO for `ms`, then returns and unsubscribes. Run **after opening the symbol's chart** (needs a BBO). Returns
`{entitled, events, new, change, delete, image, distinctOrders, hasExchOrdId, hasPriority, alerts[], sample[]}` where
`sample[]` rows are `[type, side, price, size, exchOrdId, priority, tsMs]`. `entitled:true` ⇒ DBO works → `@mbo` is usable.

### `@kline_<interval>` — live candles

> Rithmic typically emits a bar when it **closes**. For a smoothly-updating *forming* candle, most clients
> build it from `@trade` (see FOOTPRINT.md).

```json
{ "stream":"cme.esu6@kline_1m",
  "data":{ "e":"kline", "s":"ESU6",
           "k":{ "t":1718000000000, "T":1718000060000, "i":"1m",
                 "o":5500.25, "h":5501.00, "l":5499.75, "c":5500.50, "v":182 } } }
```

| Field (inside `k`) | Meaning |
|--------------------|---------|
| `t` / `T`          | Candle open / close time (ms) |
| `i`                | Interval |
| `o h l c`          | Open, high, low, close |
| `v`                | Volume |

---

## Intervals

Format: `<number><unit>`.

| Unit | Meaning        | Examples            |
|------|----------------|---------------------|
| `s`  | seconds        | `1s 5s 10s 30s`     |
| `m`  | minutes        | `1m 2m 5m 15m`      |
| `h`  | hours          | `1h 4h`             |
| `d` / `w` | daily / weekly | `1d 1w`        |
| `t`  | tick bars      | `100t`              |
| `v`  | volume bars    | `1000v`             |
| `r`  | range bars     | `4r`                |

---

## Errors

A failed reply carries `error` (message) + a `status` ≥ 400. **Exceptions also carry a `code`** so API users can
investigate: a Rithmic `OMException` → the numeric **`rp`** (R|API+ code) plus `omdesc` (short text) and `detail`
(raised-in class / method); any other exception → its .NET type name.

```json
{ "id":1, "status":400, "error":"thiếu symbol" }
{ "id":7, "status":500, "error":"unknown request", "code":14, "omdesc":"unknown request",
  "detail":"Raised in : com.omnesys.rapi.IhConn | Method : subscribeBar | Error : unknown request" }
{ "id":3, "status":500, "error":"...", "code":"IOException" }
{ "error":"bad stream: CME.ESU6@oops" }
```

| status | Meaning |
|--------|---------|
| 200    | OK |
| 400    | Bad request (missing/invalid params) — `error` only |
| 500    | Server/Rithmic exception — `error` + `code` (+ `omdesc`/`detail` for `OMException`) |

### Rithmic `OMException` codes (`code` / `rp`)

| rp | meaning | typical cause |
|----|---------|---------------|
| 6  | bad input | request params the plant rejects |
| 7  | no data | nothing for the requested window |
| 8  | already exists | duplicate subscribe / state already set (usually benign) |
| 9  | in use | another request of the same kind in progress |
| 10 | in progress | request still running |
| 11 | no handle | connection/handle not ready yet (transient at startup) |
| 13 | permission | not entitled to that data |
| 14 | unknown request | request type not served right now — e.g. **tick history on weekends/maintenance** (bar history still works) |
| 1001 | io error | plant I/O (internal, login) |
| 1002 | authentication error | login/credentials |

### Notices (push, not a reply)

Background failures the gateway can't tie to a request `id` are pushed on the **`notice` stream**:
```json
{ "stream":"notice", "data":{ "level":"error", "msg":"Đồng bộ tick COMEX.GCQ6: unknown request (rp=14) — …", "detail":"…(optional full text)" } }
```
The web app shows a banner — **red** `error` (hard, e.g. permission), **yellow** `warn` (transient/will auto-retry, e.g. a
weekend tick stall), **blue** `info` (diagnostic); clicking copies `detail`. First-chance `OMException`s from rapiplus's own
login/encoding machinery are filtered out (not our error), and benign subscribe-lifecycle transients (rp 6/8/11) log to the
server console only.

---

## Notes & gotchas

- **Uppercase symbols.** Server upper-cases them, but send uppercase to be safe (Rithmic is case-sensitive).
- **klines warm-up.** First request per instrument+interval costs ~2.5 s; subsequent ones are immediate.
- **Off-hours / weekends.** A short window may return 0 candles/trades because the market is closed. Widen
  `startTime` to reach the last session, then keep the most recent N client-side.
- **Forming candle.** `@kline` updates mainly on bar close; aggregate `@trade` for a live current bar.
- **Contract month.** Futures roll quarterly (H/M/U/Z). e.g. `ESU6` = E-mini S&P, Sep 2026.

---

## Examples

### JavaScript (browser / Node)

```js
const ws = new WebSocket("ws://127.0.0.1:9001/");
let id = 1;
const pending = new Map();
const rpc = (method, params) => new Promise(res => {
  const myId = id++; pending.set(myId, res);
  ws.send(JSON.stringify(params ? {method, params, id:myId} : {method, id:myId}));
});

ws.onmessage = (e) => {
  const m = JSON.parse(e.data);
  if (m.stream) {                       // push event
    if (m.stream.endsWith("@trade")) console.log("trade", m.data.p, m.data.q, m.data.side);
    return;
  }
  if (pending.has(m.id)) { pending.get(m.id)(m.result); pending.delete(m.id); }
};

ws.onopen = async () => {
  const now = Date.now();
  const candles = await rpc("klines", {exchange:"CME", symbol:"ESU6", interval:"1m",
                                       startTime: now - 6*86400000, endTime: now, limit:200});
  console.log("got", candles.length, "candles");
  ws.send(JSON.stringify({method:"SUBSCRIBE", params:["CME.ESU6@trade","CME.ESU6@bookTicker"], id:id++}));
};
```

### Python (`websockets`)

```python
import asyncio, json, time, websockets

async def main():
    async with websockets.connect("ws://127.0.0.1:9001/") as ws:
        now = int(time.time() * 1000)
        await ws.send(json.dumps({"method":"klines","id":1,"params":{
            "exchange":"CME","symbol":"ESU6","interval":"1m",
            "startTime": now - 6*86400000, "endTime": now, "limit":200}}))
        await ws.send(json.dumps({"method":"SUBSCRIBE","id":2,"params":["CME.ESU6@trade"]}))
        async for raw in ws:
            m = json.loads(raw)
            if m.get("stream","").endswith("@trade"):
                d = m["data"]; print("trade", d["p"], d["q"], d["side"])
            elif m.get("id") == 1:
                print("candles:", len(m["result"]))

asyncio.run(main())
```

### C# (`ClientWebSocket`)

```csharp
using System.Net.WebSockets; using System.Text; using System.Text.Json;

var ws = new ClientWebSocket();
await ws.ConnectAsync(new Uri("ws://127.0.0.1:9001/"), default);

async Task Send(object o) {
    var b = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(o));
    await ws.SendAsync(b, WebSocketMessageType.Text, true, default);
}
long now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
await Send(new { method="klines", id=1, @params=new {
    exchange="CME", symbol="ESU6", interval="1m",
    startTime=now-6*86400000L, endTime=now, limit=200 } });
await Send(new { method="SUBSCRIBE", id=2, @params=new[]{ "CME.ESU6@trade" } });

var buf = new byte[1<<20];
while (ws.State == WebSocketState.Open) {
    var r = await ws.ReceiveAsync(buf, default);
    var doc = JsonDocument.Parse(Encoding.UTF8.GetString(buf, 0, r.Count)).RootElement;
    if (doc.TryGetProperty("stream", out var s) && s.GetString().EndsWith("@trade")) {
        var d = doc.GetProperty("data");
        Console.WriteLine($"trade {d.GetProperty("p")} x{d.GetProperty("q")} {d.GetProperty("side")}");
    }
}
```

## Changelog

Current API version: **1.9.1**. Follows SemVer — breaking changes bump the MAJOR number.

### 1.9.1 — 2026-06-29
- **In-app docs landing + offline build prompt.** The gateway now serves a docs index at **`/index.html`** (card grid + a "download all .md merged" button for feeding an LLM; `/` stays the web chart); the app's **Docs** button opens it instead of jumping straight to api.html. A **Build prompt** page (**`/build.html`**, rendered from `BUILD_PROMPT.md`, with a Copy button + a sample screenshot) ships embedded too — a paste-and-go prompt that has another AI read this API and rebuild the "Market Structure" dashboard. Both work in doc-only mode (no Rithmic login).
- **Feedback note for AI consumers.** The build prompt and the merged-.md header now ask any AI reading the API to surface problems / improvement ideas for the user to review rather than changing the gateway itself.

### 1.9.0 — 2026-06-28
- **Raw R|API+ completed + REngine renaming.** Every order/PnL/account/instrument/option callback, replay and report is now documented with verified payload shapes (PII redacted). **Pull** RPC names switched to the **REngine method** (`listExchanges`, `getAccounts`, `searchInstrument`, `getOptionList`, `getInstrumentByUnderlying`, `getVolumeAtPrice`, `getProductRmsInfo`, `getStrategyList`/`getStrategyInfo`, `getEquityOptionStrategyList`, `listBinaryContracts`, `getEasyToBorrowList`, `getAuxRefData`, `getUserProfile`, `listTradeRoutes`, `listOrderHistoryDates`); the old Binance-style names keep working as **dispatch aliases** (webapp/chart unchanged). `instrumentInfo` keeps its name (merges `getRefData`+`getPriceIncrInfo`). `replaySingleHistoricalOrder` exposed. See **RAW.md**.
- **Order entry folded into RAW.md §3 ④ / `raw.html#order`** — the standalone `orders.html` + `ORDERS.md` were removed (full params + value maps now live in RAW.md). **`examples.html`** — minimal client code in 9 languages (Python / C# / JS / TS / C++ / Java / Go / Rust / Kotlin) for the three call kinds (stream / RPC-inline / RPC-trigger).
- **api.html decluttered + composite→raw map.** New **#rawmap** section lists which Raw R|API+ each composite entrypoint calls (alias / merge / stream / local). Removed the pure-alias entrypoints (`account`/`exchangeInfo` → raw.html `getAccounts`/`listExchanges`) and the redundant Account/PnL/RMS table (raw passthrough → raw.html); kept the gateway-only `connections`.
- **App.** API docs are now viewable **without a Rithmic login** (the gateway serves api.html/raw.html in a doc-only mode even during maintenance); MainForm split into separate **Web Chart** / **Docs** buttons.

### 1.8.0 — 2026-06-26
- **Remaining rapiplus pull/subscribe methods exposed — PnL, order/execution history, RMS, strategy, misc.** `subscribePnl` / `replayPnl`, `replayOpenOrders` / `replayAllOrders` / `replayExecutions` / `replayBrackets` / `replayHistoricalOrders` / `replaySingleOrder` / `orderHistoryDates`, `replayQuotes`, `productRms`, `strategyList` / `strategyInfo` / `equityOptionStrategies` / `binaryContracts`, `easyToBorrow`, `auxRefData`, `assignedUsers`, `subscribeAutoLiquidate`, `subscribeUser`, `listEnvironments` / `environment`. Query methods reply inline; subscribe/replay ack and stream results on the matching RAW channel (`@onpnlupdate`, `@onorderreplay`, `@onexecutionreplay`, …). Account auto-resolved. (Credential change, market-maker quoting, OCA/batch-order-list and user-defined-spread builders are intentionally not exposed.) The interactive **api.html** gains live **Execute** buttons for these read-only RPCs (none for order-entry actions). Plus a **Raw event inspector** panel in the web app to discover any callback's fields live.
- **Order entry — `placeOrder` / `modifyOrder` / `cancelOrder` / `cancelAllOrders` / `exitPosition` (`flatten`) / `bracketOrder` / `subscribeOrders` / `tradeRoutes`.** Live order actions wrapping `REngine.sendOrder` (market/limit/stop/stop-limit), `modifyOrder`, `cancelOrder`, `exitPosition`, `sendBracketOrder`. Account is resolved from `{fcmId,ibId,accountId}` / `accountId` / first active; trade route auto-picked from the cached `OnTradeRoute` list (override with `tradeRoute`). The RPC reply is an accept-ack only — working/filled/rejected outcomes arrive on the RAW report streams (`@OnStatusReport` / `@OnFillReport` / `@OnRejectReport` / `@OnBracketUpdate`). ⚠ Real-money; trading entitlement required; local-only socket. Full params + value maps: **RAW.md §3 ④** / `raw.html#order`.
- **Raw R|API+ passthrough — every rapiplus callback as a socket (1-to-1).** A reflective `OnAny` bus turns **each** Rithmic R|API+ push callback into its own stream `<exch>.<sym>@<CallbackName>` (symbol-scoped) or `*.*@<CallbackName>` (global: account / order / PnL / exchange-list / user…). Payload = the Info struct reflected to JSON (`e` = callback name + all fields). Curated streams are unchanged; the raw bus is additive and ref-counted (zero cost when unsubscribed). New 1-to-1 **pull** RPCs `listIbs` / `listUsers` / `userProfile` / `listAgreements`. ⚠ Account/order/PnL/user clusters need the trading entitlement and expose account data — trusted/local use only. See **RAW.md**.
- **`@mboraw` — raw DBO event stream.** Passthrough of every Rithmic DBO event (`New`/`Change`/`Delete`/`Image` with `ExchOrdId`, side, price, size, priority, exchange ts), riding the same per-price window as `@mbo`. See **MBO.md**. (DOM `@depth` stays aggregated-only — no raw form.)

### 1.7.0 — 2026-06-22
- **Continuous contract — `continuousInfo` + `cont` param.** Back-adjusted rollover chain: pick a **root** (e.g. `ES`) and the gateway stitches the locally-stored contracts (`ESM6`→`ESU6`→…) into one seamless series, **back-adjusting** older prices by the cumulative settlement gap at each roll. `continuousInfo` returns `{front, segments:[{symbol, start, end, offset}]}`; the client subscribes `front` live and draws roll markers. Passing `cont:{root, rule, adjust, rolls}` to `klines`, `aggTrades`, `bigTrades` and `volumeProfileRange` makes **candles *and* orderflow** (footprint / VP / big-trade) continuous + back-adjusted. `rule` = `date` (roll at expiry) or `volume` (roll when the next contract's daily volume overtakes); `rolls` overrides individual roll dates. See **CONTINUOUS.md**.
- **Volume Profile — `volumeProfile` + `volumeProfileRange`.** `volumeProfile` returns Rithmic's native current-session volume-at-price (`getVolumeAtPrice`) — `{rows:[[price,vol]], total, vpoc, rp}`. `volumeProfileRange` builds a **multi-session** profile from the local tick store: composite (one merged profile) or `split:1` (one profile per session, each carrying its time band). `days` counts **trading sessions** (weekends skipped); closed sessions self-backfill once. Powers the VP overlay's three modes (daily / comp / sess) + VPOC + Value Area. See **VOLUMEPROFILE.md**.
- **`rawTicks` (debug) documented.** Diagnostic probe of how deep Rithmic's tick history actually reaches (`trades` → current session; `bars` → ~90 days), returning a head/tail sample. See `rawTicks`.
- **Docs parity.** `syncHistory` and `chainSub`/`chainUnsub` now have their own sections in the interactive HTML reference (previously API.md-only).

### 1.6.0 — 2026-06-19
- **MBO (Market-By-Order) — `@mbo` / `@mbo<N>` + `mboProbe`.** Order-by-order book on Rithmic **DBO** (per-price `subscribeDbo`, rolling ±20-tick window around BBO, ~20 Hz). Levels carry `[price, orderCount, totalSize, sizes[]]` (sizes queue-ordered by priority). `mboProbe` verifies the **separate DBO entitlement** + dumps raw events. Powers the DOM "order-by-order" mode. See **MBO.md**.
- **`@bookTicker` gains `l` (last trade).** Now includes the last-trade price — including the snapshot Rithmic sends on (re)subscribe (`CallbackType.Image`) — so illiquid instruments (option strikes) show a last without waiting for a live trade. The Image print is still **not** written to the tick store / footprint (no ghost trades).
- **`chainSub` / `chainUnsub` — bulk option-chain subscribe (`subscribeByUnderlying`).** One call subscribes all options of an underlying/expiration instead of one `subscribe` per strike; quotes/OI/last still route via each strike's `@bookTicker`. Used by GEX (OI sweep) + options popup → far fewer subscribe calls (dodges the per-strike storm / "in use" throttle).

### 1.5.0 — 2026-06-18
- **Local tick-store management RPCs documented — `syncTicks`, `tickInfo`, `syncStatus`** (+ `syncHistory` for bars). `syncTicks` backfills closed-session ticks into the store recent-first (paced, single worker); `priority:1` jumps the queue; `expMs` auto-finalizes an expired contract after its last session is captured. `tickInfo` summarizes what's on disk per symbol; `syncStatus` reports `pct` against peak queue depth. Powers the "⤓ Lịch sử" sync popup.
- **Smarter sync bounds.** The tick archiver **skips Sat/Sun** sessions (CME closed) without calling Rithmic — far fewer `OMException`s — and stops descending **~10 calendar days before the first liquid session** (the contract's effective start, auto-detected from where real volume begins), instead of dragging through a contract's dead pre-listing tail. Sessions under ~100 ticks are dropped as pre-liquid.
- **`bigTrades` gains `merge` (ms).** `0` (default) keeps the precise matching-event grouping; `> 0` switches to an **ATAS-style cumulative window** — merge all same-side aggressive prints within `merge` ms → fewer, larger bubbles (bypasses the per-session cache). Lets a client dial bubble aggregation to match ATAS's look. See `bigTrades`.
- **Tick store v2 (on-disk).** One `.tick` file **per CME session** (32-byte `FXTK` header + 29-byte records with `tu` in-record), a per-symbol `tick.coverage` index (replaces all per-session `.fxv`), and the `.fxu` sidecar is gone (`tu` moved in-record — fixes the old `.fxt`/`.fxu` alignment bug). Folder renamed `Database`→`TickDb`; old layouts **auto-migrate** with zero tick loss. Wire format (`@trade`/`aggTrades`/`bigTrades`) is unchanged.
- **1m candles derived from local ticks.** Closed sessions with stored ticks build their 1m bars locally (and m5/15m/1h/4h aggregate from that), verified byte-for-byte identical to Rithmic klines — so reopening/timeframe-switching reads locally instead of re-hitting Rithmic.
- **`optionChain` documented.** The product+month options-chain call (Rithmic `getOptionList`, `instrumentByUnderlying`'s fallback) now has its own section — same `rows = [symbol, strike, "C"/"P", expiration, productCode]` shape; per-strike OI/bid-ask via `@bookTicker`. See `optionChain`, OPTIONS.md.

### 1.4.0 — 2026-06-17
- **`instrumentByUnderlying` (options chain) documented.** Two-step (`expiration:""` → expiry dates; `expiration:"<CCYYMMDD>"` → strikes), **serialized** server-side (single-in-flight at Rithmic — no more mutual timeouts), Event Contracts filtered (`/^EC/`), `optionChain` monthly fallback. Powers the Options popup + **GEX / 0DTE** (Black-76, dealer gamma, Call/Put Wall, γ-flip). See OPTIONS.md.

### 1.3.0 — 2026-06-15
- **`bigTrades` (new) — server-side cumulative-trade detection.** Gateway groups raw trades by matching event (`tu`) and returns only the bubbles `[t, side, vol, firstPrice, lastPrice]` ≥ `min`, so a client can draw big trades for a whole session without pulling millions of ticks. Source = Rithmic replay → local store (with `tu` from the `.fxu` sidecar). See `bigTrades`, BIGTRADES.md.
- **Microsecond match time `tu` — matching-event grouping (true ATAS parity).** `@trade` and `aggTrades` now carry `tu`, the exchange match time in microseconds (`SourceSsboe`·1e6 + `SourceUsecs`). Clients group cumulative trades by identical `tu` — what ATAS's `CumulativeTrade` keys on (its `.Time`). Rithmic also supplies `AggressorExchOrdId` (`o`), but grouping by `o` over-merges an order that fills across several match events (drifts higher than ATAS), so `o` is only a fallback when `tu` is missing (e.g. old local-store replay; ~50 ms window last). Replaces the earlier ms-only time-window heuristic that clipped a sweep's tail at reversals. `aggTrades` rows are now `[t,p,q,side,o,tu]`.
- **`instrumentInfo` (new).** Authoritative tick size, price decimals, expiration, point value… from Rithmic — clients no longer infer the tick from prices (fixes formatting for instruments like GC 0.1).
- **`searchSymbol` (new) + persistent symbol cache.** Keyword instrument search, served from a disk-backed cache and refreshed once per run in the background; expired contracts auto-pruned (rollover). `instrumentInfo` shares the same store (fewer calls per login). Empty exchange searches CME/CBOT/NYMEX/COMEX sequentially.
- **Live `@depth`.** The gateway now maintains the order book from per-level quotes (`OnBidQuote`/`OnAskQuote`) plus snapshot and emits `@depth` ~30 Hz continuously (no more frozen book) — powers the DOM ladder.
- **Full-session tick backfill.** Opening a chart backfills the whole current **trading session** (CME Globex roll 17:00 CT) into the local store (patches gaps from the app being off mid-session). The tick store partitions **one file per session** (not per UTC day), so a session isn't split across midnight; `bigTrades` history covers ~3 days.
- Web chart: **DOM ladder** (ATAS SmartDOM-style — Today/Bid/Sells/Buys/Ask/Chg, auto-center, historical day from the store); **exchange dropdown**; footprint imbalance fixed to **diagonal** (ATAS); **hollow/solid candles by zoom** when footprint is on.

### 1.2.0 — 2026-06-15
- **Cumulative-trade id.** `@trade` and `aggTrades` now expose `o` — the aggressor exchange order id. Prints from one aggressive order share it, so clients can rebuild **ATAS-style cumulative trades** (a sweep through several levels = one logical trade) instead of approximating with a time window. See `@trade`, `aggTrades`, RECIPES.md §1.
- **Upstream auto-reconnect.** If the gateway's Rithmic link drops, it reconnects (~30 s retries, no attempt cap) and **auto-restores every active subscription** — clients keep their socket open and do **not** re-`SUBSCRIBE`. See *Connection lifecycle & reconnect*.
- **Local tick store.** Live trades are recorded to disk; `aggTrades` serves footprint/big-trade history from the store when Rithmic can't replay a past session (works even after market close, survives restarts).
- `@trade` now filters the on-(re)subscribe snapshot (Rithmic `Image`) — only real trade updates are streamed, no stale phantom print at an old timestamp.
- Web chart: footprint renders natively on Lightweight Charts (buy/sell ladder, POC, delta); closed bars built from a conflated live feed self-heal from authoritative replay.

### 1.1.0 — 2026-06-14
- `klines`: more reliable historical candles when rapidly switching interval, symbol or date.
- Web chart moved to TradingView Lightweight Charts (candles/volume/crosshair/smooth zoom), added a history-days selector and big-trade markers.
- Faster login: the System/Gateway list is cached locally (instant on later launches, no per-launch license-server round-trip).
- Stream `@trade` gains an `r` field (gateway receive time) → the web chart shows a feed-latency badge (ms) for diagnosing delays.

### 1.0.0 — 2026-06-13
- Initial release. Streams: `@trade`, `@bookTicker`, `@depth[N]`, `@kline_<interval>`.
- Requests: `klines`, `aggTrades`, `instrumentInfo`, `searchSymbol`, `ping`, `SUBSCRIBE`/`UNSUBSCRIBE` (`account`/`exchangeInfo` → RAW.md: `getAccounts`/`listExchanges`).
- Footprint = `aggTrades` (history) + `@trade` (live). Timestamps are Unix milliseconds everywhere.

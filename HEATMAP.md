# Fxaurum Rithmic — Heatmap (Bookmap-style liquidity)

A **liquidity heatmap** over **time × price**, modelled on **Bookmap**: each pixel column is a moment in time, each row a
price; cell brightness = resting order size at that (time, price). Overlaid: a **best bid/ask line**, **trade bubbles**, and a
**current order-book depth histogram** on the right edge. Drawn as an LWC primitive **under** the candles (so footprint/candles
can sit on top when enabled).

> Companion docs: **API.md** (`@depth`, `@bookTicker`, `@trade`) · **DOM.md** (same book, ladder view) · **MBO.md**
> (order-by-order / DBO) · **STORAGE.md** (caching + how to call the API cheaply) · **RECIPES.md**.

> **Two modes.** By default the heat body is the **aggregated `@depth`** book sampled into columns (this doc, §1–§4). Toggle
> **⚙ → MBO** to switch the body to **per-order lifetime bars** built from Rithmic **DBO** (`@mboraw`): every resting order
> becomes a rectangle whose **width = its lifetime**, lane-stacked when orders overlap at a price — see **§5**. (Time-weighted
> EMA + iceberg/spoof classification remain on the **MBO.md** roadmap; the lifetime-bar visualization is built.)

---

## 1. Which sockets to call

The heatmap combines **3 live sources** (no history replay — it builds forward from the moment you open it):

| Goal | Call | Why |
|---|---|---|
| **Heat cells** (resting liquidity) | `@depth<N>` (stream) | full book per snapshot; sampled every **300 ms** into a column. `N` = depth levels (shared with the DOM). |
| **Best bid/ask line** | `@bookTicker` (stream) | drawn from **every** quote change (not the 300 ms sample) so the step line is tight and trade bubbles sit exactly on it. |
| **Trade bubbles** | `@trade` (stream) | a translucent dot per print; radius ∝ √qty, blue = buy / red = sell. |
| **Current depth histogram** (right edge) | `@depth<N>` (latest column) | the resting bid/ask size per price **right now**, drawn as horizontal bars from the price axis. |

**Minimum viable heatmap** = `@depth` (cells + depth histogram) + `@bookTicker` (line). `@trade` only adds bubbles. Depth
requires **L2 (market depth)** entitlement from Rithmic.

> **MBO mode** swaps the heat-body source: instead of `@depth` it subscribes **`@mboraw`** (per-order DBO lifetime events →
> the bars), **`@mbo<N>`** (per-level sizes → the right-edge ladder **and** the reconcile below), and keeps **`@depth1000`**
> (BBO for crossed-order cleanup). Because event streams can **drift** (a missed `New`/`Delete`), the per-order bars are
> **reconciled against `@mbo` every 300 ms** — the same authoritative per-level sizes the DOM shows — so the heatmap's
> per-order picture **matches the DOM exactly** (see §5). `@bookTicker` / `@trade` are unchanged. Needs **DBO entitlement**
> (separate from L2 — verify with `mboProbe`, see **MBO.md**). The MBO and DOM toggles **share** these streams but are
> independent: turning one off keeps whatever streams the other still needs (MBO on the heatmap works with the DOM panel closed).

---

## 2. Layers (bottom → top)

1. **Heat cells** — for each 300 ms column, every book level is a colored rectangle. Brightness = `log(size+1)/log(max)` raised
   to **gamma** (contrast). Columns gap-fill (each fills from the previous column's x) so there are **no black stripes** when the
   sample timer jitters. Always stretched to the **full pane width** (oldest at left, newest at right).
2. **Bid/ask line** — step line from `@bookTicker`. **Left of the yellow line = solid; right of it (the depth zone) = dashed.**
   Holds the last value flat to the right edge when no new tick.
3. **Current order-book depth** (Bookmap right edge) — horizontal bars of the latest book: **red = ask** (above price), **green =
   bid** (below). Bar length ∝ size; the largest bar reaches the **"yellow line"** (left bound of the zone). Size numbers shown
   when rows are tall enough.
4. **Trade bubbles** — translucent circles at (trade time, trade price).

**Colormap** (low → high liquidity): dark navy → blue → cyan → **white** → yellow → orange → **red**. Gamma (default **2.2**)
darkens mid-tones so only large liquidity lights up (less yellow wash).

---

## 3. Settings (gear ⚙ next to the Heatmap toggle)

| Setting | Meaning | Default |
|---|---|---|
| Cửa sổ thời gian (window) | How many **minutes** of recent heatmap to keep in the buffer | 5 |
| Độ tương phản màu (contrast / gamma) | Higher = darker, only big liquidity bright (less yellow); lower = more color | 2.2 |
| Hiện nến (show candles) | Off = pure Bookmap (price = the bid/ask line, no candles/footprint). On = candles/footprint drawn over the heat | off |
| Sổ lệnh hiện tại (current depth) | The right-edge depth histogram on/off | on |
| Bề rộng sổ lệnh (depth width %) | How far the "yellow line" sits from the price axis (the largest bar touches it) | 33% |
| Độ sâu MBO (per-order) | `@depth` is **always full** (1000/side ≈ the whole book). This number is the **per-order `@mbo<N>` window** (N levels each side that get per-order DBO). **`0` = full** (subscribe the whole book, like ATAS). Shared with the DOM. | 0 (full) |
| Đường kẻ giữa tick (tick lines) | MBO mode: zebra background + a thin blue-grey border on **each price tick**, so every price is a clear lane. Off = no grid. | on |
| **MBO** | Switch the heat body to **per-order bars** (subscribes `@mboraw` + `@mbo`, reconciled to match the DOM). Off = the `@depth` cells above. | off |
| Ẩn lệnh < (MBO min size) | MBO mode: only draw orders with size ≥ this (hide the size-1 noise). `0` = off. | 0 |
| Tô màu lệnh ≥ (MBO big highlight) | MBO mode: orders ≥ this are colored by **side**; smaller orders stay neutral white. `0` = off. | 0 |
| Màu bid / Màu ask (MBO) | The two highlight colors for big **bid** / big **ask** orders (color pickers). | green / red |

---

## 4. Interaction

| Action | Result |
|---|---|
| **Mouse wheel** — *@depth mode* | Zoom the **time window**, newest pinned to the right edge; always full-width. Up = zoom in, down = out (max = buffer). |
| **Mouse wheel** — *MBO mode* | Zoom **around the cursor**: the timestamp under the mouse stays pixel-fixed, so the bid/ask line **and** the order bars scale around it (like a normal chart). Window 4 s … buffer (default 15 s). Wheel with the cursor near the right edge stays **live** (now pinned at the yellow line). |
| **Drag left ↔ right** (MBO mode) | Pan into the past. The bid/ask line **freezes** at the anchor (live quotes after that point aren't drawn) until you go back to live. |
| **Drag the price axis (Y)** (MBO mode) | Zoom the **price range** — drag **up = expand** (taller bars, fewer ticks), **down = compress** (more ticks). Adjusts `heatYTicks` directly (LWC's own axis scaling is turned off in MBO so it can't snap back), so the zoom **holds on release**; price stays auto-centered. Also: **wheel on the axis** / **Shift+wheel**. |
| **Drag up ↕ down** (body, MBO mode) | Pan the **price range** up/down (also sets `heatYHold`). |
| **Auto-center on price** (MBO, live) | While not held, the Y range keeps **price centered**: if the market drifts toward the top/bottom edge the view re-centers automatically (so a falling price doesn't slide out of view). |
| **"⟶ hiện tại" button** | Appears whenever you've zoomed/panned the heatmap off live **on either axis** (time pan/zoom **or** a held Y view), or scrolled the candles away. Click → back to **live**: now re-pinned at the yellow line, time window reset to 15 s, Y auto-centering re-enabled, candles scrolled to the latest. |
| **Double-click** | Same as the button — back to live / reset zoom (time **and** Y). |
| Mouse wheel (candles shown) | Falls through to normal LWC candle zoom. |

When **"Hiện nến" is off** the chart hides candles and the footprint numbers (pure Bookmap); turn it on to overlay
footprint/candles on the heat.

---

## 5. MBO mode (per-order lifetime bars)

Toggle **⚙ → MBO**. The heat body changes from `@depth` cells to **one rectangle per resting order**, so you can read each
order's **lifetime** — the upgrade the aggregated heatmap can't show (it only knows the level total, not which orders made it).

- **Per-order bar** — each order (keyed by `ExchOrdId` from `@mboraw`) is a rectangle at its **price row**, with **x = its life**
  on the time axis: left edge = first seen (`New`/`Image`), right edge = removed (`Delete`) or **now** if still resting. So a 1-lot
  order that rests 8 s is a long thin bar; a 50-lot order that flashes for 200 ms is a short fat one. **Alive = bright, dead = faded.**
- **Matches the DOM (reconcile).** `@mboraw` is an *event* stream, so it can **drift** — a missed `New`/`Image` leaves an order
  out; a missed `Delete` leaves a phantom in. Every 300 ms the live set is **reconciled against `@mbo`** (the same authoritative
  per-level `sizes[]` the DOM shows): a size present in `@mbo` but not in the bars is **added** (with the **real size from `@mbo`**
  — never a guessed split; lifetime starts at reconcile time), and a live bar `@mbo` no longer lists is **closed**. Real orders are
  matched first so a late real `New` supersedes any stand-in. Result: the per-order picture **agrees with the DOM tick-for-tick**,
  while the real `@mboraw` ids/lifetimes are kept (needed for the iceberg roadmap). *(`@mbo` caps its `sizes[]` at 64/level, so on
  the rare level with >64 orders only the top-64 are reconciled and no bar is wrongly closed.)*
- **Lanes** — orders that overlap *in time* at the same price stack into **vertical lanes** within the tick row (greedy interval
  packing; a lane is reused once its order dies). This is the DOM's "3 blocks of 2/3/5 at one price" given a **time dimension**.
- **Dead-order cap.** A busy price can churn hundreds of orders a minute; keeping every dead bar bloats memory and crowds the
  lanes. Per price we keep the **40 largest** dead orders (`HEAT_DEADCAP`) and drop the smaller 1-lot noise — cutting by **size,
  not time**, so the *history of the meaningful (large) orders is preserved* while quiet prices keep everything. **Live/resting
  orders are never capped.**
- **Number** — each bar prints its **order size** when the lane is tall and the bar wide enough.
- **Big-order highlight** — orders ≥ *Tô màu lệnh* render in the **side color** (bid = green, ask = red, both configurable);
  everything else is neutral white. *Ẩn lệnh <* drops small orders entirely.
- **Per-tick bands** — every price tick gets a zebra background + a thin blue-grey **border line**, so each price row is a clear
  lane the bars sit inside.
- **Right edge** — the heat body is **clipped to the left of the yellow line**; the right zone keeps the live ladder, now drawn
  from **`@mbo` totals** (= the DOM **Vol** column, so the two agree). The bid/ask line is **solid left** of the yellow line and
  **dashed flat** to its right.
- **Time axis** — a **fixed-scale stretch** window (`heatViewMs`, default **15 s**) with *now* pinned at the yellow line and
  history sliding smoothly left. Zoom is **cursor-anchored** and you can **drag** to review the past (see §4); the **⟶ hiện tại**
  button / double-click return to live.

> **MBO and DOM are independent.** Both consume `@mbo`/`@mboraw`/`@depth`, but each toggle only tears down the streams **it
> alone** needs — turning the DOM off no longer blanks the heatmap (and vice-versa).

---

## 6. Gateway / rendering internals

- The book is the same `DomBook` the DOM uses (maintained from `OnBidQuote`/`OnAskQuote` + snapshot, emitted as `@depth<N>`,
  ~30 Hz). The client samples it every **`HEAT_MS` = 300 ms** into `heatCols` (one column = `{t, book:Map(tick→size), bb, ba}`).
- The bid/ask line uses a separate **`heatQuotes`** array fed by every `@bookTicker` change (finer than the 300 ms cells) — this
  is why bubbles align with the line. Both are pruned to the window buffer.
- X is a **full-width stretch** `(T − viewStart)/viewMs × paneWidth`; there is no per-bar X axis (LWC X is bar-quantized) so the
  heat is hand-drawn in `useBitmapCoordinateSpace`. The right `~3.5%` is reserved as a gap before the price axis for the line/bubbles.
- See **STORAGE.md** for how `@depth<N>` is trimmed per-subscription (so the heatmap and DOM can share one stream at a chosen depth).
- **MBO mode** instead feeds on `@mboraw`: the client holds `mboOrders` (`ExchOrdId → {price, size, side, t0, t1, segs[]}`), set
  on `New`/`Image`, split into a new **segment** on each `Change` (so a 12→6 resize keeps the 12 history), closed (`t1`) on
  `Delete`. Bars and lanes are laid out per render from that map; the right-edge ladder reads `@mbo` level totals. On
  `(re)subscribe` the gateway pushes the current `MboBook` as `Image` events so a chart that enables MBO mid-session sees the
  already-resting orders. Same stretch X as above (`xRight` = the yellow line).
- Each 300 ms sample runs, in order: **crossed-order freeze** (a bar whose price the market has crossed is closed at that point,
  using the `@depth` BBO), **`reconcileMbo()`** (add-missing / close-phantom against `@mbo`; stand-ins get a synthetic `syn:…` id
  and are re-derived each tick, so they self-clean), the **buffer-window prune** (drop dead orders older than the window), and the
  **per-price dead cap** (`HEAT_DEADCAP`, largest-kept). The renderer additionally **culls by the visible Y range** —
  `coordinateToPrice(0..paneHeight)` — so only on-screen price rows are grouped/drawn (full book is ~1375 levels but the screen
  shows a few dozen), which is what makes Y drag/zoom smooth.

---

## 7. Bookmap parity

| Bookmap feature | Status |
|---|---|
| Liquidity heatmap (time × price) | ✅ |
| Bookmap colormap + contrast control | ✅ |
| Best bid/ask line | ✅ |
| Trade bubbles (size ∝ volume) | ✅ |
| Current order-book depth (right edge) | ✅ |
| Time-window wheel zoom (cursor-anchored in MBO) | ✅ |
| Candles/footprint overlay (optional) | ✅ |
| Volume dots by aggressor | ✅ |
| **MBO per-order lifetime bars (lanes + size)** | ✅ |
| **MBO per-order reconciled to the DOM (no drift)** | ✅ |
| **Full-book per-order MBO (0 = whole book, like ATAS)** | ✅ |
| Y-axis drag pan/zoom + auto-center on price | ✅ |
| Historical heatmap replay | ⬜ (live-only; builds forward) |
| Iceberg / spoof classification | ⬜ (MBO.md roadmap — kept `@mboraw` ids make it possible) |

---

## 8. Changelog

- **2026-07-01** — **MBO per-order now matches the DOM.** The lifetime bars are **reconciled against `@mbo` every 300 ms**
  (add-missing with real `@mbo` sizes / close-phantom), fixing the `@mboraw` drift where a resting order (e.g. a 15-lot) showed on
  the DOM but was absent from the heatmap. Per-price **dead-order cap** (`HEAT_DEADCAP`, keeps the **largest**, drops 1-lot noise —
  cuts by size not time, so large-order history survives). Full-book per-order MBO (**Độ sâu MBO = 0** → subscribe the whole book,
  like ATAS; `@depth` always full). **Y-axis** drag pan/zoom with hold + **auto-center on price**; the **⟶ hiện tại** button now
  also clears a held Y view. Tick-line grid is a setting. `@mboraw` order ids/lifetimes are **kept** (basis for the iceberg roadmap).
- **2026-06-30** — **MBO mode** (per-order lifetime bars): `⚙ → MBO` switches the heat body from `@depth` cells to one rectangle
  per resting order (width = lifetime, from `@mboraw`), lane-stacked when orders overlap at a price, with in-bar size numbers,
  *min size* / *big-order highlight* (side-colored, pickers) filters, per-tick zebra+border bands, and the right-edge ladder
  sourced from `@mbo` totals (matches the DOM Vol column). Interaction: **cursor-anchored wheel zoom**, drag-to-review, a
  **⟶ hiện tại** button (+ double-click) to jump back to live. DOM and heatmap toggles are now **independent** (turning one off
  keeps the streams the other needs).
- **2026-06-16** — Heatmap: Bookmap colormap + gamma contrast, full-pane-width stretch, bid/ask line drawn from `@bookTicker`
  (tight; bubbles aligned), solid-left / dashed-right of the yellow line, current order-book depth histogram on the right edge
  (red ask / green bid, size labels), time-window mouse-wheel zoom + double-click reset, "show candles" overlay toggle, shared
  `@depth<N>` depth level (with the DOM).

# Fxaurum Rithmic — Heatmap (Bookmap-style liquidity)

A **liquidity heatmap** over **time × price**, modelled on **Bookmap**: each pixel column is a moment in time, each row a
price; cell brightness = resting order size at that (time, price). Overlaid: a **best bid/ask line**, **trade bubbles**, and a
**current order-book depth histogram** on the right edge. Drawn as an LWC primitive **under** the candles (so footprint/candles
can sit on top when enabled).

> Companion docs: **API.md** (`@depth`, `@bookTicker`, `@trade`) · **DOM.md** (same book, ladder view) · **MBO.md**
> (order-by-order / DBO) · **STORAGE.md** (caching + how to call the API cheaply) · **RECIPES.md**.

> **MBO Heatmap (planned, MBO.md Phase 2):** this heatmap samples the **aggregated** `@depth` book. A second heatmap built on
> the **MBO** book engine (Rithmic DBO) is on the roadmap — time-weighted EMA of resting size (real resting brightens, flicker
> fades) plus **spoof subtraction** (young, unfilled orders excluded). See **MBO.md** §7.

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
| Độ sâu sổ lệnh (depth levels) | `N` levels each side pulled via `@depth<N>` — **shared with the DOM ladder** | 100 |

---

## 4. Interaction

| Action | Result |
|---|---|
| **Mouse wheel** (candles hidden) | Zoom the heatmap **time window** (newest pinned to the right edge); always stays full-width. Wheel up = zoom in (nearer/finer), down = zoom out (max = data in buffer). |
| **Double-click** | Reset zoom to the full buffer. |
| Mouse wheel (candles shown) | Falls through to normal LWC candle zoom. |

When **"Hiện nến" is off** the chart hides candles and the footprint numbers (pure Bookmap); turn it on to overlay
footprint/candles on the heat.

---

## 5. Gateway / rendering internals

- The book is the same `DomBook` the DOM uses (maintained from `OnBidQuote`/`OnAskQuote` + snapshot, emitted as `@depth<N>`,
  ~30 Hz). The client samples it every **`HEAT_MS` = 300 ms** into `heatCols` (one column = `{t, book:Map(tick→size), bb, ba}`).
- The bid/ask line uses a separate **`heatQuotes`** array fed by every `@bookTicker` change (finer than the 300 ms cells) — this
  is why bubbles align with the line. Both are pruned to the window buffer.
- X is a **full-width stretch** `(T − viewStart)/viewMs × paneWidth`; there is no per-bar X axis (LWC X is bar-quantized) so the
  heat is hand-drawn in `useBitmapCoordinateSpace`. The right `~3.5%` is reserved as a gap before the price axis for the line/bubbles.
- See **STORAGE.md** for how `@depth<N>` is trimmed per-subscription (so the heatmap and DOM can share one stream at a chosen depth).

---

## 6. Bookmap parity

| Bookmap feature | Status |
|---|---|
| Liquidity heatmap (time × price) | ✅ |
| Bookmap colormap + contrast control | ✅ |
| Best bid/ask line | ✅ |
| Trade bubbles (size ∝ volume) | ✅ |
| Current order-book depth (right edge) | ✅ |
| Time-window wheel zoom | ✅ |
| Candles/footprint overlay (optional) | ✅ |
| Volume dots by aggressor | ✅ |
| Historical heatmap replay | ⬜ (live-only; builds forward) |
| Large-lot / iceberg detection | ⬜ |

---

## 7. Changelog

- **2026-06-16** — Heatmap: Bookmap colormap + gamma contrast, full-pane-width stretch, bid/ask line drawn from `@bookTicker`
  (tight; bubbles aligned), solid-left / dashed-right of the yellow line, current order-book depth histogram on the right edge
  (red ask / green bid, size labels), time-window mouse-wheel zoom + double-click reset, "show candles" overlay toggle, shared
  `@depth<N>` depth level (with the DOM).
